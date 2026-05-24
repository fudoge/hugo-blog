---
title: "AWS ALB기반의 Gateway API 구축"
description: AWS Load Balancer Controller를 기반으로 ALB Gateway API를 만들어보자.
date: 2026-05-11T13:39:24+09:00
image: aws-lbc-design.png
math: 
license: 
hidden: false
comments: true
draft: false

tags:
    - AWS
    - Kubernetes
    - EKS
    - Gateway API
    - ALB
categories:
    - AWS
    - Kubernetes
---

AWS Load Balancer Controller는 쿠버네티스 환경에서 Elastic Load Balancer를 관리할 수 있게 해준다. \
Ingress, LoadBalancer등의 타입을 지원하며, AWS 상에서는 Application Load Balancer, Network Load Balancer, Classic Load Balancer 등이 프로비저닝된다.

---
## ⚙️ 동작방식

### Ingress의 경우

#### 동작 순서

![Design of AWS Load Balancer Controller](aws-lbc-design.png)

1개의 ingress 리소스에 대해, 다음과 같이 동작한다:

1. Controller가 API Server로부터 ingress 이벤트를 확인한다. \
조건 만족 시, AWS 리소스의 생성을 시작한다.
2. ALB가 새로운 Ingress 리스로 생성된다. \
인터넷에 노출되거나(internet-facing), 내부(internal)일 수 있다.
3. AWS에서 ingress 리소스에 따른 타겟 그룹이 생긴다.
4. 정해진 포트에 따라 리스너가 생성된다. 기본적으로 80 또는 443이 사용된다. \
annotation으로 인증서를 붙일 수도 있다.
5. ingress 리소스에 따라서 rule이 생긴다. \
알맞은 Kubernetes Service에 트래픽이 도달하게 된다.

또한, Load Balancer Controller는 다음의 작업도 한다:
- k8s에서 ingress가 지워진 경우, 관련 AWS 자원 제거
- k8s에서 ingress가 변경 시, AWS 자원 수정
- controller가 재시작된 경우, 복구되도록 ingress와 관련된 AWS 자원들을 조합

#### 트래픽 처리

Load Balancer Controller는 두 가지의 트래픽 모드를 지원한다:
- **Instance mode(기본):** ALB에서 Kubernetes Node로 보내서 Service의 NodePort로 보낸다.
- **IP mode:** ALB에서 직접적으로 Kubernetes Pods로 트래픽을 흘린다. CNI가 POD ip에 직접 접근하는 것을 지원해줘야 한다.
    - hop 감소로 더 나은 성능을 보인다.
    - `alb.ingress.kubernetes.io/target-type: ip` annotation으로 설정한다.

### Gateway API의 경우

![Gateway API](gateway-reconcile.png)

다음과 같은 통합 루프가 일어난다:
1. **API Monitoring**: Controller가 API Server를 통해 지속적으로 Gateway API 리소스를 모니터링
2. **Queueing**: 확인된 리소스들이 내부 큐에 추가된다.
3. **Processing**: 각 큐의 요소들에 대해서, 
    - 연결된 GatewayClass가 관리하는 controllerName인지 확인한다. 즉, GatewayClass리소스의 `spec.controllerName`이 `gateway.k8s.aws/alb`이거나, `gateway.k8s.aws/nlb`인지 확인한다.
    - Gateway API 정의는 AWS 리소스의 NLB/ALB Listener, Rule, Target Group, Addon등으로 매핑된다.
    - 매핑된 리소스들은 AWS의 실제 상태와 비교된다. \
      desired 상태와 다르면, controller는 AWS API를 호출하여 상태를 동기화한다.
4. **Status Updates**: 상태를 맞춘 후, Gateway 리소스의 `status`필드에 상태를 업데이트한다. \
이는 로드밸런서 DNS 이름, ARN등 프로비저닝된 AWS 리소스에 대한 실시간 피드백과 Gateway가 승인되고 programmed된 상태인지 알려준다.

> **NOTE: Gateway API인데, NLB도 왜 지원하나요?** \
> Gateway API는 실제로 L7라우터로만이 아닌, L4라우터를 대신할 수도 있습니다. \
> 즉, `type: LoadBalancer`의 대체를 할 수도 있습니다. \
> Gateway별로 서로 다른 레벨의 Route 규칙을 붙일 수 없습니다. \
> 예를 들어, HTTPRoute와 TCPRoute는 같은 Gateway에서 사용될 수 없습니다. 
> - L7 Gateway: HTTPRoute, GRPCRoute 
> - L4 Gateway: TCPRoute, UDPRoute, TLSRoute

Gateway API도 똑같이 Instance mode와 IP mode를 지원한다.


---
## 🚀 Load Balancer Controller 설치하기

### Helm을 통한 설치
아래 명령으로 repository를 추가한다.
```bash
helm repo add eks https://aws.github.io/eks-charts
```

```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --version 3.33.0 \
  --set clusterName = <cluster_name> \
  --set region = <aws_region> \
  --set vpcId = <vpc_id> \
  --set serviceAccount.create = true \
  --set serviceAccount.name = aws-load-balancer-controller \
  --set controllerConfig.featureGates.ALBGatewayAPI = true # 중요: ALB Gateway API에 필수
```

또는, `values.yaml`에 helm values를 넣고 설치하면 된다.
```yaml
# values.yaml
clusterName: <clutser_name>
region: <aws_region>
vpcId: <vpc_id>

serviceAccount:
  create: true
  name: aws-load-balancer-controller

controllerConfig:
  featureGates:
    ALBGatewayAPI: true
```

```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --version 3.33.0 \
  -f values.yaml
```

### 이외의 values들

위의 helm value들과 별개로, 다음의 값들을 고려할 수 있다.
- `autoscaling`: controller에게 hpa를 제공한다. 변경이 잦다면, 필요할 수도 있다. 
- `enableWafv2`: WAF v2와 연동할 수 있게 된다.
- `enableCertManager`: admission webhook에 쓰이는 TLS인증서를 cert-manager로 관리하게 된다.
  
자세한 건 [여기](https://github.com/kubernetes-sigs/aws-load-balancer-controller/blob/main/helm/aws-load-balancer-controller/README.md)를 참조.

---
## 📝 Hands-on Example

ALB Gateway API를 만들어서, L7규칙기반 라우팅을 제공할 것이다. \
ACM을 붙여 TLS Termination을 적용하고, WAF ACL 규칙도 적용시키고, S3 access log도 붙여볼 것이다. \
또한, 외부에서는 `/metrics`경로를 접근하지 못하도록 정책도 만들어볼 것이다.

### 필수 요소

- EKS 클러스터 
- AWS Load Balancer Controller(설치는 위 참조)
    - AWS API에 접근하기 위한 [IAM Policy](https://github.com/kubernetes-sigs/aws-load-balancer-controller/blob/main/docs/install/iam_policy.json)를 가진 Role을 IRSA / Pod Identity로 Load Balancer Controller가 Assume해야 함
- [Gateway API CRD](https://github.com/kubernetes-sigs/gateway-api)에서 설치 manifest를 받는다. \

### 추가 요소

#### Route53 Hosted Zone & ACM Certificate & External-DNS

추가 요소로 분류되어있긴 하지만, 사실상 필수이다. \
실제 운영에서는 ALB만으로는 쓰지 않고, 커스텀 도메인을 붙인다.

아래는 Route53 hosted zone 구성 + ACM 인증서를 발급하는 Terraform 코드 예시이다. \
AWS 콘솔에서 도메인을 구매했다면, Hosted Zone이 자동으로 생기는데, 그 때에는 `import`로 가져와야 한다.
```hcl
resource "aws_route53_zone" "main" {
  name = var.root_domain_name
}

resource "aws_acm_certificate" "alb" {
  domain_name               = var.root_domain_name
  subject_alternative_names = var.certificate_subject_alternative_names
  validation_method         = "DNS"

  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_route53_record" "alb_acm_validation" {
  for_each = {
    for dvo in aws_acm_certificate.alb.domain_validation_options :
    dvo.domain_name => {
      name   = dvo.resource_record_name
      record = dvo.resource_record_value
      type   = dvo.resource_record_type
    }
  }

  allow_overwrite = true
  zone_id         = aws_route53_zone.main.zone_id
  name            = each.value.name
  type            = each.value.type
  ttl             = 60
  records         = [each.value.record]
}

resource "aws_acm_certificate_validation" "alb" {
  certificate_arn         = aws_acm_certificate.alb.arn
  validation_record_fqdns = [for record in aws_route53_record.alb_acm_validation : record.fqdn]
}

## Variables
variable "root_domain_name" {
  description = "Primary public DNS zone managed in Route 53."
  type        = string
  default     = "example.com"
}

variable "certificate_subject_alternative_names" {
  description = "Additional DNS names to include in the shared ACM certificate."
  type        = list(string)
  default     = ["*.example.com"]

  validation {
    condition     = length(var.certificate_subject_alternative_names) > 0
    error_message = "At least one subject alternative name must be provided."
  }
}
```

External-DNS는 [여기](https://artifacthub.io/packages/helm/external-dns/external-dns)에서 받을 수 있다.

아래는 Helm values 예시이다:
```yaml
provider:
  name: aws

serviceAccount:
  create: true
  name: external-dns

# Source 종류는 다음을 참고:
# https://kubernetes-sigs.github.io/external-dns/latest/docs/sources/about/
sources:
  - gateway-httproute

env:
  - name: AWS_DEFAULT_REGION
    value: ap-northeast-2

domainFilters:
  - example.com

policy: upsert-only

registry: txt
txtOwnerId: my-web
```

필요한 IAM Policy는 아래와 같다. \
이 Policy를 IRSA / Pod identity로 적용해주면 된다.
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "route53:ChangeResourceRecordSets",
        "route53:ListResourceRecordSets",
        "route53:ListTagsForResources"
      ],
      "Resource": [
        "arn:aws:route53:::hostedzone/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "route53:ListHostedZones"
      ],
      "Resource": [
        "*"
      ]
    }
  ]
}
```

External DNS는 다음 로직으로 동작한다:
1. ExernalDNS가 DNS이름 후보를 HTTPRoute등에서 추출
    - `spec.hostname`
    - `external-dns.alpha.kubernetes.io/hostname`
2. HTTPRoute의 status의 parents로 Gateway 식별
3. Gateway에서 Route를 Accept했는지 확인
4. Gateway의 listener와 Route의 정보들 매칭
5. Gateway.status.addresses[].value들을 확인해서 로드밸런서 주소 확인
6. DNS레코드 생성

#### WAFv2 regional ACL

Reginal 스코프의 WAFv2 ACL을 만들어서 ALB에 추가할 수 있다. \
**여기서는 ACL만 만들고, Association은 만들지 않는다.** \
**만들어진 ACL에 Load Balancer Controller가 association을 만드는 형식이다.**

#### S3 for access logging

ALB의 액세스 로그는 S3에 직접 저장되게 할 수 있다. (Cloudwatch 로그그룹 아님) \
다음의 옵션들이 권장된다:
- Lifecycle Policy
- Block Public Access
- KMS암호화(필요 시)

여기서, ALB가 S3에 로그를 쓰기 위해선, 다음의 정책이 S3 버킷에 적용되어야 한다:
```json
{
  "Version":"2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "logdelivery.elasticloadbalancing.amazonaws.com"
      },
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::amzn-s3-demo-bucket/prefix/AWSLogs/123456789012/*"
    }
  ]
}
```

### GatewayClass

`spec.controllerName`에서 `gateway.k8s.aws/alb`를 선택해야 한다.

```yaml
# GatewayClass.yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: GatewayClass
metadata:
  name: aws-alb-gateway-class
spec:
  controllerName: gateway.k8s.aws/alb

```

### LoadBalancerConfiguration

`LoadBalancerConfiguration`은 AWS Load Balancer Controller를 설치하면 이용할 수 있는 CRD로, AWS ELB의 추가 설정을 세팅할 수 있다. \
`Gateway` 리소스에서는 Gateway API 스펙에 맞는 옵션들을 붙이지만, 여기서는 실제 AWS-specific한 내용들이 들어간다.


```yaml
apiVersion: gateway.k8s.aws/v1beta1
kind: LoadBalancerConfiguration
metadata:
  name: alb-config
  namespace: kube-system
spec:
  # 외부 퍼블릭 IP를 가질지, 내부 IP를 가질지
  scheme: internet-facing
  ipAddressType: ipv4
  # 인증서 정보 추가
  listenerConfigurations:
    - protocolPort: HTTPS:443
      defaultCertificate: <ARN of ACM Certificate>
      # sslPolicy에 대해서는 다음을 참고: https://docs.aws.amazon.com/elasticloadbalancing/latest/application/describe-ssl-policies.html
      sslPolicy: ELBSecurityPolicy-TLS13-1-2-2021-06
  # WAF 추가 (regional WAFv2 ACL의 arn을 추가)
  wafV2:
    webACL: <ARN of wafv2 regional web acl)
  # 로드밸런서가 S3 에 access 로깅을 하도록 설정
  loadBalancerAttributes:
    - key: access_logs.s3.enabled
      value: "true"
    - key: access_logs.s3.bucket
      value: <s3 bucket name>
    - key: access_logs.s3.prefix
      value: ""
  # 어느 subnet에 붙을지에 대해 사용
  loadBalancerSubnets:
    - identifier: <subent-id>
```

`loadBalancerSubnets`가 비어있다면, AWS Subnet은 Tag로 다음의 값을 가져야 한다:
- Public Load Balancer인 경우(internet-facing): "kubernetes.io/role/elb" = "1"
- Private Load Balancer인 경우(internal) "kubernetes.io/role/internal-elb" = "1"

### Gateway

Gateway에서는 GatewayClass의 이름을 참조해야 한다. \
또한, LoadBalancerConfiguration의 이름을 참조하여 해당 로드 밸런서의 추가 옵션을 이용할 수 있다. \
Gateway 리소스가 생기면, 실제 AWS ELB가 프로비저닝되며, Listener에 매핑된다.

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: Gateway
metadata:
  name: aws-alb-gateway
  namespace: kube-system
spec:
  # GatewayClass의 이름을 참조해야 함
  gatewayClassName: aws-alb-gateway-class
  # ALB의 추가 옵션을 참조하여 이용 가능
  infrastructure:
    parametersRef:
      kind: LoadBalancerConfiguration
      name: alb-config
      group: gateway.k8s.aws
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      hostname: www.example.com
      allowedRoutes:
        namespaces:
          from: Selector
          selector:
            matchLabels:
              # namespace의 이름이 my-web이여야 함
              kubernetes.io/metadata.name: my-web
    - name: https
      protocol: HTTPS
      port: 443
      hostname: www.example.com
      allowedRoutes:
        namespaces:
          from: Selector
          selector:
            matchLabels:
              kubernetes.io/metadata.name: my-web
```

### HTTPRoute

HTTPRoute는 실제 Rule과 Target Group으로 매핑된다. \
여기서는 두 개의 Rule이 있는데, 하나는, `/metrics`요청이 외부에서 오면, 차단시키는 것이다. \
이는 아래에서 더 자세히 다룰 것이다. \
두 번째의 Rule은 루트 경로 (`/`)에 대해 웹 서버로 포워딩한다. \
여러 prefix에 매칭되는 경우, PathPrefix는 더 상세한 (more specific) 경로가 우선 선호된다.

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: my-web
  namespace: my-web
  labels:
    app: my-web
    app.kubernetes.io/name: my-web
spec:
  # Gatway의 리스너와 매핑
  parentRefs:
    - name: my-alb-gateway # Gateway 이름
      namespace: kube-system
      sectionName: http # Listener 이름
    - name: my-alb-gateway
      namespace: kube-system
      sectionName: https
  # 받고자 하는 hostname. 여기서의 hostname으로 External-DNS가 레코드를 생성함.
  hostnames:
    - www.example.com
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /metrics
      filters:
        - type: ExtensionRef
          extensionRef:
            group: gateway.k8s.aws
            kind: ListenerRuleConfiguration
            name: deny-metrics
    - matches:
        - path:
            type: PathPrefix
            value: /
      # 라우팅할 service 이름
      backendRefs:
        - name: my-web
          port: 80
```

### ListenerRuleConfiguration

아래는 `/metrics`가 연결하는 Rule로, 고정된 403에러를 응답하도록 한다.

```yaml
apiVersion: gateway.k8s.aws/v1beta1
kind: ListenerRuleConfiguration
metadata:
  name: deny-metrics
  namespace: my-web
spec:
  actions:
    - type: "fixed-response"
      fixedResponseConfig:
        statusCode: 403
        contentType: "text/plain"
        messageBody: "Forbidden for public access"
```

### TargetGroupConfiguration

타겟 그룹에 대해 ELB는 헬스 체크가 필요하다. \
헬스 체크 엔드포인트를 넣는다. \
또한, Ingress에서 annotation으로 IP인지, Instance인지를 정했듯이, targetType으로 대상을 정한다.

```yaml
apiVersion: gateway.k8s.aws/v1beta1
kind: TargetGroupConfiguration
metadata:
  name: my-web
  namespace: my-web
spec:
  targetReference:
    group: ""
    kind: Service
    name: my-web
  defaultConfiguration:
    # Pod IP에 직접 라우팅
    targetType: ip
    healthCheckConfig:
      healthCheckPath: /helathz/
      healthCheckProtocol: http
      matcher:
        httpCode: 200-399
```

---
## 🌐 결과 예시

아래는 설정들을 적용한 뒤의 Load Balancer Controller가 생성한 리소스 맵의 예시이다.
![Resource Map](resource_map.png)

타겟 그룹에서 healthy한 파드가 없는 이유는 스크린샷 당시에 워커 노드를 모두 제거해서 그런데, \
healthy한 타겟이 있는 경우, Pod IP들이 각 타겟 그룹에서 보인다. \
프로젝트 당시 편의를 위해 Grafana, ArgoCD등도 퍼블릭으로 노출했었다.

Route53에서도 레코드가 추가된 것을 볼 수 있다.
![Route53 Record](route53_record.png)

버킷의 S3 acceess log도 잘 쌓이는 것을 볼 수 있다.
![S3 access logs](s3_access_log.png)

---

## Referenes
- [Kubernetes sigs - aws-load-balancer-controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/)
- [AWS Docs - Enable access logs for ALB](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/enable-access-logging.html)
- [External DNS - Gateway API Route Sources](https://kubernetes-sigs.github.io/external-dns/latest/docs/sources/gateway-api/#supported-api-versions)
- [External DNS - Gateway Sources](https://kubernetes-sigs.github.io/external-dns/latest/docs/sources/gateway/)
