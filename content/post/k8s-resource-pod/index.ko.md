---
title: "Kubernetes 리소스: Pod"
description: Kubernetes의 가장 작은 실행 단위인 Pod에 대해 알아보자  
date: 2025-07-16T00:57:50+09:00
image: "k8s-logo.webp"
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

## 📦 Pod란

Pod(파드)는 쿠버네티스 애플리케이션의 가장 작은 기본 실행 단위이다.   
Pod에서 쓰이는 대표적인 컨테이너 런타임은 Containerd이다.  

파드 당 1개 이상의 컨테이너를 가질 수 있지만, 단일 책임 원칙을 따르는 것이 좋으므로, 1:1이 관례이지만, 파드 당 여러개의 컨테이너가 포함되는 경우가 있다.  
(ex. 사이드카로 로그 포워더가 함꼐 동작)   

Pod는 원자적인 단위로, 오로지 하나의 노드에만 존재한다. 노드를 걸쳐서 존재할 수 없다.  
파드는 고유의 IP를 가지며, 동일 Pod내의 컨테이너들은 네트워크 및 볼륨을 공유할 수 있다.  
Pod들 간에는 같은 flat Network를 구성하여, 서로 통신이 가능하다.  
Pod는 마치 하나의 논리적 호스트로 동작한다고 생각하면 된다.  

---
## 📝 YAML로 Pod 생성하기
`kubectl run`으로 Pod를 생성할 수 있지만, 자원을 선언적으로 관리하기 위해 주로 YAML파일을 이용한다.  

```YAML
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels: 
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: busybox
    command: ['sh', '-c', 'echo Hello Kubernetes! && sleep infinity']
```

위 마니페스트는 `busybox`라는 Unix 시스템에서 많이 쓰이는 명령어들이 담긴 컨테이너를 담은 Pod를 선언하는데, echo로 메시지를 출력하고 무한히 잠든다.  

- `apiVersion`: 리소스를 정의할 때 사용되는 API그룹과 버전을 나타내는 필수 필드.  
    - `apiVersion`: `<APIGroup>/<verion>`의 꼴로 작성되지만, core API의 경우, 그룹을 명시하지 않아도 된다.  
    - 버전에는 3가지 종류가 있다: 
        - Alpha: 기본적으로 비활성화 되어있고, 버그가 있을 수 있는 불안정한 버전이다.  다음 릴리즈에서 삭제 또는 변경이 얼마든지 이루어질 수 있다. (ex. v1aplha1)
        - Beta: 기본적으로 활성화 되어있다. 충분한 테스트를 거쳐 `Alpha`보다 상대적으로 안전하다. (ex. v2beta3)
        - Stable: 가장 안정된 버전이다. vX와 같이 표기된다. (ex. v1)
- `kind`: 리소스의 타입을 명시
- `metadata`: 이름이나 레이블 등의 메타데이터를 명시
- `spec`: 자원의 스펙을 명시(Pod의 경우 내부 컨테이너들)


아래 명령어로 적용할 수 있다:
```bash
kubectl apply -f <yaml_file>.yaml
```

`apply` 명렁은 마니페스트를 적용하는데 사용되고, 
`create`는 최초 생성에만 사용될 수 있다.

---
## ⌨️ Pod관련 기본 명령어


```bash
# Pod 조회
kubectl get po # pod, pods도 가능

# 상세 정보 출력
kubectl get po -o wide # option: wide

# label까지 출력
kubectl get po --show-labels

# CLI를 이용한 POD 생성
kubectl run <POD-NAME> --image=<IMAGE-NAME> --port=<SERVICE-PORT>

# 로그 보기
kubectl log <POD-NAME>

# Pod에서 정보를 yaml로 보기
kubectl get pod <POD-NAME> -o yaml

# 네임스페이스에서 거의 모든 리소스 삭제
kubectl delete all --all
```


---
## 🏁 요약
Pod는 쿠버네티스에서 가장 작고 원자적인 실행 단위이며, 1개 이상의 컨테이너를 가진다.  
각 Pod는 자신의 IP주소를 가지고, Pod 내의 컨테이너들은 같은 네트워크 및 볼륨을 공유할 수 있다.  
`YAML`파일에서 Pod의 `spec`은 컨테이너들이다.
