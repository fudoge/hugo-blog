---
title: "Kubernetes: 리소스 제어와 HPA(Horizontal Pod Auto-Scaler)"
description: 쿠버네티스에서 리소스 제어를 어떻게 지정하는지와, Pod의 수평적 확장에 대해 알아보자
date: 2025-07-26T00:00:28+09:00
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

## 리소스 제어
Pod를 명시할 때, 컨테이너가 어느 정도의 리소스를 요구하는지 명세할 수 있다.  
`kube-scheduler`는 이 요구를 읽고, 적절한 노드를 찾아 스케줄링한다.  
컨테이너의 리소스 한계를 명시하면, `kubelet`이 해당 컨테이너가 리소스 제한을 넘지 않도록 제한시킬 수 있다.

두 가지의 리소스 명세가 있다:
- `Requests`
- `Limits`

**Requests**, 즉 요구되는 자원량은 `kube-scheduler`가 인식하여 스케줄링 할 때 쓰이고, 컨테이너의 실행에 보장된다.  
그러나, **Limits**, 즉 제한은 다르다. `kubelet`과 컨테이너 런타임, 그리고 커널에 의해 제한받는데, 다음과 같다:
- CPU의 부족 시, 애플리케이션은 스로틀링이 걸리며 느려진다. 애플리케이션의 부하가 강하더라고, 더 많은 CPU 자원을 사용할 수 없다.  
- 메모리의 부족은 말이 다르다. out of memory, 즉 OOM으로 인한 `kill`이 일어난다. 
그러나, 메모리가 초과된다고 즉각적인 OOMkilled가 되는 것이 아니다.  
대신, 일시적으로 메모리가 초과되는 것을 허용하지만, 커널에서 메모리가 부족함이 인지되는순간 해당 컨테이너가 죽는 것이다.  
즉, 죽이는 것은 쿠버네티스가 아닌, 노드의 커널에서 죽이는 것이다.  


- **Requests**를 사용하지 않으면, 자원이 부족한 노드에 배치되었다가 제대로 동작하지 못할 수 있다.  
CPU 할당을 위해 경쟁하지만, 항상 후순위에 밀린다. 
- **Limits**를 사용하지 않으면, 더 초과해서 사용할 수 있고, 컨테이너들은 더 많은 CPU시간을 할당받기 위해 경쟁한다.  
메모리의 경우는 계속 사용되다가, 메모리 부족 시, OOMkilled의 우선 대상이 된다.
- 둘 다 사용하지 않으면, **Besteffort QoS** 로 동작하다가, 보통 메모리 초과로 가장 먼저 죽는 대상이 되거나, 평소에는 CPU 할당의 경쟁에서 항상 밀리며 지낸다.  
평소에는 **Requests**라도 써주는 것이 좋다.  

리소스 제한은 **컨테이너** 단위이다. 

## 리소스 표기

### CPU의 리소스 표기
1CPU는 1개의 물리적 CPU 코어 1개, 또는 VM의 논리적 CPU 코어 1개를 말한다.  
CPU는 정수 또는 소수로 나타내진다.  
`0.5`면 코어 절반을 쓴다는 뜻이고, `1`이면 코어 1개를 사용한다는 의미이다.  
뒤에 `m`을 붙이면, millicore라는 뜻으로, `0.5`코어 = `500m`이다.


### 메모리의 리소스 표기
메모리는 바이트 단위로 작성된다.  
정수, SI표준, IEC표준, 부동소수점 방식 모두 가능하다.  
```bash
128974848 # 일반 표기
129e6 # 부동소수점 표기(129,000,000)
129M # SI표준(129,000,000)
128974848000m = # milli-byte 표기(1 milli-byte = 1/1000 byte)
123Mi = # IEC표준 (123 MiB = 123 * 2^20 바이트)
```
물론, 메모리 실제 구조상 **IEC표준을 쓰는게 좋다**. 


## 리소스 할당 예시


```yaml
``apiVersion: v1
kind: Pod
metadata:
  name: frontend
spec:
  containers:
  - name: app
    image: images.my-company.example/app:v4
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
  - name: log-aggregator
    image: images.my-company.example/log-aggregator:v6
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"`
```

두 컨테이너는 각각 0.25CPU, 64Mi를 최소로 요구하고 있고, 최대 0.5CPU, 128Mi를 최대로 요구하고 있다.  


Kubernetes 1.32에서의 신기능인데, Pod의 자원을 명시적으로 할당하고, Pod안에서 자원을 명시하지 않은 컨테이너의 자원 제한을 Pod내에서 컨트롤 할 수 있게 되었다.  
기본적으로 비활성화된 기능이고, `PodLevelResources` 피처 게이트를 세팅하면 가능하다고 한다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-resources-demo
  namespace: pod-resources-example
spec:
  resources:
    limits:
      cpu: "1"
      memory: "200Mi"
    requests:
      cpu: "1"
      memory: "100Mi"
  containers:
  - name: pod-resources-demo-ctr-1
    image: nginx
    resources:
      limits:
        cpu: "0.5"
        memory: "100Mi"
      requests:
        cpu: "0.5"
        memory: "50Mi"
  - name: pod-resources-demo-ctr-2
    image: fedora
    command:
    - sleep
    - inf 
```

## 어떻게 Pod가 스케줄링 되는가
Pod의 생성 요청이 들어오면, 어느 노드에서 실행할지를 정한다.  
**Requests**에 명시된 CPU와 메모리의 자원량을 기반으로 스케줄링되고, **Requests**들의 합이 노드의 상한선을 넘지만 않으면 된다. **(Limits는 넘을 수 있다)**
노드의 실제 자원의 실제 사용량이 얼마 되지 않더라도, Requests의 합이 넘게 된다면, Pod를 해당 노드에 스케줄하지 않는다.  

`kubectl describe node` 명령어로 노드에서 얼마나 더 자원 할당이 가능한지 확인할 수 있다.  


## HPA(Horizontal Pods AutoScaling)
Control Plane의 Horizontal Controller가 **HorizontalPodAutoscaler(HPA)**리소스를 만들어서 수평 스케일링을 수행해준다.  
Deployemnt나 Statefulset등에 소속된 Pod들의 scale-out을 지원한다.  

### 어떻게 동작하는가
Horizontal Pod AutoScaler가 간헐적으로 RC/Deployment등을 감시한다.  
`kube-controller-manager`의 `--horizontal-pod-autoscaler-sync-period`라는 파라미터 값에 있는 숫자를 주기도 감시한다.  
기본값은 15초이다.  



### 





## 요약

