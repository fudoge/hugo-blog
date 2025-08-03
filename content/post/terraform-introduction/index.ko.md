---
title: "Terraform 시작하기: 기본 cli 명령어 + AWS EC2 인스턴스 생성"
description: "Terraform: Getting started with AWS Provider"
date: 2025-08-03T15:42:44+09:00
image: terraform-logo.png
math: 
license: 
hidden: false
comments: true
draft: false

tags: 
    - Terraform
    - AWS

Categories:
    - Terraform
---

## 🧑‍💻 Terraform 이란

**HashiCorp Terraform** 은 사람이 읽을 수 있는 구성 파일을 사용하여 버전 관리와 재사용 및 공유 기능을 제공하는, 클라우드 및 온프레미스의 리소스를 모두 정의가능한 **IaC(Infrastructure as Code)** 도구이다.   
**HCL(HashiCorp Configuration Language)** 을 이용하여 인프라를 코드로 관리하고, 선언형으로 동작하여 인프라 관리자가 리소스 간의 복잡한 의존 관계를 구성하는 것에 대한 부담을 크게 줄인다.  


Terraform은 처음에 MPL 2.0 라이선스를 따라서 오픈소스였지만, 2023년 BUSL(Business Source License) 1.1로 전환되었다.
대안으로는 
- 범용 언어(GPL, Generic Purpose Language)기반의 **Pulumi**
- 커뮤니티 버전으로 포크되어 CNCF에 의해 관리되고 있는 **Opentofu**
등이 있다. 


---
## 🚀 Terraform 동작방식

![Terraform 동작방식](tf-workflow.png)

Terraform은 클라우드 및 기타 플랫폼의 Target API에 접근하여 자원을 관리한다.  
**Provider**는 이러한 API에 접속하도록 돕는다.  

[Terraform Registry](https://registry.terraform.io/)에 AWS, Azure, GCP 등, 여러 Provider가 있는 것을 확인할 수 있다.  

![Terraform 실행과정](tf-execution.png)
Terraform은 크게 세 가지의 작업 단계로 구성된다:
- **Write**: 여러 클라우드 제공자 및 서비스에 대한 자원을 정의할 수 있다.  
- **Plan**: Terraform은 작성된 구성 파일을 확인하여, 실제 실행계획을 정리한다.  
- **Apply**: Plan대로 실제 작업을 수행한다.  만약 VPC를 수정하고 EC2 인스턴스를 변경한다면,  알아서 VPC부터 생성하고 EC2 인스턴스들을 생성한다.  


---
## ✅ Terraform의 특징

### 특장점
- **선언형 구성 방식**  
사용자는 무엇을 생성할지에 대한 상태만 정의하여, desired state를 보장받을 수 있고, 멱등한 결과를 얻는다.  
- **상태 관리**  
리소스의 상태가 `.tfstate`에 저장되고, 원격 저장소에 연동 가능하다.  
- **다양한 클라우드 및 온프레미스 인프라 지원**  
AWS, Azure, GCP, VMWare, OpenStack, Kubernetes 등, 다양한 Provider가 지원된다. 
- **모듈화 및 재사용 가능**  
`module`을 통해 코드 재사용이 가능하고, 구성의 표준화에 용이하다. 
- **의존성 자동 처리**  
사용자는 자원의 생성 순서를 고려할 필요 없이, Terraform이 자동으로 자원의 의존 관계를 파악하여 적절한 순서로 적용한다.  
즉, VPC 생성 -> 서브넷 생성 -> EC2 인스턴스 생성과 같은 과정이 자동으로 보장된다. 
- **코드 기반 버전 관리**  
Git으로 보전 컨트롤이 가능하다.  
- **CLI기반 및 간결한 문법**  
HCL은 매우 직관적이고 간결한 문법이다.  


### 단점 및 주의할 점
- **상태 파일 보안**  
`.tfstate`에 credentials이 평문으로 저장되어, 잘 관리해야 한다.
- **상태 충돌**  
여러 사용자가 동시에 상태를 변경하려고 시도할 시, 위험하다.  
잠금이 필수이다.  


---
## 💾 Terraform 설치

OS 및 환경에 맞게 설치해주면 된다.   
[Terraform 설치](https://developer.hashicorp.com/terraform/install)  

AWS CLI도 설치해주자.  
[AWS CLI 설치](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)  


AWS에서 IAM 계정을 생성하고, 액세스 키와 시크릿 액세스 키를 발급받아서 설정해주자.  
IAM계정의 권한은 현재 연습을 위해 `AdministratorAccess`를 할당했지만, 실제로는 최대한 적은 권한을 갖는 것이 좋다.  
```bash
aws configure
```

---
## 📚 요약
- **Terraform** 은 선언형의 IaC도구이며, HCL이라는 DSL(Domain Specific Language)로 상태를 정의하여 리소스를 관리하도록 해준다.  
- 다양한 **Provider** 를 통해 다양한 종류의 인프라를 정의하며 관리할 수 있고, 퍼블릭 클라우드부터 멀티클라우드나 하이브리드 클라우드, 온프레미스까지도 관리할 수 있다.  
- 버전 컨트롤에 용이하고, 자동으로 서버 자원을 프로비저닝할 수 있지만, state의 보안 및 동기화 문제를 잘 고려해야 한다.  

