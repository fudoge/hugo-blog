---
title: "Kubernetes: 리소스 제어와 HPA(Horizontal Pod Auto-Scaler)"
description: 쿠버네티스에서 리소스 제어를 어떻게 지정하는지 알아보고, Pod의 수평적 확장에 대해 알아보자
date: 2025-07-26T00:00:28+09:00
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

## 🎨 리소스 제어
Pod를 명시할 때, 컨테이너가 어느 정도의 리소스를 요구하는지 명세할 수 있다.  
`kube-scheduler`는 이 요구를 읽고, 적절한 노드를 찾아 스케줄링한다.  
컨테이너의 리소스 한계를 명시하면, `kubelet`이 해당 컨테이너가 리소스 제한을 넘지 않도록 제한시킬 수 있다.

두 가지의 리소스 명세가 있다:
- `Requests`
- `Limits`

**Requests**, 즉 요구되는 자원량은 `kube-scheduler`가 인식하여 스케줄링 할 때 쓰이고, 컨테이너의 실행에 보장된다.  
그러나, **Limits**, 즉 제한은 다르다. `kubelet`과 컨테이너 런타임, 그리고 커널에 의해 제한받는데, 다음과 같다:
- CPU의 부족 시, 애플리케이션은 **스로틀링** 이 걸리며 느려진다. 애플리케이션의 부하가 강하더라고, 더 많은 CPU 자원을 사용할 수 없다.  
- 메모리의 부족은 말이 다르다. out of memory, 즉 OOM으로 인한 `kill`이 일어난다. 
그러나, 메모리가 초과된다고 즉각적인 OOMkilled가 되는 것이 아니다.  
대신, 일시적으로 메모리가 초과되는 것을 허용하지만, **커널에서 메모리가 부족함이 인지되는순간 해당 컨테이너가 죽는 것이다.**  
즉, 죽이는 것은 쿠버네티스가 아닌, 노드의 커널에서 죽이는 것이다.  


- **Requests**를 사용하지 않으면, 자원이 부족한 노드에 배치되었다가 제대로 동작하지 못할 수 있다.  
CPU 할당을 위해 경쟁하지만, 항상 후순위에 밀린다. 
- **Limits**를 사용하지 않으면, 더 초과해서 사용할 수 있고, 컨테이너들은 더 많은 CPU시간을 할당받기 위해 경쟁한다.  
메모리의 경우는 계속 사용되다가, 메모리 부족 시, OOMkilled의 우선 대상이 된다.
- 둘 다 사용하지 않으면, **Besteffort QoS** 로 동작하다가, 보통 메모리 초과로 가장 먼저 죽는 대상이 되거나, 평소에는 CPU 할당의 경쟁에서 항상 밀리며 지낸다.  
평소에는 **Requests**라도 써주는 것이 좋다.  

리소스 제한은 **컨테이너** 단위이다. 

---
## 🔬 리소스 표기

### CPU의 리소스 표기
1CPU는 1개의 물리적 CPU 코어 1개, 또는 VM의 논리적 CPU 코어 1개를 말한다.  
CPU는 **정수 또는 소수** 로 나타내진다.  
`0.5`면 코어 절반을 쓴다는 뜻이고, `1`이면 코어 1개를 사용한다는 의미이다.  
뒤에 `m`을 붙이면, **millicore** 라는 뜻으로, `0.5`코어 = `500m`이다.


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

---
## 🏃 리소스 할당 예시


```yaml
apiVersion: v1
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

## ✅ 어떻게 Pod가 스케줄링 되는가
Pod의 생성 요청이 들어오면, 어느 노드에서 실행할지를 정한다.  
**Requests**에 명시된 CPU와 메모리의 자원량을 기반으로 스케줄링되고, **Requests**들의 합이 노드의 상한선을 넘지만 않으면 된다. **(Limits는 넘을 수 있다)**
노드의 실제 자원의 실제 사용량이 얼마 되지 않더라도, Requests의 합이 넘게 된다면, Pod를 해당 노드에 스케줄하지 않는다.  

`kubectl describe node` 명령어로 노드에서 얼마나 더 자원 할당이 가능한지 확인할 수 있다.  


## 👯‍♂️ HPA(Horizontal Pods AutoScaling)
Control Plane의 Horizontal Controller가 **HorizontalPodAutoscaler(HPA)** 리소스를 만들어서 수평 스케일링을 수행해준다.  
Deployemnt나 Statefulset등에 소속된 Pod들의 scale-out을 지원한다.  

### 동작 방식
Horizontal Pod AutoScaler가 간헐적으로 **RC/Deployment** 등을 감시한다.  
`kube-controller-manager`의 `--horizontal-pod-autoscaler-sync-period`라는 파라미터 값에 있는 숫자를 주기도 감시한다.  
기본값은 15초이다. **클러스터 전체의 HPA들을 동일하게, 그리고 전역적으로 감지하고 평가하고, 오토스케일 한다.**  
예측가능한 스케줄링 및 자원할당을 위해서이다.  

각 감시마다, `controller-manager`는 각각의 `HorizontalPodAutoscaler`에 따른 자원 메트릭들을 질의한다.  
`scaleTargetRef`에 정의된 타겟 리소스를 찾아서, 타겟 리소스의 `.spec.selector`에 따라서 Pod들을 조회한다.  
그 뒤, Pod-당 질의되는 메트릭 또는 커스텀 메트릭 등을 조회한다.  

- Pod당 수집되는 자원 메트릭(ex. CPU)은 `HorizontalPodAutoscaler`에 의해 타겟된 각 Pod로부터 메트릭 API를 호출해서 수집한다.  
    - 만약 **target utilization value(목표 사용률)** 이 정해지면, 메트릭들은 사용률료 계산되어 목표량과 비교된다.  
    - 만약 **target raw value** 로 정해지면, 메트릭들은 실제 사용량을 기반으로 계산된다.    
    Pod의 CPU request를 정하지 않았다면, HPA는 CPU 사용률을 계산할 수 없다.   
    즉, AutoScaler 계산에서 제외되고, 자동 스케일링이 비정상적으로 작동할 수 있다.  
- Pod당 수집되는 커스텀 메트릭에 대해서는, 컨트롤러 함수들이 per-pod 리소스 메트릭의 수집과 같이 사용되지만, **utilization values로는 사용이 불가능하고, raw value로만 사용이 가능하다.**  
즉, 커스텀 메트릭에서는 적절한 임계값 설정을 위해서는, 경험기반 또는 트래픽 데이터 분석을 통한 적절한 수치를 정해야 한다.  
- 오브젝트(Ingress, Service) 메트릭, 또는 외부(클라우드 서비스나 기타 쿠버 클러스터 외부 시스템, 예를 들면 ELB 등)메트릭에 대해서는, **단일 값**으로 주어진다.   
(현재 Value)/(목표 Value) ratio만큼 현재 레플리카 수에 곱해서 Pod의 수가 조정된다.  
`autoscaling/v2` API version에서는, Pod수로 나뉘어져서, per-pod로도 계산가능하다.  
즉, (현재 Value)/(pod 개수)로 나눈 값으로 target value와 비교할 수 있다는 의미이다.  


### 레플리카 계산

메트릭을 계산한 뒤, 새로 정해지는 레플리카의 개수는 다음과 같다: 

$$
desiredReplicas = ceil \lceil currentReplicas \times \frac{currentMetricValue}{desiredMetricValue} \rceil
$$

예를 들어, 현재 메트릭 값이 `200m`이고, 목표 값이 `100m`이라면, 레플리카의 값은 기존의 두 배가 될 것이다.  
$200 \div 100 = 2.0$이기 때문이다. 
반대로, 현재 메트릭 값이 `50m`이라 하자. $50 \div 100 = 0.5$로, Pod의 개수가 절반 줄어들 것이다.  
그러나, 이러한 `ratio`가 1.0에 가까우면, 스케일링을 생략할 수도 있다.  
**기본 허용(tolerance)** 는 `0.1`로, `0.9 ~ 1.1`인 경우, 생략한다는 의미이다.  
이는 **너무 잦은 레플리카 수의 변경을 방지** 한다.  
아래와 같이 설정 가능하다: 
```yaml
behavior:
  scaleUp:
    tolerance: 0.05 # 5% tolerance for scale up
```
또는, `--horizontal-pod-autoscaler-tolerance=<tolerance>`를 통해서 조정 가능하다.  
`kubectl`로 하는 것이 아닌, `kube-controller-manager`의 실행 `command`의 `arg`로 넣어야 한다.  


**일부 상황(startup 등)에서는 Pod의 메트릭 값이 비정상이더라도 관용할 필요가 있다.**  
부정확한 메트릭이 측정되기 때문이다.  
- `--horizontal-pod-autoscaler-initial-readiness-delay`플래그의 값을 기반으로 Pod의 startup을 대기해주는데, 기본 30초이다.  
- `--horizontal-pod-autoscaler-cpu-initialization-period` 플래그의 값 설정을 통해서 해당 시간 내로 Ready 상태로 바뀌었다고 하더라도, ratio 계산에 쓰이지 않는다.  
JAVA 애플리케이션 등의 startup 과정에서는 조금 특이한 사용량이 나타나서 혼란을 줄 수 있기 때문이다.  
즉, 만약 Pod의 startup부분이 높은 CPU사용을 한다면, 
    - `startupProbe`를 높은 CPU사용량인 동안 계속 붙잡아두는 방법 또는
    - `readinessProbe`가 CPU spike가 진정되면 `Ready`를 보고하도록 설정(`initialDealySeconds`)  
추가로, `--horizontal-pod-autoscaler-cpu-initialization-period`옵션도 같이 사용해주면 좋다.



**만약, 메트릭 수집에 실패한 노드가 있다면, 더 보수적으로 계산한다.**  
측정 실패한 Pod에 대해서는, '가정값'을 넣는다.  
- `scale-in`을 고려할 때에는 누락된 Pod가 100%, 즉 최대치를 쓴다고 가정하고,  
- `scale-out`을 고려할 때에는 누락된 Pod가 0%, 즉 최소치를 사용한다고 가정한다.    

이렇게 보수적으로 잡고 계산하는 이유는, 이렇게 계산하는 것이 기존 레플리카와의 차이가 적어서, 잘못된 계산이더라도 **레플리카 수가 들쭉날쭉 바뀌는 것을 방지** 하기 때문이다.  

만약, **여러 메트릭이 고려된다면, 각 메트릭에 대해서 새로운 `desiredReplicas`가 구해지고, 이들 중 최대값**으로 정해진다.  

HPA가 스케일링을 하기 이전, 스케일 계산기록들은 모두 기록된다.  
그리고, Pod를 줄여야 할 상황일 경우, 이전의 기록된 스케일 계산기록들 중, 현재 레플리카보다 작은 레플리카 제안값들 중 최대값을 찾아서, 그 값 정도로만 규모를 줄인다.  
이 이유는, **점진적인 개수 줄이기**를 위함이다.  

예를 들어, 현재 레플리카가 10개, 줄여야 하는 상황에서 이전에 제안받은 추천값들이 `[5, 8, 7, 6]`인 경우, 8개로 줄어든다는 의미이다.
`--horizontal-pod-autoscaler-downscale-stabilization`플래그의 값을 조정해서 설정 가능하다.  
기본은 5분이다.  

---
## 📦 Container 수준의 메트릭 수집
Kubernetes 1.30부터 적용된다.  
만약 Pod에 웹 애플리케이션 하나와, 로그 전송기 하나가 있다고 하면, 우리의 관심사는 웹 애플리케이션 뿐일 것이다.  
**컨테이너의 리소스를 메트릭으로 세울 수 있다.**
```yaml
type: ContainerResource
containerResource:
  name: cpu
  container: application
  target:
    type: Utilization
    averageUtilization: 60
```

위 예시에서는, `application` 컨테이너의 메트릭에만 관심을 쓰도록 한다.

---
## ❗️ HPA 사용 시 주의할 점
- 기존 Deployemnt 또는 Statefulset을 이용하면서 명시한 spec.replicas의 값은 제거하는 것이 좋다. 새로 apply하면서 충돌이 일어날 수 있기 때문이다.  
- 또한 Deployemnt의 롤링 업데이트와는 매끄럽게 호환되지만, Statefulset과 HPA의 연동은 꽤 까다롭다(여기서 다루진 않는다)

---
## 🐣 HPA 예시

`kubectl`로 간단하게 HPA를 구성할 수 있다:
```bash
kubectl autoscale deployment my-deployment --cpu-percent=50 --min=1 --max=5
```

또는, yaml로 선언 가능하다.
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 10
  metrics: 
    - type: Resource
      resource: 
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
```

---
## 📚 요약
- **Requests** 는 최소 요구되는 자원 할당량, 총합이 노드의 가용량을 넘을 수 없다
- **Limits** 는 최대 할당 가능한 자원의 양, 총합이 노드의 가용량을 넘을 수는 있다
- CPU는 초과할 수 없고 대신 스로톨링, 메모리는 limits를 초과할 수 있으나, 노드의 메모리 압박 시, 곧 쓰러질 후보가 된다
- 쿠버네티스의 HPA 철학: **늘릴 땐 과감하게, 줄일 땐 신중하게**
- HPA관련 `arg`값들은 주로 `kube-controller-manager`의 매개변수 플래그값에 넣어주면 된다
