---
title: "Kubernetes: Affinity를 이용한 스케줄링"
description: 쿠버네티스에서 Affinity를 이용한 스케줄링 방법에 대해 알아보자.
date: 2025-07-29T22:28:57+09:00
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

## 😽 Affinity 개요

Affinity는 Pod들의 배치 제약을 정의하는 기능으로, 선호 및 비선호의 정도를 정한다.  

Pod의 `spec.affinity`로 값이 지정된다.   
두 가지의 조건이 있다:  
- `requiredDuringSchedulingIgnoredDuringExecution`: Pod가 반드시 정의된 선호 노드로 배치되어야 함
**(하드 제약)**
- `preferredDuringSchedulingIgnoredDuringExecution`: 가능하다면 선호 노드로 배치되지만, 자원 부족 등의 이유로 선호 노드에 배치가 될 수 없는 경우, 조건에 맞지 않는 타 노드에 배치될 수 있음.  
**(소프트 제약)**
이 경우, 조건에 weight라는 프로퍼티가 존재하는데, **숫자가 클수록 우선순위가 높은 조건** 이다.  1~100의 범위를 가진다.

---
## 🏠 Node Affinity

**어느 노드에 Pod를 할당할지 정하게 된다.**
```yaml
affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd  
```

이 예시에서, `[ssd]`에 포함되는 disktype 값을 가지는 label을 가진 노드에만 Pod가 배치되어야 한다.

---
## 👫 Pod Affinity

어떤 Pod가 놓인 노드 근처(같은 노드 또는 같은 가용 영역)에 함께 스케줄링되도록 지시할 수 있다.  
데이터 전송 지연을 줄이고, 서비스 간 통신 비용을 낮추고자 할 때 사용된다.  
**서로 다른 AZ간 통신은 곧 추가 비용이기 때문이다**
```yaml
affinity:
  podAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
            - key: app
              operator: in
              values:
                - frontend
        topologyKey: "kubernetes.io/hostname"
        # topologyKey: "failure-domain.beta.kubernetes.io/zone" # 같은 AZ
```

위 예시는 frontend와 같은 노드에서 같이 실행되기를 원한다.

---
## 💔 Pod Anti-Affinity

특정 label을 가진 Pod들과 같은 노드 또는 AZ내에 **공존하지 않게 제약** 을 걸 수 있다.  
```yaml
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions: 
              - key: role
                operator: in
                values:
                  - db
          topologyKey: "failure-domain.beta.kubernetes.io/zone"
```

위 예시는 AZ내의 `role: db` Pod와는 떨어져 배치되기를 원한다.

---
## 📚 요약

- Affinity는 Pod들의 스케줄링을 제어하는데 매우 중요하고, label을 잘 달아둬야 한다.  
Pod의 `spec.affinity`에 정의된다. 
- `requiredDuringSchedulingIgnoredDuringExecution`을 하드 제약이고,
- `preferredDuringSchedulingIgnoredDuringExecution`는 소프트 제약이다.  
- `nodeAffinity`는 노드를 정하는 데 쓰인다.  
- `podAffinity`는 주로 서비스들 간 위치를 가깝게 구성하게 하기 위해 사용된다.  
- `podAntiAffinity`는 주로 서비스들을 서로 다른 AZ에 분산시키는 용도로 사용된다.  
