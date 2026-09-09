---
title: "Linux Network - TPROXY 기초"
description: "Linux 커널의 Netfilter hook, nftables TPROXY expression, skb socket 할당, fwmark 기반 local route 흐름을 소스 코드와 실습으로 따라가보자"
date: 2026-09-09T18:19:13+09:00
lastmod: 2026-09-09T18:19:13+09:00
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

Transparent proxy는 client가 proxy의 존재를 명시적으로 알지 못해도, 중간의 host가 트래픽을 proxy로 넘겨 처리하는 방식이다.  
HTTP proxy처럼 client가 proxy 주소를 직접 설정하는 방식이 아니라, router나 node가 지나가는 트래픽을 가로채 proxy process로 전달한다.  

이때 단순히 DNAT 또는 REDIRECT로 목적지 주소와 포트를 proxy 주소로 바꿀 수도 있다.  
하지만 NAT 기반 방식은 packet의 destination을 실제로 바꾸기 때문에, proxy가 원래 목적지 주소를 정확히 다루기 어렵거나 추가 조회에 의존해야 한다.  
특히 UDP처럼 connection state만으로 원래 목적지를 안정적으로 복원하기 어려운 경우에는 이 차이가 더 중요하다.  

TPROXY(Transparent Proxy)는 이 문제를 NAT 없이 푼다.  
packet의 destination address/port는 유지한 채, Netfilter 단계에서 transparent listener socket을 찾아 `skb->sk`에 붙이고, fwmark 기반 policy routing으로 packet을 local delivery 시킨다.  

이번 글에서는 이전 글에서 살펴본 IPv4 ingress/routing 흐름 위에 TPROXY가 어디서 개입하는지 Linux 커널 소스 코드와 간단한 실습으로 따라가보자.  

---
## 🪝 NF_HOOK의 동작 자세히 보기

### nf_hook

이전 글에서는 `ip_rcv()`가 IPv4 datagram을 검증한 뒤, `NF_HOOK()`을 `PREROUTING` hook으로 실행한다는 점을 확인했다.  

`NF_HOOK()` wrapper는 내부에서 `nf_hook()`을 호출한다.  

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

`nf_hook()`은 현재 hook 지점에 등록된 hook entry가 있으면 평가를 시작한다.  
실제 순회와 verdict 처리는 `nf_hook_slow()`가 맡는다.  

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

	// hook 엔트리가 있으면 실행
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

`nf_hook_slow()`는 hook entry를 순서대로 평가하고, 각 callback이 반환한 verdict에 따라 다음 동작을 결정한다.  
실제 평가는 `nf_hook_entry_hookfn()`이 맡는다.  
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
		// 여기서의 break는 switch문에 대한 break이므로, 상위 for loop는 계속된다.
		// ACCEPT가 전체 verdict를 ACCEPT하는 것이 아닌, 현재 entry를 PASS한다는 것을
		// 여기서 더 직관적으로 볼 수 있다.
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

`nf_hook_entry_hookfn()`은 entry에 붙은 callback을 호출하는 wrapper이다.  
`entry->hook`은 함수 포인터로, 실제 hook 함수에 연결된다.  

nftables callback이든, iptables callback이든, 또는 다른 Netfilter callback이 실행될 수 있다.  

```c
// source: include/linux/netfilter.h
static inline int
nf_hook_entry_hookfn(const struct nf_hook_entry *entry, struct sk_buff *skb,
		     struct nf_hook_state *state)
{
	return entry->hook(entry->priv, skb, state);
}
```

### nftables 기준으로 hook 살펴보기

`nft_chain_filter_ipv4`를 보자.  
`hooks` 배열에 `nft_do_chain_ipv4`가 붙어있는 것을 볼 수 있다.  
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

nft의 base chain의 hook을 초기화하는 `nft_basechain_hook_init()`을 보자.  
결과적으로 `ops->hook`에 hook callback이 등록된다.  

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
	// ipv4에서는 결국 ops->hook = nft_do_chain_ipv4로 됨
	ops->hook		= hook->type->hooks[ops->hooknum];
	ops->hook_ops_type	= NF_HOOK_OP_NF_TABLES;
}

```

`nft_do_chain_ipv4()`는 실제 core인 `nft_do_chain()`을 실행한다.  
다른 family에서도 비슷한 구조이다.  

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

`nft_do_chain()`은 nftables rule 평가의 core이다.  
각 rule과 expression을 순회하며 평가 결과를 만들고, verdict를 Netfilter core로 반환한다.  

```c
// source: net/netfilter/nf_tables_core.c
unsigned int
nft_do_chain(struct nft_pktinfo *pkt, void *priv)
{
	// ...
	
	// 첫 rule가져오기
	rule = (struct nft_rule_dp *)blob->data;
next_rule:
	regs.verdict.code = NFT_CONTINUE;
	// 각 rule에 대해 평가
	for (; !rule->is_last ; rule = nft_rule_next(rule)) {
		// 각 expression 순회
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

	// verdict를 netfilter에 리턴
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
	
	// base chain의 default policy대로 리턴
	return nft_base_chain(basechain)->policy;
}
EXPORT_SYMBOL_GPL(nft_do_chain);
```

---
## 🧲 TPROXY expression

TPROXY expression은 `.eval = nft_tproxy_eval`로 등록된다.  

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

`nft_do_chain()`에서 `.eval`이 실행된다.  

```c
// source: net/netfilter/nf_tables_core.c
			// ...
			else if (expr->ops != &nft_payload_fast_ops ||
				 !nft_payload_fast_eval(expr, &regs, pkt))
				expr_call_ops_eval(expr, &regs, pkt);
		// ...
```

`nft_tproxy_eval()`에서는 IPv4 패킷의 경우 `nft_tproxy_eval_v4()`가 실행된다.  
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

`nft_tproxy_eval_v4()`가 실제 TPROXY의 코어이다.  
기존 연결로 매칭되는 socket이 있으면 그 socket을 찾고, 없으면 TPROXY target 주소와 포트에서 대기 중인 listener를 찾는다.  
조건에 맞는 transparent socket을 찾으면 그 socket을 `skb`에 할당한다.  

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
	// 원래 패킷의 4-tuple기준으로 이미 연결된 socket이 있는지 찾기
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
		// 이미 있는 커넥션이 없으면, TPROXY target 주소와 포트에 listening 중인 socket 조회
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

`nf_tproxy_assign_sock()`은 찾은 socket을 `skb`에 할당한다.  

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

이제 transparent proxy로 포워딩, 즉 local route가 된 이후 TCP 계층에서 무슨 일이 일어나는지 보자.  
`__inet_lookup_skb()`가 실행된다.  

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

`__inet_lookup_skb()`에서는 먼저 `inet_steal_sock()`이 실행된다.  

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

`inet_steal_sock()`에서는 `skb_steal_sock()`이 실행된다.  
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

`skb_steal_sock()`은 `skb->sk`에 들어 있던 socket을 가져오고, `skb` 쪽의 socket 참조를 비운다.  
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

이후, `tcp_v4_rcv()`에서는 `tcp_v4_do_rcv()`로 실제 처리를 한다.  

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

TCP의 경우 listener가 SYN을 처리하면서 request socket을 만들 때, 원래 `skb`의 IP/TCP header를 기준으로 주소와 포트를 채운다.  
```c
// source: net/ipv4/tcp_ipv4.c
static void tcp_v4_init_req(struct request_sock *req,
			    const struct sock *sk_listener,
			    struct sk_buff *skb)
{
	struct inet_request_sock *ireq = inet_rsk(req);
	struct net *net = sock_net(sk_listener);

	// saddr, daddr을 skb로부터
	sk_rcv_saddr_set(req_to_sk(req), ip_hdr(skb)->daddr);
	sk_daddr_set(req_to_sk(req), ip_hdr(skb)->saddr);
	RCU_INIT_POINTER(ireq->ireq_opt, tcp_v4_save_options(net, skb));
}
```

port 번호는 `tcp_openreq_init()`에서 초기화된다.  
이 함수는 TCP request socket에 필요한 다른 필드들도 함께 초기화한다.  
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
## 🧭 전체 흐름 정리

앞에서 본 코드는 Netfilter와 TCP socket 처리에 집중되어 있었다.  
fwmark 기반 local route까지 함께 넣어 정리하면 전체 흐름은 다음과 같다:  

```text
NIC
 ↓
...
 ↓
ip_rcv()
 ↓
NF_HOOK(NF_INET_PRE_ROUTING)
 ↓
nftables
  ↓
nft_do_chain()
  ↓
[tproxy to :PORT]
  ↓
nft_tproxy_eval_v4()
  ↓
nf_tproxy_get_sock_v4(... ESTABLISHED)
  ↓ 없으면
nf_tproxy_get_sock_v4(... LISTENER)
  ↓
nf_tproxy_sk_is_transparent()
  ↓
nf_tproxy_assign_sock()
    skb_orphan(skb)
    skb->sk = sk
    skb->destructor = sock_edemux
 ↓
[meta mark set 0x..]             <-- fwmark
 ↓
NF_ACCEPT
 ↓
ip_rcv_finish()
 ↓
ip_route_input_noref()
   ↑
   skb->mark를 보고 policy routing
   ip rule fwmark -> local route
 ↓
dst_input()
 ↓
ip_local_deliver()
 ↓
NF_INET_LOCAL_IN
 ↓
ip_local_deliver_finish()
 ↓
ip_protocol_deliver_rcu()
 ↓
tcp_v4_rcv()
 ↓
__inet_lookup_skb()
 ↓
inet_steal_sock()
 ↓
skb_steal_sock()
   sk = skb->sk
   skb->sk = NULL
   skb->destructor = NULL
 ↓
그 sk로 tcp_v4_do_rcv()
 ↓
proxy socket
```

---
## 🧪 실습: TPROXY

단일 서버에서 진행해보자.  

client는 별도의 network namespace에서 실행한다.  
`ip` 명령어와 network namespace가 익숙하지 않다면 [이전 글]({{< relref "post/linux-network-ip" >}})을 먼저 참조하자.  
```bash
# client namespace 생성
sudo ip netns add cli

# veth pair
sudo ip link add veth-tp type veth peer name veth-cli
sudo ip link set veth-cli netns cli

sudo ip addr add 10.10.0.1/24 dev veth-tp
sudo ip link set veth-tp up

sudo ip netns exec cli ip addr add 10.10.0.2/24 dev veth-cli
sudo ip netns exec cli ip link set lo up
sudo ip netns exec cli ip link set veth-cli up

sudo ip netns exec cli ip route add default via 10.10.0.1
```

`ip rule`로 fwmark가 `1`인 패킷은 100번 routing table을 보도록 만든다.  
table 100에는 `lo`로 향하는 local default route를 넣는다.  
```bash
sudo ip rule add fwmark 1 lookup 100
sudo ip route add local 0.0.0.0/0 dev lo table 100
```

nftables rule을 통해 mark를 추가한다.  
`PREROUTING` 단계에서 `veth-tp`로 들어온 TCP 80번 트래픽을 `:50080` transparent proxy로 보내고, mark를 `1`로 설정한다.  
```bash
# table 추가
sudo nft add table inet tp

# base chain 추가
sudo nft 'add chain inet tp prerouting {
    type filter hook prerouting priority mangle;
    policy accept;
}'

# veth-tp dev에서 트래픽이 오고 목적지 포트가 80이면 :50080 TPROXY로 보내고 mark를 1로 설정
sudo nft 'add rule inet tp prerouting \
    iifname "veth-tp" tcp dport 80 \
    tproxy to :50080 meta mark set 1 accept'
```

아래는 간단한 TPROXY 서버 구현체이다.  
실제 환경에서는 Envoy, HAProxy, Squid 등이 쓰인다.  
```python
# tproxy.py
import socket

# 커널이 transparent socket을 인식
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

서버를 root namespace에서 실행해준다.  
```bash
sudo python3 tproxy.py
```

client ns에서 testnet주소와 같은 무작위 주소로 연결해보자.  
```bash
sudo ip netns exec cli curl -v http://198.51.100.10/
```

TPROXY 서버에도 로그가 기록된 것을 볼 수 있다.  
TPROXY의 child socket, 즉 `accept()`로 받아 client와 연결을 맺은 socket에서 local 주소가 `198.51.100.10:80`임을 볼 수 있다.  
```bash
# 읽기 쉽게 HTTP request payload는 줄바꿈해서 표기
peer = ('10.10.0.2', 39112) 
local = ('198.51.100.10', 80) 
received = b'
	GET / HTTP/1.1\r\n
	Host: 198.51.100.10\r\n
	User-Agent: curl/8.18.0\r\n
	Accept: */*\r\n\r\n'
```

socket에 원래 목적지 주소가 보존되는 것은 routing 이후에 우연히 만들어지는 동작이 아니다.  
`PREROUTING` 단계에서 먼저 transparent listener를 찾고, `skb->sk`에 TPROXY listener를 넣은 뒤, fwmark에 의해 local route가 이루어진다.  
그 뒤 L4 계층에서 `skb->sk`가 steal되고, proxy는 원래 목적지 주소 기준의 socket으로 트래픽을 받는다.  

첫 SYN패킷이 도착하는 과정이 다음과 같다:  
```text
SYN skb
10.0.0.2:39112 -> 198.51.100.10:80
        │
        ▼
PREROUTING
        │
        ├─ TPROXY
        │    skb->sk = listener(:50080)
        │
        └─ mark = 1
        │
        ▼
route lookup
        │
        │ skb->daddr = 198.51.100.10
        │ skb->saddr = 10.0.0.2
        │ skb->mark  = 1
        ▼
table 100
local 0/0
        │
        ▼
RTN_LOCAL
        │
        ▼
ip_local_deliver()
        │
        ▼
TCP
skb_steal_sock()
        │
        ▼
listener로 SYN 처리
        │
        ▼
request_sock
        │
        ▼
child socket
```


---
## 🔎 Cilium에서의 TPROXY

TPROXY를 활용하는 대표적인 케이스로 Kubernetes CNI 중 하나인 Cilium이 있다.  

Cilium의 L7 policy traffic은 node-local Envoy를 거친다.  
아래 Kubernetes Node에서는 proxy traffic용 fwmark와 local route가 구성된 것을 볼 수 있다.  
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

다음으로는 iptables rule을 보자.  
`0x200/0xf00` mark를 하는 것을 볼 수 있다.  
이 노드에서는 FQDN egress를 포함한 L7 rule에서 node-local Envoy로 향하는 proxy traffic을 mark 기반 local route와 함께 처리한다.  
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
