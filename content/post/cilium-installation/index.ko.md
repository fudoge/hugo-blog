---
title: "Cilium - 클러스터에 설치하기"
description: 
date: 2025-09-22T23:29:21+09:00
image: cilium-logo.png
math: 
license: 
hidden: false
comments: true
draft: false
---

이번에는 Cilium을 쿠버네티스에 설치할 것이다.  
여기서는 Cilium 1.18.1, Kubernetes 1.34를 사용한다

---
## ℹ️ 설치 방법

설치 방법에는 두 가지가 있다:

1. **Cilium CLI tool**  
단순한 설치가 가능하다.
2. **Helm Chart**  
더 디테일한 환경 설정도 가능하다.  
 [Cilium documentation](https://docs.cilium.io/en/stable/installation/k8s-install-helm/#installation-using-helm)을 참고해서 설치 할 수 있다.

우선 **Cilium CLI Tool** 을 이용해서 설치한다.

---
## 🧾 요구사항

외부 CNI가 설치되기를 위해서, 쿠버네티스 클러스터가 준비되어있어야 한다.  
`kubectl`역시 필요하다.

아래 섹션을 통해 kind환경에서 간단하게 할 수 있지만, [이 레포지토리](https://github.com/fudoge/ansible-kubeadm)를 참고하면, `ansible`을 이용해서 쉽고 빠르게 쿠버네티스를 구성할 수 있다.

---
## 🐳 Kind세팅
만약, 별도 클러스터가 있다면, 건너뛰어도 좋다.   
여기서는 간단하게 `Kind`로 쿠버네티스를 구성하는 법을 설명한다.

요구사항:

- [Docker설치](https://docs.docker.com/engine/install/ubuntu/)
- [kind설치](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)

아래는 3-노드에 기본 CNI가 비활성화된 kind클러스터 yaml이다.

```yaml
# kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
networking:
  disableDefaultCNI: true
```

```bash
kind create cluster --config=kind-config.yaml
```

kind는 클러스터를 만들고, `kubectl` 컨텍스트에 자동으로 컨텍스트를 추가한다.  
컨텍스트를 확인해 보자. `kind-kind`라고 나오면 성공.

```bash
kubectl config current-context
```

아직은 CNI가 없기에, `NotReady`상태일 것이다.
```bash
kubectl get nodes
```

---
## 💻 Cilium CLI 설치

[Cilium CLI를 로컬에 설치](https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/#install-the-cilium-cli)한다.

```bash
18:36:03 in ~ …
➜ cilium version
cilium-cli: v0.18.7 compiled with go1.25.0 on linux/amd64
cilium image (default): v1.18.1
cilium image (stable): v1.18.1
cilium image (running): unknown. Unable to obtain cilium version. Reason: release: not found
```

---
## 🚀 Cilium 설치
이제 CLI로 Cilium을 설치할 수 있다:
```bash
18:36:08 in ~ …
➜ cilium install
ℹ️  Using Cilium version 1.18.1
🔮 Auto-detected cluster name: kubernetes
🔮 Auto-detected kube-proxy has been installed

```

설치되는 데에 시간이 좀 걸린다. `cilium status --wait`으로 설치될 때까지 기다려보자.

```bash
18:37:49 in ~ …
➜ cilium status --wait
    /¯¯\
 /¯¯\__/¯¯\    Cilium:             OK
 \__/¯¯\__/    Operator:           OK
 /¯¯\__/¯¯\    Envoy DaemonSet:    OK
 \__/¯¯\__/    Hubble Relay:       disabled
    \__/       ClusterMesh:        disabled

DaemonSet              cilium                   Desired: 1, Ready: 1/1, Available: 1/1
DaemonSet              cilium-envoy             Desired: 1, Ready: 1/1, Available: 1/1
Deployment             cilium-operator          Desired: 1, Ready: 1/1, Available: 1/1
Containers:            cilium                   Running: 1
                       cilium-envoy             Running: 1
                       cilium-operator          Running: 1
                       clustermesh-apiserver
                       hubble-relay
Cluster Pods:          2/2 managed by Cilium
Helm chart version:    1.18.1
Image versions         cilium             quay.io/cilium/cilium:v1.18.1@sha256:65ab17c052d8758b2ad157ce766285e04173722df59bdee1ea6d5fda7149f0e9: 1
                       cilium-envoy       quay.io/cilium/cilium-envoy:v1.34.4-1754895458-68cffdfa568b6b226d70a7ef81fc65dda3b890bf@sha256:247e908700012f7ef56f75908f8c965215c26a27762f296068645eb55450bda2: 1
                       cilium-operator    quay.io/cilium/operator-generic:v1.18.1@sha256:97f4553afa443465bdfbc1cc4927c93f16ac5d78e4dd2706736e7395382201bc: 1

18:38:32 in ~ took 35.0s …
➜
```

**Hubble** 을 미리 활성화해 놓자. 나중에 쓸 것이다.
이 명령어는 Hubble을 위해 Cilium 에이전트를 재구성하고 재실행한다.  
또한 클러스터 전역에 네트워크 관찰성을 위해 Hubble 컴포넌트를 활성화한다.
```bash
cilium hubble enable --ui
```

`cilium status`명령으로 Hubble 컴포넌트를 확인할 수 있다:
```bash
➜ cilium status
    /¯¯\
 /¯¯\__/¯¯\    Cilium:             OK
 \__/¯¯\__/    Operator:           OK
 /¯¯\__/¯¯\    Envoy DaemonSet:    OK
 \__/¯¯\__/    Hubble Relay:       OK
    \__/       ClusterMesh:        disabled

DaemonSet              cilium                   Desired: 2, Ready: 2/2, Available: 2/2
DaemonSet              cilium-envoy             Desired: 2, Ready: 2/2, Available: 2/2
Deployment             cilium-operator          Desired: 1, Ready: 1/1, Available: 1/1
Deployment             hubble-relay             Desired: 1, Ready: 1/1, Available: 1/1
Deployment             hubble-ui                Desired: 1, Ready: 1/1, Available: 1/1
Containers:            cilium                   Running: 2
                       cilium-envoy             Running: 2
                       cilium-operator          Running: 1
                       clustermesh-apiserver
                       hubble-relay             Running: 1
                       hubble-ui                Running: 1
Cluster Pods:          7/7 managed by Cilium
Helm chart version:    1.18.1
Image versions         cilium             quay.io/cilium/cilium:v1.18.1@sha256:65ab17c052d8758b2ad157ce766285e04173722df59bdee1ea6d5fda7149f0e9: 2
                       cilium-envoy       quay.io/cilium/cilium-envoy:v1.34.4-1754895458-68cffdfa568b6b226d70a7ef81fc65dda3b890bf@sha256:247e908700012f7ef56f75908f8c965215c26a27762f296068645eb55450bda2: 2
                       cilium-operator    quay.io/cilium/operator-generic:v1.18.1@sha256:97f4553afa443465bdfbc1cc4927c93f16ac5d78e4dd2706736e7395382201bc: 1
                       hubble-relay       quay.io/cilium/hubble-relay:v1.18.1@sha256:7e2fd4877387c7e112689db7c2b153a4d5c77d125b8d50d472dbe81fc1b139b0: 1
                       hubble-ui          quay.io/cilium/hubble-ui-backend:v0.13.2@sha256:a034b7e98e6ea796ed26df8f4e71f83fc16465a19d166eff67a03b822c0bfa15: 1
                       hubble-ui          quay.io/cilium/hubble-ui:v0.13.2@sha256:9e37c1296b802830834cc87342a9182ccbb71ffebb711971e849221bd9d59392: 1

```

---
## ✅ Cilium 운영 검증

Cilium CLI도구는 또한 연결성 테스트를 특정 쿠버네티스 네임스페이스 내에서 할 수 있다.

```bash
19:01:23 in ~ …
➜ cilium connectivity test --request-timeout 30s --connect-timeout 10s
✨ [kubernetes] Deploying client3 deployment...
✨ [kubernetes] Deploying echo-other-node service...
✨ [kubernetes] Deploying other-node deployment...
✨ [host-netns] Deploying kubernetes daemonset...
✨ [host-netns-non-cilium] Deploying kubernetes daemonset...
🔭 Enabling Hubble telescope...
⚠️  Unable to contact Hubble Relay, disabling Hubble telescope and flow validation: rpc error: code = Unavailable desc = connection error: desc = "transport: Error while dialing: dial tcp 127.0.0.1:4245: connect: connection refused"
ℹ️  Expose Relay locally with:
   cilium hubble enable
   cilium hubble port-forward&
ℹ️  Cilium version: 1.18.1
🏃[cilium-test-1] Running 123 tests ...

✅ [cilium-test-1] All 77 tests (662 actions) successful, 46 tests skipped, 1 scenarios skipped.

```

연결 테스트에는 수많은 테스트들이 있다.  
새로운 버전의 Cilium을 받는다면, 기본 연결 테스트의 수가 더 늘어난 형태일 수도 있다.  
처음에는 많은 대기시간이 필요하다. 실행을 위해 컨테이너 이미지도 받아오는 등의 초기 세팅과정이 들어있기 때문이다.  
최소 10분은 예상하고 시작해야 한다.

> **테스트를 하려면..**  
> 연결 테스트는 최소 두 개 이상의 워커 노드를 요구한다.   
> 연결 테스트 파드는 control-plane에서 열리지 않는다.  

---
## ☸️ kubectl로 조회하기

Cilium이 설치되면, `kubectl`로 노드들이 준비된 것을 확인할 수 있다.

```bash
18:46:50 in ~ …
➜ kubectl get nodes
NAME      STATUS   ROLES           AGE   VERSION
master    Ready    control-plane   51m   v1.34.1
worker1   Ready    <none>          15m   v1.34.1

19:02:18 in ~ …
➜ kubectl get daemonsets -A
NAMESPACE       NAME                    DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR                AGE
cilium-test-1   host-netns              2         2         2       2            2           <none>                       55s
cilium-test-1   host-netns-non-cilium   0         0         0       0            0           cilium.io/no-schedule=true   55s
kube-system     cilium                  2         2         2       2            2           kubernetes.io/os=linux       25m
kube-system     cilium-envoy            2         2         2       2            2           kubernetes.io/os=linux       25m
kube-system     kube-proxy              2         2         2       2            2           kubernetes.io/os=linux       51m

19:02:26 in ~ …
➜ kubectl get deployments -A
NAMESPACE       NAME              READY   UP-TO-DATE   AVAILABLE   AGE
cilium-test-1   client            1/1     1            1           16m
cilium-test-1   client2           1/1     1            1           16m
cilium-test-1   client3           1/1     1            1           64s
cilium-test-1   echo-other-node   1/1     1            1           64s
cilium-test-1   echo-same-node    1/1     1            1           16m
kube-system     cilium-operator   1/1     1            1           25m
kube-system     coredns           2/2     2            2           51m
kube-system     hubble-relay      1/1     1            1           23m
kube-system     hubble-ui         1/1     1            1           23m
```

성공적으로 Cilium이 설치된 것을 확인할 수 있다!  
