---
title: "Linux Network - Routing(1)"
description: "A hands-on overview of ip route, ip rule, routing tables, policy routing, and fwmark-based routing"
date: 2026-09-04T11:48:02+09:00
image: pbr-topology.webp
math: true
license:
hidden: false
comments: true
draft: false

tags:
    - Linux
    - Network
    - Routing
    - iproute2
    - nftables

categories:
    - Network
---

**Routing in Linux is not just about setting a single default gateway.**  
A routing table can contain several types of route entries, and with `ip rule`, Linux can choose routes based not only on the destination address but also on **packet metadata such as source address, incoming interface, and fwmark**.  

In this post, we will look at the basic behavior of `ip route` and `ip rule`, then build both **ordinary routing and fwmark-based policy routing** labs with network namespaces.  

---
## 🧭 Reading the `ip route` man page

### Route type

Route entries handled by `ip route` can have several types.  
In normal setups, you mostly see `unicast` routes, but Linux can also install routes that **intentionally drop packets or reject them for policy reasons**.  

- **unicast**
    - A normal route entry used to forward packets toward a destination.
    - This includes routes through a gateway as well as routes for destinations directly reachable on the same link.
- **unreachable**
    - Explicitly states that the destination is unreachable.
    - The packet is discarded, and an ICMP host unreachable message is returned.
    - Userspace receives an `EHOSTUNREACH` error.
- **blackhole**
    - Silently drops the packet.
    - No ICMP message is sent.
    - Userspace receives an `EINVAL` error.
- **prohibit**
    - States that the destination is unreachable.
    - Compared with `unreachable`, the stronger meaning is that the destination must not be reached due to policy.
    - Userspace receives an `EACCES` error.
- **local**
    - Means the destination is the Linux host itself.
    - The packet is not forwarded externally and is delivered to the local networking stack.
- **broadcast**
    - Means the destination is a broadcast address.
- **throw**
    - A special route type commonly used with policy routing.
    - When this route is selected, lookup in the current table terminates as if no route had been found.
    - Without policy routing, it behaves like a missing route in the routing table: the packet is dropped and an ICMP net unreachable message is generated.
    - The local sender receives an `ENETUNREACH` error.
- **nat**
    - No longer supported since Linux 2.6.
    - In the past, a route itself could carry address translation semantics.
- **anycast**
    - Not implemented at the moment.
    - Represents an anycast destination address.
    - Incoming traffic to the destination behaves like `local`, but using the address as a source address is not allowed.
- **multicast**
    - A special type used for multicast routing.
    - It usually does not appear in ordinary routing tables.

### Route table

Linux groups routes into **routing tables**.  
Tables are identified by numbers from $1$ to $2^{32}-1$, and names can be assigned in `/usr/lib/iproute2/rt_tables` or `/etc/iproute2/rt_tables`.  
If the same table name or ID exists in both files, `/etc/iproute2/rt_tables` has higher priority.  

Ordinary routes are added to the **`main(id: 254)` table** by default.  
The kernel also uses this table for normal routing decisions.  
Table IDs `0`, `253`, `254`, and `255` are reserved for built-in tables.  
When policy routing is used, multiple routing tables can be used together for different purposes.  

There is another important table that is not usually visible in day-to-day configuration.  
It is the **`local(id: 255)` table**.  
This table contains routes for local addresses and broadcast addresses.  
The kernel maintains it automatically, so administrators usually do not need to edit it directly.  

### Managing routes with `ip route`

All of the following commands show route entries in the `main` table.  

```bash
ip r
ip route
ip route show
ip route show table main
```

You can also inspect the `local` table.  
As described above, it contains local and broadcast routes.  

```bash
ip route show table local
```

To show route entries from all tables, use `table all`.  

```bash
ip route show table all
```

Now add `blackhole`, `unreachable`, and `prohibit` routes manually.  

```bash
ip route add blackhole 10.0.1.0/24
ip route add unreachable 10.0.2.0/24
ip route add prohibit 10.0.3.0/24
```

If you send `ping` to each destination, the error observed from userspace differs by route type.  

```bash
# Blackhole
root@ubuntu:~# ping 10.0.1.1
ping: connect: Invalid argument # EINVAL

# Unreachable
root@ubuntu:~# ping 10.0.2.1
ping: connect: No route to host # EHOSTUNREACH

# Prohibit
root@ubuntu:~# ping 10.0.3.1
ping: Do you want to ping broadcast? Then -b. If not, check your local firewall rules

root@ubuntu:~# ping -b 10.0.3.1
WARNING: pinging broadcast address
ping: connect: Permission denied # EACCES
```

For `prohibit`, `ping` may initially print a message related to pinging a broadcast address.  
With the `-b` option, you can confirm the final `Permission denied` result, which corresponds to `EACCES`.  

Next, check how routing is evaluated for a specific destination address.  
For a packet going to `8.8.8.8`, `ip route get` shows which route, interface, and source IP would actually be used.  

```bash
# Which route/interface/source IP will be used for a packet to 8.8.8.8?
root@ubuntu:~# ip route get 8.8.8.8
8.8.8.8 via 192.168.10.1 dev eth0 src 192.168.10.10 uid 0
    cache

# Which route will be used if the source IP is explicitly set to 192.168.10.10?
# This is useful when checking policy routing behavior.
root@ubuntu:~# ip route get 8.8.8.8 from 192.168.10.10
8.8.8.8 from 192.168.10.10 via 192.168.10.1 dev eth0 uid 0
    cache

# Show the actual route entry matched from the FIB perspective.
root@ubuntu:~# ip route get fibmatch 8.8.8.8
default via 192.168.10.1 dev eth0 proto static
```

**FIB stands for Forwarding Information Base.**  
It is the internal routing data structure the kernel queries when deciding where to send a packet.  
Linux uses an LC-trie(Level-Compressed trie) family structure to find the **longest prefix match**.  

You can clean up the route entries added above with the following commands.  

```bash
ip route del blackhole 10.0.1.0/24
ip route del unreachable 10.0.2.0/24
ip route del prohibit 10.0.3.0/24
```

---
## 🧪 Hands-on: Routing between different networks

Let's use network namespaces to create a router between two separate networks and make both sides reachable from each other.  

![](routing.webp)

The topology above can be built with the following commands.  

```bash
# Add namespaces
ip netns add n1
ip netns add router
ip netns add n2

# Create two veth pairs
ip link add veth-n1 type veth peer name veth-r1
ip link add veth-n2 type veth peer name veth-r2

# Move each interface into the target namespace.
# n1 and n2 get one interface each, while router gets the opposite ends.
ip link set veth-n1 netns n1
ip link set veth-r1 netns router
ip link set veth-r2 netns router
ip link set veth-n2 netns n2

# Assign IP addresses to each network interface
ip netns exec n1 ip addr add 10.0.1.2/24 dev veth-n1
ip netns exec router ip addr add 10.0.1.1/24 dev veth-r1
ip netns exec router ip addr add 10.0.2.1/24 dev veth-r2
ip netns exec n2 ip addr add 10.0.2.2/24 dev veth-n2

# Bring up loopback in each namespace
ip netns exec n1 ip link set lo up
ip netns exec router ip link set lo up
ip netns exec n2 ip link set lo up

# Bring up the veth interfaces
ip netns exec n1 ip link set veth-n1 up
ip netns exec router ip link set veth-r1 up
ip netns exec router ip link set veth-r2 up
ip netns exec n2 ip link set veth-n2 up

# Enable IPv4 forwarding in the router namespace
ip netns exec router sysctl -w net.ipv4.ip_forward=1

# Add routes to reach the opposite network
ip netns exec n1 ip route add 10.0.2.0/24 via 10.0.1.1
ip netns exec n2 ip route add 10.0.1.0/24 via 10.0.2.1
```

Now send pings from `n1` to `n2`, and from `n2` to `n1`.  

```bash
# n1 -> n2
ip netns exec n1 ping 10.0.2.2

# n2 -> n1
ip netns exec n2 ping 10.0.1.2
```

When the lab is done, delete the namespaces.  

```bash
ip netns del n1
ip netns del router
ip netns del n2
```

---
## 🚦 Routing policy

In some cases, the destination address alone is not enough to choose the right route.  
You may want to route based on other packet fields such as source address, incoming interface, TOS, or fwmark.  
This is called policy routing.  

Linux uses the **RPDB(Routing Policy Database)** for policy routing.  
The RPDB consists of multiple rules, and each rule is composed of a selector and an action.  

In the RPDB, **a lower priority value means a higher priority**.  
Each rule's selector compares fields such as source address, destination address, incoming interface, TOS, and fwmark against the packet.  
If the selector matches the packet, the action of that rule is executed.  

The important point is that **RPDB lookup does not always stop just because an action was executed**.  
Lookup stops only when the action returns a **final result**.  
The final result may be a normal route, or it may be a failure result such as `unreachable`, `prohibit`, or `blackhole`.  
If no route is found in a table, lookup can continue to the next rule.  

At startup, the kernel creates the default RPDB with the following three rules.  

1. Priority: `0`, Selector: match anything, Action: lookup `local(id: 255)` table
    - A special table for local and broadcast addresses.
2. Priority: `32766`, Selector: match anything, Action: lookup `main(id: 254)` table
    - The ordinary routing table.
    - Administrators can add, remove, and modify route entries in this table.
3. Priority: `32767`, Selector: match anything, Action: lookup `default(id: 253)` table
    - Empty by default.
    - Reserved as a final fallback table.
    - It can be removed, and it is not commonly used today.

The action of an `ip rule` is not limited to `lookup table`.  
Just like route types, it can also perform actions such as `blackhole`, `unreachable`, or `prohibit`.  

### Creating a rule manually

Let's create a priority 100 rule that looks up table `id = 100` when `source ip = 10.0.1.0/24`.  
Then add a default route through `192.168.10.1` to table 100.  

```bash
ip rule add priority 100 from 10.0.1.0/24 lookup 100
ip route add table 100 default via 192.168.10.1
```

Check the new rule and table.  

```bash
ip rule
ip route show table 100
```

Now run `route get` with a source address that matches the selector.  

```bash
# LOCAL OUTPUT
# If this host sends a packet to 8.8.8.8 with source IP 10.0.1.2,
# which route should be used?
# This can fail if 10.0.1.2 is not a local address on the host.
ip route get 8.8.8.8 from 10.0.1.2

# FORWARDING
# If a packet with source IP 10.0.1.2 arrives on eth0,
# which route should be used to forward it to 8.8.8.8?
ip route get 8.8.8.8 from 10.0.1.2 iif eth0
```

For the forwarding-style `route get` query to work properly, IPv4 forwarding must be enabled.  

```bash
sysctl -w net.ipv4.ip_forward=1
```

Now add the following rule.  

```bash
ip rule add priority 200 from 10.0.1.0/25 lookup 200
```

The new rule has a longer prefix than the existing `10.0.1.0/24` rule.  
However, because its priority number is larger, the priority 100 rule is evaluated first.  
In other words, **prefix length matters when selecting a route inside the same table**, while **the lookup order between tables is determined by `ip rule` priority**.  

You can clean up the rule and table configuration with the following commands.  

```bash
ip rule del priority 200
ip rule del priority 100
ip route flush table 100
```

---
## 🏷️ Packet metadata: mark

Packets can carry **metadata that is used for routing decisions**.  
With this metadata, policy routing can be configured more flexibly.  

The mark discussed here is not the 8-bit QoS marking value stored in the IPv4 DS field or IPv6 Traffic Class.  
It is the **`mark` field inside the Linux kernel's `struct sk_buff`**.  
In other words, **this value is not carried in the packet when it leaves the host.**  

### Packet mark

The following is an example of setting a packet mark with nftables.  

```bash
# Set the packet's skb mark to 123 at the output hook.
nft add table ip mangle
nft 'add chain ip mangle output { type route hook output priority mangle; policy accept; }'
nft add rule ip mangle output meta mark set 123
```

### Packet mark and conntrack mark

A mark can be set on a single packet, but it can also be stored in a conntrack entry.  
With conntrack marks, **the mark can be maintained at the flow level rather than per packet**, and the same decision can be reused in both directions of a connection.  

```bash
# Set skb->mark = 1 for packets passing through the forward chain.
nft add rule filter forward meta mark set 1

# Store the packet mark into the conntrack mark.
nft add rule filter forward ct mark set meta mark
```

---
## 🧬 Hands-on: Policy routing with fwmark

Now let's build the following network topology and practice fwmark-based policy routing.  

![](pbr-topology.webp)

In this lab, the client sends requests to the same server IP, **`203.0.113.10`**.  
`203.0.113.0/24` is the **TEST-NET-3 prefix** reserved by RFC 5737 for documentation and examples.  
It does not represent a real Internet host here; it is an example address that is safe to use in labs and technical documents.  
The default route points to WAN1, while **only traffic with TCP destination port `8443` receives fwmark `0x2` and is routed through WAN2**.  

The final state we want to confirm is as follows.  

- **Requests to `203.0.113.10:8080`** have no mark and follow the main table toward WAN1.
- **Requests to `203.0.113.10:8443`** get fwmark `0x2` from nftables, then look up table 100 through `ip rule` and go out through WAN2.
- In conntrack output, **the `dport=8080` flow appears with `mark=0`**, while **the `dport=8443` flow appears with `mark=2`**.

First, create the namespaces and veth pairs.  

```bash
# Create namespaces
ip netns add c
ip netns add r
ip netns add wan1
ip netns add wan2

ip -n c link set lo up
ip -n r link set lo up
ip -n wan1 link set lo up
ip -n wan2 link set lo up

# Client <-> Router
ip link add c0 type veth peer name r-c
ip link set c0 netns c
ip link set r-c netns r

ip -n c addr add 10.0.0.2/24 dev c0
ip -n r addr add 10.0.0.1/24 dev r-c

ip -n c link set c0 up
ip -n r link set r-c up

# Router <-> WAN1
ip link add w1-r type veth peer name r-w1

ip link set w1-r netns wan1
ip link set r-w1 netns r

ip -n wan1 addr add 10.1.0.2/30 dev w1-r
ip -n r addr add 10.1.0.1/30 dev r-w1

ip -n wan1 link set w1-r up
ip -n r link set r-w1 up

# Router <-> WAN2
ip link add w2-r type veth peer name r-w2

ip link set w2-r netns wan2
ip link set r-w2 netns r

ip -n wan2 addr add 10.2.0.2/30 dev w2-r
ip -n r addr add 10.2.0.1/30 dev r-w2

ip -n wan2 link set w2-r up
ip -n r link set r-w2 up

# Assign the same server IP to WAN1 and WAN2
ip -n wan1 addr add 203.0.113.10/32 dev lo
ip -n wan2 addr add 203.0.113.10/32 dev lo

# The client sends all traffic to the router.
ip -n c route add default via 10.0.0.1

# Return routes from the WAN namespaces back to the client network
ip -n wan1 route add 10.0.0.0/24 via 10.1.0.1
ip -n wan2 route add 10.0.0.0/24 via 10.2.0.1

# Put the WAN1 path in the default main table.
ip -n r route add 203.0.113.10/32 \
    via 10.1.0.2 dev r-w1

# Put the WAN2 path in table 100.
ip -n r route add 203.0.113.10/32 \
    via 10.2.0.2 dev r-w2 \
    table 100
ip -n r route add 10.0.0.0/24 \
    dev r-c \
    table 100

# Packets with fwmark 0x2 look up table 100.
ip -n r rule add priority 100 \
    fwmark 0x2 table 100

# Enable IPv4 forwarding in the router namespace.
ip netns exec r \
    sysctl -w net.ipv4.ip_forward=1
```

Now run `ip route get` inside the router namespace `r`.  

```bash
# An unmarked packet uses the WAN1 path from the main table.
ip netns exec r \
    ip route get 203.0.113.10

# A packet marked with 0x2 uses the WAN2 path from table 100.
ip netns exec r \
    ip route get 203.0.113.10 mark 0x2
```

Next, use nftables to mark TCP `8443` traffic.  
The key idea is to **inspect the first packet, set a packet mark for routing, and save that value into the conntrack mark** so later packets in the same connection keep the same routing decision.  

**`meta mark` is the packet mark**, or the current packet's `skb->mark` value.  
On the other hand, **`ct mark` is the mark stored in the conntrack entry**.  
A TCP connection consists of many packets, so if only the first SYN packet matches the condition and later packets lose the mark, **routing can become inconsistent.**  
To avoid that, we **store the value in `ct mark`** and restore it back to **`meta mark`** when packets from the same connection arrive later.  

The `priority mangle` part of the base chain is also important.  
In nftables, multiple base chains can attach to the same hook, and priority decides the order in which those chains run.  
Here, the packet mark must be set **before the routing decision** so that `ip rule fwmark 0x2` can take effect, so the chain is attached at the mangle priority, which is commonly used for packet modification.  

```bash
# Create the inet-family pbr table in the r(router) namespace.
ip netns exec r nft add table inet pbr

# Create a base chain at the prerouting stage.
# priority mangle places this chain where the mark can be set before routing decision.
ip netns exec r nft 'add chain inet pbr prerouting {
    type filter hook prerouting priority mangle;
    policy accept;
}'

# Restore the current packet's fwmark(meta mark) from the conntrack mark(ct mark).
# In other words, after the initial connection setup, restore skb mark from ct mark.
ip netns exec r nft add rule inet pbr prerouting meta mark set ct mark

# If the input interface is r-c,
# the conntrack state is new,
# the protocol is TCP,
# and the destination port is 8443, set fwmark to 0x2.
# In other words, when a TCP 8443 SYN arrives from the client side,
# mark this connection as traffic that should go through WAN2.
ip netns exec r nft add rule inet pbr prerouting \
    iifname "r-c" \
    ct state new \
    tcp dport 8443 \
    meta mark set 0x2

# Under the same condition, store the packet mark into the conntrack mark.
# Later packets in this connection can restore fwmark from ct mark.
ip netns exec r nft add rule inet pbr prerouting \
    iifname "r-c" \
    ct state new \
    tcp dport 8443 \
    ct mark set meta mark
```

Next, start web servers in the WAN1 and WAN2 namespaces.  
Both servers use the same IP, but WAN1 listens on `8080` and WAN2 listens on `8443`.  

```bash
# WAN1
mkdir -p /tmp/w1
echo 'I am WAN1' > /tmp/w1/index.html
ip netns exec wan1 \
    python3 -m http.server 8080 \
    --bind 203.0.113.10 \
    --directory /tmp/w1

# In another terminal, WAN2
mkdir -p /tmp/w2
echo 'I am WAN2' > /tmp/w2/index.html
ip netns exec wan2 \
    python3 -m http.server 8443 \
    --bind 203.0.113.10 \
    --directory /tmp/w2
```

Then, from another terminal, send HTTP requests from the client namespace.  

```bash
# Send a request to WAN1
ip netns exec c \
    curl 203.0.113.10:8080

# Send a request to WAN2
ip netns exec c \
    curl 203.0.113.10:8443
```

While sending requests, it is useful to run `tcpdump` in the router namespace and check which interface the traffic exits from.  

```bash
ip netns exec r \
    tcpdump -ni r-w1

ip netns exec r \
    tcpdump -ni r-w2
```

Finally, check conntrack information.  

```bash
root@ubuntu:~# ip netns exec r conntrack -L -o extended
ipv4 2 tcp 6 111 TIME_WAIT src=10.0.0.2 dst=203.0.113.10 sport=52262 dport=8443 src=203.0.113.10 dst=10.0.0.2 sport=8443 dport=52262 [ASSURED] mark=2 use=1
ipv4 2 tcp 6 118 TIME_WAIT src=10.0.0.2 dst=203.0.113.10 sport=42574 dport=8080 src=203.0.113.10 dst=10.0.0.2 sport=8080 dport=42574 [ASSURED] mark=0 use=1
conntrack v1.4.9 (conntrack-tools): 2 flow entries have been shown.
```

You can see that **the `dport=8443` flow has `mark=2`**, while **the `dport=8080` flow has `mark=0`**.  
In other words, **only TCP 8443 traffic looks up table 100 through the fwmark-based rule and exits through the WAN2 path.**  

You can clean up the lab environment with the following commands.  

```bash
ip netns del c
ip netns del r
ip netns del wan1
ip netns del wan2
```

---
## 📚 References

- [ip-route(8) - Linux manual page](https://man7.org/linux/man-pages/man8/ip-route.8.html)
- [LC-trie implementation notes](https://docs.kernel.org/networking/fib_trie.html)
- [ip-rule(8) - Linux manual page](https://man7.org/linux/man-pages/man8/ip-rule.8.html)
- [struct sk_buff documentation](https://docs.kernel.org/networking/skbuff.html)
- [nftables - Setting packet metainformation](https://wiki.nftables.org/wiki-nftables/index.php/Setting_packet_metainformation)
- [RFC 5737 - IPv4 Address Blocks Reserved for Documentation](https://www.rfc-editor.org/rfc/rfc5737.html)
