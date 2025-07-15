---
title: "로컬 쿠버네티스 구성 및 쿠버네티스 기본 제어 가이드: kubectl, kubectx, kubens"
description: "kind기반의 로컬 클러스터 생성 및 kubectl 기본적인 명령어 사용법"
date: 2025-07-15T00:28:57+09:00
image: k8s-logo.webp
math: 
license: 
hidden: false
comments: true
draft: false

tags:
    - Kubernetes
    - Container
categories:
    - Kubernetes
---

`kubectl`은 kube-apiserver에 api 요청을 전송하기 위한 클라이언트 프로그램이다.  
`kind`는 로컬 컴퓨터에서 쿠버네티스 클러스터를 쉽고 빠르게 구성하게 해주는 도구이다.  
`kubectx`는 `kubectl`이 어느 클러스터에 접속할지, 어느 사용자와 네임스페이스를 사용할지에 대한 context를 쉽고 빠르게 전환할 수 있도록 돕는다.  
`kubens`는 네임스페이스의 전환을 돕는 도구이다.  

---
## 🏗️ 설치

[kubectl, kind설치](https://kubernetes.io/ko/docs/tasks/tools/)
kubectx, kubens 설치
```bash
git clone https://github.com/ahmetb/kubectx /opt/kubectx && ln -s /opt/kubectx/kubectx /usr/local/bin/kubectx && ln -s /opt/kubectx/kubens /usr/local/bin/kubens
```

## 🐳 kind 구성

`kind`설치가 완료되었다면, `kind`에 실제 클러스터를 만들어주자.  
아래 yaml을 작성해주면 된다.  

```yaml

kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: cwave-cluster
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  image: kindest/node:v1.32.5
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
- role: worker
  image: kindest/node:v1.32.5
- role: worker
  image: kindest/node:v1.32.5
networking:
  serviceSubnet: "10.120.0.0/16"
  podSubnet: "10.110.0.0/16"
```

이 yaml은 2개의 노드를 가진다.
아래 명령어를 통해서 워커 노드를 2개 생성해주자.
```bash
kubectl apply -f <yaml_file>
```

노드들의 리스트를 가져와보자. 
```bash
kubectl get no
```

## 🛠️ 명령어 기본 사용 가이드

### kubectl
`kubectl`의 기본 명령은 다음과 같다:
```bash
kubectl [command] [TYPE] [NAME] [flags]
```

- `command`: 하나 이상의 리소스에서 수행하려는 동작을 지정, 주로 동사이다.
    - `apply`
    - `get`
    - `create`
    - `describe`
    - `delete`
    - 등등..
- `TYPE`: 리소스 타입을 지정한다. 단수형, 복수형, 약어 모두 지원한다.
    - `po`
    - `svc`
    - `deploy`
    - `rs`
    - `sts`
    - `no`
    - `ns`
    - `pv`
    - `pvc`
    - `ds`
    - 등등..
- `NAME`: 여러 개의 리소스를 지정하여 작업할 수 있다. 
- `flags: 여러 개의 선택적 플래그를 지정할 수 있다.  `


`kubectl`의 기본 설정 파일은 `~/.kube/config`에 존재한다.  
여기서 API 엔드포인트를 정할 수 있다.

### kubectx, kubens

`kubectx`는 쿠버네티스의 context를 쉽게 전환시켜 준다.   
하나의 `kubectl`로 여러 클러스터를 넘나들기 편하게 해준다.  

`kubectx`로 리스트를 불러올 수 있고, 현재 상태도 확인 가능하다.  
`kubectx <context>`로 컨텍스트 전환이 가능하다.


`kubens`는 namespace 전환을 쉽게 해준다.  
`kubens`로 리스트를 볼 수 있고, `kubens <namespace>`로 쉽게 네임스페이스 전환이 가능하다.  
