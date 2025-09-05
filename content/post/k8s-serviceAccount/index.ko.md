---
title: "Kubernetes: ServiceAccount"
description:  ServiceAccount에 대해 알아보자.
date: 2025-08-17T21:18:44+09:00
image: k8s-logo.webp
math: true
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

## 🔐 ServiceAccount란
**ServiceAccount** 란, 쿠버네티스 내에서 고유한 식별을 제공하는 **kube-apiserver**에 접근하는 자원을 위한 계정이다.  
쉽게 생각하면, Pod의 쿠버네티스 클러스터 내 계정이라고 볼 수 있다.  

ServiceAccount는 다음의 특징을 가진다:
- **Namespaced:**  
각 ServiceAccount는 쿠버네티스 네임스페이스에 종속된다.  
모든 네임스페이스는 생성 이후, 기본 ServiceAccount를 가진다.  
- **Lightweight:**  
ServiceAccount는 클러스터에 존재하고, Kubernetes API에 정의된다.  
특정 작업의 허용을 위해, 빠르게 ServiceAccount를 만들어줄 수 있다.  
- **Portable:**  
복잡한 워크로드 내에서의 설정 번들에서 시스템 컴포넌트로 사용될 수 있다.  

ServiceAccount는 User Account와는 다르다.  
기본적으로 User Account는 쿠버네티스의 API Server에 존재하지 않는다.  
대신, API Server는 사용자 자격 증명을 불투명 데이터로 다룬다.  
사용자 계정을 다양한 방법으로 인증 가능하다.  
일부 쿠버네티스 배포판은 확장 API를 제공할 수 있다.  


| Description | ServiceAccount | User or Group |
| --------------- | --------------- | --------------- |
| Location | Kubernetes API(ServiceAccount Object로 존재) | 외부 |
| Access Control | Kubernetes RBAC 또는 인가 메커니즘 | Kubernetes RBAC 또는 다른 자격증명 및 접근 메커니즘들 |
| Intended Use | 워크로드 및 자동화 | 사람 |



### Default ServiceAccount
쿠버네티스 클러스터가 생성되면, 자동으로 `default` ServiceAccount가 만들어진다.  
가지고 있는 권한은 딱히 없다.  
만약 제거된다면, 자동으로 다시 생성된다.  
**RBAC** 가 켜져 있다면, "API-discovery", 즉 API조회 수준의 작업밖에 할 수 없다.  

Pod를 띄울 때 아무 SA를 주지 않으면, 해당 SA가 붙는다(`spec.serviceAccountName`으로 SA를 줄 수 있다)  
이게 취약점이 될 수 있는데,  기본 SA 토큰이 주입되기 때문이다.  
그래서, SA를 잘 지정하거나, `automountServiceAccount`를 false로 하여 자동 토큰 마운트를 비활성화할 필요가 있다.  


---
## 🧱 ServiceAccount의 사용 사례

다음과 같은 상황에서 자격 증명을 주는 것이 필요할 수도 있다:
- Pod들이 Kubernetes API와 통신이 필요한 경우
    - Secrets에 저장된 정보를 Read-Only로 제공
    - namespace간 통신
- 외부 서비스와 Pod들이 통신하는 경우
    - 클라우드 제공자 또는 신뢰 관계에 있는 외부 제공자와의 상호작용
- `imagePullSecret`을 사용하여 프라이빗 이미지 레지스트리에 제공하는 경우
- Kubernetes API 서버와 통신이 필요한 외부 서비
- 서드 파티 보안 소프트웨에서의 사용

## 🔌  ServiceAccount를 사용하는 방법

ServiceAccount를 사용하기 위해서는 다음을 따르면 된다:
1. `ServiceAccount` 오브젝트를 생성
2. `RBAC`와 같은 인가 메커니즘을 ServiceAccount Object에 적용
3. Pod 생성에서 `ServiceAccount` 할당

### ServiceAccount에 권한 할당하기
**role-based access control(RBAC)** 메커니즘으로 각 ServiceAccount에 최소권한을 할당할 수 있다.  
Role을 만들어서, ServiceAccount에 Role을 bind해주면 된다.  
Pod들이 필요한 기능에서의 권한만 사용하도록 할당을 해줄 수 있다.   

### Namespace간 접근
**RBAC** 를 이용하여 하나의 namespace에서 다른 namespace의 자원을 불러올 수 있도록 한다.  
예를 들어, 모니터링 및 보안 도구들은 별도 네임스페이스에서 클러스터의 애플리케이션들로부터 여러 상태를 불러올 수 있는 것이다.  


### ServiceAccount를 Pod에 적용하기
`spec.serviceAccountName`으로 Pod에 ServiceAccount를 적용가능하다.  
v1.22 이후로, 특정 볼륨으로 토큰이 마운트된다.  이 토큰은 수명이 짧고, 자동으로 업데이트되게 되어있다.  
이전에는 Secret으로 정적 토큰이 긴 수명을 가졌다.  

### ServiceAccount credentials 받아오기
만약 일반적이지 않는 위치에 ServiceAccount를 마운트 시키고 싶거나, 다른 이유로 자격 증명을 별도 저장하고 싶다면, 다음의 방법들이 있다:
- `TokenRequest API`: 애플리케이션으로 API 요청을 보내는 방법이다.  
토큰은 수명이 짧고, 자동으로 갱신 요청을 직접 해야 한다.  
Kubernetes API에 접근하기 어려운 애플리케이션이라면, 사이드카 컨테이너를 붙여서 공유 볼륨을 이용하면 되겠다.  
- `Token Volume Projection`: v1.20 이후로 가능한데, Pod spec에 `projected volume`을 지정하는 것이다.  
kubelet은 해당 volume에 ServiceAccount 토큰을 주입하고, 자동 갱신해준다.  
현재 가장 많이 쓰이는 방법이다.  

이외에도, `Service Account Token Secrets`, `LegacyServieAccountNoAutoGeneration feature gate`도 있지만, 이들은 권장되지 않고, 버전 지원이 안 될수도 있다.  

> **Note**
> 클러스터 밖에서 실행되는데 자격 증명이 필요하다면, `TokenRequest API`또는 별도의 자격 증명 이용 방법이 권장된다.  


---
## ✅ ServiceAccount credentials 인가  
ServiceAccount는 **JWT(Json Web Token)** 을 사용한다.  
API 서버에 접근하고자 하면, `Authorization: Bearer <token>`을 http 헤더에 붙이면 된다.  
API 서버는 다음의 방법으로 토큰을 검증한다:  
1. 토큰 서명 검증
2. 토큰이 만료되었는지 검증
3. 참조 유효성 확인(해당 ServiceAccount및 관련 오브젝트가 존재하는지 체크)
4. 토큰이 현재 유효한지 검증(nbf 검증)
5. Audience Claim을 확인

`TokenRequest API`로 발급된 토큰은 Pod 라이프사이클과 묶인 bound token을 제공한다.  Pod가 삭제되면 같이 무효화되고, 특정 짧은 유효기간에 자동 회전(새 토큰으로 교체)된다.  

---
## 💂 별도의 검증 방법
외부에서 SA에 대한 검증이 필요하다면, 다음의 방법이 있다:
- `TokenReview API`(권장)
- OIDC discovery

Object Lifecycle과 동기화된 토큰 검증이 가능해서 권장된다.  

---
## 🟰 대안
- `Webhook Token Authentication`등의 별도 검증 방안을 사용할 수 있다.  
- Pod에 별도 자격 증명을 부여할 수 있다. (X.509등)
- 클러스터 외부 OIDC 및 IAM 등 이용
- 기타 등등...


---
## 📚 요약
- ServiceAccount는 쿠버네티스 리소스가 사용하는 계정이다.  
- 토큰은 JWT를 이용한다.  
- 발급은 `TokenRequest API` 이용 또는 `Token Volume Projection`이 권장된다. 단기 자격증명이여서 수명이 짧에 보안에 좋다.  
    - 오브젝트와 생명 주기가 동기화되어, 오브젝트의 존재 여부를 판단할 수도 있으며, 오브젝트 삭제 시에는 바로 만료 처리되어 안전하다.  
- 외부에서 토큰 검증을 하려면, `TokenReview API`를 이용하는 것이 권장된다.  
- `TokenRequest API`를 이용하려면, `admin`인증서, 또는 클라우드 IAM, 기존 권한있는 SA 토큰 등이 필요하다.  
