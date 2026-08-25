---
title: "Linux Network - nftables"
description: "Netfilter hook 흐름과 nftables chain, conntrack, Docker bridge NAT/port forwarding 구조를 정리해보자"
date: 2026-08-26T01:49:01+09:00
image: netfilter-hook.webp
math: false
license: 
hidden: false
comments: true
draft: false

tags:
    - Linux
    - Network
    - nftables
    - Docker

categories:
    - Network
---

Linux에서 방화벽, NAT, 포트 포워딩을 이해하려면 결국 **Netfilter hook에서 패킷이 어디를 지나가는지**를 알아야 한다.
이번 글에서는 nftables의 base chain과 priority를 먼저 보고, Docker bridge network가 만드는 NAT/port forwarding 규칙을 직접 흉내 내본다.

---
## 🪝 Netfilter hooks

아래 그림은 Linux networking의 packet flow를 보여준다.
![](netfilter-hook.webp)
- Driver RX path / Driver TX path:
	- RX path:
		- RX = Receive를 의미.
		- NIC로부터 frame을 받은 뒤, NIC driver가 Linux networking stack으로 올리기까지의 과정
	- TX path:
		- TX = Transmit을 의미.
		- Linux networking stack에서 NIC driver로 보내는 과정
- Ingress
	- 들어오는 패킷이 networking stack까지 들어가기 직전의 지점
	- Ingress hook은 prerouting보다 더 빠른 단계에서 필터링 가능
		- fragmented datagram들이 재조립되기 이전에 판단됨
		- 즉, UDP의 Destination port를 매칭하는 등은 첫 fragment나 unfragmented packet에서만 가능하므로, 포트 매칭을 하기에는 부적합함
- Egress
	- 나가는 패킷이 networking stack의 처리를 마치고, network interface로 내보내기 직전
	- Ingress처럼 별도의 hook이 실행될 수 있음

그림의 여러 경우의 수를 살펴보자.
- ARP 질의를 받은 경우:
	1. RX path
	2. Ingress
	3. Bridge Port? -> No, Protocol Type? -> ARP
	4. ARP Input hook을 거침
	5. ARP Handler 처리 (응답 생성)
	6. Output hook을 거침
	7. Egress
	8. TX path
- 웹 서버가 HTTP 요청을 받은 경우:
	1. RX path
	2. Ingress
	3. Bridge Port? -> No, Protocol Type? -> IP
	4. IP Prerouting Hook
	5. Routing Decision
	6. IP Input Hook
	7. TCP socket
	8. Local Process (Web server, 아래부터는 새로운 outbound packet임)
	9. TCP socket
	10. Routing Decision
	11. IP Output Hook
	12. IP Postrouting Hook
	13. Egress
	14. TX path
- 같은 host의 Docker 기본 브릿지 네트워크에 소속한 `172.17.0.2` 컨테이너 A로부터 `172.17.0.3` 컨테이너 B로 향하는 트래픽이 온 경우
	- A는 B의 MAC 주소를 ARP를 통해 알아낸 뒤 veth로 송신
	1. RX path
	2. Ingress
	3. Bridge Port? -> Yes
	4. Prerouting Bridge
	5. Bridge Decision(dst MAC == b의 MAC)
	6. Forward Bridge
	7. Postrouting Bridge
	8. Egress
	9. TX path
- Docker의 기본 브릿지 네트워크에 소속한 컨테이너 A가 인터넷에 접속하길 원함(ex. 8.8.8.8)
	- A가 보내는 frame의 dst MAC은 docker0의 MAC
	1. RX path
	2. Ingress
	3. Bridge Port? -> Yes
	4. Bridge Prerouting
	5. Bridge Decision 
		- dst MAC == docker0 (local)
	6. Bridge INPUT
	7. IP Prerouting
	8. IP Routing Decision
		- dst = 8.8.8.8
		- default route -> eth0
	9. IP Forward
	10. IP Postrouting
		- MASQUERADE(NAT)
		- src 172.17.0.2 -> 192.168.10.10
	11. Egress(eth0)
	12. TX path

간단히 정리하면, 기본적인 플로우는 아래와 같다.
- **외부 -> Local**
  PREROUTING -> Routing -> INPUT
- **외부 -> 외부 (Router)**
  PREROUTING -> Routing -> FORWARD -> POSTROUTING
- **Local -> 외부**
  Routing -> OUTPUT -> POSTROUTING

트래픽을 포워딩시키고 싶으면, 아래 명령으로 허용한다.
```bash
echo 1 > /proc/sys/net/ipv4/ip_forward
```

iptables와 비교했을 때 nftables의 큰 차이는 **각 hook에 대응되는 기본 chain이 미리 만들어져 있지 않다**는 점이다.
따라서 직접 Netfilter hook에 연결하는 **base chain**을 만들어야 한다.

---
## 🧩 nftables hook/priority

iptables와 달리, `INPUT`, `OUTPUT` 등의 사전 정의된 chain은 없다.
대신 특정 processing step에서 패킷을 필터링하려면, base chain을 생성해서 적절한 Netfilter hook에 붙여야 한다.

### Base Chain 추가

Base chain은 Netfilter hooks에 등록되는 체인들이다.
이 체인들이 Linux의 TCP/IP stack에서 패킷들을 보고 처리할 수 있다.

기본적인 base chain 문법은 다음과 같다.
```nft
add chain [<family>] <table_name> <chain_name> { type <type> hook <hook> [device <device>] priority <priority> ; [policy <policy> ;] [comment <comment> ;]}
```

아래의 예시는 필터 테이블 ip input에 base chain을 추가하는 모습이다.
```bash
nft 'add chain ip filter input { type filter hook input priority 0; }'
```

- 이 `add chain` 명령은 input chain을 등록하고, 이 체인을 input hook에 연결한다.
	- 이 체인은 로컬 시스템 프로세스를 목적지로 하는 패킷, 즉 이 컴퓨터로 들어오는 패킷을 확인한다.
- **priority는 중요하다. chain의 실행 순서를 정하기 때문이다.**
	- Input hook에 여러 체인이 있다면, 작은 순서대로 동작한다.
	- 두 개의 base chain이 같은 priority를 가지는 것은 가능하지만, 매번 평가가 달라질 수 있어 불확실하다.
	- 여기서 `filter`는 테이블의 이름으로 쓰였을 뿐, chain의 type은 아니다.

데스크톱과 같은 포워딩하지 않는 컴퓨터에서도 output chain을 등록 가능하다.
이 output chain은 로컬 컴퓨터의 프로세스에서 생성되어 외부로 나가는 패킷을 처리한다.
```bash
nft 'add chain ip filter output { type filter hook output priority 0; }'
```

이제 incoming(로컬 프로세스로 오는) 패킷과 outgoing(로컬 프로세스에 의해 생긴) 트래픽을 필터링할 수 있다.
> **NOTE**
> 만약 중괄호(`{}`)로 감싸지지 않은 체인을 포함하면, 아무 패킷을 보지 않는 regular chain을 만드는 것이다. (`iptables -N chain-name`과 같음)

nftables 0.5부터, default policy를 추가할 수 있다:
iptables에서처럼, accept와 drop, 두 가지의 기본 정책이 가능하다.
```bash
nft 'add chain ip filter output { type filter hook output priority 0; policy accept; }'
```

chain을 ingress hook에 추가할 때는, 체인이 붙을 디바이스를 지정하는 것이 중요하다.
```bash
nft 'add chain netdev filter eth0_filter { type filter hook ingress device eth0 priority 0; }'
```

#### Base Chain의 종류
Base chain에는 세 가지가 있다:
- **filter**: 
	- 패킷을 필터링할 때 사용. 허용 및 차단 등이 가능. 
	- `arp`, `bridge`, `ip`, `ip6`, `inet` table family에서 지원된다.
- **route**: 
	- 관련 IP 헤더 필드나 packet mark가 변경되었을 때, 라우팅 경로를 다시 계산(리라우팅)할 때 사용한다.
	- `iptables`의 `mangle`과 비슷하다.
	- 대신, route 체인 타입은 output hook에서만 사용한다.
	- 다른 hook에서는 route 말고 filter를 사용해야 한다.
	- `ip`, `ip6`, `inet` 테이블 패밀리에서 지원된다.
- **nat**: 
	- NAT(Network Address Translation)를 수행할 때 사용.
	- 하나의 flow에 대해, 첫 번째 패킷만 `nat`체인을 통과한다.
	- 이후 패킷들은 거치지 않는다.
	- `ip`, `ip6`, `inet` table family에서 지원된다.

#### Base chain hook들

Base chain에서 다음의 hook들을 사용할 수 있다.
- **ingress**: NIC드라이버에서 패킷이 올라온 직후의 패킷
- **prerouting**: 시스템으로 들어온 모든 패킷을 봄. 라우팅 패킷 이전
- **input**:  로컬 시스템 및 프로세스가 목적지인 패킷을 봄
- **forward**: 이 컴퓨터가 최종 목적지가 아닌 패킷을 봄
- **output**: 로컬 프로세스가 생성한 패킷을 봄
- **postrouting**:  패킷이 시스템 밖으로 나가기 직전. 즉 forward된 것 또는 output된 것의 이후

#### Base chain 우선순위

각 nftables base chain에는 priority 값이 저장되는데, 같은 hook에 연결된 다른 base chain, flowtable, Netfilter 내부 처리 작업들과 비교해서 어떤 순서로 실행될지를 결정한다.

예를 들어, prerouting hook에 -300짜리 priority가 붙으면 connection tracking(CONNTRACK)보다 더 먼저 실행된다.
> **NOTE**
> 패킷이 accept되고 다른 체인이 있고, 그 체인이 같은 hook type에 늦은 priority를 가진다면, 다음 체인으로 계속 진행된다.
> 즉, accept는 최종 허용을 의미하지 않는다.
> 그러나, drop은 즉시 최종 결정된다.

다음의 룰셋을 보자:
```bash
table ip filter {
        # This chain is evaluated first due to priority
        chain services {
                type filter hook input priority 0; policy accept;

                # If matched, this rule will prevent any further evaluation
                tcp dport http drop

                # If matched, and despite the accept verdict, the packet proceeds to enter the chain below
                tcp dport ssh accept

                # Likewise for any packets that get this far and hit the default policy
        }

        # This chain is evaluated last due to priority
        chain input {
                type filter hook input priority 1; policy drop;
                # All ingress packets end up being dropped here!
        }
}
```

두 chain 모두 input hook에 연결되어 있다.
실행 순서는 priority가 작은 순이므로, services -> input의 순서를 가진다.

- HTTP 패킷이 들어오는 경우
	- services 체인에서 drop (`tcp dport http drop`에 의해)
- SSH 패킷이 들어오는 경우
	- services 체인에서 accept(`tcp dport ssh accept`에 의해)
	- 그러나, input 체인에서 `drop`
- 그 외 패킷
	- services에서 허용(기본 accept)
	- 그러나 input체인에서 `drop`
- 만약 chain input의 priority를 -1로 두면?
	- services에 도달하기 이전에 input에서 drop
#### Base chain 정책

더 이상 평가할 규칙이 없는 경우, 두 가지 정책이 가능하다:
- accept: 계속해서 네트워크 스택을 통과하도록 함 (기본값)
- drop: 패킷이 base chain의 끝에 도달하면 패킷을 버림

### Regular Chain 추가

다음의 문법으로 regular chain을 만들 수 있다:
```bash
add chain [family] <table_name> <chain_name> [comment <comment>]
```

Chain의 이름은 아무렇게나 가능하다.
hook keyword가 regular chain에 없음을 주목하자.

Netfilter hook에 붙지 않기에, regular chain만으로는 아무 트래픽을 보지 못한다.
그러나, 1개 이상의 base chain은 이러한 chain으로 jump 또는 goto가 가능하다.
- `jump`: 함수 호출처럼 갔다 돌아옴
- `goto`: 원래 체인으로 돌아오지 않음

즉, regular chain은 **규칙을 재사용하고 모듈화하는 데** 쓰일 수 있다.

### Chain 제거

아래 문법으로 chain을 지운다:
```bash
delete chain [family] <table_name> <chain_name>
```

조건은 지우려는 체인이 비어있어야 한다는 것이다. 그렇지 않으면 아래와 같이 지워질 수 없다.
```bash
nft 'delete chain ip filter input'
<cmdline>:1:1-28: Error: Could not delete chain: Device or resource busy
delete chain ip filter input
^^^^^^^^^^^^^^^^^^^^^^^^^^^^
```

참고로, 다른 체인에서 참조되는 regular chain의 경우 이 역시 지워질 수 없으니, 참조하고 있는 체인에서도 참조를 끊어야 한다.

### Chain 비우기

아래의 문법으로 chain의 rule을 모두 없앨 수 있다.

```bash
flush chain [family] <table_name> <chain_name>
```

---
## 🔎 conntrack

conntrack은 netfilter의 userspace cli 도구이다.
기존의 `/proc/net/ip_conntrack` 인터페이스를 대체하며, 이 도구로 Linux Kernel Netfilter의 connection tracking subsystem인 `nf_conntrack`을 검색, 리스트, 관찰, 유지할 수 있다.
현재 추적 중인 연결들을 덤프하거나, state table의 연결 상태를 지우거나, 새로운 연결을 추가할 수 있다.

```bash
# 현재 테이블 dump
conntrack -L

# 더 많은 정보와 함께 조회
conntrack -L -o extended

# port=TCP 커넥션들만 필터
conntrack -L -p tcp

# 전체 테이블 flush
conntrack -F

# 이벤트 추적
conntrack -E
```

---
## 🧵 nft monitor trace

기존의 `iptables method -J TRACE`의 개선판이라고 보면 되는 기능이다.
패킷이 nftables 안에서 어떻게 처리되는지를 추적할 수 있다.

두 가지 단계로 나뉜다:
- ruleset에서 추적하도록 켜기
- nft tool에서 이벤트를 추적

### nftrace 활성화하기

아래 룰은 패킷의 메타데이터에 `nftrace=1`을 설정한다.
```bash
meta nftrace set 1
```

트래픽이 많은 경우, 매칭 스코프를 좁히길 원할 수도 있다.
그래서 보통 조건을 붙여 필터링하는데, 아래처럼 하면 tcp만 가능해진다.
```bash
ip protocol tcp meta nftrace set 1
```

### Chain에서 tracing 켜기

tracing을 위해서는, 전용 chain을 만드는 것이 권장되는데, 가장 맨 앞에서 켜는 것이 좋다.
추적 이전의 처리 과정은 trace될 수 없기 때문이다.
앞에서 붙이면 거의 모든 일을 관찰할 수 있다.

관찰이 끝나면 **해당 chain만 지우면 된다.**

### 이벤트 트레이싱하기

아래 명령어로 추적 정보를 본다:
```bash
nft monitor trace
```


---
## 🧱 예시: 직접 NAT 만들어보기

아래의 nft는 eth1을 내부 네트워크로 연결하고, eth0을 외부 인터넷에 연결한 NAT gateway로 동작한다고 생각하면 된다.
```nginx
# nat.nft

# inet family에서 'firewall'이라는 이름의 table 생성
table inet firewall {
	# 'forward' chain: 
	# 여기서의 forward는 chain의 이름일 뿐, 의미를 가지지 않는다.
    chain forward {
		# 필터링용 체인임을 의미
		# forward지점에 연결
		# 일반적인 필터링의 위치에서 실행
		# 기본적으로 deny
        type filter hook forward priority filter; policy drop;

		# eth1 -> eth0트래픽에 대해 승인
        iifname "eth1" oifname "eth0" accept
		# eth0 -> eth1에 대해서, ct(conntrack)에서의 추적 상태가 established, related인 경우 허용
		# 주목할 점은, ct가 보는 established와 TCP state machine의 ESTABLISHED는 다르다는 것이다.
		# ct의 established 상태는 그저 현재 연결에 추적에 대한 응답이라고 보면 된다.
		# 즉, UDP 패킷의 응답도 ct에서는 established로 추적한다.
		# related는 기존 트래픽과 연결된 새로운 커넥션을 말하는데, FTP data 연결이나, ICMP error등이 있다.
        iifname "eth0" oifname "eth1" ct state related,established accept
    }

	# 'postrouting' chain:
	# 이 체인명 역시 이름이 postrouting이라는 것 뿐, 의미를 가지는 게 아님.
    chain postrouting {
		# NAT 처리 체인
		# 라우팅 연결이 끝난 뒤, 패킷이 실제로 나가기 전 단계에 연결.
		# source NAT 처리가 되는 표준 우선순위
		# 기본 승인(NAT체인은 보통 기본 승인)
        type nat hook postrouting priority srcnat; policy accept;

		# 아웃바운드 인터페이스가 `eth0`인 경우, `eth0`의 주소로 masquerade
        oifname "eth0" masquerade
    }
}
```

적용:
```bash
nft -f nat.nft
```

명령형으로 구성하면 다음과 같다:
```bash
nft add rule inet firewall postrouting oifname "eth0" masquerade
nft add rule inet firewall forward iifname "eth1" oifname "eth0" accept
nft add rule inet firewall forward \
    iifname "eth0" oifname "eth1" ct state related,established accept
```

번외로, iptables에서는 다음과 같이 작성한다:
```bash
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
iptables -A FORWARD -i eth1 -o eth0 -j ACCEPT
iptables -A FORWARD -i eth0 -o eth1 -m state --state RELATED,ESTABLISHED -j ACCEPT

```


그러나 룰만으로는 안 되고, 다음과 같이 머신에서 forwarding을 활성화해야 한다.

일시적으로 활성화:
```bash
sysctl -w net.ipv4.ip_forward=1
```

영구적 활성화:
```bash
cat <<EOF | sudo tee /etc/sysctl.d/99-router.conf > /dev/null
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
EOF

sudo sysctl --system
```

---
## 🐳 실습: Docker bridge network를 모방한 NAT + Port forward

### `docker0` 네트워크 살펴보기

SNAT의 경우는 위의 예시와 비슷하게 구성하면 되겠다.
그러면, Port forwarding, 즉 DNAT는 어떻게 처리할까?
Docker에서는 어떻게 NAT 및 Port forwarding을 처리하는지 알아보자.

아래는 `nft list ruleset`으로 조회한 결과이다:
- Docker Engine 29 기준, 아직 native nftables backend는 experimental이라, 아래는 `iptables-nft`가 만들어준 nftables 규칙이다.
- `counter packets 0 bytes 0`은 룰에 붙은 카운터 값으로, 조건이 아닌 통계 정보이다.
```nginx
# Warning: table ip nat is managed by iptables-nft, do not touch!
table ip nat {
        chain DOCKER {
				# TCP destination port가 8080이고 docker0에서 온 패킷이 아니라면,
				# 172.17.0.2:80포트로 목적지 주소 변환
                iifname != "docker0" tcp dport 8080 counter packets 0 bytes 0 dnat to 172.17.0.2:80
        }

		# DNAT는 보통 prerouting 단계에서 이루어짐
		# 그러나, 이 체인의 이름은 의미를 가지지 않음. 실제 hook이 붙는 건 base chain에서.
        chain PREROUTING {
				# base chain: default accept
                type nat hook prerouting priority dstnat; policy accept;
				# FIB(forwarding information base)를 조회하여 
				# Linux routing 관점에서 Destination IP가 호스트 자신의 주소라면
				# DOCKER chain으로 jump
                fib daddr type local counter packets 0 bytes 0 jump DOCKER
        }

        chain OUTPUT {
				# output base chain: default accept
                type nat hook output priority dstnat; policy accept;
				# destination IP가 loopback 대역이 아니고
				# FIB조회 결과 목적지가 이 호스트의 local 주소라면
				# DOCKER chain으로 jump
				# loopback이 대상인 경우는 docker-proxy가 중개
                ip daddr != 127.0.0.0/8 fib daddr type local counter packets 0 bytes 0 jump DOCKER
        }

        chain POSTROUTING {
				# Postrouting: 나가는 패킷에 대해 (SNAT): default accept
                type nat hook postrouting priority srcnat; policy accept;
				# source address가 172.17.0.0/16, 나가는 인터페이스가 docker0가 아니라면, 
				# source IP를 해당 출력 인터페이스의 IP로 SNAT(MASQUERADE)
                ip saddr 172.17.0.0/16 oifname != "docker0" counter packets 0 bytes 0 masquerade
        }
}
# Warning: table ip filter is managed by iptables-nft, do not touch!
table ip filter {
        chain DOCKER {
				# destination ip가 172.17.0.2에 docker0로부터 오는 패킷이 아니며
				# docker0으로 나가는 패킷이고, destination port가 80이면 허용
                ip daddr 172.17.0.2 iifname != "docker0" oifname "docker0" tcp dport 80 counter packets 0 bytes 0 accept
				# not docker0 -> docker0트래픽에서 앞의 허용 규칙에 매칭되지 않으면 drop
                iifname != "docker0" oifname "docker0" counter packets 0 bytes 0 drop
        }

        chain DOCKER-FORWARD {
				# 각 chain으로 jump하여 실행한 뒤, 돌아와서 다음 과정을 수행함
                counter packets 0 bytes 0 jump DOCKER-CT
                counter packets 0 bytes 0 jump DOCKER-INTERNAL
                counter packets 0 bytes 0 jump DOCKER-BRIDGE
                iifname "docker0" counter packets 0 bytes 0 accept
        }

        chain DOCKER-BRIDGE {
				# Forwarding결과, 출력 인터페이스가 docker0이면, DOCKER chain으로 jump
                oifname "docker0" counter packets 0 bytes 0 jump DOCKER
        }

        chain DOCKER-CT {
				# CONNTRACK: established또는 related의 경우, 
				# 즉, 보낸 패킷이 돌아올 때의 응답 또는 FTP data transfer 등의 경우 허용
				# 앞에서의 NAT예제에서도 사용한 규칙
                oifname "docker0" ct state related,established counter packets 0 bytes 0 accept
        }

        chain DOCKER-INTERNAL {
				# docker network create --internal로 만든 네트워크의 외부 격리 구현
        }

        chain FORWARD {
				# 모든 처리 이후 문제 없다면 accept
                type filter hook forward priority filter; policy accept;
                counter packets 0 bytes 0 jump DOCKER-USER
                counter packets 0 bytes 0 jump DOCKER-FORWARD
        }

        chain DOCKER-USER {
			# User 전용 체인
        }
}
# --- ip6 family는 생략
table ip raw {
		# Direct remote access를 차단하기 위한 체인
		# 즉, port-forwarding으로만 트래픽을 노출하게 하려는 의도
        chain PREROUTING {
                type filter hook prerouting priority raw; policy accept;
                ip daddr 172.17.0.2 iifname != "docker0" counter packets 0 bytes 0 drop
        }
}
```

아래는 Docker Engine 29 버전에서 experimental 기능인 native nftables backend를 이용했다.
iptables backend로 생성된 것과 어느 정도 유사함을 볼 수 있다.

```nginx

# /etc/docker/daemon.json에서
# { "firewall-backend": "nftables" }를 추가

table ip nat {  
	chain PREROUTING {  
		type nat hook prerouting priority dstnat; policy accept;  
	}  
  
	chain OUTPUT {  
		type nat hook output priority dstnat; policy accept;  
	}  
  
	chain POSTROUTING {  
		type nat hook postrouting priority srcnat; policy accept;  
	}  
}  

table ip filter {  
	chain FORWARD {  
		type filter hook forward priority filter; policy accept;  
		counter packets 0 bytes 0 jump DOCKER-USER  
	}  
  
	chain DOCKER-USER {  
	}  
}  

table ip raw {  
	chain PREROUTING {  
		type filter hook prerouting priority raw; policy accept;  
	}  
}  

table ip docker-bridges {  
	map filter-forward-in-jumps {  
		type ifname : verdict  
		elements = { "docker0" : jump filter-forward-in__docker0 }  
	}  
  
	map filter-forward-out-jumps {  
		type ifname : verdict  
		elements = { "docker0" : jump filter-forward-out__docker0 }  
	}  
  
	map nat-postrouting-in-jumps {  
		type ifname : verdict  
		elements = { "docker0" : jump nat-postrouting-in__docker0 }  
	}  
  
	map nat-postrouting-out-jumps {  
		type ifname : verdict  
		elements = { "docker0" : jump nat-postrouting-out__docker0 }  
	}  
  
	chain filter-FORWARD {  
		type filter hook forward priority filter; policy accept;  
		oifname vmap @filter-forward-in-jumps  
		iifname vmap @filter-forward-out-jumps  
	}  
  
	chain nat-OUTPUT {  
		type nat hook output priority dstnat; policy accept;  
		ip daddr != 127.0.0.0/8 fib daddr type local counter packets 0 bytes 0 jump nat-prerouting-and-output  
	}  
  
	chain nat-POSTROUTING {  
		type nat hook postrouting priority srcnat; policy accept;  
		iifname vmap @nat-postrouting-out-jumps  
		oifname vmap @nat-postrouting-in-jumps  
	}  
  
	chain nat-PREROUTING {  
		type nat hook prerouting priority dstnat; policy accept;  
		fib daddr type local counter packets 0 bytes 0 jump nat-prerouting-and-output  
	}  
  
	chain nat-prerouting-and-output {  
		iifname != "docker0" tcp dport 8080 counter packets 0 bytes 0 dnat to 172.17.0.2:80 comment "DNAT"  
	}  
  
	chain raw-PREROUTING {  
		type filter hook prerouting priority raw; policy accept;  
		ip daddr 172.17.0.2 iifname != "docker0" counter packets 0 bytes 0 drop comment "DROP DIRECT ACCESS"  
	}  
  
	chain filter-forward-in__docker0 {  
		ct state established,related counter packets 0 bytes 0 accept  
		iifname "docker0" counter packets 0 bytes 0 accept comment "ICC"  
		ip daddr 172.17.0.2 tcp dport 80 counter packets 0 bytes 0 accept  
		counter packets 0 bytes 0 drop comment "UNPUBLISHED PORT DROP"  
	}  
  
	chain filter-forward-out__docker0 {  
		ct state established,related counter packets 0 bytes 0 accept  
		counter packets 0 bytes 0 accept comment "OUTGOING"  
	}  
  
	chain nat-postrouting-in__docker0 {  
	}  
  
	chain nat-postrouting-out__docker0 {  
		oifname != "docker0" ip saddr 172.17.0.0/16 counter packets 0 bytes 0 masquerade comment "MASQUERADE"  
		}  
}
```


`docker-proxy`라는 프로세스가 포트 포워딩하는 포트를 점유함을 볼 수 있다.
```bash
root@ubuntu:~# docker run -d --name nginx -p 8080:80 nginx
8e241826e0620f38049fb96f5c4c5781c89ff3acd8e61545953310aec4cc5479 
root@ubuntu:~# ss -ntlp 
State Recv-Q Send-Q Local Address:Port Peer Address:Port Process 
...
LISTEN 0 4096 0.0.0.0:8080 0.0.0.0:* users:(("docker-proxy",pid=11562,fd=8)) 
...
```

하나 주목할 점은, 방화벽 룰에서는 **loopback 주소는 처리하지 않는다**는 것이다.
`ss`로 보았듯이, docker-proxy가 중개해준다.
그 이유는, loopback, 과거 커널/네트워크 호환 등의 문제를 kernel NAT만으로는 해결하기 힘들어서, 여전히 docker-proxy를 이용한다고 한다.

그러나 현대 Linux routing에서는 hairpin 상황을 처리할 수 있어졌다.
`route_localnet`을 활성화해주면 된다.
`userland-proxy`를 비활성화해보자.
```json
# /etc/docker/daemon.json
{
  "userland-proxy": false
}
```

적용을 위해 재실행해준다.
```bash
systemctl restart docker
```

`docker ps`를 해보면, IPv6 포트 포워딩은 안 되는 것을 볼 수 있다.
`[::1]:8080->80`과 같은 IPv6 -> IPv4 변환은 `route_localnet`으로도 여전히 한계가 있다고 한다.

```bash
root@cp-1:~# docker ps 
CONTAINER ID IMAGE COMMAND CREATED STATUS PORTS NAMES 
618abe9aff42 nginx "/docker-entrypoint.…" 11 seconds ago Up 11 seconds 0.0.0.0:8080->80/tcp nginx
```

`ss` 명령어에서는, `0.0.0.0:8080`을 다른 프로세스가 처리하지 못하게 dockerd가 reserve한다.
여기서도 `[::1]:8080`은 점유하지 않는 것을 볼 수 있다.

```bash
root@cp-1:~# ss -ntlp 
State Recv-Q Send-Q Local Address:Port Peer Address:Port Process 
LISTEN 0 4096 0.0.0.0:8080 0.0.0.0:* users:(("dockerd",pid=19866,fd=30)) 
```

docker0의 `route_localnet`이 `1`로 활성화되었다.
이제 `127.0.0.0/8` 대역을 대상으로 하는 트래픽도 라우팅 가능해진다.
```bash
root@cp-1:~# sysctl -a | grep docker0.route_localnet 
net.ipv4.conf.docker0.route_localnet = 1
```

`nft list ruleset`으로 다시 규칙들을 확인해보자.
```nginx
table inet nat {  
	# ...
	chain DOCKER { 
		tcp dport 8080 counter packets 0 bytes 0 dnat to 172.17.0.2:80 
	}
	
	chain OUTPUT {  
		type nat hook output priority dstnat; policy accept;  
		# daddr != 127.0.0.1/8조건이 사라져있다,
		fib daddr type local counter packets 0 bytes 0 jump DOCKER  
	}  
	
	chain POSTROUTING {
		type nat hook postrouting priority srcnat; policy accept; 
		# 추가: host -> host port -> container로 연결되도록 함. 
		# src = 127.0.0.1, dst = 127.0.0.1:8080에서
		# OUTPUT DNAT를 거치고, src = 127.0.0.1, dst = 172.17.0.2:80으로 바뀐 이후
		# 이 rule을 만나서 MASQUERADE되어
		# src = 172.17.0.1, dst = 172.17.0.2:80으로 변환.
		oifname "docker0" fib saddr type local counter packets 0 bytes 0 masquerade 
		ip saddr 172.17.0.0/16 oifname != "docker0" counter packets 0 bytes 0 masquerade 
		# 추가: 컨테이너 -> host port -> 자기 자신으로 통하도록 
		# src = 172.17.0.2, dst = HostIP:8080으로 하는 경우
		# OUTPUT에 의해  src = 172.17.0.2 dst = 172.17.0.2:80
		# 이는 Host의 conntrack/NAT경로를 정상적으로 돌아오지 못할 수 있음.
		# 그래서 이 rule에서는 다음처럼 바꿈:
		# src = 172.17.0.1, dst = 172.17.0.2:80
		ip saddr 172.17.0.2 ip daddr 172.17.0.2 tcp dport 80 counter packets 0 bytes 0 masquerade
	}
	
	# ...
}  
# ...
```

우리는 **userland-proxy를 비활성화한 버전**을 따라해볼 것이다.

### 네트워크 토폴로지

`ns1`이라는 netns를 만들고, veth pair를 생성해 연결한다.
Docker와 유사한 토폴로지를 만들어준다.
```bash
ip link add mydocker0 type bridge
ip addr add 172.18.0.1/16 dev mydocker0 broadcast 172.18.255.255
ip link set mydocker0 up

ip link add veth-host1 type veth peer name veth-ns1

ip link set veth-host1 master mydocker0
ip link set veth-host1 up
ip netns add ns1
ip link set veth-ns1 netns ns1
ip netns exec ns1 ip addr add 172.18.0.2/16 dev veth-ns1 broadcast 172.18.255.255
ip netns exec ns1 ip link set veth-ns1 up
ip netns exec ns1 ip link set lo up
ip netns exec ns1 ip route add default via 172.18.0.1
ip netns exec ns1 nginx

# 선택: netns를 하나 더 만들기
ip link add veth-host2 type veth peer name veth-ns2

ip link set veth-host2 master mydocker0
ip link set veth-host2 up

ip netns add ns2
ip link set veth-ns2 netns ns2
ip netns exec ns2 ip addr add 172.18.0.3/16 dev veth-ns2 broadcast 172.18.255.255
ip netns exec ns2 ip link set veth-ns2 up
ip netns exec ns2 ip link set lo up
ip netns exec ns2 ip route add default via 172.18.0.1
ip netns exec ns2 nginx
```

### 포워딩 및 local route 활성화
커널 파라미터도 IPv4 포워딩 및 local route가 가능하게 만들어준다.
```bash
sysctl -w net.ipv4.ip_forward=1
sysctl -w net.ipv4.conf.mydocker0.route_localnet=1
```

영구적으로 원한다면:
```bash
cat <<EOF | sudo tee /etc/sysctl.d/99-router.conf > /dev/null
net.ipv4.ip_forward = 1
net.ipv4.conf.mydocker0.route_localnet = 1
EOF

sudo sysctl --system
```

### NAT를 지원하도록 nft 설정하기

아래처럼 규칙을 만들면 된다.
```nginx
table ip mynat {
    chain raw_prerouting {
        type filter hook prerouting priority raw; policy accept;

        # 외부에서 container IP 직접 routing 차단
        ip daddr { 172.18.0.2, 172.18.0.3 } \
            iifname != "mydocker0" \
            drop
    }

    chain prerouting {
        type nat hook prerouting priority dstnat; policy accept;

        fib daddr type local tcp dport 8080 \
            dnat to 172.18.0.2:80

        fib daddr type local tcp dport 8081 \
            dnat to 172.18.0.3:80
    }

    chain output {
        type nat hook output priority dstnat; policy accept;

        fib daddr type local tcp dport 8080 \
            dnat to 172.18.0.2:80

        fib daddr type local tcp dport 8081 \
            dnat to 172.18.0.3:80
    }

    chain forward {
        type filter hook forward priority filter; policy drop;

        ct state invalid drop

        # container 쪽으로 돌아가는 응답
        oifname "mydocker0" ct state established,related accept

        # container-originated forwarding
        iifname "mydocker0" accept

        # published port를 통해 DNAT된 신규 연결
        oifname "mydocker0" ct status dnat accept
    }

    chain postrouting {
        type nat hook postrouting priority srcnat; policy accept;

        # Host -> published port -> container
        oifname "mydocker0" \
            fib saddr type local \
            masquerade

        # Container -> published port -> same bridge
        iifname "mydocker0" \
            oifname "mydocker0" \
            ct status dnat \
            masquerade

        # Container -> outside
        ip saddr 172.18.0.0/24 \
            oifname != "mydocker0" \
            masquerade
    }
}
```

### 테스트

`curl`을 해보자. `ns1`의 nginx로부터 응답을 받는다.
**DNAT가 잘 되는 것을 볼 수 있다.**
```bash
# localhost로
curl localhost:8080

# Host가 가진 다른 IP로
curl 192.168.10.101:8080
```

`ns1`에서 인터넷에 연결할 수 있는지도 확인해보자.
```bash
ip netns exec ns1 ping 8.8.8.8
```

`conntrack`으로 추적도 해보자.
```bash
root@ubuntu:~# conntrack -L -p tcp
tcp 6 8 TIME_WAIT 
	# 원래 방향 tuple
	src=127.0.0.1 dst=127.0.0.1 sport=60712 dport=8080 
	# 응답 방향 tuple
	src=172.18.0.2 dst=172.18.0.1 sport=80 dport=60712
	[ASSURED]
```
---
## 📚 References

- https://wiki.nftables.org/wiki-nftables/index.php/Netfilter_hooks
- https://wiki.nftables.org/wiki-nftables/index.php/Configuring_chains
- https://conntrack-tools.netfilter.org/manual.html
- https://man.archlinux.org/man/conntrack.8.en
- https://kernel-internals.org/net/conntrack/
- https://wiki.nftables.org/wiki-nftables/index.php/Ruleset_debug/tracing
- https://docs.kernel.org/networking/ip-sysctl.html
- https://docs.kernel.org/networking/kapi.html
- https://docs.kernel.org/networking/nf_flowtable.html
- https://wiki.nftables.org/wiki-nftables/index.php/Quick_reference-nftables_in_10_minutes
- https://docs.docker.com/engine/network/firewall-iptables
- https://docs.docker.com/engine/network/firewall-nftables/
- https://docs.docker.com/reference/cli/dockerd
- https://github.com/moby/moby/issues/45629
