---
title: "Cilium과 노드의 tailscaled가 충돌한 이유"
description: "Cilium L7 TPROXY 트래픽이 tailscaled가 활성화한 src_valid_mark와 충돌해 IP_LOCAL_SOURCE로 드롭된 사례를 커널 tracing과 FIB lookup으로 분석한다"
date: 2026-09-14T15:43:19+09:00
lastmod: 2026-09-30T10:42:15+09:00
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

홈랩을 개편하면서 k3s 클러스터의 CNI로 **Cilium**을 설치하고, 원격 접속을 위해 각 노드에 **Tailscale**을 설치했다.  
두 구성 요소는 각각 정상적으로 보였지만, 함께 사용하자 Cilium의 L7 트래픽이 타임아웃되는 문제가 발생했다.

이 글에서는 패킷이 Cilium의 TPROXY로 리다이렉트된 뒤 커널의 source validation에서 `IP_LOCAL_SOURCE`로 드롭되는 과정을 추적한다.  
분석 결과, tailscaled가 활성화한 `net.ipv4.conf.all.src_valid_mark=1`과 Cilium의 **fwmark 기반 TPROXY routing**이 특정 조건에서 충돌하고 있었다.

아래 글을 먼저 읽으면 `ip` 명령어부터 Netfilter, policy routing, source validation, TPROXY까지 이어지는 흐름을 이해하는 데 도움이 된다.

- [`ip` 명령어와 network namespace]({{< relref "post/linux-network-ip" >}})
- [Netfilter와 nftables]({{< relref "post/linux-network-nft" >}})
- [Routing(1): `ip rule`, fwmark, policy routing]({{< relref "post/linux-network-route-1" >}})
- [Routing(2): FIB lookup과 source validation]({{< relref "post/linux-network-route-2" >}})
- [TPROXY 기초]({{< relref "post/linux-network-tproxy-basic" >}})

---
## 🚨 문제 상황

재현 환경은 다음과 같다.

- **Cilium:** v1.20.1
- **Ubuntu:** 26.04 LTS
- **Kubernetes:** v1.37.0
- **클러스터 구성:** 단일 노드

`cilium connectivity test`를 실행하자 **80개 중 26개 테스트가 실패**했다. 실패한 항목은 공통적으로 Cilium의 proxy를 거치는 **L7 트래픽 테스트**였고, 요청은 exit code `28`, 즉 타임아웃으로 종료됐다.

아래는 당시의 전체 실패 로그다.

```bash
📋 Test Report [cilium-test-1]
 ❌ 26/80 tests failed (70/318 actions), 57 tests skipped, 0 scenarios skipped:
Test [echo-ingress-l7]:
 🟥 echo-ingress-l7/pod-to-pod-with-endpoints:curl-ipv4-0-public: cilium-test-1/client2-7899b7dd9c-9bcss (10.217.0.246) -> curl-ipv4-0-public (10.217.0.70:8080): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -4 -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null http://10.217.0.70:8080/public" failed: command failed (pod=cilium-test-1/client2-7899b7dd9c-9bcss, container=client2): command terminated with exit code 28
 🟥 echo-ingress-l7/pod-to-pod-with-endpoints:curl-ipv4-0-private: cilium-test-1/client2-7899b7dd9c-9bcss (10.217.0.246) -> curl-ipv4-0-private (10.217.0.70:8080): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -4 -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null http://10.217.0.70:8080/private" failed with unexpected exit code: command failed (pod=cilium-test-1/client2-7899b7dd9c-9bcss, container=client2): command terminated with exit code 28 (expected 22, found 28)
 🟥 echo-ingress-l7/pod-to-pod-with-endpoints:curl-ipv4-0-privatewith-header: cilium-test-1/client2-7899b7dd9c-9bcss (10.217.0.246) -> curl-ipv4-0-privatewith-header (10.217.0.70:8080): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -4 -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null -H X-Very-Secret-Token: 42 http://10.217.0.70:8080/private" failed: command failed (pod=cilium-test-1/client2-7899b7dd9c-9bcss, container=client2): command terminated with exit code 28
Test [echo-ingress-l7-named-port]:
 🟥 echo-ingress-l7-named-port/pod-to-pod-with-endpoints:curl-ipv4-1-public: cilium-test-1/client2-7899b7dd9c-9bcss (10.217.0.246) -> curl-ipv4-1-public (10.217.0.70:8080): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -4 -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null --retry 3 --retry-all-errors --retry-delay 3 http://10.217.0.70:8080/public" failed: command failed (pod=cilium-test-1/client2-7899b7dd9c-9bcss, container=client2): command terminated with exit code 28
 🟥 echo-ingress-l7-named-port/pod-to-pod-with-endpoints:curl-ipv4-1-private: cilium-test-1/client2-7899b7dd9c-9bcss (10.217.0.246) -> curl-ipv4-1-private (10.217.0.70:8080): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -4 -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null http://10.217.0.70:8080/private" failed with unexpected exit code: command failed (pod=cilium-test-1/client2-7899b7dd9c-9bcss, container=client2): command terminated with exit code 28 (expected 22, found 28)
 🟥 echo-ingress-l7-named-port/pod-to-pod-with-endpoints:curl-ipv4-1-privatewith-header: cilium-test-1/client2-7899b7dd9c-9bcss (10.217.0.246) -> curl-ipv4-1-privatewith-header (10.217.0.70:8080): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -4 -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null --retry 3 --retry-all-errors --retry-delay 3 -H X-Very-Secret-Token: 42 http://10.217.0.70:8080/private" failed: command failed (pod=cilium-test-1/client2-7899b7dd9c-9bcss, container=client2): command terminated with exit code 28
Test [client-egress-l7-method]:
 🟥 client-egress-l7-method/pod-to-pod-with-endpoints:curl-ipv4-1-public: cilium-test-1/client2-7899b7dd9c-9bcss (10.217.0.246) -> curl-ipv4-1-public (10.217.0.70:8080): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -4 -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null -X POST http://10.217.0.70:8080/public" failed: command failed (pod=cilium-test-1/client2-7899b7dd9c-9bcss, container=client2): command terminated with exit code 28
 🟥 client-egress-l7-method/pod-to-pod-with-endpoints:curl-ipv4-1-private: cilium-test-1/client2-7899b7dd9c-9bcss (10.217.0.246) -> curl-ipv4-1-private (10.217.0.70:8080): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -4 -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null -X POST http://10.217.0.70:8080/private" failed: command failed (pod=cilium-test-1/client2-7899b7dd9c-9bcss, container=client2): command terminated with exit code 28
 🟥 client-egress-l7-method/pod-to-pod-with-endpoints:curl-ipv4-1-privatewith-header: cilium-test-1/client2-7899b7dd9c-9bcss (10.217.0.246) -> curl-ipv4-1-privatewith-header (10.217.0.70:8080): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -4 -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null -X POST -H X-Very-Secret-Token: 42 http://10.217.0.70:8080/private" failed: command failed (pod=cilium-test-1/client2-7899b7dd9c-9bcss, container=client2): command terminated with exit code 28
Test [client-egress-l7-method-port-range]:
 🟥 client-egress-l7-method-port-range/pod-to-pod-with-endpoints:curl-ipv4-0-public: cilium-test-1/client2-7899b7dd9c-9bcss (10.217.0.246) -> curl-ipv4-0-public (10.217.0.70:8080): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -4 -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null -X POST http://10.217.0.70:8080/public" failed: command failed (pod=cilium-test-1/client2-7899b7dd9c-9bcss, container=client2): command terminated with exit code 28
 🟥 client-egress-l7-method-port-range/pod-to-pod-with-endpoints:curl-ipv4-0-private: cilium-test-1/client2-7899b7dd9c-9bcss (10.217.0.246) -> curl-ipv4-0-private (10.217.0.70:8080): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -4 -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null -X POST http://10.217.0.70:8080/private" failed: command failed (pod=cilium-test-1/client2-7899b7dd9c-9bcss, container=client2): command terminated with exit code 28
 🟥 client-egress-l7-method-port-range/pod-to-pod-with-endpoints:curl-ipv4-0-privatewith-header: cilium-test-1/client2-7899b7dd9c-9bcss (10.217.0.246) -> curl-ipv4-0-privatewith-header (10.217.0.70:8080): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -4 -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null -X POST -H X-Very-Secret-Token: 42 http://10.217.0.70:8080/private" failed: command failed (pod=cilium-test-1/client2-7899b7dd9c-9bcss, container=client2): command terminated with exit code 28
Test [client-egress-l7]:
 🟥 client-egress-l7/pod-to-pod:curl-ipv4-1: cilium-test-1/client2-7899b7dd9c-9bcss (10.217.0.246) -> cilium-test-1/echo-same-node-68d4675b56-ppnsl (10.217.0.70:8080): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -4 -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null http://10.217.0.70:8080" failed: command failed (pod=cilium-test-1/client2-7899b7dd9c-9bcss, container=client2): command terminated with exit code 28
 🟥 client-egress-l7/pod-to-world:http-to-one.one.one.one.-ipv4-1: cilium-test-1/client2-7899b7dd9c-9bcss (10.217.0.246) -> one.one.one.one.-http (one.one.one.one.:80): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -4 -H Host: one.one.one.one -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null --retry 3 --retry-all-errors --retry-delay 3 http://one.one.one.one.:80" failed: command failed (pod=cilium-test-1/client2-7899b7dd9c-9bcss, container=client2): command terminated with exit code 28
Test [client-egress-l7-port-range]:
 🟥 client-egress-l7-port-range/pod-to-pod:curl-ipv4-1: cilium-test-1/client2-7899b7dd9c-9bcss (10.217.0.246) -> cilium-test-1/echo-same-node-68d4675b56-ppnsl (10.217.0.70:8080): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -4 -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null http://10.217.0.70:8080" failed: command failed (pod=cilium-test-1/client2-7899b7dd9c-9bcss, container=client2): command terminated with exit code 28
 🟥 client-egress-l7-port-range/pod-to-world:http-to-one.one.one.one.-ipv4-1: cilium-test-1/client2-7899b7dd9c-9bcss (10.217.0.246) -> one.one.one.one.-http (one.one.one.one.:80): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -4 -H Host: one.one.one.one -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null --retry 3 --retry-all-errors --retry-delay 3 http://one.one.one.one.:80" failed: command failed (pod=cilium-test-1/client2-7899b7dd9c-9bcss, container=client2): command terminated with exit code 28
Test [client-egress-l7-named-port]:
 🟥 client-egress-l7-named-port/pod-to-pod:curl-ipv4-1: cilium-test-1/client2-7899b7dd9c-9bcss (10.217.0.246) -> cilium-test-1/echo-same-node-68d4675b56-ppnsl (10.217.0.70:8080): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -4 -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null http://10.217.0.70:8080" failed: command failed (pod=cilium-test-1/client2-7899b7dd9c-9bcss, container=client2): command terminated with exit code 28
 🟥 client-egress-l7-named-port/pod-to-world:http-to-one.one.one.one.-ipv4-1: cilium-test-1/client2-7899b7dd9c-9bcss (10.217.0.246) -> one.one.one.one.-http (one.one.one.one.:80): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -4 -H Host: one.one.one.one -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null --retry 3 --retry-all-errors --retry-delay 3 http://one.one.one.one.:80" failed: command failed (pod=cilium-test-1/client2-7899b7dd9c-9bcss, container=client2): command terminated with exit code 28
Test [client-egress-tls-sni]:
 🟥 client-egress-tls-sni/pod-to-world:https-to-one.one.one.one.-ipv4-0: cilium-test-1/client-88bb5f9f8-jhjdz (10.217.0.196) -> one.one.one.one.-https (one.one.one.one.:443): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -4 -H Host: one.one.one.one -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null --retry 3 --retry-all-errors --retry-delay 3 https://one.one.one.one.:443" failed: command failed (pod=cilium-test-1/client-88bb5f9f8-jhjdz, container=client): command terminated with exit code 28
 🟥 client-egress-tls-sni/pod-to-world:https-to-one.one.one.one.-index-ipv4-0: cilium-test-1/client-88bb5f9f8-jhjdz (10.217.0.196) -> one.one.one.one.-https-index (one.one.one.one.:443): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -4 -H Host: one.one.one.one -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null --retry 3 --retry-all-errors --retry-delay 3 https://one.one.one.one.:443/index.html" failed: command failed (pod=cilium-test-1/client-88bb5f9f8-jhjdz, container=client): command terminated with exit code 28
 🟥 client-egress-tls-sni/pod-to-world:https-to-one.one.one.one.-ipv4-1: cilium-test-1/client2-7899b7dd9c-9bcss (10.217.0.246) -> one.one.one.one.-https (one.one.one.one.:443): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -4 -H Host: one.one.one.one -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null --retry 3 --retry-all-errors --retry-delay 3 https://one.one.one.one.:443" failed: command failed (pod=cilium-test-1/client2-7899b7dd9c-9bcss, container=client2): command terminated with exit code 28
 🟥 client-egress-tls-sni/pod-to-world:https-to-one.one.one.one.-index-ipv4-1: cilium-test-1/client2-7899b7dd9c-9bcss (10.217.0.246) -> one.one.one.one.-https-index (one.one.one.one.:443): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -4 -H Host: one.one.one.one -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null --retry 3 --retry-all-errors --retry-delay 3 https://one.one.one.one.:443/index.html" failed: command failed (pod=cilium-test-1/client2-7899b7dd9c-9bcss, container=client2): command terminated with exit code 28 Test [client-egress-tls-sni-denied]:
 🟥 client-egress-tls-sni-denied/pod-to-world:https-to-one.one.one.one.-ipv4-0: cilium-test-1/client-88bb5f9f8-jhjdz (10.217.0.196) -> one.one.one.one.-https (one.one.one.one.:443): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -4 -H Host: one.one.one.one -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null https://one.one.one.one.:443" failed with unexpected exit code: command failed (pod=cilium-test-1/client-88bb5f9f8-jhjdz, container=client): command terminated with exit code 28 (expected 35, found 28)
 🟥 client-egress-tls-sni-denied/pod-to-world:https-to-one.one.one.one.-index-ipv4-0: cilium-test-1/client-88bb5f9f8-jhjdz (10.217.0.196) -> one.one.one.one.-https-index (one.one.one.one.:443): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -4 -H Host: one.one.one.one -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null https://one.one.one.one.:443/index.html" failed with unexpected exit code: command failed (pod=cilium-test-1/client-88bb5f9f8-jhjdz, container=client): command terminated with exit code 28 (expected 35, found 28)
 🟥 client-egress-tls-sni-denied/pod-to-world:https-to-one.one.one.one.-ipv4-1: cilium-test-1/client2-7899b7dd9c-9bcss (10.217.0.246) -> one.one.one.one.-https (one.one.one.one.:443): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -4 -H Host: one.one.one.one -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null https://one.one.one.one.:443" failed with unexpected exit code: command failed (pod=cilium-test-1/client2-7899b7dd9c-9bcss, container=client2): command terminated with exit code 28 (expected 35, found 28)
 🟥 client-egress-tls-sni-denied/pod-to-world:https-to-one.one.one.one.-index-ipv4-1: cilium-test-1/client2-7899b7dd9c-9bcss (10.217.0.246) -> one.one.one.one.-https-index (one.one.one.one.:443): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -4 -H Host: one.one.one.one -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null https://one.one.one.one.:443/index.html" failed with unexpected exit code: command failed (pod=cilium-test-1/client2-7899b7dd9c-9bcss, container=client2): command terminated with exit code 28 (expected 35, found 28)
Test [client-egress-tls-sni-wildcard]:
 🟥 client-egress-tls-sni-wildcard/pod-to-world:https-to-one.one.one.one.-ipv4-0: cilium-test-1/client-88bb5f9f8-jhjdz (10.217.0.196) -> one.one.one.one.-https (one.one.one.one.:443): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -4 -H Host: one.one.one.one -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null --retry 3 --retry-all-errors --retry-delay 3 https://one.one.one.one.:443" failed: command failed (pod=cilium-test-1/client-88bb5f9f8-jhjdz, container=client): command terminated with exit code 28
 🟥 client-egress-tls-sni-wildcard/pod-to-world:https-to-one.one.one.one.-index-ipv4-0: cilium-test-1/client-88bb5f9f8-jhjdz (10.217.0.196) -> one.one.one.one.-https-index (one.one.one.one.:443): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -4 -H Host: one.one.one.one -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null --retry 3 --retry-all-errors --retry-delay 3 https://one.one.one.one.:443/index.html" failed: command failed (pod=cilium-test-1/client-88bb5f9f8-jhjdz, container=client): command terminated with exit code 28
 🟥 client-egress-tls-sni-wildcard/pod-to-world:https-to-one.one.one.one.-ipv4-1: cilium-test-1/client2-7899b7dd9c-9bcss (10.217.0.246) -> one.one.one.one.-https (one.one.one.one.:443): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -4 -H Host: one.one.one.one -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null --retry 3 --retry-all-errors --retry-delay 3 https://one.one.one.one.:443" failed: command failed (pod=cilium-test-1/client2-7899b7dd9c-9bcss, container=client2): command terminated with exit code 28
 🟥 client-egress-tls-sni-wildcard/pod-to-world:https-to-one.one.one.one.-index-ipv4-1: cilium-test-1/client2-7899b7dd9c-9bcss (10.217.0.246) -> one.one.one.one.-https-index (one.one.one.one.:443): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -4 -H Host: one.one.one.one -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null --retry 3 --retry-all-errors --retry-delay 3 https://one.one.one.one.:443/index.html" failed: command failed (pod=cilium-test-1/client2-7899b7dd9c-9bcss, container=client2): command terminated with exit code 28
Test [client-egress-tls-sni-wildcard-denied]:
 🟥 client-egress-tls-sni-wildcard-denied/pod-to-world-2:https-k8s.io.-ipv4-0: cilium-test-1/client-88bb5f9f8-jhjdz (10.217.0.196) -> k8s.io.-https (k8s.io.:443): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -4 -H Host: k8s.io -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null https://k8s.io.:443" failed with unexpected exit code: command failed (pod=cilium-test-1/client-88bb5f9f8-jhjdz, container=client): command terminated with exit code 28 (expected 35, found 28)
 🟥 client-egress-tls-sni-wildcard-denied/pod-to-world-2:https-k8s.io.-ipv4-1: cilium-test-1/client2-7899b7dd9c-9bcss (10.217.0.246) -> k8s.io.-https (k8s.io.:443): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -4 -H Host: k8s.io -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null https://k8s.io.:443" failed with unexpected exit code: command failed (pod=cilium-test-1/client2-7899b7dd9c-9bcss, container=client2): command terminated with exit code 28 (expected 35, found 28)
Test [client-egress-tls-sni-random-wildcard]:
 🟥 client-egress-tls-sni-random-wildcard/pod-to-world:https-to-one.one.one.one.-ipv4-0: cilium-test-1/client-88bb5f9f8-jhjdz (10.217.0.196) -> one.one.one.one.-https (one.one.one.one.:443): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -4 -H Host: one.one.one.one -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null --retry 3 --retry-all-errors --retry-delay 3 https://one.one.one.one.:443" failed: command failed (pod=cilium-test-1/client-88bb5f9f8-jhjdz, container=client): command terminated with exit code 28
 🟥 client-egress-tls-sni-random-wildcard/pod-to-world:https-to-one.one.one.one.-index-ipv4-0: cilium-test-1/client-88bb5f9f8-jhjdz (10.217.0.196) -> one.one.one.one.-https-index (one.one.one.one.:443): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -4 -H Host: one.one.one.one -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null --retry 3 --retry-all-errors --retry-delay 3 https://one.one.one.one.:443/index.html" failed: command failed (pod=cilium-test-1/client-88bb5f9f8-jhjdz, container=client): command terminated with exit code 28
 🟥 client-egress-tls-sni-random-wildcard/pod-to-world:https-to-one.one.one.one.-ipv4-1: cilium-test-1/client2-7899b7dd9c-9bcss (10.217.0.246) -> one.one.one.one.-https (one.one.one.one.:443): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -4 -H Host: one.one.one.one -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null --retry 3 --retry-all-errors --retry-delay 3 https://one.one.one.one.:443" failed: command failed (pod=cilium-test-1/client2-7899b7dd9c-9bcss, container=client2): command terminated with exit code 28
 🟥 client-egress-tls-sni-random-wildcard/pod-to-world:https-to-one.one.one.one.-index-ipv4-1: cilium-test-1/client2-7899b7dd9c-9bcss (10.217.0.246) -> one.one.one.one.-https-index (one.one.one.one.:443): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -4 -H Host: one.one.one.one -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null --retry 3 --retry-all-errors --retry-delay 3 https://one.one.one.one.:443/index.html" failed: command failed (pod=cilium-test-1/client2-7899b7dd9c-9bcss, container=client2): command terminated with exit code 28
Test [client-egress-tls-sni-random-wildcard-denied]:
 🟥 client-egress-tls-sni-random-wildcard-denied/pod-to-world-2:https-k8s.io.-ipv4-0: cilium-test-1/client2-7899b7dd9c-9bcss (10.217.0.246) -> k8s.io.-https (k8s.io.:443): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -4 -H Host: k8s.io -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null https://k8s.io.:443" failed with unexpected exit code: command failed (pod=cilium-test-1/client2-7899b7dd9c-9bcss, container=client2): command terminated with exit code 28 (expected 35, found 28)
 🟥 client-egress-tls-sni-random-wildcard-denied/pod-to-world-2:https-k8s.io.-ipv4-1: cilium-test-1/client-88bb5f9f8-jhjdz (10.217.0.196) -> k8s.io.-https (k8s.io.:443): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -4 -H Host: k8s.io -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null https://k8s.io.:443" failed with unexpected exit code: command failed (pod=cilium-test-1/client-88bb5f9f8-jhjdz, container=client): command terminated with exit code 28 (expected 35, found 28)
Test [client-egress-tls-sni-double-wildcard]:
 🟥 client-egress-tls-sni-double-wildcard/pod-to-world:https-to-one.one.one.one.-ipv4-0: cilium-test-1/client-88bb5f9f8-jhjdz (10.217.0.196) -> one.one.one.one.-https (one.one.one.one.:443): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -4 -H Host: one.one.one.one -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null --retry 3 --retry-all-errors --retry-delay 3 https://one.one.one.one.:443" failed: command failed (pod=cilium-test-1/client-88bb5f9f8-jhjdz, container=client): command terminated with exit code 28
 🟥 client-egress-tls-sni-double-wildcard/pod-to-world:https-to-one.one.one.one.-index-ipv4-0: cilium-test-1/client-88bb5f9f8-jhjdz (10.217.0.196) -> one.one.one.one.-https-index (one.one.one.one.:443): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -4 -H Host: one.one.one.one -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null --retry 3 --retry-all-errors --retry-delay 3 https://one.one.one.one.:443/index.html" failed: command failed (pod=cilium-test-1/client-88bb5f9f8-jhjdz, container=client): command terminated with exit code 28
 🟥 client-egress-tls-sni-double-wildcard/pod-to-world:https-to-one.one.one.one.-ipv4-1: cilium-test-1/client2-7899b7dd9c-9bcss (10.217.0.246) -> one.one.one.one.-https (one.one.one.one.:443): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -4 -H Host: one.one.one.one -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null --retry 3 --retry-all-errors --retry-delay 3 https://one.one.one.one.:443" failed: command failed (pod=cilium-test-1/client2-7899b7dd9c-9bcss, container=client2): command terminated with exit code 28
 🟥 client-egress-tls-sni-double-wildcard/pod-to-world:https-to-one.one.one.one.-index-ipv4-1: cilium-test-1/client2-7899b7dd9c-9bcss (10.217.0.246) -> one.one.one.one.-https-index (one.one.one.one.:443): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -4 -H Host: one.one.one.one -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null --retry 3 --retry-all-errors --retry-delay 3 https://one.one.one.one.:443/index.html" failed: command failed (pod=cilium-test-1/client2-7899b7dd9c-9bcss, container=client2): command terminated with exit code 28
Test [client-egress-tls-sni-double-wildcard-denied]:
 🟥 client-egress-tls-sni-double-wildcard-denied/pod-to-world-2:https-k8s.io.-ipv4-0: cilium-test-1/client-88bb5f9f8-jhjdz (10.217.0.196) -> k8s.io.-https (k8s.io.:443): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -4 -H Host: k8s.io -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null https://k8s.io.:443" failed with unexpected exit code: command failed (pod=cilium-test-1/client-88bb5f9f8-jhjdz, container=client): command terminated with exit code 28 (expected 35, found 28)
 🟥 client-egress-tls-sni-double-wildcard-denied/pod-to-world-2:https-k8s.io.-ipv4-1: cilium-test-1/client2-7899b7dd9c-9bcss (10.217.0.246) -> k8s.io.-https (k8s.io.:443): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -4 -H Host: k8s.io -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null https://k8s.io.:443" failed with unexpected exit code: command failed (pod=cilium-test-1/client2-7899b7dd9c-9bcss, container=client2): command terminated with exit code 28 (expected 35, found 28)
Test [client-egress-l7-tls-headers-sni]:
 🟥 client-egress-l7-tls-headers-sni/pod-to-world-with-tls-intercept:https-to-one.one.one.one.-0: cilium-test-1/client-88bb5f9f8-jhjdz (10.217.0.196) -> one.one.one.one.-https (one.one.one.one.:443): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -H Host: one.one.one.one -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null --cacert /tmp/test-ca.crt -H X-Very-Secret-Token: 42 --retry 5 --retry-delay 0 --retry-all-errors https://one.one.one.one.:443" failed: command failed (pod=cilium-test-1/client-88bb5f9f8-jhjdz, container=client): command terminated with exit code 28
 🟥 client-egress-l7-tls-headers-sni/pod-to-world-with-tls-intercept:https-to-one.one.one.one.-1: cilium-test-1/client2-7899b7dd9c-9bcss (10.217.0.246) -> one.one.one.one.-https (one.one.one.one.:443): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -H Host: one.one.one.one -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null --cacert /tmp/test-ca.crt -H X-Very-Secret-Token: 42 --retry 5 --retry-delay 0 --retry-all-errors https://one.one.one.one.:443" failed: command failed (pod=cilium-test-1/client2-7899b7dd9c-9bcss, container=client2): command terminated with exit code 28
Test [client-egress-l7-tls-headers-other-sni]:
 🟥 client-egress-l7-tls-headers-other-sni/pod-to-world-with-tls-intercept:https-to-one.one.one.one.-0: cilium-test-1/client-88bb5f9f8-jhjdz (10.217.0.196) -> one.one.one.one.-https (one.one.one.one.:443): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -H Host: one.one.one.one -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null --cacert /tmp/test-ca.crt -H X-Very-Secret-Token: 42 https://one.one.one.one.:443" failed with unexpected exit code: command failed (pod=cilium-test-1/client-88bb5f9f8-jhjdz, container=client): command terminated with exit code 28 (expected 35, found 28)
 🟥 client-egress-l7-tls-headers-other-sni/pod-to-world-with-tls-intercept:https-to-one.one.one.one.-1: cilium-test-1/client2-7899b7dd9c-9bcss (10.217.0.246) -> one.one.one.one.-https (one.one.one.one.:443): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -H Host: one.one.one.one -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null --cacert /tmp/test-ca.crt -H X-Very-Secret-Token: 42 https://one.one.one.one.:443" failed with unexpected exit code: command failed (pod=cilium-test-1/client2-7899b7dd9c-9bcss, container=client2): command terminated with exit code 28 (expected 35, found 28)
Test [client-egress-l7-set-header]:
 🟥 client-egress-l7-set-header/pod-to-pod-with-endpoints:curl-ipv4-1-auth-header-required: cilium-test-1/client2-7899b7dd9c-9bcss (10.217.0.246) -> curl-ipv4-1-auth-header-required (10.217.0.70:8080): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -4 -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null -X POST --retry 3 --retry-all-errors --retry-delay 3 http://10.217.0.70:8080/auth-header-required" failed: command failed (pod=cilium-test-1/client2-7899b7dd9c-9bcss, container=client2): command terminated with exit code 28
Test [client-egress-l7-set-header-port-range]:
 🟥 client-egress-l7-set-header-port-range/pod-to-pod-with-endpoints:curl-ipv4-1-auth-header-required: cilium-test-1/client2-7899b7dd9c-9bcss (10.217.0.246) -> curl-ipv4-1-auth-header-required (10.217.0.70:8080): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -4 -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null -X POST --retry 3 --retry-all-errors --retry-delay 3 http://10.217.0.70:8080/auth-header-required" failed: command failed (pod=cilium-test-1/client2-7899b7dd9c-9bcss, container=client2): command terminated with exit code 28
Test [to-fqdns]:
 🟥 to-fqdns/pod-to-world:http-to-one.one.one.one.-ipv4-0: cilium-test-1/client-88bb5f9f8-jhjdz (10.217.0.196) -> one.one.one.one.-http (one.one.one.one.:80): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -4 -H Host: one.one.one.one -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null --retry 3 --retry-all-errors --retry-delay 3 http://one.one.one.one.:80" failed: command failed (pod=cilium-test-1/client-88bb5f9f8-jhjdz, container=client): command terminated with exit code 28
 🟥 to-fqdns/pod-to-world:http-to-one.one.one.one.-ipv4-1: cilium-test-1/client2-7899b7dd9c-9bcss (10.217.0.246) -> one.one.one.one.-http (one.one.one.one.:80): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -4 -H Host: one.one.one.one -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null --retry 3 --retry-all-errors --retry-delay 3 http://one.one.one.one.:80" failed: command failed (pod=cilium-test-1/client2-7899b7dd9c-9bcss, container=client2): command terminated with exit code 28
Test [to-fqdns-with-proxy]:
 🟥 to-fqdns-with-proxy/pod-to-world:http-to-one.one.one.one.-ipv4-0: cilium-test-1/client-88bb5f9f8-jhjdz (10.217.0.196) -> one.one.one.one.-http (one.one.one.one.:80): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -4 -H Host: one.one.one.one -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null --retry 3 --retry-all-errors --retry-delay 3 http://one.one.one.one.:80" failed: command failed (pod=cilium-test-1/client-88bb5f9f8-jhjdz, container=client): command terminated with exit code 28
 🟥 to-fqdns-with-proxy/pod-to-world:http-to-one.one.one.one.-ipv4-1: cilium-test-1/client2-7899b7dd9c-9bcss (10.217.0.246) -> one.one.one.one.-http (one.one.one.one.:80): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -4 -H Host: one.one.one.one -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null --retry 3 --retry-all-errors --retry-delay 3 http://one.one.one.one.:80" failed: command failed (pod=cilium-test-1/client2-7899b7dd9c-9bcss, container=client2): command terminated with exit code 28
Test [to-fqdns-with-ccec-listener]:
 🟥 to-fqdns-with-ccec-listener/pod-to-world:http-to-one.one.one.one.-ipv4-0: cilium-test-1/client-88bb5f9f8-jhjdz (10.217.0.196) -> one.one.one.one.-http (one.one.one.one.:80): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -4 -H Host: one.one.one.one -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null http://one.one.one.one.:80" failed: command failed (pod=cilium-test-1/client-88bb5f9f8-jhjdz, container=client): command terminated with exit code 28
 🟥 to-fqdns-with-ccec-listener/pod-to-world:https-to-one.one.one.one.-ipv4-0: cilium-test-1/client-88bb5f9f8-jhjdz (10.217.0.196) -> one.one.one.one.-https (one.one.one.one.:443): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -4 -H Host: one.one.one.one -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null https://one.one.one.one.:443" failed with unexpected exit code: command failed (pod=cilium-test-1/client-88bb5f9f8-jhjdz, container=client): command terminated with exit code 28 (expected 35, found 28)
 🟥 to-fqdns-with-ccec-listener/pod-to-world:https-to-one.one.one.one.-index-ipv4-0: cilium-test-1/client-88bb5f9f8-jhjdz (10.217.0.196) -> one.one.one.one.-https-index (one.one.one.one.:443): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -4 -H Host: one.one.one.one -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null https://one.one.one.one.:443/index.html" failed with unexpected exit code: command failed (pod=cilium-test-1/client-88bb5f9f8-jhjdz, container=client): command terminated with exit code 28 (expected 35, found 28)
 🟥 to-fqdns-with-ccec-listener/pod-to-world:http-to-one.one.one.one.-ipv4-1: cilium-test-1/client2-7899b7dd9c-9bcss (10.217.0.246) -> one.one.one.one.-http (one.one.one.one.:80): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -4 -H Host: one.one.one.one -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null http://one.one.one.one.:80" failed: command failed (pod=cilium-test-1/client2-7899b7dd9c-9bcss, container=client2): command terminated with exit code 28
 🟥 to-fqdns-with-ccec-listener/pod-to-world:https-to-one.one.one.one.-ipv4-1: cilium-test-1/client2-7899b7dd9c-9bcss (10.217.0.246) -> one.one.one.one.-https (one.one.one.one.:443): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -4 -H Host: one.one.one.one -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null https://one.one.one.one.:443" failed with unexpected exit code: command failed (pod=cilium-test-1/client2-7899b7dd9c-9bcss, container=client2): command terminated with exit code 28 (expected 35, found 28)
 🟥 to-fqdns-with-ccec-listener/pod-to-world:https-to-one.one.one.one.-index-ipv4-1: cilium-test-1/client2-7899b7dd9c-9bcss (10.217.0.246) -> one.one.one.one.-https-index (one.one.one.one.:443): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -4 -H Host: one.one.one.one -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null https://one.one.one.one.:443/index.html" failed with unexpected exit code: command failed (pod=cilium-test-1/client2-7899b7dd9c-9bcss, container=client2): command terminated with exit code 28 (expected 35, found 28)
Test [client-egress-l7-tls-deny-without-headers]:
 🟥 client-egress-l7-tls-deny-without-headers/pod-to-world-with-tls-intercept:https-to-one.one.one.one.-0: cilium-test-1/client-88bb5f9f8-jhjdz (10.217.0.196) -> one.one.one.one.-https (one.one.one.one.:443): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -H Host: one.one.one.one -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null --cacert /tmp/test-ca.crt --retry 5 --retry-delay 0 --retry-all-errors https://one.one.one.one.:443" failed with unexpected exit code: command failed (pod=cilium-test-1/client-88bb5f9f8-jhjdz, container=client): command terminated with exit code 28 (expected 22, found 28)
 🟥 client-egress-l7-tls-deny-without-headers/pod-to-world-with-tls-intercept:https-to-one.one.one.one.-1: cilium-test-1/client2-7899b7dd9c-9bcss (10.217.0.246) -> one.one.one.one.-https (one.one.one.one.:443): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -H Host: one.one.one.one -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null --cacert /tmp/test-ca.crt --retry 5 --retry-delay 0 --retry-all-errors https://one.one.one.one.:443" failed with unexpected exit code: command failed (pod=cilium-test-1/client2-7899b7dd9c-9bcss, container=client2): command terminated with exit code 28 (expected 22, found 28)
Test [client-egress-l7-tls-headers]:
 🟥 client-egress-l7-tls-headers/pod-to-world-with-tls-intercept:https-to-one.one.one.one.-0: cilium-test-1/client2-7899b7dd9c-9bcss (10.217.0.246) -> one.one.one.one.-https (one.one.one.one.:443): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -H Host: one.one.one.one -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null --cacert /tmp/test-ca.crt -H X-Very-Secret-Token: 42 --retry 5 --retry-delay 0 --retry-all-errors https://one.one.one.one.:443" failed: command failed (pod=cilium-test-1/client2-7899b7dd9c-9bcss, container=client2): command terminated with exit code 28
 🟥 client-egress-l7-tls-headers/pod-to-world-with-tls-intercept:https-to-one.one.one.one.-1: cilium-test-1/client-88bb5f9f8-jhjdz (10.217.0.196) -> one.one.one.one.-https (one.one.one.one.:443): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -H Host: one.one.one.one -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null --cacert /tmp/test-ca.crt -H X-Very-Secret-Token: 42 --retry 5 --retry-delay 0 --retry-all-errors https://one.one.one.one.:443" failed: command failed (pod=cilium-test-1/client-88bb5f9f8-jhjdz, container=client): command terminated with exit code 28
Test [client-egress-l7-extra-tls-headers]:
 🟥 client-egress-l7-extra-tls-headers/pod-to-world-with-extra-tls-intercept:https-to-one.one.one.one.-0: cilium-test-1/client-88bb5f9f8-jhjdz (10.217.0.196) -> one.one.one.one.-https (one.one.one.one.:443): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -H Host: one.one.one.one -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null --cacert /tmp/test-ca.crt -H X-Very-Secret-Token: 42 --retry 5 --retry-delay 0 --retry-all-errors https://one.one.one.one.:443" failed: command failed (pod=cilium-test-1/client-88bb5f9f8-jhjdz, container=client): command terminated with exit code 28
 🟥 client-egress-l7-extra-tls-headers/pod-to-world-with-extra-tls-intercept:https-to-k8s.io.-0: cilium-test-1/client-88bb5f9f8-jhjdz (10.217.0.196) -> k8s.io.-https (k8s.io.:443): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -H Host: k8s.io -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null --cacert /tmp/test-ca.crt -H X-Very-Secret-Token: 42 --retry 5 --retry-delay 0 --retry-all-errors https://k8s.io.:443" failed: command failed (pod=cilium-test-1/client-88bb5f9f8-jhjdz, container=client): command terminated with exit code 28
 🟥 client-egress-l7-extra-tls-headers/pod-to-world-with-extra-tls-intercept:https-to-one.one.one.one.-1: cilium-test-1/client2-7899b7dd9c-9bcss (10.217.0.246) -> one.one.one.one.-https (one.one.one.one.:443): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -H Host: one.one.one.one -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null --cacert /tmp/test-ca.crt -H X-Very-Secret-Token: 42 --retry 5 --retry-delay 0 --retry-all-errors https://one.one.one.one.:443" failed: command failed (pod=cilium-test-1/client2-7899b7dd9c-9bcss, container=client2): command terminated with exit code 28
 🟥 client-egress-l7-extra-tls-headers/pod-to-world-with-extra-tls-intercept:https-to-k8s.io.-1: cilium-test-1/client2-7899b7dd9c-9bcss (10.217.0.246) -> k8s.io.-https (k8s.io.:443): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -H Host: k8s.io -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null --cacert /tmp/test-ca.crt -H X-Very-Secret-Token: 42 --retry 5 --retry-delay 0 --retry-all-errors https://k8s.io.:443" failed: command failed (pod=cilium-test-1/client2-7899b7dd9c-9bcss, container=client2): command terminated with exit code 28
Test [client-egress-l7-tls-headers-port-range]:
 🟥 client-egress-l7-tls-headers-port-range/pod-to-world-with-tls-intercept:https-to-one.one.one.one.-0: cilium-test-1/client-88bb5f9f8-jhjdz (10.217.0.196) -> one.one.one.one.-https (one.one.one.one.:443): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -H Host: one.one.one.one -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null --cacert /tmp/test-ca.crt -H X-Very-Secret-Token: 42 --retry 5 --retry-delay 0 --retry-all-errors https://one.one.one.one.:443" failed: command failed (pod=cilium-test-1/client-88bb5f9f8-jhjdz, container=client): command terminated with exit code 28
 🟥 client-egress-l7-tls-headers-port-range/pod-to-world-with-tls-intercept:https-to-one.one.one.one.-1: cilium-test-1/client2-7899b7dd9c-9bcss (10.217.0.246) -> one.one.one.one.-https (one.one.one.one.:443): command "curl --silent --fail --show-error --connect-timeout 2 --max-time 10 -H Host: one.one.one.one -w %{local_ip}:%{local_port} -> %{remote_ip}:%{remote_port} = %{response_code}\n --output /dev/null --cacert /tmp/test-ca.crt -H X-Very-Secret-Token: 42 --retry 5 --retry-delay 0 --retry-all-errors https://one.one.one.one.:443" failed: command failed (pod=cilium-test-1/client2-7899b7dd9c-9bcss, container=client2): command terminated with exit code 28
[cilium-test-1] 26 tests failed
root@cp-1:~#
```

---
## 🔍 디버깅 과정

### 최소 재현 환경 구성

먼저 테스트용 Pod와 `CiliumNetworkPolicy`를 생성한다. DNS L7 규칙이 적용된 트래픽을 의도적으로 Cilium proxy로 보내기 위한 구성이다.

```bash
# Namespace 생성
kubectl create ns cilium-test

# Pod 생성
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

# CiliumNetworkPolicy 생성
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

이어서 `src_valid_mark`를 활성화해 tailscaled가 설치된 노드와 같은 조건을 만든다.

```bash
sysctl -w net.ipv4.conf.all.src_valid_mark=1
```

Pod에서 DNS 조회를 실행하면 **타임아웃이 발생**한다.

```bash
kubectl -n cilium-test exec curl -- nslookup google.com
```

### Cilium monitor로 proxy redirect 확인

Pod IP와 Cilium endpoint ID를 찾은 뒤 해당 endpoint의 트래픽을 모니터링한다.

```bash
# Pod IP 알아내기
kubectl -n cilium-test get pod curl -o wide

# Pod endpoint 알아내기
kubectl -n kube-system exec cilium-<hash> -- \
    cilium-dbg endpoint list | grep <POD-IP>

EP="<POD_ENDPOINT>"

kubectl -n kube-system exec -it cilium-<hash> -- \
    cilium-dbg monitor --related-to="$EP" -v -v
```

다른 터미널에서 DNS 조회를 다시 실행한다.

```bash
kubectl -n cilium-test exec curl -- nslookup google.com
```

monitor 로그에는 endpoint `1885`의 policy verdict가 `action redirect`, `to-proxy`로 기록된다. 즉, DNS L7 규칙에 따라 패킷이 노드의 TPROXY로 넘어간다.

```text
Policy verdict log: flow 0x98fc9bfe local EP ID 1885, remote ID 19110, proto 17, egress, action redirect, auth: disabled, match L3-L4, 10.217.0.124:49595 -> 10.217.0.76:53 udp
-----------------------------------------------------------------------------
CPU 01: MARK 0x98fc9bfe FROM 1885 to-proxy: 70 bytes (70 captured), state new, , identity 16010->unknown, orig-ip 0.0.0.0, to proxy-port 45867
Ethernet        {Contents=[..14..] Payload=[..62..] SrcMAC=4e:aa:8b:40:68:2b DstMAC=c2:51:7e:42:84:e1 EthernetType=IPv4 Length=0}
IPv4    {Contents=[..20..] Payload=[..36..] Version=4 IHL=5 TOS=0 Length=56 Id=33348 Flags=DF FragOffset=0 TTL=64 Protocol=UDP Checksum=41463 SrcIP=10.217.0.124 DstIP=10.217.0.76 Options=[] Padding=[]}
UDP     {Contents=[..8..] Payload=[..28..] SrcPort=49595 DstPort=53(domain) Length=36 Checksum=5807}
```

### Kernel tracing으로 drop reason 확인

이제 `kfree_skb` tracepoint를 활성화해 커널이 패킷을 버리는 이유를 확인한다.

```bash
TRACE=/sys/kernel/tracing

mkdir -p "$TRACE/instances/cilium-srcmark"
T="$TRACE/instances/cilium-srcmark"

echo 0 > "$T/tracing_on"
echo > "$T/trace"

echo 1 > "$T/events/skb/kfree_skb/enable"
echo 1 > "$T/tracing_on"
```

tracing을 켠 상태에서 DNS 조회를 한 번 더 실행한다.

```bash
kubectl -n cilium-test exec curl -- nslookup google.com
```

타임아웃이 발생하면 tracing을 중지하고 `IP_LOCAL_SOURCE`를 검색한다.

```bash
echo 0 > "$T/tracing_on"

grep -i 'IP_LOCAL_SOURCE' "$T/trace"
```

다음과 같이 패킷이 `IP_LOCAL_SOURCE` 사유로 드롭된 것을 확인할 수 있다.

```text
        nslookup-113430  [001] ..s1. 164463.161391: kfree_skb: skbaddr=0000000001fff167 rx_sk=0000000000000000 protocol=2048 location=ip_rcv_finish_core+0x233/0x360 reason: IP_LOCAL_SOURCE
        nslookup-113430  [001] ..s1. 164463.161397: kfree_skb: skbaddr=0000000001fff167 rx_sk=0000000000000000 protocol=2048 location=ip_rcv_finish_core+0x233/0x360 reason: IP_LOCAL_SOURCE
```

### 커널 source validation 코드 추적

먼저 `fib_validate_source()`가 `rp_filter=0`일 때도 `__fib_validate_source()`로 들어갈 수 있는지 확인해야 한다. `rp_filter=0`은 RPF를 끄는 설정이지, 이 함수의 모든 출발지 검사를 건너뛴다는 뜻은 아니다. 이 환경에서는 Cilium이 추가한 custom `ip rule`이 있으므로, `accept_local=0`인 패킷은 fast path에서 바로 반환하지 않고 full check로 이동한다. 해당 조건은 글 아래의 9월 29일 재확인 부분에서 코드와 함께 살펴본다.

full check인 `__fib_validate_source()`를 보면 드롭 원인이 드러난다. reverse FIB lookup 결과가 local route인데 `accept_local=0`이면 커널은 패킷을 `IP_LOCAL_SOURCE`로 드롭한다.

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

	fl4.flowi4_oif = 0;
	fl4.flowi4_l3mdev = l3mdev_master_ifindex_rcu(dev);
	fl4.flowi4_iif = oif ? : LOOPBACK_IFINDEX;
	fl4.daddr = src;
	fl4.saddr = dst;
	fl4.flowi4_dscp = dscp;
	fl4.flowi4_scope = RT_SCOPE_UNIVERSE;
	fl4.flowi4_tun_key.tun_id = 0;
	fl4.flowi4_flags = 0;
	fl4.flowi4_uid = sock_net_uid(net, NULL);
	fl4.flowi4_multipath_hash = 0;

	no_addr = idev->ifa_list == NULL;

	// `src_valid_mark=1`이면 fwmark도 reverse lookup에 반영
	fl4.flowi4_mark = IN_DEV_SRC_VMARK(idev) ? skb->mark : 0;
	if (!fib4_rules_early_flow_dissect(net, skb, &fl4, &flkeys)) {
		fl4.flowi4_proto = 0;
		fl4.fl4_sport = 0;
		fl4.fl4_dport = 0;
	} else {
		swap(fl4.fl4_sport, fl4.fl4_dport);
	}

	if (fib_lookup(net, &fl4, &res, 0))
		goto last_resort;
	if (res.type != RTN_UNICAST) {
		if (res.type != RTN_LOCAL) {
			reason = SKB_DROP_REASON_IP_INVALID_SOURCE;
			goto e_inval;

		// Local route이고 `accept_local=0`이면 IP_LOCAL_SOURCE로 drop
		} else if (!IN_DEV_ACCEPT_LOCAL(idev)) {
			reason = SKB_DROP_REASON_IP_LOCAL_SOURCE;
			goto e_inval;
		}
	}

// ...
```

그렇다면 reverse lookup 결과가 왜 local route가 됐을까? 원인은 Cilium이 만든 다음 policy routing rule과 routing table에 있다.

```bash
root@cp-1:~# ip rule
9:      from all fwmark 0x200/0xf00 lookup 2004
100:    from all lookup local
32766:  from all lookup main
32767:  from all lookup default

root@cp-1:~# ip route show table 2004
local default dev lo proto kernel scope host
```

패킷의 fwmark가 `0x200/0xf00`에 매칭되면 main table보다 먼저 **table 2004**를 조회한다. table 2004에는 모든 목적지를 `lo`로 보내는 `local default` route가 있다.

이 rule은 Cilium의 TPROXY routing을 위한 것이다. Cilium은 L7 rule 처리가 필요한 패킷에 mark를 설정하고, 각 노드의 cilium-agent가 관리하는 Envoy proxy로 보낸다.

**`src_valid_mark=1`이면 source validation의 reverse FIB lookup에도 패킷 mark가 반영된다.** 그 결과 Cilium의 table 2004가 선택되고, route type이 `RTN_LOCAL`로 판정된다. 이때 `accept_local=0`이므로 패킷이 드롭된다.

### `src_valid_mark`를 활성화한 주체

`src_valid_mark`를 활성화한 주체는 **tailscaled**였다. Tailscale의 Linux router 구현은 패킷의 fwmark를 source validation에 반영하기 위해 이 sysctl을 `1`로 설정한다.

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
## 🧪 A/B 테스트

`src_valid_mark`와 `accept_local`의 조합을 바꿔가며 테스트한 결과는 다음과 같다.

| `src_valid_mark` | `accept_local` | Reverse FIB에 `0x200` 반영 | Reverse FIB 결과 | DNS 결과 | Drop reason |
| --- | --- | --- | --- | --- | --- |
| `0` | `0` | X | 정상 | 정상 | 없음 |
| `0` | `1` | X | 정상 | 정상 | 없음 |
| `1` | `0` | O | `table 2004 → local dev lo → RTN_LOCAL` | **Timeout** | `IP_LOCAL_SOURCE` |
| `1` | `1` | O | `table 2004 → local dev lo → RTN_LOCAL` | 정상 | 없음 |

오직 **`src_valid_mark=1`이면서 `accept_local=0`인 조합**에서 DNS 요청이 실패한다. 이 결과는 앞서 확인한 커널 코드의 분기와 정확히 일치한다.

---
## 🧭 패킷 드롭 흐름 정리

전체 흐름을 한 줄씩 연결하면 다음과 같다.

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

이 사례에서는 Cilium이 L7 처리를 위해 설정한 `skb->mark=0x200`이 tailscaled가 활성화한 `src_valid_mark=1` 때문에 reverse lookup에 참여했다. 이 lookup이 Cilium의 local route를 선택하고, 기본값인 `accept_local=0`과 만나면서 `IP_LOCAL_SOURCE` drop으로 이어졌다.

따라서 비슷한 현상은 Tailscale에만 국한되지 않을 수 있다. 노드에서 **`net.ipv4.conf.all.src_valid_mark=1`을 설정하는 다른 도구**도 동일한 Cilium routing 조건과 만나면 영향을 줄 가능성이 있다.

---
## 🛠️ 해결 방안과 권장 구성

Cilium을 사용하는 Kubernetes 노드에 Tailscale을 직접 설치해야 한다면, 먼저 **L7/FQDN policy와 `src_valid_mark`의 상호작용을 확인하는 것이 좋다.** 재현 환경에서는 해당 `lxc*` 인터페이스의 `accept_local=1`로 증상을 해소할 수 있었지만, source validation 정책을 바꾸는 조치이므로 적용 전 영향 범위를 충분히 검토해야 한다.

클러스터 서비스에 접근하려는 목적이라면 [Tailscale Kubernetes Operator](https://tailscale.com/docs/kubernetes-operator)를 사용해 Tailscale 연결을 Kubernetes 리소스로 관리하는 방법도 고려할 수 있다.

노드 SSH 접속이 목적이라면 별도 장비의 **Subnet Router**를 통해 노드 대역으로 접근하는 것도 하나의 대안이다.

---
## 📣 GitHub Issue 제보

Cilium 저장소에 이 현상을 [Issue #48706](https://github.com/cilium/cilium/issues/48706)으로 제보했다. 제보 당시의 분석은 재현 결과와 커널 코드에 근거한 것이었고, 이후 PR 논의와 maintainer 답변을 통해 추가로 확인한 내용을 아래에 기록했다.

### 관련 PR 논의 (09.21~22, Closed without merge)

9월 22일 새벽, 한 사용자가 내가 제기한 이슈를 해결하는 PR을 열었다.

변경 내용은 veth/netkit으로 생성되는 `lxc*` 인터페이스에서 `accept_local`을 활성화하는 것이었다.

앞선 A/B 테스트에서도 `lxc*` 인터페이스의 `accept_local`을 활성화하면 트래픽이 정상적으로 흐르는 것을 확인했다.

PR 작성자는 Cilium이 이미 eBPF 프로그램의 `is_valid_lxc_src_ipv4()`에서 커널의 `rp_filter`를 대신해 검증하므로 추가적인 위험은 없다고 주장했다.

이후 한 maintainer가 질문했다.

- Cilium이 `rp_filter`를 비활성화하는데도 왜 `fib_validate_source()`가 실행되고 RPF가 진행되는가?
- Cilium이 전역 `src_valid_mark` 설정을 덮어써서 마크를 검사하지 않도록 하면 되지 않는가? 굳이 `accept_local`을 활성화해야 하는가?

PR 작성자는 다음과 같이 답했다.

- **`rp_filter=0`인데도 `fib_validate_source()`가 실행되는 이유:** 리눅스 커널은 `IN_DEV_MAXCONF()` 매크로를 통해 `conf.all.rp_filter`와 `conf.<dev>.rp_filter` 중 큰 값을 사용한다. 대부분의 현대 리눅스 배포판에서 `net.ipv4.conf.all.rp_filter`의 기본값은 1(strict) 또는 2(loose)이다. 따라서 Cilium이 장치의 `rp_filter`를 0으로 설정해도, 커널은 `max(all, dev) = max(1, 0) = 1`로 평가해 RPF와 `fib_validate_source()`를 실행한다는 설명이었다.
- **`lxc*` 장치에서 `src_valid_mark=0`으로 덮어쓸 수 없는 이유:** PR 작성자도 처음에는 이 방법을 생각했다고 한다. 하지만 `src_valid_mark`는 `IN_DEV_ORCONF()` 매크로로 평가되므로, Tailscale 같은 프로그램이 `net.ipv4.conf.all.src_valid_mark=1`로 설정하면 장치의 값을 0으로 바꿔도 평가 결과는 1이 된다고 설명했다.
- **`accept_local`이 유일한 해결책이라고 본 이유:** 전역 노드 설정의 `rp_filter`와 `src_valid_mark` 때문에 `fib_validate_source()`를 피할 수 없으므로, `accept_local`을 활성화해야 한다는 주장이었다.

별개로 이 PR은 작성자와 maintainer 사이의 AI 정책 관련 논쟁으로 인해 병합 없이 닫혔다.

### maintainer의 이슈 답변 (09.22~현재)

9월 22일, maintainer가 이슈에 답변했다.

- 앞선 PR 논의를 거친 뒤, Tailscale이 `net.ipv4.conf.all.src_valid_mark=1`로 설정해서는 안 된다고 생각한다고 했다. Cilium뿐 아니라 다른 네트워킹 스택 소프트웨어와의 상호 운용성도 깨뜨린다는 것이다.
- Tailscale이 특정 장치(예: WireGuard)와 해당 장치의 라우팅을 담당한다면, 그 장치에서 `src_valid_mark=1`로 설정하는 것은 이해할 수 있다고 했다. 하지만 현재 설정은 Cilium이 노드 내에서 프록시 트래픽을 전달하며 Kubernetes Pod와 통신하는 과정과 충돌한다.
- [커널 문서](https://www.kernel.org/doc/html/latest/networking/ip-sysctl.html)는 이 사례에서 `src_valid_mark=0`이 필요한 이유를 다음과 같이 설명한다.

> src_valid_mark - BOOLEAN
>
> 0 - The fwmark of the packet is not included in reverse path route lookup. This allows for asymmetric routing configurations utilizing the fwmark in only one direction, e.g., transparent proxying.
>
> 1 - The fwmark of the packet is included in reverse path route lookup. This permits rp_filter to function when the fwmark is used for routing traffic in both directions.
>
> This setting also affects the utilization of fmwark when performing source address selection for ICMP replies, or determining addresses stored for the IPOPT_TS_TSANDADDR and IPOPT_RR IP options.
>
> The max value from conf/{all,interface}/src_valid_mark is used.
>
> Default value is 0.

maintainer는 다음과 같은 의견도 덧붙였다.

- [커널 문서](https://www.kernel.org/doc/html/latest/networking/ip-sysctl.html)는 `accept_local`을 다음과 같이 설명한다.

> accept_local - BOOLEAN
>
> Accept packets with local source addresses. In combination with suitable routing, this can be used to > direct packets between two local interfaces over the wire and have them accepted properly. default FALSE

- 여기서 말하는 "local source address"의 의미는 명확하지 않지만, IP가 다른 netns에 있더라도 FIB 분류 결과는 local인 것으로 추정했다.
- 이 설정은 적용 범위가 제한적이며, 이 시나리오에서 Pod에서 호스트 스택으로 향하는 트래픽의 출발지 주소가 '유효한지'에 초점을 맞춘다고 봤다.
- 해당 트래픽은 이미 Cilium의 Pod egress BPF 프로그램을 통과한다. 따라서 추가적인 reverse path 보호로 관련 필터링을 적용할 수 있으며, 기본적으로 이미 적용되고 있다고 설명했다.
- 내 A/B 테스트 결과를 근거로, 이 설정을 활성화하는 것이 이 문제의 완화책이 될 수 있다고 봤다.

9월 23일, 나는 다음과 같이 답했다.

- 논의가 Tailscale에 국한되어 있었지만, NetBird도 `net.ipv4.conf.all.src_valid_mark=1`을 설정한다. Cilium 이슈 #41991과 관련 NetBird 이슈 #4575에서도 해결 방향이 아직 정리되지 않았다.
- Tailscale에서는 `conf.all.src_valid_mark=1` 설정을 제거하고 per-interface `iif` policy routing rule로 대체하려는 draft PR(#19860)이 진행 중이다.
- 이제는 이 문제를 Tailscale만의 문제로 볼지, `src_valid_mark`를 전역으로 활성화하는 네트워킹 소프트웨어 전반과의 호환성을 고려할지가 궁금하다.
- 앞서 논의한 완화책을 구현하던 PR이 닫힌 상태이므로, 당장 새 PR을 만들지는 않고 좀 더 지켜보겠다.

이후 재현 과정을 다시 살펴보다가 PR 논의에 일부 오류가 있음을 발견해 9월 29일에 추가 코멘트를 남겼다.

**`rp_filter=0`인데도 `fib_validate_source()`가 실행되는 이유는 배포판의 전역 기본 설정 때문이 아니다.** 많은 리눅스 배포판에서 `all.rp_filter`의 기본값이 1 또는 2인 것은 맞지만, Cilium은 이를 0으로 변경한다.

다음은 Ubuntu 26.04 LTS의 기본 설정이다.
```bash
root@cp-1:~# sysctl net.ipv4.conf.all.rp_filter
net.ipv4.conf.all.rp_filter = 2

root@cp-1:~# sysctl net.ipv4.conf.eth0.rp_filter
net.ipv4.conf.eth0.rp_filter = 2
```

kubeadm으로 클러스터를 초기화하고 Cilium 1.20.1을 설치하면 다음과 같이 바뀐다.
```bash
root@cp-1:~# sysctl net.ipv4.conf.all.rp_filter
net.ipv4.conf.all.rp_filter = 0

root@cp-1:~# sysctl net.ipv4.conf.eth0.rp_filter
net.ipv4.conf.eth0.rp_filter = 2

# veth pair 하나 예시
root@cp-1:~# sysctl net.ipv4.conf.lxcf3591bba91d5.rp_filter
net.ipv4.conf.lxcf3591bba91d5.rp_filter = 0
```

즉, `lxc*` 인터페이스에서 `rp_filter`의 평가값은 0이 될 수 있다.

그렇다면 `rp_filter`의 평가값이 0일 때도 `fib_validate_source()`가 실행되는지 살펴보자.
```c
// source: net/ipv4/fib_frontend.c
/* Ignore rp_filter for packets protected by IPsec. */
int fib_validate_source(struct sk_buff *skb, __be32 src, __be32 dst,
			dscp_t dscp, int oif, struct net_device *dev,
			struct in_device *idev, u32 *itag)
{
    // IPsec 패킷이면 0, 아니면 `rp_filter` 평가값을 사용
	int r = secpath_exists(skb) ? 0 : IN_DEV_RPFILTER(idev);
	struct net *net = dev_net(dev);

	if (!r && !fib_num_tclassid_users(net) &&
	    (dev->ifindex != oif || !IN_DEV_TX_REDIRECTS(idev))) {
		if (IN_DEV_ACCEPT_LOCAL(idev))
			goto ok;

		/* with custom local routes in place, checking local addresses
		 * only will be too optimistic, with custom rules, checking
		 * local addresses only can be too strict, e.g. due to vrf
		 */
        // ⭐ r=0이어도 custom local route 또는 custom rule이 있으면 full_check(__fib_validate_source())로 이동한다.
		if (net->ipv4.fib_has_custom_local_routes ||
		    fib4_has_custom_rules(net))
			goto full_check;
		/* Within the same container, it is regarded as a martian source,
		 * and the same host but different containers are not.
		 */
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

**`__fib_validate_source()`에 진입하면, 이 사례에서는 `rpf` 값과 관계없이 `IP_LOCAL_SOURCE`로 드롭된다.** 해당 분기를 살펴보자.
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
// ...

	// src_valid_mark=1이면 패킷의 mark를 reverse lookup에 반영한다.
	fl4.flowi4_mark = IN_DEV_SRC_VMARK(idev) ? skb->mark : 0;
	if (!fib4_rules_early_flow_dissect(net, skb, &fl4, &flkeys)) {
		fl4.flowi4_proto = 0;
		fl4.fl4_sport = 0;
		fl4.fl4_dport = 0;
	} else {
		swap(fl4.fl4_sport, fl4.fl4_dport);
	}

	if (fib_lookup(net, &fl4, &res, 0))
		goto last_resort;
	if (res.type != RTN_UNICAST) {
		if (res.type != RTN_LOCAL) {
			reason = SKB_DROP_REASON_IP_INVALID_SOURCE;
			goto e_inval;
		} else if (!IN_DEV_ACCEPT_LOCAL(idev)) {
			// 이 사례는 아래의 rpf 검사 전에 IP_LOCAL_SOURCE로 드롭된다.
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
	if (dev_match) {
		ret = FIB_RES_NHC(res)->nhc_scope >= RT_SCOPE_HOST;
		return ret;
	}
	if (no_addr)
		goto last_resort;
	if (rpf == 1)
		goto e_rpf;
// ...
```

---
## 📚 References

- [Linux kernel source](https://github.com/torvalds/linux)
- [Cilium source](https://github.com/cilium/cilium)
- [Cilium command cheatsheet](https://github.com/cilium/cilium/blob/main/Documentation/cheatsheet.rst)
- [Linux kernel — ftrace](https://docs.kernel.org/trace/ftrace.html)
- [Linux kernel — Event Tracing](https://docs.kernel.org/trace/events.html)
- [Tailscale Kubernetes Operator](https://tailscale.com/docs/kubernetes-operator)
