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

**장점:**
- 설정이 단순

**단점:**
- 서비스에 미치는 영향이 큼(모두 중단 후 다시 시작하기까지  잠시동안의 다운타임이 있다)

### Ramped
Incremental 또는 Rolling Update라고도 불린다.  
순차적으로 Pod들을 교체한다. 
- `maxSurge`: 최대 몇 개의 Pod들이 replicas보다 더 추가가능한가
- `maxUnavailable`: 최대 몇 개까지 서비스에서 제외해도 되는다
- 즉, 동시에 재가동될 수 있는 개수는 `maxSurge + maxUnavailable`개이다. 
- `Sticky Session`으로 버전간 돌아다니지 않도록 조정해야 한다.

**장점:**
- 설정 및 사용이 쉬움
- 신규 버전이 천천히 배포
- 데이터 Rebalancing이 진행되는 상태유지 어플리케이션에 유리

**단점:**
- 롤아웃/롤백이 오래걸림
- 트래픽에 대한 통제가 어렵다


### Blue/Green
새 버전으로 똑같은 양으로 띄워서 한 번에 전환시킨다.  
운영 환경이나 개발 환경이 거의 동일한 경우, 빠른 전환이 가능하다.  
또는, 서비스 관계가 복잡한 경우, 사용에 용이하다.  

**장점:**
- 간단한 롤아웃/롤백
- 종속성 지옥으로부터의 자유


**단점:**
- 비용이 2배.


### Canary
조금씩만 전환하여 안전하다는 판단이 들면, 크게 전환시킨다.  

**장점:**
- 일부 사용자를 위해 출시된 버전을 사용할 경우
- 오류율 및 성능 모니터링에 편리
- 비교적 빠른 롤백

**단점:**
- 느린 롤아웃
- Sticky Session 필요
- 정확한 트래픽 이용에는 서비스 메시 필요


### A/B Testing
지역, 언어, 회원정보, 버전, 헤더, 쿼리 파라미터 등의 정보를 기반으로 트래픽을 분류한다.  
즉, 사용자별로 차이를 두어 서비스한다.  

**장점:**
- 여러 버전의 어플리케이을 동시에 서비스 가능 
- 트래픽 분배에 대한 완전한 제어가 가능  
- 전환율을 개선하기 위해 비즈니스 목적으로 사용가능

**단점:**
- 지능형 로드밸런서가 필요  
- 세션에 대한 오류 해결이 어려움


### Shadow
Mirror 또는 Dark라고도 불린다.
임시 버전도 동일한 요청을 같이 복사해서 받는다.  

**장점:**
- 운영계에서 테스트에 좋음
- 사용자 영향 적음
- 안정성이 조건을 충족하기까지 롤아웃 없음

**단점:**
- 설정 복잡
- 비용이 비쌈
- 특정 서비스(걸제 등)은 Mocking필요

### 배포 전략 요약
- 다운타임을 허용해도 된다면, `Recreate`가 꽤 괜찮다
- `recreate`와 `ramped`는 Deployment기능에 설정으로 포함되어 있다. (`kubectl apply`로 충분하다.)
- `ramped`와 `blue/green`은 일반적으로 사용성이 좋고, 적용하기 쉽다.  
- `blue/green`과 `shadow`는 추가적인 인스턴스를 필요로 하여 비용이 비싸다.  
- `canary`와 `A/B Testing`은 릴리즈에 대한 확신이 없거나, 대규모 업데이트에 용이하다. 
- `canary`, `A/B Testing`, `shadow`는 모두 `Istio`와 같은 추가적인 컴포넌트가 필요하다. 


## Deployment 사용해보기
여기서는 기본적인 `recreate`와 `rolling update`를 실습해본다.

### 기본 Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels: 
    app: nginx
spec: 
  replicas: 3
  selector: 
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

생성 및 확인
```bash
kubectl apply -f nginx-deployment.yaml
kubectl get deployment
```

이제, 업데이트를 한번 해 볼 것이다.
아래 명령어에서 `--record`옵션을 넣으면, 명령어 자체가 `change-cause`가 되어 업데이트 메시지로 남는데, **deprecated** 기능이므로 권장되진 않는다.
```bash
kubectl set image deployment/nginx-deployment nginx=nginx:latest ## 또는 yaml파일을 수정 후 apply
```

어노테이션으로 현재 버전에 대해 업데이트 로그를 남길 것이다.
`kubernetes.io/change-cause` Key는 업데이트 로그 메시지를 기록하는 특수 주석 Key이다.
```bash
kubectl annotate deployment/nginx-deployment kubernetes.io/change-cause="update to nginx latest"
```

업데이트 관련 명령어들은 다음과 같다:
```bash
# 업데이트 상태 확인
kubectl rollout status deployment/nginx-deployment

# 업데이트 이력 확인 (change cause 포함)
kubectl rollout history deployment/nginx-deployment

# 상세 이력 확인
kubectl rollout history deployment/nginx-deployment --revision=2

# 상세 정보 확인
kubectl describe deployment nginx-deployment
```

롤백 명령어들은 다음과 같다:
```bash
## 이전으로 롤백
kubectl rollout undo deployment/nginx-deployment

## 특정 버전으로 롤백
kubectl rollout undo deployment/nginx-deployment --to-revision=1
```

다시 한번 히스토리를 봐보자. 새로 append되어있지만, `change cause`로는 롤백되었다는 메시지가 남겨있다.
```bash
kubectl rollout history deployment/nginx-deployment
```

Deployment를 스케일링 해보자.
```bash
kubectl scale deployment nginx-deployment --replicas=5 ## 5개로 레플리카 조정
```

결과를 확인해보자.
```bash
kubectl get po,deploy
```

### Rolling Update
Rolling Update는 쿠버네티스의 롤아웃의 기본 전략이다.  
즉, 이전 예시도 롤링 업데이트였다는 의미이다.  
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels: 
    app: nginx
spec: 
  replicas: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  selector: 
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

업데이트를 진행해주자.
```bash
kubectl set image deployment/nginx-deployment nginx=nginx:latest --record
```

만약 다른터미널에서 아래 명령어를 켜놓는다면, 감시하기 편하다.
```bash
kubectl get po -w # watch: 계속 출력
```

### Recreate

`strategy: Recreate`로 Recreate전략을 쉽게 취할 수 있다.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels: 
    app: nginx
spec: 
  replicas: 3
  strategy: Recreate
  selector: 
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```



## 마무리
- 다양한 배포 전략이 있으니, 때와 상황에 맞게 잘 사용해야 한다.  
- K8s에서는 기본적으로 `Recreate`와 `RollingUpdate`를 지원한다. 기본은 `RollingUpdate`이다.  
- `kubectl rollout`명령어로 스케일링 명령을 사용 가능하다.  
- `kubernetes.io/change-cause`는 버전 컨트롤을 돕는 특별 메시지를 위한 Key값이다.


