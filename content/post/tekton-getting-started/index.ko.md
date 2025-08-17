---
title: "Tekton 시작하기: 소개 및 설치"
description: Tekton 시작하기
date: 2025-08-17T23:02:54+09:00
image: tekton-logo.png
math: 
license: 
hidden: false
comments: true
draft: false
---

---
## 🧑‍🚀 Tekton이란
**Tekton** 은 Kubernetes Native CI/CD 솔루션이다.  
Tekton은 쿠버네티스의 확장으로 설치되며, 쿠버네티스에서 커스텀 리소스로 존재하여, CLI로도 다뤄질 수 있다.

---
## 💜 Tekton의 장점
- **Customizable:**  
Task, Pipeline, Step, Workspace, Resource 등이 모두 **CRD(Custom Resource Definition)** 로 되어 있어서, 조합 및 확장가능하다.  
- **Reusable:**  
같은 Task를 **여러 Pipeline에서 재사용** 할 수 있고, 파라미터나 워크스페이스를 적용하면 다양한 워크플로우에 재사용할 수 있다.  
`Tekton Hub`에서 공개된 Task도 가져다 쓸 수 있다.  
- **Expandable:**  
미리 만들어진 Task들로 파이프라인을 구성할 수 있고, 새로운 Task를 조직해서 표준으로 공유 가능하다.  
- **Standardized:**  
쿠버네티스 네이티브 방식으로 편하게 관리할 수 있다.  
`kubectl`로 관리 가능하고, Knative 또는 ArgoCD와의 궁합이 좋다.  
- **Scalable:**  
쿠버네티스 자원이기에, 노드만 추가된다면 파이프라인도 더 확장된다.  
리소스 제한도 직접 지정 가능하다.  

즉, 가볍고, 유연하고, 확장 가능한 파이프라인을 구성할 수 있다.  
게다가, RBAC, 네임스페이스 격리 의 쿠버네티스 장점들이 그대로 녹아있다. 


---
## 🆚 Tekton vs Other Solutions
물론 Tekton이 완벽한 것은 아니지만, 다른 솔루션들 대비해서 여러 강점들을 보인다:  


**Jenkins** 는 강력하지만, 오래 된 도구이며, 플러그인 및 에이전트 관리에 대한 복잡함과, Groovy에 대한 진입 장벽이 있다.  
- 쿠버네티스에서 Jenkins를 이용할 수도 있지만, Tekton 대비 리소스 오버헤드가 크다.  
- 그에 비해, Tekton은 마치 서버리스처럼 클러스터에서 동작한다.   
- 또한, Tekton은 YAML로 관리되어, 파이프라인을 더 간결한 코드로 관리할 수 있다.  

**GitHub Actions** 나 **GitLab CI** 도 강력하지만...
- 무료로 제공받는 Runner는 캐시 효율이 떨어진다.  
- 자체 Runner를 사용하면, 별도의 서버 관리의 불편함이 생긴다.  

Tekton도 단점이 있다.
- 러닝 커브(GitHub Actions, GitLab CI보다 어려움)  
- 레퍼런스 및 커뮤니티 규모 작음
- UI 제한적


---
## 👥 Tekton 구성요소
- **Tekton Pipelines**    
쿠버네티스 CRD로, CI 파이프라인을 정의하고 실행하는 핵심 컴포넌트 
- **Tekton Triggers**  
이벤트(Webhook 등)로 파이프라인을 자동으로 시작하게 함
- **Tekton CLI**  
Tekton 리소스를 조회/실행/로그확인 하는 CLI도구
- **Tekton Dashboard**  
파이프라인/태스크 상태와 로그를 확인할 수 있는 웹 UI
- **Tekton Catalog**  
커뮤니티에서 공유되는 표준 Task/Pipeline 저장소
- **Tekton Hub**  
Catalog의 웹 포털
- **Tekton Operator**  
Tekton 설치 및 업데이트를 오퍼레이터 패턴으로 관리
- **Tekton Chain**  
빌드 아키텍트의 서명 및 프로비넌스 기록을 자동화하는 공급망 보안 컴포넌트


### 기본 개념
- **Task:**  
    - 여러 Step(컨테이너 실행 단위)를 정의
    - Step들은 하나의 Pod에서 순차적으로 실행되며, 하나의 Step 결과를 다음의 Step에서 활용가능(공유 볼륨 또는 워크스페이스)
- **Pipeline:**
    - 여러 단계의 복잡한 워크로드
    - Task간 순서를 정의 가능하며, 이전 Task 결과를 다음 Task에서 활용가능
    - 복잡한 CI/CD를 정의하는 상위 개념  
- **TaskRun:**
    - Task가 실행된 인스턴스 및 기록 
    - 실행 로그, 성공 및 실패 상태, 사용된 파라미터 등을 담음
- **PipelineRun:**
    - Pipeline의 실행 인스턴스
    - 내부적으로 여러 TaskRun이 실행되며, 전체 실행의 상태 및 결과 기록
    - CI/CD파이프라인의 실행 히스토리로 남음


활용의 구분은 다음과 같다:
- **Task:**
    - 작은 작업
    - 단독 실행 가능한 단순 워크로드
- **Pipeline:**
    - 여러 단계의 복잡한 워크로드
    - 조건 분기, 에러 핸들링 등의 다양한 패턴 지원
    - 엔드 투 엔드 CI/CD


---
## 🚀 Tekton 설치

[Install Tekton Pipeline](https://tekton.dev/docs/installation/pipelines/)  

[Install Tekton CLI](https://tekton.dev/docs/cli/)

---
## 📦 StorageClass 정의하기

Tekton은 영속적인 스토리지가 필요하다.  
`StorageClass`를 정의하여 PV를 할당받을 수 있도록 해야 한다.  

AWS EKS기준, 아래와 같다:
- 먼저, EKS Pod Identity 에이전트를 설치하여 Pod에게 IAM 권한을 부여하도록 해야 한다.
- 그 뒤, CSI 드라이버를 구성해줘야 한다.  

`StorageClass` 정의 예시는 다음과 같다:
```YAML
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  fsType: ext4
  encrypted: "true"
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```


---
## 📚 요약
- Tekton은 쿠버네티스 네이티브의 CI/CD 파이프라인 솔루션이다.  
- 쿠버네티스에서 경량으로 동작하며, 쿠버네티스의 RBAC, namespace등의 기능, 오토스케일링 기능 등, 자원 할당 및 제한, 노드 선택 등의 다양한 기능을 사용할 수 있다. 
- 러닝커브와 부족한 UI, 적은 레퍼런스가 단점이다.  
