---
title: "Linux Network - Routing(2)"
description: "Trace Linux kernel IPv4 ingress routing decisions, FIB lookup, source validation, and local delivery through source code"
date: 2026-09-07T21:02:18+09:00
image:
math: false
license:
hidden: false
comments: true
draft: false

tags:
    - Linux
    - Network
    - Routing
    - Kernel
    - Netfilter

categories:
    - Network
---

This time, let's look at how routing and packet processing work inside the Linux kernel.  
Linux kernel source code is not easy to read, so this post focuses on the core flow while leaving out many internal optimizations, caches, and error-handling details.  

---
## 🧭 From Ingress to Forwarding

First, let's follow what happens when an IPv4 packet enters the Linux kernel.  

`ip_rcv()` receives and processes the socket buffer (`skb`) passed up from L2.  
The main IPv4 packet validation happens in `ip_rcv_core()`.  

```c
// source: net/ipv4/ip_input.c
/*
 * IP receive entry point
 */
int ip_rcv(struct sk_buff *skb, struct net_device *dev, struct packet_type *pt,
	   struct net_device *orig_dev)
{
	// Get network namespace information
	// reference:
	// - linux/include/linux/netdevice.h
	// - linux/include/net/net_namespace.h
	struct net *net = dev_net(dev);

	// Basic IPv4 packet validation
	skb = ip_rcv_core(skb, net);
	// Drop if validation failed
	if (skb == NULL)
		return NET_RX_DROP;

	// Netfilter: PREROUTING stage.
	// If no hook is registered, or hook evaluation allows processing to continue,
	// the NF_HOOK() wrapper calls ip_rcv_finish() through okfn()
	return NF_HOOK(NFPROTO_IPV4, NF_INET_PRE_ROUTING,
		       net, NULL, skb, dev, NULL,
		       ip_rcv_finish);
}
```

`ip_rcv_core()` validates the IPv4 datagram.  

```c
// source: net/ipv4/ip_input.c
/*
 * 	Main IP Receive routine.
 */
static struct sk_buff *ip_rcv_core(struct sk_buff *skb, struct net *net)
{
	const struct iphdr *iph;
	int drop_reason;
	u32 len;

	// ...
	
	drop_reason = SKB_DROP_REASON_NOT_SPECIFIED;
	if (!pskb_may_pull(skb, sizeof(struct iphdr)))
		goto inhdr_error;

	iph = ip_hdr(skb);

	/*
	 *	RFC1122: 3.2.1.2 MUST silently discard any IP frame that fails the checksum.
	 *
	 *	Is the datagram acceptable?
	 *
	 *	1.	Length at least the size of an ip header
	 *	2.	Version of 4
	 *	3.	Checksums correctly. [Speed optimisation for later, skip loopback checksums]
	 *	4.	Doesn't have a bogus length
	 */
	 
	 // ...
	 
	/* Our transport medium may have padded the buffer out. Now we know it
	 * is IP we can trim to the true length of the frame.
	 * Note this now means skb->len holds ntohs(iph->tot_len).
	 */
	if (pskb_trim_rcsum(skb, len)) {
		__IP_INC_STATS(net, IPSTATS_MIB_INDISCARDS);
		goto drop;
	}

	// Set the L4 (transport) header start position
	iph = ip_hdr(skb);
	skb->transport_header = skb->network_header + iph->ihl*4;

	/* Remove any debris in the socket control block */
	memset(IPCB(skb), 0, sizeof(struct inet_skb_parm));
	IPCB(skb)->iif = skb->skb_iif;

	/* Must drop socket now because of tproxy. */
	if (!skb_sk_is_prefetched(skb))
		skb_orphan(skb);

	return skb;

csum_error:
	drop_reason = SKB_DROP_REASON_IP_CSUM;
	__IP_INC_STATS(net, IPSTATS_MIB_CSUMERRORS);
inhdr_error:
	if (drop_reason == SKB_DROP_REASON_NOT_SPECIFIED)
		drop_reason = SKB_DROP_REASON_IP_INHDR;
	__IP_INC_STATS(net, IPSTATS_MIB_INHDRERRORS);
drop:
	kfree_skb_reason(skb, drop_reason);
out:
	return NULL;
}
```

After validation in `ip_rcv_core()`, `NF_HOOK()` runs the `PREROUTING` hook.  
If no hook is registered, or hook evaluation allows processing to continue, the `NF_HOOK()` wrapper calls the function passed as `okfn()`.  
In `ip_rcv()`, that `okfn` is `ip_rcv_finish()`.  
```c
// source: include/linux/netfilter.h
static inline int
NF_HOOK(uint8_t pf, unsigned int hook, struct net *net, struct sock *sk, struct sk_buff *skb,
	struct net_device *in, struct net_device *out,
	int (*okfn)(struct net *, struct sock *, struct sk_buff *))
{
	int ret = nf_hook(pf, hook, net, sk, skb, in, out, okfn);
	if (ret == 1)
		ret = okfn(net, sk, skb);
	return ret;
}
```

`ip_rcv_finish()` calls `ip_rcv_finish_core()`.  

```c
// source: net/ipv4/ip_input.c
static int ip_rcv_finish(struct net *net, struct sock *sk, struct sk_buff *skb)
{
	struct net_device *dev = skb->dev;
	int ret;

	/* if ingress device is enslaved to an L3 master device pass the
	 * skb to its handler for processing
	 */
	skb = l3mdev_ip_rcv(skb);
	if (!skb)
		return NET_RX_SUCCESS;

	ret = ip_rcv_finish_core(net, skb, dev, NULL);
	if (ret != NET_RX_DROP)
		ret = dst_input(skb);
	return ret;
}
```


The main job of `ip_rcv_finish_core()` is to start the routing decision.  
```c
// source: net/ipv4/ip_input.c
static int ip_rcv_finish_core(struct net *net,
			      struct sk_buff *skb, struct net_device *dev,
			      const struct sk_buff *hint)
{
	const struct iphdr *iph = ip_hdr(skb);
	struct rtable *rt;
	int drop_reason;

	// ...

	/*
	 *	Initialise the virtual path cache for the packet. It describes
	 *	how the packet travels inside Linux networking.
	 */
	// Routing Decision
	if (!skb_valid_dst(skb)) {
		drop_reason = ip_route_input_noref(skb, iph->daddr, iph->saddr,
						   ip4h_dscp(iph), dev);
		if (unlikely(drop_reason))
			goto drop_error;
	} else {
		struct in_device *in_dev = __in_dev_get_rcu(dev);

		if (in_dev && IN_DEV_ORCONF(in_dev, NOPOLICY))
			IPCB(skb)->flags |= IPSKB_NOPOLICY;
	}

	// ...

	return NET_RX_SUCCESS;

drop:
	kfree_skb_reason(skb, drop_reason);
	return NET_RX_DROP;

drop_error:
	if (drop_reason == SKB_DROP_REASON_IP_RPFILTER)
		__NET_INC_STATS(net, LINUX_MIB_IPRPFILTER);
	goto drop;
}

```

`ip_route_input_noref()` runs `ip_route_input_rcu()`.  
> **NOTE:**
> RCU means Read-Copy-Update. It is a synchronization mechanism used in Linux kernel read-mostly situations.
> It lets readers safely keep reading existing objects, while old objects can be reclaimed after a grace period. A common publishing pattern is replacing a pointer with one that points to a new object.

```c
// source: net/ipv4/route.c
enum skb_drop_reason ip_route_input_noref(struct sk_buff *skb, __be32 daddr,
					  __be32 saddr, dscp_t dscp,
					  struct net_device *dev)
{
	enum skb_drop_reason reason;
	struct fib_result res;

	rcu_read_lock();
	reason = ip_route_input_rcu(skb, daddr, saddr, dscp, dev, &res);
	rcu_read_unlock();

	return reason;
}
EXPORT_SYMBOL(ip_route_input_noref);
```

On the normal non-multicast path, `ip_route_input_rcu()` calls `ip_route_input_slow()`.  
```c
// source: net/ipv4/route.c
/* called with rcu_read_lock held */
static enum skb_drop_reason
ip_route_input_rcu(struct sk_buff *skb, __be32 daddr, __be32 saddr,
		   dscp_t dscp, struct net_device *dev,
		   struct fib_result *res)
{

	// multicast handling...

	return ip_route_input_slow(skb, daddr, saddr, dscp, dev, res);
}
```

The core routing work happens in `ip_route_input_slow()`.  

Here, the kernel runs `fib_lookup()`.  
It also checks whether forwarding is enabled and branches into local delivery or forwarding depending on the route result type.  

```c
// source: net/ipv4/route.c
static enum skb_drop_reason
ip_route_input_slow(struct sk_buff *skb, __be32 daddr, __be32 saddr,
		    dscp_t dscp, struct net_device *dev,
		    struct fib_result *res)
{
	enum skb_drop_reason reason = SKB_DROP_REASON_NOT_SPECIFIED;
	struct in_device *in_dev = __in_dev_get_rcu(dev);
	struct flow_keys *flkeys = NULL, _flkeys;
	struct net    *net = dev_net(dev);
	struct ip_tunnel_info *tun_info;
	int		err = -EINVAL;
	unsigned int	flags = 0;
	u32		itag = 0;
	struct rtable	*rth;
	struct flowi4	fl4;
	bool do_cache = true;

	// ...

	/*
	 *	Now we are ready to route packet.
	 */
	fl4.flowi4_l3mdev = 0;
	fl4.flowi4_oif = 0;
	fl4.flowi4_iif = dev->ifindex;
	fl4.flowi4_mark = skb->mark;
	fl4.flowi4_dscp = dscp;
	fl4.flowi4_scope = RT_SCOPE_UNIVERSE;
	fl4.flowi4_flags = 0;
	fl4.daddr = daddr;
	fl4.saddr = saddr;
	fl4.flowi4_uid = sock_net_uid(net, NULL);
	fl4.flowi4_multipath_hash = 0;

	if (fib4_rules_early_flow_dissect(net, skb, &fl4, &_flkeys)) {
		flkeys = &_flkeys;
	} else {
		fl4.flowi4_proto = 0;
		fl4.fl4_sport = 0;
		fl4.fl4_dport = 0;
	}

	// Look up the destination through FIB
	err = fib_lookup(net, &fl4, res, 0);
	if (err != 0) {
		if (!IN_DEV_FORWARD(in_dev))
			err = -EHOSTUNREACH;
		goto no_route;
	}
	
	// ...

	err = -EINVAL;
	// For a LOCAL route, validate the source and move to the local-delivery route path
	if (res->type == RTN_LOCAL) {
		reason = fib_validate_source_reason(skb, saddr, daddr, dscp,
						    0, dev, in_dev, &itag);
		if (reason)
			goto martian_source;
		goto local_input;
	}

	// Check whether forwarding is enabled on the device
	if (!IN_DEV_FORWARD(in_dev)) {
		err = -EHOSTUNREACH;
		goto no_route;
	}
	
	// If it is not an RTN_UNICAST route that can be forwarded, treat it as an invalid destination
	if (res->type != RTN_UNICAST) {
		reason = SKB_DROP_REASON_IP_INVALID_DEST;
		goto martian_destination;
	}

make_route:
	reason = ip_mkroute_input(skb, res, in_dev, daddr, saddr, dscp,
				  flkeys);
	// ...
}
```

`ip_mkroute_input()` calls `__mkroute_input()`, which creates a routing cache entry.  
```c
// source: net/ipv4/route.c
static enum skb_drop_reason
ip_mkroute_input(struct sk_buff *skb, struct fib_result *res,
		 struct in_device *in_dev, __be32 daddr,
		 __be32 saddr, dscp_t dscp, struct flow_keys *hkeys)
{
	// ...

	/* create a routing cache entry */
	return __mkroute_input(skb, res, in_dev, daddr, saddr, dscp);
}
```

In `__mkroute_input()`, the function pointer for `ip_forward` is assigned to `rth->dst.input`.  
Then the nexthop information is stored, and the lookup result dst is attached to the skb.  
```c
// source: net/ipv4/route.c
/* called in rcu_read_lock() section */
static enum skb_drop_reason
__mkroute_input(struct sk_buff *skb, const struct fib_result *res,
		struct in_device *in_dev, __be32 daddr,
		__be32 saddr, dscp_t dscp)
{
	enum skb_drop_reason reason = SKB_DROP_REASON_NOT_SPECIFIED;
	struct fib_nh_common *nhc = FIB_RES_NHC(*res);
	// Output device taken from nexthop information
	struct net_device *dev = nhc->nhc_dev;
	struct fib_nh_exception *fnhe;
	struct rtable *rth;
	int err;
	struct in_device *out_dev;
	bool do_cache;
	u32 itag = 0;

	// source validation
	err = fib_validate_source(skb, saddr, daddr, dscp, FIB_RES_OIF(*res),
				  in_dev->dev, in_dev, &itag);
	if (err < 0) {
		reason = -err;
		ip_handle_martian_source(in_dev->dev, in_dev, skb, daddr,
					 saddr);

		goto cleanup;
	}

	// ...

	// Packets using this route will be sent to ip_forward()
	rth->dst.input = ip_forward;

	// Set next-hop information
	rt_set_nexthop(rth, daddr, res, fnhe, res->fi, res->type, itag,
		       do_cache);
			   
	lwtunnel_set_redirect(&rth->dst);
	// Store the lookup result dst in skb->dst
	skb_dst_set(skb, &rth->dst);
out:
	reason = SKB_NOT_DROPPED_YET;
cleanup:
	return reason;
}
```

`ip_forward()` implements IPv4 forwarding behavior.  
```c
// source: net/ipv4/ip_forward.c
int ip_forward(struct sk_buff *skb)
{
	u32 mtu;
	struct iphdr *iph;	/* Our header */
	struct rtable *rt;	/* Route we use */
	struct ip_options *opt	= &(IPCB(skb)->opt);
	struct net *net;
	SKB_DR(reason);

	skb_forward_csum(skb);
	net = dev_net(skb->dev);

	/*
	 *	According to the RFC, we must first decrease the TTL field. If
	 *	that reaches zero, we must reply an ICMP control message telling
	 *	that the packet's lifetime expired.
	 */
	if (ip_hdr(skb)->ttl <= 1)
		goto too_many_hops;

	// ...
	
	rt = skb_rtable(skb);

	if (opt->is_strictroute && rt->rt_uses_gateway)
		goto sr_failed;

	__IP_INC_STATS(net, IPSTATS_MIB_OUTFORWDATAGRAMS);

	// Check the MTU on the forwarding path. If needed, send ICMP Fragmentation Needed and drop
	IPCB(skb)->flags |= IPSKB_FORWARDED;
	mtu = ip_dst_mtu_maybe_forward(&rt->dst, true);
	if (ip_exceeds_mtu(skb, mtu)) {
		IP_INC_STATS(net, IPSTATS_MIB_FRAGFAILS);
		icmp_send(skb, ICMP_DEST_UNREACH, ICMP_FRAG_NEEDED,
			  htonl(mtu));
		SKB_DR_SET(reason, PKT_TOO_BIG);
		goto drop;
	}

	/* We are about to mangle packet. Copy it! */
	if (skb_cow(skb, LL_RESERVED_SPACE(rt->dst.dev)+rt->dst.header_len))
		goto drop;
	iph = ip_hdr(skb);

	/* Decrease ttl after skb cow done */
	ip_decrease_ttl(iph);
	
	// ...

	// FORWARD HOOK
	return NF_HOOK(NFPROTO_IPV4, NF_INET_FORWARD,
		       net, NULL, skb, skb->dev, rt->dst.dev,
		       ip_forward_finish);
	
	// ...
}
```

### Summary

The overall flow can be summarized as follows:  
```c
// source: summary of net/ipv4/ip_input.c, net/ipv4/route.c, net/ipv4/ip_forward.c
ip_rcv()
  ↓
ip_rcv_core()
  ↓
NF_HOOK(PREROUTING)
  ↓
ip_rcv_finish()
  ↓
ip_rcv_finish_core()
  ↓
ip_route_input_noref()
  ↓
ip_route_input_rcu()
  ↓
ip_route_input_slow()
  ↓
build flowi4
  ↓
fib_lookup() 
  ↓
check res->type
  ├─ RTN_LOCAL
  │    → local_input
  │    → build dst/rtable for local delivery
  │
  └─ RTN_UNICAST + forwarding enabled
       ↓
     ip_mkroute_input()
       ↓
     __mkroute_input()
       ↓
     output device / source validation
       ↓
     create rth
       ↓
     rth->dst.input = ip_forward
       ↓
     rt_set_nexthop()
       ↓
     skb_dst_set(skb, &rth->dst)
```

Looking back at `ip_rcv_finish()`, we have the following code.  
If `ip_rcv_finish_core()` does not return `DROP`, it runs `dst_input(skb)`.  
That calls `skb->dst->input(skb)`, and because `rth->dst.input = ip_forward` was stored earlier, `ip_forward(skb)` is eventually executed.  
```c
// source: net/ipv4/ip_input.c
	// ...
	ret = ip_rcv_finish_core(net, skb, dev, NULL);
	if (ret != NET_RX_DROP)
		ret = dst_input(skb);
	return ret;
```

---
## 🧩 FIB Rules and fwmark

Now let's look at the source code related to policy routing, focusing on marks.  

Before looking at `fib_lookup()`, recall the basic structure of `ip rule`.  
```bash
# source: iproute2/ip/iprule.c
$ ip rule 
0: from all lookup local 
32766: from all lookup main 
32767: from all lookup default
```

The code below shows a similar structure.  
In `ip rule`, the `main` and `default` tables had lower priorities, and the kernel fallback order follows the same idea.  
```c
// source: include/net/ip_fib.h
static inline int fib_lookup(struct net *net, struct flowi4 *flp,
			     struct fib_result *res, unsigned int flags)
{
	struct fib_table *tb;
	int err = -EAGAIN;

	flags |= FIB_LOOKUP_NOREF;
	if (net->ipv4.fib_has_custom_rules)
		return __fib_lookup(net, flp, res, flags);

	rcu_read_lock();

	res->tclassid = 0;
	
	// table main (32766)

	tb = rcu_dereference_rtnl(net->ipv4.fib_main);
	if (tb)
		err = fib_table_lookup(tb, flp, res, flags);

	if (err != -EAGAIN)
		goto out;

	// table default (32767)

	tb = rcu_dereference_rtnl(net->ipv4.fib_default);
	if (tb)
		err = fib_table_lookup(tb, flp, res, flags);

	if (err == -EAGAIN)
		err = -ENETUNREACH;
out:
	rcu_read_unlock();

	return err;
}
```

The `local` table does not appear directly in this code.  
When there are no custom rules, the `local` table is handled through a trie structure that aliases the `main` table. Once a custom rule is added, `fib_unmerge()` separates it.  
I will not go deeper into that here, but you can see traces of it in `fib_new_table()` and `fib_unmerge()` in `net/ipv4/fib_frontend.c`.  

`__fib_lookup()` calls `fib_rules_lookup()`.  
```c
// source: net/ipv4/fib_rules.c
int __fib_lookup(struct net *net, struct flowi4 *flp,
		 struct fib_result *res, unsigned int flags)
{
	struct fib_lookup_arg arg = {
		.result = res,
		.flags = flags,
	};
	int err;

	/* update flow if oif or iif point to device enslaved to l3mdev */
	l3mdev_update_flow(net, flowi4_to_flowi(flp));

	err = fib_rules_lookup(net->ipv4.rules_ops, flowi4_to_flowi(flp), 0, &arg);
#ifdef CONFIG_IP_ROUTE_CLASSID
	if (arg.rule)
		res->tclassid = ((struct fib4_rule *)arg.rule)->tclassid;
	else
		res->tclassid = 0;
#endif

	if (err == -ESRCH)
		err = -ENETUNREACH;

	return err;
}
```

`fib_rules_lookup()` iterates over each rule and evaluates whether it matches through `fib_rule_match()`.  
```c
// source: net/core/fib_rules.c
int fib_rules_lookup(struct fib_rules_ops *ops, struct flowi *fl,
		     int flags, struct fib_lookup_arg *arg)
{
	struct fib_rule *rule;
	int err;

	rcu_read_lock();

	list_for_each_entry_rcu(rule, &ops->rules_list, list) {
jumped:
		if (!fib_rule_match(rule, ops, fl, flags, arg))
			continue;
			
		// ...
out:
	rcu_read_unlock();

	return err;
}
```

In `fib_rule_match()`, the rule is matched against several conditions, including the mark:  
```c
// source: net/core/fib_rules.c
static int fib_rule_match(struct fib_rule *rule, struct fib_rules_ops *ops,
			  struct flowi *fl, int flags,
			  struct fib_lookup_arg *arg)
{
	int iifindex, oifindex, ret = 0;

	iifindex = READ_ONCE(rule->iifindex);
	if (iifindex && !fib_rule_iif_match(rule, iifindex, fl))
		goto out;

	oifindex = READ_ONCE(rule->oifindex);
	if (oifindex && !fib_rule_oif_match(rule, oifindex, fl))
		goto out;

	// mark
	// rule->mark:      fwmark value from ip rule
	// rule->mark_mask: mask value after fwmark in ip rule
	// fl->flowi_mark:  mark used for route lookup
	// The condition below computes a bitwise xor between the rule fwmark and flowi_mark,
	// then applies the mask to keep only the relevant bits. If the result is 0, it matches and continues.
	// fwmark is masked so that only a specific bit range is evaluated.
	// Other bits may be used for purposes other than rule matching.
	if ((rule->mark ^ fl->flowi_mark) & rule->mark_mask)
		goto out;

	if (rule->tun_id && (rule->tun_id != fl->flowi_tun_key.tun_id))
		goto out;

	if (rule->l3mdev && !l3mdev_fib_rule_match(rule->fr_net, fl, arg))
		goto out;

	if (uid_lt(fl->flowi_uid, rule->uid_range.start) ||
	    uid_gt(fl->flowi_uid, rule->uid_range.end))
		goto out;

	ret = INDIRECT_CALL_MT(ops->match,
			       fib6_rule_match,
			       fib4_rule_match,
			       rule, fl, flags);
out:
	return (rule->flags & FIB_RULE_INVERT) ? !ret : ret;
}
```

---
## 🛡️ Source validation

We have seen `fib_validate_source()` several times in the previous snippets.  
This logic checks whether the source IP is valid.  

Some cases take a fast path, while systems with custom `ip rule` entries run a full check.  
You can also see that if the source IP is one of the host's own addresses while forwarding, the packet is treated as suspicious and dropped.  

```c
// source: net/ipv4/fib_frontend.c
int fib_validate_source(struct sk_buff *skb, __be32 src, __be32 dst,
			dscp_t dscp, int oif, struct net_device *dev,
			struct in_device *idev, u32 *itag)
{
	// RPF: Reverse Path Filtering
	//     Checks whether the reverse-path lookup uses the ingress interface, then filters accordingly
	//     0: No source validation
	//     1: Strict mode as defined in RFC 3704.
	//     2: Loose mode as defined in RFC 3704.
	// secpath_exists: checks whether the packet is protected by IPsec or similar; if so, set to 0.
	//     Applying RPF to IPsec or similar protected packets is inappropriate
	// If the packet is not protected, use the interface sysctl rp_filter value (net.ipv4.<iface>.rp_filter)
	int r = secpath_exists(skb) ? 0 : IN_DEV_RPFILTER(idev);
	struct net *net = dev_net(dev);

	if (!r && !fib_num_tclassid_users(net) &&
	    (dev->ifindex != oif || !IN_DEV_TX_REDIRECTS(idev))) {
		
		// Fast success if local routes are accepted
		// Corresponds to the `accept_local` sysctl
		if (IN_DEV_ACCEPT_LOCAL(idev))
			goto ok;
		/* with custom local routes in place, checking local addresses
		 * only will be too optimistic, with custom rules, checking
		 * local addresses only can be too strict, e.g. due to vrf
		 */
		// If there is a custom local route or custom rule, validate through full_check
		if (net->ipv4.fib_has_custom_local_routes ||
		    fib4_has_custom_rules(net))
			goto full_check;
		
		/* Within the same container, it is regarded as a martian source,
		 * and the same host but different containers are not.
		 */
		
		// If the source IP is one of the host's own addresses, this is suspicious, so drop
		// NOTE: this is FORWARD, not OUTPUT
		if (inet_lookup_ifaddr_rcu(net, src))
			return -SKB_DROP_REASON_IP_LOCAL_SOURCE;

ok:
		*itag = 0;
		return 0;
	}

full_check:
	return __fib_validate_source(skb, src, dst, dscp, oif, dev, r, idev,
				     itag);
}
```

The full-check path, `__fib_validate_source()`, performs a reverse FIB lookup.  
Two lookups are involved. The first finds the best path, and the second checks additional information for comparison against the ingress interface.  

```c
// source: net/ipv4/fib_frontend.c
/* Given (packet source, input interface) and optional (dst, oif, tos):
 * - (main) check, that source is valid i.e. not broadcast or our local
 *   address.
 * - figure out what "logical" interface this packet arrived
 *   and calculate "specific destination" address.
 * - check, that packet arrived from expected physical interface.
 * called with rcu_read_lock()
 */
static int __fib_validate_source(struct sk_buff *skb, __be32 src, __be32 dst,
				 dscp_t dscp, int oif, struct net_device *dev,
				 int rpf, struct in_device *idev, u32 *itag)
{
	struct net *net = dev_net(dev);
	enum skb_drop_reason reason;
	struct flow_keys flkeys;
	int ret, no_addr;
	struct fib_result res;
	struct flowi4 fl4;
	bool dev_match;

	// Build the reverse flow
	// - Reverse the interface direction
	// The reverse-path oif is not decided yet, so set it to 0
	// The reverse-path iif can come from the original oif, but if it is unknown because there was no previous FIB lookup, use the loopback interface instead
	fl4.flowi4_oif = 0;
	fl4.flowi4_l3mdev = l3mdev_master_ifindex_rcu(dev);
	fl4.flowi4_iif = oif ? : LOOPBACK_IFINDEX;
	// - Reverse the IP addresses too
	fl4.daddr = src;
	fl4.saddr = dst;
	
	// ...

	// Load the mark value as well
	// Requires the src_valid_mark sysctl (net.ipv4.conf.<iface>.src_valid_mark)
	fl4.flowi4_mark = IN_DEV_SRC_VMARK(idev) ? skb->mark : 0;
	// Reverse L4 information too
	if (!fib4_rules_early_flow_dissect(net, skb, &fl4, &flkeys)) {
		fl4.flowi4_proto = 0;
		fl4.fl4_sport = 0;
		fl4.fl4_dport = 0;
	} else {
		swap(fl4.fl4_sport, fl4.fl4_dport);
	}

	// Run reverse-path lookup
	if (fib_lookup(net, &fl4, &res, 0))
		goto last_resort;
	if (res.type != RTN_UNICAST) {
		if (res.type != RTN_LOCAL) {
			reason = SKB_DROP_REASON_IP_INVALID_SOURCE;
			goto e_inval;
		// If the route type is LOCAL but accept_local (net.ipv4.conf.<iface>.accept_local) is 0, drop
		} else if (!IN_DEV_ACCEPT_LOCAL(idev)) {
			reason = SKB_DROP_REASON_IP_LOCAL_SOURCE;
			goto e_inval;
		}
	}
	fib_combine_itag(itag, &res);

	dev_match = fib_info_nh_uses_dev(res.fi, dev);
	/* This is not common, loopback packets retain skb_dst so normally they
	 * would not even hit this slow path.
	 */
	dev_match = dev_match || (res.type == RTN_LOCAL &&
				  dev == net->loopback_dev);
	
	// If dev_match passes, allow immediately
	if (dev_match) {
		ret = FIB_RES_NHC(res)->nhc_scope >= RT_SCOPE_HOST;
		return ret;
	}
	if (no_addr)
		goto last_resort;
	// If rp_filter is 1 (strict), drop
	if (rpf == 1)
		goto e_rpf;
	
	// Now set oif to the ingress interface
	fl4.flowi4_oif = dev->ifindex;

	// Second lookup
	// This time, oif is set to ingress
	// Check whether there is a valid path
	// Check whether the source is reachable through the ingress interface
	// A value greater than 0 means the source can be treated as direct/link reachable
	ret = 0;
	if (fib_lookup(net, &fl4, &res, FIB_LOOKUP_IGNORE_LINKSTATE) == 0) {
		if (res.type == RTN_UNICAST)
			ret = FIB_RES_NHC(res)->nhc_scope >= RT_SCOPE_HOST;
	}
	return ret;

last_resort:
	if (rpf)
		goto e_rpf;
	*itag = 0;
	return 0;

e_inval:
	return -reason;
e_rpf:
	return -SKB_DROP_REASON_IP_RPFILTER;
}
```

---
## 📍 Local routing

Let's look at `ip_route_input_slow()` again.  

When the route type is `LOCAL`, source validation is performed.  
If `accept_local` is disabled at this stage, the packet may be dropped. If validation succeeds, execution jumps to `local_input`.  


```c
// source: net/ipv4/route.c
static enum skb_drop_reason
ip_route_input_slow(struct sk_buff *skb, __be32 daddr, __be32 saddr,
		    dscp_t dscp, struct net_device *dev,
		    struct fib_result *res)
{

	// (fib lookup...)
	
	err = -EINVAL;
	if (res->type == RTN_LOCAL) {
		reason = fib_validate_source_reason(skb, saddr, daddr, dscp,
						    0, dev, in_dev, &itag);
		if (reason)
			goto martian_source;
		goto local_input;
	}

// ...

out:
	return reason;

// ...

local_input:
	if (IN_DEV_ORCONF(in_dev, NOPOLICY))
	IPCB(skb)->flags |= IPSKB_NOPOLICY;

	// If a cache entry exists, there is no need to touch the table
	do_cache &= res->fi && !itag;
	if (do_cache) {
		struct fib_nh_common *nhc = FIB_RES_NHC(*res);

		rth = rcu_dereference(nhc->nhc_rth_input);
		if (rt_cache_valid(rth)) {
			skb_dst_set_noref(skb, &rth->dst);
			reason = SKB_NOT_DROPPED_YET;
			goto out;
		}
	}

	// Store the destination in rtable, the route result object
	// Pass RTCF_LOCAL as a flag
	rth = rt_dst_alloc(ip_rt_get_dev(net, res),
			   flags | RTCF_LOCAL, res->type, false);
	if (!rth)
		goto e_nobufs;

	// This is an ingress route, so using dst as an output path would be a bug
	rth->dst.output= ip_rt_bug;
	
	// ...

	// Mark rtable as a route created by input/ingress route lookup
	rth->rt_is_input = 1;

	// ...
	
	// Also handle the case where UNREACHABLE reaches local_input
	// local_input does not only process local routes;
	// it is also a common area that creates an input rtable and attaches it to skb
	if (res->type == RTN_UNREACHABLE) {
		rth->dst.input= ip_error;
		rth->dst.error= -err;
		rth->rt_flags	&= ~RTCF_LOCAL;
	}
	
	// ..

	// Store the state that this has a route for local delivery
	skb_dst_set(skb, &rth->dst);
	reason = SKB_NOT_DROPPED_YET;
	goto out;

no_route:
	RT_CACHE_STAT_INC(in_no_route);
	res->type = RTN_UNREACHABLE;
	res->fi = NULL;
	res->table = NULL;
	goto local_input;
	
	// ...
}
```

Looking closely at `rt_dst_alloc()`, when the `RTCF_LOCAL` flag is set, it stores `rt->dst.input = ip_local_deliver`.  
`ip_local_deliver()` is the actual handler connected through that function pointer.  
```c
// source: net/ipv4/route.c
struct rtable *rt_dst_alloc(struct net_device *dev,
			    unsigned int flags, u16 type,
			    bool noxfrm)
{
	struct rtable *rt;

	rt = dst_alloc(&ipv4_dst_ops, dev, DST_OBSOLETE_FORCE_CHK,
		       (noxfrm ? DST_NOXFRM : 0));

	if (rt) {
		rt->rt_genid = rt_genid_ipv4(dev_net(dev));
		rt->rt_flags = flags;
		rt->rt_type = type;
		rt->rt_is_input = 0;
		rt->rt_iif = 0;
		rt->rt_pmtu = 0;
		rt->rt_mtu_locked = 0;
		rt->rt_uses_gateway = 0;
		rt->rt_gw_family = 0;
		rt->rt_gw4 = 0;

		rt->dst.output = ip_output;
		if (flags & RTCF_LOCAL)
			rt->dst.input = ip_local_deliver;
	}

	return rt;
}
EXPORT_SYMBOL(rt_dst_alloc);
```

`ip_local_deliver()` passes the packet to the upper protocol layer.  
You can see that it enters the `NF_INET_LOCAL_IN` stage through `NF_HOOK()`.  
After passing the hook, it calls `ip_local_deliver_finish()`.  
```c
// source: net/ipv4/ip_input.c
/*
 * 	Deliver IP Packets to the higher protocol layers.
 */
int ip_local_deliver(struct sk_buff *skb)
{
	/*
	 *	Reassemble IP fragments.
	 */
	struct net *net = dev_net(skb->dev);

	if (ip_is_fragment(ip_hdr(skb))) {
		if (ip_defrag(net, skb, IP_DEFRAG_LOCAL_DELIVER))
			return 0;
	}

	return NF_HOOK(NFPROTO_IPV4, NF_INET_LOCAL_IN,
		       net, NULL, skb, skb->dev, NULL,
		       ip_local_deliver_finish);
}
EXPORT_SYMBOL(ip_local_deliver);
```

The actual point where the packet moves up to L4 is `ip_local_deliver_finish()`.  
```c
// source: net/ipv4/ip_input.c
static int ip_local_deliver_finish(struct net *net, struct sock *sk, struct sk_buff *skb)
{
	if (unlikely(skb_orphan_frags_rx(skb, GFP_ATOMIC))) {
		__IP_INC_STATS(net, IPSTATS_MIB_INDISCARDS);
		kfree_skb_reason(skb, SKB_DROP_REASON_NOMEM);
		return 0;
	}

	skb_clear_delivery_time(skb);
	// Decapsulate so the skb points to the L4 header
	__skb_pull(skb, skb_network_header_len(skb));

	rcu_read_lock();
	// Send to the actual upper protocol layer
	ip_protocol_deliver_rcu(net, skb, ip_hdr(skb)->protocol);
	rcu_read_unlock();

	return 0;
}
```


### RTN_LOCAL vs RT_SCOPE_HOST

In the local table, you can see `scope host`.  

```bash
# source: iproute2/ip/iproute.c
ip route show table local
local 100.85.76.63 dev tailscale0 proto kernel scope host src 100.85.76.63
local 127.0.0.0/8 dev lo proto kernel scope host src 127.0.0.1
local 127.0.0.1 dev lo proto kernel scope host src 127.0.0.1
broadcast 127.255.255.255 dev lo proto kernel scope link src 127.0.0.1
local 172.17.0.1 dev docker0 proto kernel scope host src 172.17.0.1
broadcast 172.17.255.255 dev docker0 proto kernel scope link src 172.17.0.1 linkdown 
local 192.168.0.16 dev wlp3s0 proto kernel scope host src 192.168.0.16
broadcast 192.168.0.255 dev wlp3s0 proto kernel scope link src 192.168.0.16
```

Scope is not a route type. It is closer to the notion of distance to the destination.  
```bash
# source: include/uapi/linux/rtnetlink.h
RT_SCOPE_UNIVERSE = 0
RT_SCOPE_LINK     = 253
RT_SCOPE_HOST     = 254
RT_SCOPE_NOWHERE  = 255
```

### Local in source IP validation

In `fib_validate_source()`, when `accept_local=1` and `rp_filter=0`, most cases are accepted through the fast path.  
```c
// source: net/ipv4/fib_frontend.c
	if (!r && !fib_num_tclassid_users(net) &&
	    (dev->ifindex != oif || !IN_DEV_TX_REDIRECTS(idev))) {
		
		if (IN_DEV_ACCEPT_LOCAL(idev))
			goto ok;
```

In the slow path, `__fib_validate_source()` first drops packets whose route type is `RTN_LOCAL` when `accept_local=0`.  
During the later interface validation, it accepts not only normal dev matches but also the special case where the route type is `RTN_LOCAL` and dev, meaning the ingress interface, is loopback.  
```c
// source: net/ipv4/fib_frontend.c
	// ...
	// RTN_UNICAST -> allow
	// RTN_LOCAL, accept_local=1 -> continue to interface validation below
	// RTN_LOCAL, accept_local=0 -> drop
	// other types -> drop
	if (fib_lookup(net, &fl4, &res, 0))
		goto last_resort;
	if (res.type != RTN_UNICAST) {
		if (res.type != RTN_LOCAL) {
			reason = SKB_DROP_REASON_IP_INVALID_SOURCE;
			goto e_inval;
		} else if (!IN_DEV_ACCEPT_LOCAL(idev)) {
			reason = SKB_DROP_REASON_IP_LOCAL_SOURCE;
			goto e_inval;
		}
	}
	fib_combine_itag(itag, &res);

	// Among the nexthops in fib_info for the route selected by this lookup,
	// is there a nexthop that uses the actual ingress dev?
	dev_match = fib_info_nh_uses_dev(res.fi, dev);
	// Exception: allow loopback because it does not match normally
	dev_match = dev_match || (res.type == RTN_LOCAL &&
				  dev == net->loopback_dev);
	// ...
```


---
## 📚 References

- https://github.com/torvalds/linux
- https://www.kernel.org/doc/html/latest/RCU/whatisRCU.html
- https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/6/html/security_guide/sect-security_guide-server_security-reverse_path_forwarding
- https://www.kernel.org/doc/html/latest/networking/ip-sysctl.html
- https://www.man7.org/linux/man-pages/man7/rtnetlink.7.html
