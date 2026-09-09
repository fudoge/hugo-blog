---
title: "Linux Network - TPROXY Basics"
description: "Follow Linux Netfilter hooks, nftables TPROXY expression evaluation, skb socket assignment, and fwmark-based local routing from kernel source to hands-on practice"
date: 2026-09-09T18:19:15+09:00
lastmod: 2026-09-09T18:19:15+09:00
slug: linux-network-tproxy-basic
image:
math: false
license:
hidden: false
comments: true
draft: false

tags:
    - Linux
    - Network
    - TPROXY
    - Netfilter
    - nftables
    - Kernel
    - Cilium

categories:
    - Network
---

Transparent proxying lets an intermediate host pass traffic to a proxy even when the client is not explicitly configured to use one.  
Instead of configuring an HTTP proxy address on the client, a router or node intercepts transit traffic and hands it to a local proxy process.  

One way to do this is DNAT or REDIRECT, where the packet destination address or port is rewritten to the proxy address.  
That works for many cases, but it also means the packet destination is actually changed, so the proxy must recover the original destination through extra mechanisms.  
This is especially important for UDP, where recovering the original destination from connection state is less reliable.  

TPROXY(Transparent Proxy) solves this without NAT.  
The packet keeps its original destination address and port, while Netfilter finds a transparent listener socket, attaches it to `skb->sk`, and fwmark-based policy routing sends the packet into local delivery.  

In this post, let's build on the IPv4 ingress/routing path from the previous posts and follow where TPROXY intervenes, using Linux kernel source code and a small hands-on lab.  

---
## 🪝 Looking Closely at NF_HOOK

### nf_hook

In the previous post, we saw that `ip_rcv()` validates an IPv4 datagram and then runs `NF_HOOK()` at the `PREROUTING` hook.  

The `NF_HOOK()` wrapper calls `nf_hook()` internally.  

```c
// source: include/linux/netfilter.h
static inline int
NF_HOOK(uint8_t pf, unsigned int hook, struct net *net, struct sock *sk,
        struct sk_buff *skb, struct net_device *in, struct net_device *out,
        int (*okfn)(struct net *, struct sock *, struct sk_buff *))
{
    int ret = nf_hook(pf, hook, net, sk, skb, in, out, okfn);

    if (ret == 1)
        ret = okfn(net, sk, skb);

    return ret;
}
```

`nf_hook()` starts evaluation when hook entries are registered at the current hook point.  
The actual hook traversal and verdict handling are done by `nf_hook_slow()`.  

```c
// source: include/linux/netfilter.h
static inline int nf_hook(u_int8_t pf, unsigned int hook, struct net *net,
			  struct sock *sk, struct sk_buff *skb,
			  struct net_device *indev, struct net_device *outdev,
			  int (*okfn)(struct net *, struct sock *, struct sk_buff *))
{
	struct nf_hook_entries *hook_head = NULL;
	int ret = 1;

	// ...

	// Run registered hook entries if any exist.
	if (hook_head) {
		struct nf_hook_state state;

		nf_hook_state_init(&state, hook, pf, indev, outdev,
				   sk, net, okfn);

		ret = nf_hook_slow(skb, &state, hook_head, 0);
	}
	rcu_read_unlock();

	return ret;
}
```

`nf_hook_slow()` evaluates hook entries in order and decides the next action from the verdict returned by each callback.  
The actual callback invocation is done by `nf_hook_entry_hookfn()`.  

```c
// source: net/netfilter/core.c
/* Returns 1 if okfn() needs to be executed by the caller,
 * -EPERM for NF_DROP, 0 otherwise.  Caller must hold rcu_read_lock. */
int nf_hook_slow(struct sk_buff *skb, struct nf_hook_state *state,
		 const struct nf_hook_entries *e, unsigned int s)
{
	unsigned int verdict;
	int ret;

	for (; s < e->num_hook_entries; s++) {
		verdict = nf_hook_entry_hookfn(&e->hooks[s], skb, state);
		switch (verdict & NF_VERDICT_MASK) {
		// This break exits only the switch statement.
		// The outer for loop continues.
		// In this sense, ACCEPT means "pass this entry",
		// not "make the whole packet finally accepted".
		case NF_ACCEPT:
			break;
		case NF_DROP:
			kfree_skb_reason(skb,
					 SKB_DROP_REASON_NETFILTER_DROP);
			ret = NF_DROP_GETERR(verdict);
			if (ret == 0)
				ret = -EPERM;
			return ret;
		case NF_QUEUE:
			ret = nf_queue(skb, state, s, verdict);
			if (ret == 1)
				continue;
			return ret;
		case NF_STOLEN:
			return NF_DROP_GETERR(verdict);
		default:
			WARN_ON_ONCE(1);
			return 0;
		}
	}

	return 1;
}
EXPORT_SYMBOL(nf_hook_slow);
```

`nf_hook_entry_hookfn()` is a small wrapper that calls the callback attached to the hook entry.  
`entry->hook` is a function pointer, so it points to the actual hook implementation.  

This callback may come from nftables, iptables, or another Netfilter user.  

```c
// source: include/linux/netfilter.h
static inline int
nf_hook_entry_hookfn(const struct nf_hook_entry *entry, struct sk_buff *skb,
		     struct nf_hook_state *state)
{
	return entry->hook(entry->priv, skb, state);
}
```

### Looking at hooks through nftables

Let's look at `nft_chain_filter_ipv4`.  
The `hooks` array maps IPv4 Netfilter hook points to `nft_do_chain_ipv4`.  

```c
// source: net/netfilter/nft_chain_filter.c
static const struct nft_chain_type nft_chain_filter_ipv4 = {
	.name		= "filter",
	.type		= NFT_CHAIN_T_DEFAULT,
	.family		= NFPROTO_IPV4,
	.hook_mask	= (1 << NF_INET_LOCAL_IN) |
			  (1 << NF_INET_LOCAL_OUT) |
			  (1 << NF_INET_FORWARD) |
			  (1 << NF_INET_PRE_ROUTING) |
			  (1 << NF_INET_POST_ROUTING),
	.hooks		= {
		[NF_INET_LOCAL_IN]	= nft_do_chain_ipv4,
		[NF_INET_LOCAL_OUT]	= nft_do_chain_ipv4,
		[NF_INET_FORWARD]	= nft_do_chain_ipv4,
		[NF_INET_PRE_ROUTING]	= nft_do_chain_ipv4,
		[NF_INET_POST_ROUTING]	= nft_do_chain_ipv4,
	},
};
```

Now look at `nft_basechain_hook_init()`, which initializes the hook for an nftables base chain.  
Eventually, the hook callback is registered in `ops->hook`.  

```c
// source: net/netfilter/nf_tables_api.c
static void nft_basechain_hook_init(struct nf_hook_ops *ops, u8 family,
				    const struct nft_chain_hook *hook,
				    struct nft_chain *chain)
{
	ops->pf			= family;
	ops->hooknum	= hook->num;
	ops->priority	= hook->priority;
	ops->priv		= chain;
	// For IPv4, this eventually becomes ops->hook = nft_do_chain_ipv4.
	ops->hook		= hook->type->hooks[ops->hooknum];
	ops->hook_ops_type	= NF_HOOK_OP_NF_TABLES;
}

```

`nft_do_chain_ipv4()` calls the real core function, `nft_do_chain()`.  
Other protocol families follow a similar structure.  

```c
// source: net/netfilter/nft_chain_filter.c
#ifdef CONFIG_NF_TABLES_IPV4  
static unsigned int nft_do_chain_ipv4(void *priv,  
					struct sk_buff *skb,  
					const struct nf_hook_state *state)  
{  
	struct nft_pktinfo pkt;  
  
	nft_set_pktinfo(&pkt, skb, state);  
	nft_set_pktinfo_ipv4(&pkt);  
  
	return nft_do_chain(&pkt, priv);  
}
```

`nft_do_chain()` is the core of nftables rule evaluation.  
It iterates over rules and expressions, evaluates them, and returns the resulting verdict to the Netfilter core.  

```c
// source: net/netfilter/nf_tables_core.c
unsigned int
nft_do_chain(struct nft_pktinfo *pkt, void *priv)
{
	// ...
	
	// Get the first rule.
	rule = (struct nft_rule_dp *)blob->data;
next_rule:
	regs.verdict.code = NFT_CONTINUE;
	// Evaluate each rule.
	for (; !rule->is_last ; rule = nft_rule_next(rule)) {
		// Iterate over expressions in the rule.
		nft_rule_dp_for_each_expr(expr, last, rule) {
			if (expr->ops == &nft_cmp_fast_ops)
				nft_cmp_fast_eval(expr, &regs);
			else if (expr->ops == &nft_cmp16_fast_ops)
				nft_cmp16_fast_eval(expr, &regs);
			else if (expr->ops == &nft_bitwise_fast_ops)
				nft_bitwise_fast_eval(expr, &regs);
			else if (expr->ops != &nft_payload_fast_ops ||
				 !nft_payload_fast_eval(expr, &regs, pkt))
				expr_call_ops_eval(expr, &regs, pkt);

			if (regs.verdict.code != NFT_CONTINUE)
				break;
		}

		switch (regs.verdict.code) {
		case NFT_BREAK:
			regs.verdict.code = NFT_CONTINUE;
			nft_trace_copy_nftrace(pkt, &info);
			continue;
		case NFT_CONTINUE:
			nft_trace_packet(pkt, &regs.verdict,  &info, rule,
					 NFT_TRACETYPE_RULE);
			continue;
		}
		break;
	}

	nft_trace_verdict(pkt, &info, rule, &regs);

	// Return the verdict to Netfilter.
	switch (regs.verdict.code & NF_VERDICT_MASK) {
	case NF_ACCEPT:
	case NF_QUEUE:
	case NF_STOLEN:
		return regs.verdict.code;
	case NF_DROP:
		return NF_DROP_REASON(pkt->skb, SKB_DROP_REASON_NETFILTER_DROP, EPERM);
	}

	switch (regs.verdict.code) {
	case NFT_JUMP:
		if (unlikely(stackptr >= NFT_JUMP_STACK_SIZE)) {
			DEBUG_NET_WARN_ON_ONCE(1);
			return NF_DROP_REASON(pkt->skb, SKB_DROP_REASON_NETFILTER_DROP, ELOOP);
		}
		jumpstack[stackptr].rule = nft_rule_next(rule);
		stackptr++;
		fallthrough;
	case NFT_GOTO:
		chain = regs.verdict.chain;
		goto do_chain;
	case NFT_CONTINUE:
	case NFT_RETURN:
		break;
	default:
		DEBUG_NET_WARN_ON_ONCE(1);
	}

	// ...
	
	// Return the base chain default policy.
	return nft_base_chain(basechain)->policy;
}
EXPORT_SYMBOL_GPL(nft_do_chain);
```

---
## 🧲 TPROXY Expression

The TPROXY expression is registered with `.eval = nft_tproxy_eval`.  

```c
// source: net/netfilter/nft_tproxy.c
static const struct nft_expr_ops nft_tproxy_ops = {
	.type		= &nft_tproxy_type,
	.size		= NFT_EXPR_SIZE(sizeof(struct nft_tproxy)),
	.eval		= nft_tproxy_eval,
	.init		= nft_tproxy_init,
	.destroy	= nft_tproxy_destroy,
	.dump		= nft_tproxy_dump,
	.validate	= nft_tproxy_validate,
};
```

`nft_do_chain()` eventually runs the expression's `.eval` callback.  

```c
// source: net/netfilter/nf_tables_core.c
			// ...
			else if (expr->ops != &nft_payload_fast_ops ||
				 !nft_payload_fast_eval(expr, &regs, pkt))
				expr_call_ops_eval(expr, &regs, pkt);
		// ...
```

For IPv4 packets, `nft_tproxy_eval()` calls `nft_tproxy_eval_v4()`.  

```c
// source: net/netfilter/nft_tproxy.c
static void nft_tproxy_eval(const struct nft_expr *expr,
			    struct nft_regs *regs,
			    const struct nft_pktinfo *pkt)
{
	const struct nft_tproxy *priv = nft_expr_priv(expr);

	switch (nft_pf(pkt)) {
	case NFPROTO_IPV4:
		switch (priv->family) {
		case NFPROTO_IPV4:
		case NFPROTO_UNSPEC:
			nft_tproxy_eval_v4(expr, regs, pkt);
			return;
		}
		break;
#if IS_ENABLED(CONFIG_NF_TABLES_IPV6)
	case NFPROTO_IPV6:
		switch (priv->family) {
		case NFPROTO_IPV6:
		case NFPROTO_UNSPEC:
			nft_tproxy_eval_v6(expr, regs, pkt);
			return;
		}
#endif
	}
	regs->verdict.code = NFT_BREAK;
}
```

`nft_tproxy_eval_v4()` is the core of IPv4 TPROXY handling.  
It first looks for a socket matching an existing connection. If none exists, it looks for a listener on the TPROXY target address and port.  
If the matched socket is transparent, the socket is assigned to the `skb`.  

```c
// source: net/netfilter/nft_tproxy.c
static void nft_tproxy_eval_v4(const struct nft_expr *expr,
			       struct nft_regs *regs,
			       const struct nft_pktinfo *pkt)
{
	const struct nft_tproxy *priv = nft_expr_priv(expr);
	struct sk_buff *skb = pkt->skb;
	const struct iphdr *iph = ip_hdr(skb);
	struct udphdr _hdr, *hp;
	__be32 taddr = 0;
	__be16 tport = 0;
	struct sock *sk;

	if ((pkt->tprot != IPPROTO_TCP &&
	     pkt->tprot != IPPROTO_UDP) || pkt->fragoff) {
		regs->verdict.code = NFT_BREAK;
		return;
	}

	hp = skb_header_pointer(skb, ip_hdrlen(skb), sizeof(_hdr), &_hdr);
	if (!hp) {
		regs->verdict.code = NFT_BREAK;
		return;
	}

	/* check if there's an ongoing connection on the packet addresses, this
	 * happens if the redirect already happened and the current packet
	 * belongs to an already established connection
	 */
	// Look for an already connected socket using the original packet 4-tuple.
	sk = nf_tproxy_get_sock_v4(nft_net(pkt), skb, iph->protocol,
				   iph->saddr, iph->daddr,
				   hp->source, hp->dest,
				   skb->dev, NF_TPROXY_LOOKUP_ESTABLISHED);

	if (priv->sreg_addr)
		taddr = nft_reg_load_be32(&regs->data[priv->sreg_addr]);
	taddr = nf_tproxy_laddr4(skb, taddr, iph->daddr);

	if (priv->sreg_port)
		tport = nft_reg_load_be16(&regs->data[priv->sreg_port]);
	if (!tport)
		tport = hp->dest;

	/* UDP has no TCP_TIME_WAIT state, so we never enter here */
	if (sk && sk->sk_state == TCP_TIME_WAIT) {
		/* reopening a TIME_WAIT connection needs special handling */
		sk = nf_tproxy_handle_time_wait4(nft_net(pkt), skb, taddr, tport, sk);
	} else if (!sk) {
		/* no, there's no established connection, check if
		 * there's a listener on the redirected addr/port
		 */
		// If no connection exists, look for a listener on the TPROXY target address and port.
		sk = nf_tproxy_get_sock_v4(nft_net(pkt), skb, iph->protocol,
					   iph->saddr, taddr,
					   hp->source, tport,
					   skb->dev, NF_TPROXY_LOOKUP_LISTENER);
	}

	// key
	if (sk && nf_tproxy_sk_is_transparent(sk))
		nf_tproxy_assign_sock(skb, sk);
	else
		regs->verdict.code = NFT_BREAK;
}
```

`nf_tproxy_assign_sock()` assigns the matched socket to the `skb`.  

```c
// source: include/net/netfilter/nf_tproxy.h
/* assign a socket to the skb -- consumes sk */
static inline void nf_tproxy_assign_sock(struct sk_buff *skb, struct sock *sk)
{
	skb_orphan(skb);
	skb->sk = sk;
	skb->destructor = sock_edemux;
}
```

Now let's see what happens later in the TCP layer after the packet is routed locally toward the transparent proxy.  
`__inet_lookup_skb()` is called from the TCP receive path.  

```c
// source: net/ipv4/tcp_ipv4.c
int tcp_v4_rcv(struct sk_buff *skb)
{
	struct net *net = dev_net_rcu(skb->dev);
	enum skb_drop_reason drop_reason;
	enum tcp_tw_status tw_status;
	int sdif = inet_sdif(skb);
	int dif = inet_iif(skb);
	const struct iphdr *iph;
	const struct tcphdr *th;
	struct sock *sk = NULL;
	bool refcounted;
	int ret;
	u32 isn;

	// ...
	
	sk = __inet_lookup_skb(skb, __tcp_hdrlen(th), th->source,
			       th->dest, sdif, &refcounted);
	
	// ...
```

`__inet_lookup_skb()` first tries `inet_steal_sock()`.  

```c
// source: include/net/inet_hashtables.h
static inline struct sock *__inet_lookup_skb(struct sk_buff *skb,
					     int doff,
					     const __be16 sport,
					     const __be16 dport,
					     const int sdif,
					     bool *refcounted)
{
	struct net *net = skb_dst_dev_net_rcu(skb);
	const struct iphdr *iph = ip_hdr(skb);
	struct sock *sk;

	sk = inet_steal_sock(net, skb, doff, iph->saddr, sport, iph->daddr, dport,
			     refcounted, inet_ehashfn);
	if (IS_ERR(sk))
		return NULL;
	if (sk)
		return sk;

	return __inet_lookup(net, skb, doff, iph->saddr, sport,
			     iph->daddr, dport, inet_iif(skb), sdif,
			     refcounted);
}
```

`inet_steal_sock()` calls `skb_steal_sock()`.  

```c
// source: include/net/inet_hashtables.h
static inline
struct sock *inet_steal_sock(struct net *net, struct sk_buff *skb, int doff,
			     const __be32 saddr, const __be16 sport,
			     const __be32 daddr, const __be16 dport,
			     bool *refcounted, inet_ehashfn_t *ehashfn)
{
	struct sock *sk, *reuse_sk;
	bool prefetched;

	sk = skb_steal_sock(skb, refcounted, &prefetched);
	if (!sk)
		return NULL;

	if (!prefetched || !sk_fullsock(sk))
		return sk;

	if (sk->sk_protocol == IPPROTO_TCP) {
		if (sk->sk_state != TCP_LISTEN)
			return sk;
	} else if (sk->sk_protocol == IPPROTO_UDP) {
		if (sk->sk_state != TCP_CLOSE)
			return sk;
	} else {
		return sk;
	}

	reuse_sk = inet_lookup_reuseport(net, sk, skb, doff,
					 saddr, sport, daddr, ntohs(dport),
					 ehashfn);
	if (!reuse_sk)
		return sk;

	/* We've chosen a new reuseport sock which is never refcounted. This
	 * implies that sk also isn't refcounted.
	 */
	WARN_ON_ONCE(*refcounted);

	return reuse_sk;
}
```

`skb_steal_sock()` takes the socket from `skb->sk` and clears the socket reference from the `skb` side.  

```c
// source: include/net/sock.h
/**
 * skb_steal_sock - steal a socket from an sk_buff
 * @skb: sk_buff to steal the socket from
 * @refcounted: is set to true if the socket is reference-counted
 * @prefetched: is set to true if the socket was assigned from bpf
 */
static inline struct sock *skb_steal_sock(struct sk_buff *skb,
					  bool *refcounted, bool *prefetched)
{
	struct sock *sk = skb->sk;

	if (!sk) {
		*prefetched = false;
		*refcounted = false;
		return NULL;
	}

	*prefetched = skb_sk_is_prefetched(skb);
	if (*prefetched) {
#if IS_ENABLED(CONFIG_SYN_COOKIES)
		if (sk->sk_state == TCP_NEW_SYN_RECV && inet_reqsk(sk)->syncookie) {
			struct request_sock *req = inet_reqsk(sk);

			*refcounted = false;
			sk = req->rsk_listener;
			req->rsk_listener = NULL;
			return sk;
		}
#endif
		*refcounted = sk_is_refcounted(sk);
	} else {
		*refcounted = true;
	}

	skb->destructor = NULL;
	skb->sk = NULL;
	return sk;
}
```

After that, `tcp_v4_rcv()` continues into `tcp_v4_do_rcv()` for actual TCP processing.  

```c
// source: net/ipv4/tcp_ipv4.c
/* The socket must have it's spinlock held when we get
 * here, unless it is a TCP_LISTEN socket.
 *
 * We have a potential double-lock case here, so even when
 * doing backlog processing we use the BH locking scheme.
 * This is because we cannot sleep with the original spinlock
 * held.
 */
int tcp_v4_do_rcv(struct sock *sk, struct sk_buff *skb)
{
	enum skb_drop_reason reason;

	reason = psp_sk_rx_policy_check(sk, skb);
	if (reason)
		goto err_discard;

	if (sk->sk_state == TCP_ESTABLISHED) { /* Fast path */
		struct dst_entry *dst;

		dst = rcu_dereference_protected(sk->sk_rx_dst,
						lockdep_sock_is_held(sk));

		sock_rps_save_rxhash(sk, skb);
		sk_mark_napi_id(sk, skb);
		if (dst && unlikely(dst != skb_dst(skb))) {
			if (sk->sk_rx_dst_ifindex != skb->skb_iif ||
			    !INDIRECT_CALL_1(dst->ops->check, ipv4_dst_check,
					     dst, 0)) {
				RCU_INIT_POINTER(sk->sk_rx_dst, NULL);
				dst_release(dst);
			}
		}
		tcp_rcv_established(sk, skb);
		return 0;
	}

	if (tcp_checksum_complete(skb))
		goto csum_err;

	if (sk->sk_state == TCP_LISTEN) {
		struct sock *nsk = tcp_v4_cookie_check(sk, skb);

		if (!nsk)
			return 0;
		if (nsk != sk) {
			reason = tcp_child_process(sk, nsk, skb);
			sock_put(nsk);
			if (reason)
				goto reset;
			return 0;
		}
	} else
		sock_rps_save_rxhash(sk, skb);

	reason = tcp_rcv_state_process(sk, skb);
	if (reason)
		goto reset;
	return 0;

reset:
	tcp_v4_send_reset(sk, skb, sk_rst_convert_drop_reason(reason));
discard:
	sk_skb_reason_drop(sk, skb, reason);
	/* Be careful here. If this function gets more complicated and
	 * gcc suffers from register pressure on the x86, sk (in %ebx)
	 * might be destroyed here. This current version compiles correctly,
	 * but you have been warned.
	 */
	return 0;

csum_err:
	reason = SKB_DROP_REASON_TCP_CSUM;
	trace_tcp_bad_csum(skb);
	TCP_INC_STATS(sock_net(sk), TCP_MIB_CSUMERRORS);
err_discard:
	TCP_INC_STATS(sock_net(sk), TCP_MIB_INERRS);
	goto discard;
}
```

For TCP, when the listener processes a SYN and creates a request socket, it fills the address and port fields from the original IP/TCP headers in the `skb`.  

```c
// source: net/ipv4/tcp_ipv4.c
static void tcp_v4_init_req(struct request_sock *req,
			    const struct sock *sk_listener,
			    struct sk_buff *skb)
{
	struct inet_request_sock *ireq = inet_rsk(req);
	struct net *net = sock_net(sk_listener);

	// Fill source and destination addresses from the skb.
	sk_rcv_saddr_set(req_to_sk(req), ip_hdr(skb)->daddr);
	sk_daddr_set(req_to_sk(req), ip_hdr(skb)->saddr);
	RCU_INIT_POINTER(ireq->ireq_opt, tcp_v4_save_options(net, skb));
}
```

The port numbers are initialized in `tcp_openreq_init()`.  
This function also initializes other fields needed by the TCP request socket.  

```c
// source: include/net/tcp.h
static void tcp_openreq_init(struct request_sock *req,
			     const struct tcp_options_received *rx_opt,
			     struct sk_buff *skb, const struct sock *sk)
{
	// ...
	ireq->ir_rmt_port = tcp_hdr(skb)->source;
	ireq->ir_num = ntohs(tcp_hdr(skb)->dest);
	// ...
}
```

---
## 🧭 Full Flow Summary

The source snippets above focus on Netfilter and TCP socket handling.  
If we include fwmark-based local routing, the full path looks like this:  

```text
NIC
 |
...
 |
ip_rcv()
 |
NF_HOOK(NF_INET_PRE_ROUTING)
 |
nftables
 |
nft_do_chain()
 |
[tproxy to :PORT]
 |
nft_tproxy_eval_v4()
 |
nf_tproxy_get_sock_v4(... ESTABLISHED)
 |
if none
 |
nf_tproxy_get_sock_v4(... LISTENER)
 |
nf_tproxy_sk_is_transparent()
 |
nf_tproxy_assign_sock()
  skb_orphan(skb)
  skb->sk = sk
  skb->destructor = sock_edemux
 |
[meta mark set 0x..]             <-- fwmark
 |
NF_ACCEPT
 |
ip_rcv_finish()
 |
ip_route_input_noref()
 ^
 skb->mark is used for policy routing
 ip rule fwmark -> local route
 |
dst_input()
 |
ip_local_deliver()
 |
NF_INET_LOCAL_IN
 |
ip_local_deliver_finish()
 |
ip_protocol_deliver_rcu()
 |
tcp_v4_rcv()
 |
__inet_lookup_skb()
 |
inet_steal_sock()
 |
skb_steal_sock()
  sk = skb->sk
  skb->sk = NULL
  skb->destructor = NULL
 |
tcp_v4_do_rcv() with that socket
 |
proxy socket
```

---
## 🧪 Hands-on: TPROXY

Let's run the lab on a single server.  

The client will run in a separate network namespace.  
If `ip` commands and network namespaces are unfamiliar, refer to the [previous post]({{< relref "post/linux-network-ip" >}}) first.  

```bash
# Create a client namespace.
sudo ip netns add cli

# Create a veth pair.
sudo ip link add veth-tp type veth peer name veth-cli
sudo ip link set veth-cli netns cli

sudo ip addr add 10.10.0.1/24 dev veth-tp
sudo ip link set veth-tp up

sudo ip netns exec cli ip addr add 10.10.0.2/24 dev veth-cli
sudo ip netns exec cli ip link set lo up
sudo ip netns exec cli ip link set veth-cli up

sudo ip netns exec cli ip route add default via 10.10.0.1
```

Add an `ip rule` so packets with fwmark `1` use routing table 100.  
In table 100, add a local default route through `lo`.  

```bash
sudo ip rule add fwmark 1 lookup 100
sudo ip route add local 0.0.0.0/0 dev lo table 100
```

Now add an nftables rule that sets the mark.  
At `PREROUTING`, TCP port 80 traffic coming from `veth-tp` is sent to the transparent proxy at `:50080`, and mark `1` is applied.  

```bash
# Add a table.
sudo nft add table inet tp

# Add a base chain.
sudo nft 'add chain inet tp prerouting {
    type filter hook prerouting priority mangle;
    policy accept;
}'

# If traffic arrives on veth-tp and the destination port is 80,
# send it to :50080 through TPROXY and set mark 1.
sudo nft 'add rule inet tp prerouting \
    iifname "veth-tp" tcp dport 80 \
    tproxy to :50080 meta mark set 1 accept'
```

Below is a minimal TPROXY server implementation.  
In real environments, Envoy, HAProxy, Squid, or similar proxies are usually used.  

```python
# tproxy.py
import socket

# Let the kernel treat this as a transparent socket.
IP_TRANSPARENT = getattr(socket, "IP_TRANSPARENT", 19)

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)

s.setsockopt(socket.SOL_IP, IP_TRANSPARENT, 1)

s.bind(("0.0.0.0", 50080))
s.listen(128)

print("transparent listener on :50080")

while True:
    c, peer = s.accept()

    print()
    print("peer     =", c.getpeername())
    print("local    =", c.getsockname())

    data = c.recv(4096)
    print("received =", repr(data))

    body = b"TPROXY OK\n"
    c.sendall(
        b"HTTP/1.1 200 OK\r\n"
        b"Connection: close\r\n"
        b"Content-Length: 10\r\n"
        b"\r\n" +
        body
    )
    c.close()
```

Run the server in the root namespace.  

```bash
sudo python3 tproxy.py
```

From the client namespace, connect to an arbitrary TEST-NET address.  

```bash
sudo ip netns exec cli curl -v http://198.51.100.10/
```

The TPROXY server should print logs.  
For the child socket accepted by `accept()`, the local address is still `198.51.100.10:80`.  

```bash
# The HTTP request payload is split across lines for readability.
peer = ('10.10.0.2', 39112) 
local = ('198.51.100.10', 80) 
received = b'
	GET / HTTP/1.1\r\n
	Host: 198.51.100.10\r\n
	User-Agent: curl/8.18.0\r\n
	Accept: */*\r\n\r\n'
```

The original destination address is not restored accidentally after routing.  
At `PREROUTING`, the transparent listener is found first and attached to `skb->sk`; then fwmark-based policy routing sends the packet to local delivery.  
After that, the L4 layer steals `skb->sk`, and the proxy receives traffic using the original destination socket address.  

The first SYN packet follows this path:  

```text
SYN skb
10.0.0.2:39112 -> 198.51.100.10:80
        |
        v
PREROUTING
        |
        +-- TPROXY
        |    skb->sk = listener(:50080)
        |
        +-- mark = 1
        |
        v
route lookup
        |
        | skb->daddr = 198.51.100.10
        | skb->saddr = 10.0.0.2
        | skb->mark  = 1
        v
table 100
local 0/0
        |
        v
RTN_LOCAL
        |
        v
ip_local_deliver()
        |
        v
TCP
skb_steal_sock()
        |
        v
SYN is processed by the listener
        |
        v
request_sock
        |
        v
child socket
```

---
## 🔎 TPROXY in Cilium

One representative use case of TPROXY is Cilium, a Kubernetes CNI.  

Cilium L7 policy traffic goes through a node-local Envoy proxy.  
On the Kubernetes node below, we can see a fwmark and local route setup for proxy traffic.  

```bash
[root@home-cp-1:~]# ip rule 
9: from all fwmark 0x200/0xf00 lookup 2004 
100: from all lookup local 
32766: from all lookup main 
32767: from all lookup default

[root@home-cp-1:~]#
ip route show table 2004 
local default dev lo proto kernel scope host
```

Next, look at the iptables rules.  
We can see rules matching mark `0x200/0xf00`.  
On this node, L7 traffic including FQDN egress is handled with proxy traffic marks and a local route toward node-local Envoy.  

```bash
[root@home-cp-1:~]# iptables -S 
-P INPUT ACCEPT 
-P FORWARD ACCEPT 
-P OUTPUT ACCEPT 
-N CILIUM_FORWARD 
-N CILIUM_INPUT 
-N CILIUM_OUTPUT 
-N KUBE-FIREWALL 
-N KUBE-KUBELET-CANARY 
-A INPUT -m comment --comment "cilium-feeder: CILIUM_INPUT" -j CILIUM_INPUT 
-A INPUT -j KUBE-FIREWALL 
-A FORWARD -m comment --comment "cilium-feeder: CILIUM_FORWARD" -j CILIUM_FORWARD 
-A OUTPUT -m comment --comment "cilium-feeder: CILIUM_OUTPUT" -j CILIUM_OUTPUT 
-A OUTPUT -j KUBE-FIREWALL 
-A CILIUM_FORWARD -o cilium_host -m comment --comment "cilium: any->cluster on cilium_host forward accept" -j ACCEPT 
-A CILIUM_FORWARD -i cilium_host -m comment --comment "cilium: cluster->any on cilium_host forward accept (nodeport)" -j ACCEPT 
-A CILIUM_FORWARD -i lxc+ -m comment --comment "cilium: cluster->any on lxc+ forward accept" -j ACCEPT 
-A CILIUM_FORWARD -i cilium_net -m comment --comment "cilium: cluster->any on cilium_net forward accept (nodeport)" -j ACCEPT 
-A CILIUM_INPUT -m mark --mark 0x200/0xf00 -m comment --comment "cilium: ACCEPT for proxy traffic" -j ACCEPT 
-A CILIUM_OUTPUT -m mark --mark 0xa00/0xe00 -m comment --comment "cilium: ACCEPT for proxy traffic" -j ACCEPT 
-A CILIUM_OUTPUT -m mark --mark 0x800/0xe00 -m comment --comment "cilium: ACCEPT for l7 proxy upstream traffic" -j ACCEPT 
-A CILIUM_OUTPUT -m mark ! --mark 0xd00/0xf00 -m mark ! --mark 0xe00/0xf00 -m mark ! --mark 0x400/0xf00 -m mark ! --mark 0xa00/0xe00 -m mark ! --mark 0x800/0xe00 -m comment --comment "cilium: host->any mark as from host" -j MARK --set-xmark 0xc00/0xf00 
-A KUBE-FIREWALL ! -s 127.0.0.0/8 -d 127.0.0.0/8 -m comment --comment "block incoming localnet connections" -m conntrack ! --ctstate RELATED,ESTABLISHED,DNAT -j DROP 
```

---
## 📚 References

- Linux kernel source: https://github.com/torvalds/linux  
- Kernel TPROXY documentation: https://www.kernel.org/doc/html/latest/networking/tproxy.html  
- Cilium L7 policy documentation: https://docs.cilium.io/en/latest/security/policy/layer7  
