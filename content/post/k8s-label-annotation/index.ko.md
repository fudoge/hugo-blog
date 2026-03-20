---
title: "Kubernetes: Label과 Annotation"
description: 쿠버네티스의 주요 메타데이터인 Label과 Annotation에 대해 알아보자
date: 2025-07-16T05:22:15+09:00
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

## 🏷️ Label
### Label이란
Label은 Pod뿐만 아니라, 쿠버네티스의 모든 리소스에 사용될 수 있는 간단하지만 강력한 기능이다.  
리소스에 부여하는 임의의 `Key/Value`쌍이다.  
`LabelSelector`를 사용해서 리소스를 선택할 때 사용된다.  
일반적으로 리소스를 만들 때 부여하지만, 만들어진 후에도 Label을 추가하거나 변경할 수 있다.  

### Label 관련 제어

`yaml`파일에서 객체 선언 시에, `metadata`프로퍼티의 `labels`에 `key/value`쌍으로 정의해주면 된다.
```yaml
apiVersion: v1
kind: Pod
metadata: 
  name: web
  labels:
    env: prod
    creation_method: manual
spec: 
  containers:
  - name: web
    image: nginx:latest
    ports:
    - containerPort: 80
```

위의 예시에서, `nginx`의 의 label은 `env: prod`와 `creation_method: manual`이 있다.  
또는, 기존에 존재하는 자원에 label을 추가할 수 있다.  
```bash
kubectl label <Resource Type> <Name> key1=value1 key2=value2 ...
```

한 번에 여러 태그를 달아줄 수 있고, 여러 자원에 한 번에 부여할 수 있다.  
기존에 존재하는 자원에 label을 덮어씌울 수도 있다. `--overwrite`옵션을 달아주면 된다.  
```bash
kubectl label <Resource Type> <Name> key=value --overwrite
```

`LabelSelector`를 이용해서 조회를 할 수도 있다.
아래 예시를 통해 `env=prod`인 label을 가진 자원들을 조회할 수 있다.
```bash
kubectl get po -l env=prod
```
`LabelSelector`는 Key값만으로도 조회할 수 있다.  
```bash
kubectl get pod -l env
```

또한, 다양한 연산식을 제공한다:
```bash
kubectl get po -l `!env` # env라는 key가 없는 Pod조회

env in (prod, env) # env가 prod나 env로 설정된 경우 조회
env notin (prod, env) # env가 prod나 env가 아닌 경우에 조회
```

`-L`옵션으로 으로 해당 Key별로 컬럼을 만들어 조회할 수 있다:
```bash
kubectl get po -L env
```

또한, 각 자원별 label들을 `--show-labels`를 통해 확인할 수 있다:
```bash
kubectl get po --show-labels # pod들을 각각의 label들과 함께 조회
```


`<key>-`를 통해서 제거가 가능하다.
```bash
kubectl label po web app- # web pod의 app을 key로하는 label 제거
```

---
## 📜 Annotation
### Annotation이란
`Key/Value`쌍으로, Label과 유사하지만, `Selector`가 없다.
그러나, 라벨보다 훨씬 많은 정보를 담을 수 있다.  
주로 쿠버네티스의 오브젝트에 상세한 설명을 추가하여 다른 사람이 알아보기 쉽게 하는 용도이다.  
빌드 정보나 만든 사람등을 기록하면 작업 시에 용이하게 사용될 수 있다.  
또한, 특정 애드온이 쿠버네티스 오브젝트에 정보를 주입할 때 사용하기도 한다.  

### Annotation 관련 제어
Annotation은 250kb까지 작성가능하지만, 짧고 간결하게 사용하는 것이 좋다. 추가 및 제거가 `Label`과 유사한 명령어 구조를 가진다.  
그러나, 주석은 Selector가 없으므로, `describe`를 통해서 볼 수 있다.  

web pod를 하나 만들어보자.
```yaml
apiVersion: v1
kind: Pod
metadata: 
  name: web
  annotations:
    description: "web pod"
    build_info: "1.0.0"
    creator_info: "John Doe"
spec: 
  containers:
  - name: web
    image: nginx:latest
    ports:
    - containerPort: 80
```

아래 명령은 web pod의 설정을 볼 수 있는데, 이 안에 주석이 추가된 것을 확인할 수 있다.  
```bash
kubectl describe po web
```

---
## ⚖️ 비교
| 항목 | Label | Annotation |
| --------------- | --------------- | --------------- |
| 용도 | 리소스 필터링, 셀렉션 | 메타데이터 저장 |
| 구조 | `key/value`구조, `value`는 63자까지 허용 | `key/value`구조, `value`는 245KB까지 가능. |
| 셀렉터 사용 | 가능 | 불가능 |
| 예시 용도 | `app=nginx`, `env=prod` | description, build info, creator info |
