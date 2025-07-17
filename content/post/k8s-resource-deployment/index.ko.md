---
title: "Kubernetes 리소스: Deployment"
description: 실제로 Pod를 띄울 때 사용되는 Deployment에 대해 알아보자
date: 2025-07-17T13:43:05+09:00
image: k8s-logo.webp
math: 
license: 
hidden: false
comments: true
draft: true

tags:
    - Kubernetes
    - Container
categories:
    - Kubernetes
---

## Deployment
**Deployment** 는 Pod 및 ReplicaSet에 대한 **선언적 업데이트**를 지원하며, 그 배포에 대한 세분화된 기능을 제공하는 쿠버네티스 오브젝트이다.  
Deployment에 상태를 선언하면, 그 상태를 따라가기 위해 Deployment Controller가 동작하여 상태를 맞춘다.  

전반적인 구조는 아래와 같다:
```plaintext
Deployment
|
ㄴ ReplicaSet
   |
   ㅏ Pod
   | ...
   ㄴ Pod
```

## 배포 전략
Deployment에 대해 더 알아보기 전에, 배포 전략에 어떤 것들이 있는지에 대해 알아본다.

### Recreate
모든 Pod들을 내리고, 새로운 버전의 Pod들을 일괄적으로 여는 전략이다.  

장점:
- 설정이 단순하다

단점: 
- 서비스에 미치는 영향이 크다(모두 중단 후 다시 시작하기까지  잠시동안의 다운타임이 있다)

### Ramped
Incremental 또는 Rolling Update라고도 불린다.  
순차적으로 Pod들을 교체한다. 



### Blue/Green

### Canary

### A/B Testing

### Shadow

## 요약


