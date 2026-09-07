---
title: "Linux Network - Routing(2)"
description: "Linux 커널의 IPv4 ingress routing decision, FIB lookup, source validation, local delivery 흐름을 소스 코드로 따라가보자"
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

이번에는 리눅스 커널에서의 routing 및 패킷 처리 동작을 알아볼 것이다.  
리눅스 커널의 소스 코드를 읽는 것이 매우 어렵지만, 내부 성능 최적화 및 캐싱, 에러 핸들링 등을 제외하고, 핵심 흐름 위주로 살펴보자.  

---
## 🧭 Ingress부터 Forward까지

우선, Linux 커널 코드에서 IPv4 패킷이 들어왔을 때 어떻게 처리되는지 알아보자.  

`ip_rcv()`는 L2에서 올라온 socket buffer(`skb`)를 받아 처리한다.  
핵심 IPv4 패킷 검증은 `ip_rcv_core()`에서 한다.  

```c
// source: net/ipv4/ip_input.c
/*
 * IP receive entry point
 */
int ip_rcv(struct sk_buff *skb, struct net_device *dev, struct packet_type *pt,
	   struct net_device *orig_dev)
{
	// network namespace 정보를 가져옴
	// reference:
	// - linux/include/linux/netdevice.h
	// - linux/include/net/net_namespace.h
	struct net *net = dev_net(dev);

	// 기본적인 IPv4패킷 검증
	skb = ip_rcv_core(skb, net);
	// 문제가 있다면 drop
	if (skb == NULL)
		return NET_RX_DROP;

	// Netfilter: PREROUTING 단계.
	// 등록된 hook이 없거나 hook 평가 결과 계속 진행 가능한 경우,
	// NF_HOOK() wrapper가 okfn()으로 ip_rcv_finish()를 호출
	return NF_HOOK(NFPROTO_IPV4, NF_INET_PRE_ROUTING,
		       net, NULL, skb, dev, NULL,
		       ip_rcv_finish);
}
```

`ip_rcv_core`에서는 IPv4 Datagram을 검증한다.  

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

	// L4(transport) header 시작 위치 설정
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

`ip_rcv_core()`에서 검증이 끝나면 `NF_HOOK()`을 `PREROUTING` hook으로 실행한다.  
등록된 hook이 없거나 hook 평가 결과 계속 진행 가능한 경우, `NF_HOOK()` wrapper는 `okfn()`으로 전달된 `ip_rcv_finish()`를 호출한다.  
즉, `ip_rcv()`에서는 `okfn` 자리에 `ip_rcv_finish()`가 들어간다.  
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

`ip_rcv_finish()`에서는 `ip_rcv_finish_core()`를 호출한다.  

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


`ip_rcv_finish_core()`의 핵심 작업은 routing decision을 시작하는 것이다.  
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

`ip_route_input_noref()`에서는 `ip_route_input_rcu()`를 실행한다.  
> **NOTE:**
> RCU: Read-Copy-Update로, Linux Kernel에서의 read-mostly situations에서의 동기화 매커니즘이다.
> reader가 기존 객체를 안전하게 읽을 수 있도록 유지하고, grace period가 지난 후 이전 객체를 reclaim할 수 있게 하는 read-mostly 동기화를 진행한다. 새 객체를 publish할 때는 포인터 교체 패턴이 흔히 사용된다.

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

`ip_route_input_rcu`에서는 일반적인 non-multicast인 경로에서 `ip_route_input_slow()`를 호출한다.  
```c
// source: net/ipv4/route.c
/* called with rcu_read_lock held */
static enum skb_drop_reason
ip_route_input_rcu(struct sk_buff *skb, __be32 daddr, __be32 saddr,
		   dscp_t dscp, struct net_device *dev,
		   struct fib_result *res)
{

	// multicast인경우 핸들링...

	return ip_route_input_slow(skb, daddr, saddr, dscp, dev, res);
}
```

`ip_route_input_slow()`에서는 라우팅의 핵심 동작이 진행된다.  

여기에서 `fib_lookup()`을 실행하는 것을 볼 수 있다.  
커널 파라미터에서 forwarding을 활성화했는지도 확인하고, 라우팅 결과의 type에 따라 local delivery 또는 forwarding 경로로 나뉜다.  

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

	// FIB lookup으로 목적지 조회
	err = fib_lookup(net, &fl4, res, 0);
	if (err != 0) {
		if (!IN_DEV_FORWARD(in_dev))
			err = -EHOSTUNREACH;
		goto no_route;
	}
	
	// ...

	err = -EINVAL;
	// LOCAL Route의 경우, source를 검증하고, local delivery용 route 구성 경로로 이동
	if (res->type == RTN_LOCAL) {
		reason = fib_validate_source_reason(skb, saddr, daddr, dscp,
						    0, dev, in_dev, &itag);
		if (reason)
			goto martian_source;
		goto local_input;
	}

	// device의 Forwarding이 켜져있는지 확인
	if (!IN_DEV_FORWARD(in_dev)) {
		err = -EHOSTUNREACH;
		goto no_route;
	}
	
	// Forwarding 가능한 RTN_UNICAST가 아니면, invalid destination으로 처리
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

`ip_mkroute_input()`는 라우팅 캐시 엔트리를 생성하는 `__mkroute_input()`를 실행한다.  
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

`__mkroute_input()`에서는 `ip_forward`의 함수 포인터 주소를 `rth->dst.input`에 연결하고, nexthop 정보를 저장한 뒤 skb의 dst에 lookup 결과의 dst를 저장한다.  
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
	// nexthop 정보에서 가져온 출력 device
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

	// 이 route를 타는 패킷은 ip_forward()로 보낼 것
	rth->dst.input = ip_forward;

	// next-hop 정보 설정
	rt_set_nexthop(rth, daddr, res, fnhe, res->fi, res->type, itag,
		       do_cache);
			   
	lwtunnel_set_redirect(&rth->dst);
	// skb의 dst에 lookup 결과의 dst를 저장
	skb_dst_set(skb, &rth->dst);
out:
	reason = SKB_NOT_DROPPED_YET;
cleanup:
	return reason;
}
```

`ip_forward()`는 IPv4 forwarding의 동작을 구현한다.  
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

	// Forwarding경로의 MTU확인. 필요한 경우 ICMP Fragmentation Needed 전송 후 drop
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

### 요약

즉, 전체 흐름의 요약은 다음과 같다:  
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
flowi4 구성
  ↓
fib_lookup() 
  ↓
res->type 확인
  ├─ RTN_LOCAL
  │    → local_input
  │    → local delivery용 dst/rtable 구성
  │
  └─ RTN_UNICAST + forwarding enabled
       ↓
     ip_mkroute_input()
       ↓
     __mkroute_input()
       ↓
     output device / source validation
       ↓
     rth 생성
       ↓
     rth->dst.input = ip_forward
       ↓
     rt_set_nexthop()
       ↓
     skb_dst_set(skb, &rth->dst)
```

이후 `ip_rcv_finish()` 쪽을 다시 보면 아래 코드가 있다.  
`ip_rcv_finish_core()`가 `DROP`을 반환하지 않으면 `dst_input(skb)`를 실행한다.  
이는 `skb->dst->input(skb)`을 호출하고, 앞에서 `rth->dst.input = ip_forward`로 저장해둔 결과에 따라 결국 `ip_forward(skb)`가 실행된다.  
```c
// source: net/ipv4/ip_input.c
	// ...
	ret = ip_rcv_finish_core(net, skb, dev, NULL);
	if (ret != NET_RX_DROP)
		ret = dst_input(skb);
	return ret;
```

---
## 🧩 FIB rules와 fwmark

이제 mark를 중심으로 policy routing에 관련된 소스 코드를 확인해보자.  

`fib_lookup()`을 보기 전에 `ip rule`의 기본 구조를 다시 보면 다음과 같다.  
```bash
# source: iproute2/ip/iprule.c
$ ip rule 
0: from all lookup local 
32766: from all lookup main 
32767: from all lookup default
```

아래 코드에서도 비슷한 구조가 보인다.  
`ip rule`에서 `main`과 `default` table이 낮은 우선순위에 있었는데, 커널 코드의 fallback 순서도 그 구조와 맞닿아 있다.  
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

대신 `local` table은 이 코드에서 직접 보이지 않는다.  
custom rule이 없는 경우에는 `local` table이 `main` table을 alias로 둔 trie 구조로 다뤄지고, custom rule이 생기면 `fib_unmerge()`를 통해 분리된다.  
이 부분은 당장 자세히 다루지는 않지만, `net/ipv4/fib_frontend.c`의 `fib_new_table()`과 `fib_unmerge()` 함수에서 그 흔적을 볼 수 있다.  

`__fib_lookup()`에서는 `fib_rules_lookup()`을 호출한다.  
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

`fib_rules_lookup()`에서는 각 rule을 순회하면서 `fib_rule_match()`로 매칭 여부를 평가한다.  
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

`fib_rule_match()`에서는 mark를 포함한 여러 조건들과 함께 rule을 매칭하는 것을 볼 수 있다:  
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
	// rule->mark:      ip rule의 fwmark값
	// rule->mark_mask: ip rule의 fwmark뒤의 mask값
	// fl->flowi_mark:  route lookup에 사용할 mark
	// 즉, 아래 조건문은 rule의 fwmark와 flowi_mark의 bitwise xor이므로,
	// 두 diff를 구한 뒤, mask로 원하는 비트 부분만 남긴 뒤, 해당 부분이 0이면 매칭에 성공하고 다음 if문으로 넘어가는 방식이다.
	// fwmark를 마스킹하는 이유는, 특정 비트구간만을 보기 위해서라고 보면 된다.
	// mark를 rule 매칭말고도 다른 활용에 쓰기 위해 다른 비트 부분을 썼을 수도 있다.
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

앞의 소스들에서 `fib_validate_source()`를 몇 번 보았다.  
source IP가 유효한지 검사하는 로직이다.  

일부 케이스에서는 빠르게 판단하려는 fast path가 있고, custom `ip rule`이 있는 경우에는 full check를 실행한다.  
또 forwarding 중에 source IP가 자기 자신이면 이상한 패킷으로 보고 drop하는 것을 볼 수 있다.  

```c
// source: net/ipv4/fib_frontend.c
int fib_validate_source(struct sk_buff *skb, __be32 src, __be32 dst,
			dscp_t dscp, int oif, struct net_device *dev,
			struct in_device *idev, u32 *itag)
{
	// RPF: Reverse Path Filtering
	//     역방향 lookup과 ingress의 인터페이스가 같은지 조회하고, 이에 따른 필터링
	//     0: No source validation
	//     1: Strict mode as defined in RFC 3704.
	//     2: Loose mode as defined in RFC 3704.
	// secpath_exists: IPSec등의 보호된 패킷인지 조회 후, 보호된 패킷이면 0으로 설정.
	//     IPSec 등의 패킷에 RPF를 적용하는 것은 부적절하기 때문
	// 보호된 패킷이 아니면, interface의 sysctl rp_filter값(net.ipv4.<iface>.rp_filter)
	int r = secpath_exists(skb) ? 0 : IN_DEV_RPFILTER(idev);
	struct net *net = dev_net(dev);

	if (!r && !fib_num_tclassid_users(net) &&
	    (dev->ifindex != oif || !IN_DEV_TX_REDIRECTS(idev))) {
		
		// Local Route가 허용이면 빠르게 성공
		// `accept_local` sysctl에 대응됨
		if (IN_DEV_ACCEPT_LOCAL(idev))
			goto ok;
		/* with custom local routes in place, checking local addresses
		 * only will be too optimistic, with custom rules, checking
		 * local addresses only can be too strict, e.g. due to vrf
		 */
		// custom local route 또는 custom rule이 있으면, full_check로 검증
		if (net->ipv4.fib_has_custom_local_routes ||
		    fib4_has_custom_rules(net))
			goto full_check;
		
		/* Within the same container, it is regarded as a martian source,
		 * and the same host but different containers are not.
		 */
		
		// source IP가 자신이 자신 Address인 것은 이상한 상황이므로 drop
		// NOTE: 지금 상황은 OUTPUT이 아니라 FORWARD이므로
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

full check 로직인 `__fib_validate_source()`에서는 역방향 FIB lookup을 실행한다.  
두 번의 lookup이 진행되는데, 하나는 best path를 찾기 위한 것이고, 그 다음 하나는 ingress interface와 비교하기 위한 부가 정보를 확인하는 과정이다.  

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

	// 역방향 flow를 구성함
	// - 인터페이스를 반대로
	// 역방향에서의 oif는 아직 안정해졌으므로 0
	// 역방향에서의 iff는 oif를 뒤집으면 되는데, 이전에 FIB lookup을 한적이 없어서 flow의 oif를 모르는 경우 등에서는 Loopback Interface로 대체
	fl4.flowi4_oif = 0;
	fl4.flowi4_l3mdev = l3mdev_master_ifindex_rcu(dev);
	fl4.flowi4_iif = oif ? : LOOPBACK_IFINDEX;
	// - IP주소도 반대로
	fl4.daddr = src;
	fl4.saddr = dst;
	
	// ...

	// mark값도 로딩
	// sysctl의 src_valid_mark(net.ipv4.conf.<iface>.src_valid_mark)가 필요
	fl4.flowi4_mark = IN_DEV_SRC_VMARK(idev) ? skb->mark : 0;
	// L4정보도 뒤집기
	if (!fib4_rules_early_flow_dissect(net, skb, &fl4, &flkeys)) {
		fl4.flowi4_proto = 0;
		fl4.fl4_sport = 0;
		fl4.fl4_dport = 0;
	} else {
		swap(fl4.fl4_sport, fl4.fl4_dport);
	}

	// 역방향 lookup 호출
	if (fib_lookup(net, &fl4, &res, 0))
		goto last_resort;
	if (res.type != RTN_UNICAST) {
		if (res.type != RTN_LOCAL) {
			reason = SKB_DROP_REASON_IP_INVALID_SOURCE;
			goto e_inval;
		// route의 type이 LOCAL인데 accept_local(net.ipv4.conf.<iface>.accept_local)이 0이면 drop
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
	
	// dev_match를 통과하면 즉시 허용
	if (dev_match) {
		ret = FIB_RES_NHC(res)->nhc_scope >= RT_SCOPE_HOST;
		return ret;
	}
	if (no_addr)
		goto last_resort;
	// rp_filter의 값이 1(strict)이면 drop
	if (rpf == 1)
		goto e_rpf;
	
	// 이제는 oif를 ingress로 정함
	fl4.flowi4_oif = dev->ifindex;

	// 2차 lookup
	// 이번에는 oif를 ingress로 정한 상태에서
	// valid path가 있는지 검사
	// source가 ingress interface 기준으로 reachable한지 확인
	// 결과가 0보다 크면 direct/link reachable한 source로 판단할 수 있음을 의미
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

다시 `ip_route_input_slow()`를 보자.  

type이 `LOCAL`인 경우 source validation을 수행한다.  
이 단계에서 `accept_local`이 꺼져 있다면 drop될 수 있고, 검증을 통과하면 `goto local_input`으로 흐름을 넘긴다.  


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

	// cache가 있으면 테이블 작업을 할 필요 없음
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

	// rtable(route결과 객체)에 destination을 저장
	// RTCF_LOCAL을 플래그로 넘김
	rth = rt_dst_alloc(ip_rt_get_dev(net, res),
			   flags | RTCF_LOCAL, res->type, false);
	if (!rth)
		goto e_nobufs;

	// ingress용 route이기에, dst를 output path로 쓰면 버그라는 의미
	rth->dst.output= ip_rt_bug;
	
	// ...

	// rtable에 input/ingress route lookup으로 만들어진 route임을 마킹
	rth->rt_is_input = 1;

	// ...
	
	// UNREACHABLE -> local_input으로 온 경우에도 핸들링
	// 즉, local_input은 단순히 local route를 처리할 뿐만 아니라, 
	// input용 rtable을 생성하여 skb에 붙이는 공통 영역임
	if (res->type == RTN_UNREACHABLE) {
		rth->dst.input= ip_error;
		rth->dst.error= -err;
		rth->rt_flags	&= ~RTCF_LOCAL;
	}
	
	// ..

	// local-delivery용 route를 가진다는 상태로 저장됨
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

`rt_dst_alloc()`을 자세히 보면 `RTCF_LOCAL` 플래그가 켜진 경우 `rt->dst.input = ip_local_deliver`를 저장한다.  
`ip_local_deliver()` 역시 함수 포인터로 연결되는 실제 처리 함수이다.  
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

`ip_local_deliver()`는 상위 프로토콜 레이어로 전달한다.  
`NF_HOOK()`으로 `NF_INET_LOCAL_IN` 단계에 들어가는 것을 볼 수 있다.  
hook을 통과하면 `ip_local_deliver_finish()`를 호출한다.  
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

실제 L4로 넘어가는 지점은 `ip_local_deliver_finish()`이다.  
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
	// 역캡슐화(L4 header를 가리키도록)
	__skb_pull(skb, skb_network_header_len(skb));

	rcu_read_lock();
	// 실제 상위 protocol layer로 보냄
	ip_protocol_deliver_rcu(net, skb, ip_hdr(skb)->protocol);
	rcu_read_unlock();

	return 0;
}
```


### RTN_LOCAL vs RT_SCOPE_HOST

local table을 보면 `scope host`라는 값을 볼 수 있다.  

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

scope는 route의 type이 아니라 destination까지의 거리와 비슷한 개념이라고 볼 수 있다.  
```bash
# source: include/uapi/linux/rtnetlink.h
RT_SCOPE_UNIVERSE = 0
RT_SCOPE_LINK     = 253
RT_SCOPE_HOST     = 254
RT_SCOPE_NOWHERE  = 255
```

### source IP validation에서의 local

`fib_validate_source()`에서 `accept_local=1`, `rp_filter=0`인 경우, 대부분의 경우 fast path에서 ok가 된다.  
```c
// source: net/ipv4/fib_frontend.c
	if (!r && !fib_num_tclassid_users(net) &&
	    (dev->ifindex != oif || !IN_DEV_TX_REDIRECTS(idev))) {
		
		if (IN_DEV_ACCEPT_LOCAL(idev))
			goto ok;
```

slow path인 `__fib_validate_source()`에서는 우선 `RTN_LOCAL`이지만 `accept_local=0`이면 drop한다.  
이후 인터페이스 검증에서는 정상적인 dev match뿐 아니라, `RTN_LOCAL` 타입이면서 dev, 즉 ingress interface가 loopback인 경우도 예외적으로 허용한다.  
```c
// source: net/ipv4/fib_frontend.c
	// ...
	// RTN_UNICAST -> 허용
	// RTN_LOCAL, accept_local=1 -> 아래 interface검증
	// RTN_LOCAL, accept_local=0 -> drop
	// 나머지 type -> drop
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

	// 이번 lookup으로 선택된 route의 fib_info안에 있는 nexthop들 중에서, 
	// 실제 ingress dev를 사용하는 nexthop이 있나?
	dev_match = fib_info_nh_uses_dev(res.fi, dev);
	// 예외 처리: loopback은 매칭에 실패하므로 허용
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
