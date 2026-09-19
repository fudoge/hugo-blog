---
title: "Investigating a Conflict Between Cilium and Node-Level tailscaled"
description: "A kernel tracing and FIB lookup analysis of Cilium L7 TPROXY traffic being dropped with IP_LOCAL_SOURCE after interacting with src_valid_mark enabled by tailscaled"
date: 2026-09-14T15:43:19+09:00
lastmod: 2026-09-14T15:43:19+09:00
slug: cilium-troubleshooting-tproxy-src-valid-mark
image:
math: false
license:
hidden: false
comments: true
draft: false

tags:
    - Kubernetes
    - Cilium
    - Tailscale
    - TPROXY
    - Linux
    - Network
    - Troubleshooting

categories:
    - Cilium
    - Network
---

While rebuilding my homelab, I installed **Cilium** as the CNI for a k3s cluster and **Tailscale** on each node for remote access.  
Both components appeared to work correctly on their own, but using them together caused Cilium L7 traffic to time out.

This post traces the packet path from Cilium's TPROXY redirect to the point where Linux source validation drops the packet with `IP_LOCAL_SOURCE`.  
The analysis showed that `net.ipv4.conf.all.src_valid_mark=1`, enabled by tailscaled, was interacting with Cilium's **fwmark-based TPROXY routing** under this specific set of conditions.

The following posts provide useful background on the path from `ip` commands and Netfilter to policy routing, source validation, and TPROXY:

- [`ip` commands and network namespaces]({{< relref "post/linux-network-ip" >}})
- [Netfilter and nftables]({{< relref "post/linux-network-nft" >}})
- [Routing (1): `ip rule`, fwmark, and policy routing]({{< relref "post/linux-network-route-1" >}})
- [Routing (2): FIB lookup and source validation]({{< relref "post/linux-network-route-2" >}})
- [TPROXY fundamentals]({{< relref "post/linux-network-tproxy-basic" >}})

---
## 🚨 The Problem

The issue was reproduced in the following environment:

- **Cilium:** v1.20.1
- **Ubuntu:** 26.04 LTS
- **Kubernetes:** v1.37.0
- **Cluster topology:** Single node

Running `cilium connectivity test` resulted in **26 failures out of 80 tests**. The failed cases all involved **L7 traffic passing through Cilium's proxy**, and the requests ended with exit code `28`, indicating a timeout.

The following excerpt shows the common failure pattern:

```text
📋 Test Report [cilium-test-1]
 ❌ 26/80 tests failed (70/318 actions), 57 tests skipped:

Test [echo-ingress-l7]:
 🟥 echo-ingress-l7/pod-to-pod-with-endpoints:curl-ipv4-0-public:
 command terminated with exit code 28

Test [client-egress-l7]:
 🟥 client-egress-l7/pod-to-pod:curl-ipv4-1:
 command terminated with exit code 28

Test [client-egress-tls-sni]:
 🟥 client-egress-tls-sni/pod-to-world:https-to-one.one.one.one.-ipv4-0:
 command terminated with exit code 28

[cilium-test-1] 26 tests failed
```

---
## 🔍 Debugging the Packet Path

### Building a minimal reproduction

First, create a test Pod and a `CiliumNetworkPolicy`. The DNS L7 rule intentionally sends the matching traffic through the Cilium proxy.

```bash
# Create the namespace
kubectl create ns cilium-test

# Create the Pod
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: curl
  name: curl
  namespace: cilium-test
spec:
  containers:
    - image: alpine/curl
      name: curl
      command:
        - "sleep"
        - "3600"
  dnsPolicy: ClusterFirst
  restartPolicy: Always
EOF

# Create the CiliumNetworkPolicy
kubectl apply -f - <<EOF
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: test-rule
  namespace: cilium-test
spec:
  endpointSelector:
    matchLabels:
      run: curl
  egress:
    - toEndpoints:
        - matchLabels:
            io.kubernetes.pod.namespace: kube-system
            k8s-app: kube-dns
      toPorts:
        - ports:
            - port: "53"
          rules:
            dns:
              - matchPattern: "*"
EOF
```

Next, enable `src_valid_mark` to reproduce the relevant host state created by tailscaled:

```bash
sysctl -w net.ipv4.conf.all.src_valid_mark=1
```

A DNS lookup from the Pod now **times out**:

```bash
kubectl -n cilium-test exec curl -- nslookup google.com
```

### Confirming the proxy redirect with Cilium monitor

Find the Pod IP and its Cilium endpoint ID, then monitor traffic related to that endpoint:

```bash
# Find the Pod IP
kubectl -n cilium-test get pod curl -o wide

# Find the Pod endpoint
kubectl -n kube-system exec cilium-<hash> -- \
    cilium-dbg endpoint list | grep <POD-IP>

EP="<POD_ENDPOINT>"

kubectl -n kube-system exec -it cilium-<hash> -- \
    cilium-dbg monitor --related-to="$EP" -v -v
```

Run the DNS lookup again in another terminal:

```bash
kubectl -n cilium-test exec curl -- nslookup google.com
```

The monitor output records the policy verdict for endpoint `1885` as `action redirect` and `to-proxy`. This confirms that the DNS L7 rule redirects the packet to the node-local TPROXY path.

```text
Policy verdict log: flow 0x98fc9bfe local EP ID 1885, remote ID 19110,
proto 17, egress, action redirect, auth: disabled, match L3-L4,
10.217.0.124:49595 -> 10.217.0.76:53 udp
-----------------------------------------------------------------------------
CPU 01: MARK 0x98fc9bfe FROM 1885 to-proxy: 70 bytes (70 captured),
state new, identity 16010->unknown, orig-ip 0.0.0.0, to proxy-port 45867
IPv4 {SrcIP=10.217.0.124 DstIP=10.217.0.76 Protocol=UDP}
UDP  {SrcPort=49595 DstPort=53(domain)}
```

### Finding the drop reason with kernel tracing

Enable the `kfree_skb` tracepoint to determine why the kernel drops the packet:

```bash
TRACE=/sys/kernel/tracing

mkdir -p "$TRACE/instances/cilium-srcmark"
T="$TRACE/instances/cilium-srcmark"

echo 0 > "$T/tracing_on"
echo > "$T/trace"

echo 1 > "$T/events/skb/kfree_skb/enable"
echo 1 > "$T/tracing_on"
```

With tracing enabled, run the DNS lookup once more:

```bash
kubectl -n cilium-test exec curl -- nslookup google.com
```

After it times out, stop tracing and search for `IP_LOCAL_SOURCE`:

```bash
echo 0 > "$T/tracing_on"

grep -i 'IP_LOCAL_SOURCE' "$T/trace"
```

The result shows that the kernel dropped the packet with the `IP_LOCAL_SOURCE` reason:

```text
nslookup-113430 [001] ..s1. 164463.161391: kfree_skb:
skbaddr=0000000001fff167 protocol=2048
location=ip_rcv_finish_core+0x233/0x360 reason: IP_LOCAL_SOURCE

nslookup-113430 [001] ..s1. 164463.161397: kfree_skb:
skbaddr=0000000001fff167 protocol=2048
location=ip_rcv_finish_core+0x233/0x360 reason: IP_LOCAL_SOURCE
```

### Tracing source validation in the kernel

The cause becomes clear in the Linux kernel source validation function, `__fib_validate_source()`. If the reverse FIB lookup returns a local route while `accept_local=0`, the kernel drops the packet with `IP_LOCAL_SOURCE`.

```c
static int __fib_validate_source(struct sk_buff *skb, __be32 src, __be32 dst,
                                 dscp_t dscp, int oif, struct net_device *dev,
                                 int rpf, struct in_device *idev, u32 *itag)
{
    struct net *net = dev_net(dev);
    enum skb_drop_reason reason;
    struct fib_result res;
    struct flowi4 fl4;

    // ...

    fl4.daddr = src;
    fl4.saddr = dst;

    // Include the fwmark in the reverse lookup when src_valid_mark=1
    fl4.flowi4_mark = IN_DEV_SRC_VMARK(idev) ? skb->mark : 0;

    // ...

    if (fib_lookup(net, &fl4, &res, 0))
        goto last_resort;
    if (res.type != RTN_UNICAST) {
        if (res.type != RTN_LOCAL) {
            reason = SKB_DROP_REASON_IP_INVALID_SOURCE;
            goto e_inval;

        // Drop a local route with IP_LOCAL_SOURCE when accept_local=0
        } else if (!IN_DEV_ACCEPT_LOCAL(idev)) {
            reason = SKB_DROP_REASON_IP_LOCAL_SOURCE;
            goto e_inval;
        }
    }

    // ...
```

Why did the reverse lookup return a local route? The answer lies in the following policy routing rule and routing table created by Cilium:

```bash
root@cp-1:~# ip rule
9:      from all fwmark 0x200/0xf00 lookup 2004
100:    from all lookup local
32766:  from all lookup main
32767:  from all lookup default

root@cp-1:~# ip route show table 2004
local default dev lo proto kernel scope host
```

When a packet's fwmark matches `0x200/0xf00`, the kernel looks up **table 2004** before the main table. Table 2004 contains a `local default` route that sends every destination to `lo`.

This rule belongs to Cilium's TPROXY routing path. When L7 policy processing is required, Cilium marks the packet and sends it to the node's Envoy proxy.

**With `src_valid_mark=1`, source validation includes the packet mark in its reverse FIB lookup.** The lookup therefore selects Cilium's table 2004 and returns `RTN_LOCAL`. Because `accept_local=0`, the packet is dropped.

### Who enabled `src_valid_mark`?

In this environment, **tailscaled** enabled `src_valid_mark`. Tailscale's Linux router implementation sets this sysctl to `1` so that source validation takes packet fwmarks into account.

```bash
❯ git clone --depth=1 https://github.com/tailscale/tailscale
❯ cd tailscale

❯ rg src_valid_mark
wgengine/router/osrouter/router_linux.go
568:            // Enable src_valid_mark so the kernel uses the packet's fwmark
573:            if err := writeSysctl("net.ipv4.conf.all.src_valid_mark", "1"); err != nil {
574:                    r.logf("warning: failed to enable src_valid_mark: %v", err)
```

---
## 🧪 A/B Test

The results of testing every combination of `src_valid_mark` and `accept_local` are summarized below:

| `src_valid_mark` | `accept_local` | Include `0x200` in reverse FIB | Reverse FIB result | DNS result | Drop reason |
| --- | --- | --- | --- | --- | --- |
| `0` | `0` | No | Normal | Works | None |
| `0` | `1` | No | Normal | Works | None |
| `1` | `0` | Yes | `table 2004 → local dev lo → RTN_LOCAL` | **Timeout** | `IP_LOCAL_SOURCE` |
| `1` | `1` | Yes | `table 2004 → local dev lo → RTN_LOCAL` | Works | None |

DNS fails only when **`src_valid_mark=1` and `accept_local=0`**. This result matches the branch observed in the kernel source.

---
## 🧭 Packet Drop Flow

The complete path can be summarized as follows:

```text
skb->mark = 0x200
        ↓
fib_validate_source()
        ↓
mark participates in reverse lookup
        ↓
rule 9 / table 2004
        ↓
local default
        ↓
RTN_LOCAL
        ↓
accept_local=0
        ↓
IP_LOCAL_SOURCE
```

In this case, the `skb->mark=0x200` set by Cilium for L7 processing participated in the reverse lookup because tailscaled had enabled `src_valid_mark=1`. The lookup selected Cilium's local route, which then combined with the default `accept_local=0` setting and resulted in an `IP_LOCAL_SOURCE` drop.

This behavior may not be limited to Tailscale. Another tool that sets `net.ipv4.conf.all.src_valid_mark=1` on the node could potentially trigger the same interaction with Cilium's routing rules.

---
## 🛠️ Mitigation and Deployment Options

If Tailscale must run directly on a Kubernetes node using Cilium, it is worth checking the interaction between **L7/FQDN policies and `src_valid_mark`** first. Setting `accept_local=1` can remove the immediate symptom, but it changes source validation behavior and should only be applied after reviewing its wider impact.

For access to cluster services, the [Tailscale Kubernetes Operator](https://tailscale.com/docs/kubernetes-operator) is another option. It manages Tailscale connectivity through Kubernetes resources instead of relying on host-level tailscaled networking.

If the main goal is SSH access to nodes, routing the node subnet through a **Subnet Router** on a separate machine is also a possible alternative.

---
## 📣 Reported to Cilium

I reported the behavior as [Cilium Issue #48706](https://github.com/cilium/cilium/issues/48706). The analysis in this post is based on the reproduction results and the observed kernel path, while further upstream confirmation is still pending.

I will update this post if the issue receives additional findings or an upstream change.

---
## 📚 References

- [Linux kernel source](https://github.com/torvalds/linux)
- [Cilium source](https://github.com/cilium/cilium)
- [Cilium command cheatsheet](https://github.com/cilium/cilium/blob/main/Documentation/cheatsheet.rst)
- [Linux kernel — ftrace](https://docs.kernel.org/trace/ftrace.html)
- [Linux kernel — Event Tracing](https://docs.kernel.org/trace/events.html)
- [Tailscale Kubernetes Operator](https://tailscale.com/docs/kubernetes-operator)
