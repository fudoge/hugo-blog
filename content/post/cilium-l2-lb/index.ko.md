---
title: "Cilium: L2 LoadBalancer를 노출해보자"
description: Cilium에서 L2 Announcement를 이용하여 LoadBalancer를 꺼내보자.
date: 2025-11-01T14:52:39+09:00
image: cilium-logo.png
math: 
license: 
hidden: false
comments: true
draft: false

tags:
    - Kubernetes
    - Cilium
categories:
    - Cilium
---

온프레미스 환경에서 L2 브로드캐스팅을 이용해서 로드밸런서를 만들어보자.

---
## 💾 설치
아래의 helm values들이 필요하다:
```YAML
installCRDs: true

l2announcements:
	enabled: true
```

```Bash
helm upgrade cilium cilium/cilium \
	-n kube-system \
	-f values.yaml
```

---
## 🏊 IP풀 및 정책 생성
우선, IP할당 풀을 만들어준다.
`cilium-ip-pool.yaml`
```YAML
apiVersion: "cilium.io/v2alpha1"
kind: CiliumLoadBalancerIPPool
metadata:
	name: "local-nw-pool"
spec:
	blocks:
		- cidr: "192.168.0.240/29" # 공유기 DHCP대역과 겹치지 않는것이 좋다
```

L2광고정책도 생성해준다.
`cilium-l2-announcement.yaml`
```YAML
apiVersion: "cilium.io/v2alpha1"
kind: CiliumL2AnnouncementPolicy
metadata:
  name: local-l2
spec:
  nodeSelector:
    matchLabels:
      name: "worker1" # 이렇게 하면, name=worker1을 가진 노드에서 광고한다.
  interfaces:
     - enp6s0 # 본인 네트워크 인터페이스로
  externalIPs: true
  loadBalancerIPs: true
```

---
## ✅ 테스트
`test.yaml`
```YAML
# https://kubernetes.io/docs/concepts/workloads/pods/
apiVersion: v1
kind: Pod
metadata:
  name: "nginx"
  namespace: default
  labels:
    app: "nginx"
spec:
  containers:
  - name: nginx
    image: "nginx:latest"
    resources:
      limits:
        cpu: 200m
        memory: 500Mi
      requests:
        cpu: 100m
        memory: 200Mi
    ports:
    - containerPort: 80
      name: http
  restartPolicy: Always
---
# https://kubernetes.io/docs/concepts/services-networking/service/
apiVersion: v1
kind: Service
metadata:
  name: nginx
  namespace: default
spec:
  selector:
    app: nginx
  type: LoadBalancer
  ports:
  - name: nginx
    protocol: TCP
    port: 80
---
```

External-IP가 나온 것을 볼 수 있다.
```Bash
16:12:28 in ~/cilium-lab/helm_install took 4.7s …
➜ kubectl get svc
NAME         TYPE           CLUSTER-IP       EXTERNAL-IP    PORT(S)        AGE
kubernetes   ClusterIP      10.96.0.1        <none>         443/TCP        21d
nginx        LoadBalancer   10.102.139.71    192.168.0.32   80:31861/TCP   21h
```

결과를 확인해보자:
```Bash
curl 192.168.0.32/index.html
```

> 주의
> L2 announcement는 한 service당 하나의 노드에만 붙을 수 있다.