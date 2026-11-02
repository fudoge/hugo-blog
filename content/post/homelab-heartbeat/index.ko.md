---
title: "홈랩의 생존여부 추적하기"
description: 
date: 2026-10-28T22:45:00+09:00
image: 
math: 
license: 
comments: true
draft: true
build:
    list: always    # Change to "never" to hide the page from the list
---

홈랩을 새로 파는 와중에, 클러스터를 어떻게 살아있게 할지 고민했다. \
healthcheck.io를 이용할까도 했지만, 현재 백업 및 일부 자동화가 AWS를 이용하는 만큼, 헬스체크 스택 자체를 AWS로 통합하기 위한 목적도 있고, \
또 학습의 목적도 일부 있다.

## 주요 아키텍처

핵심 아이디어는 다음과 같다:
1. Alertmanager에서 Watchdog 알림을 AWS API Gateway로 전송
2. API Gateway에서 Lambda함수 실행
3. Lambda함수에서 커스텀 메트릭 발행
4. CloudWatch Alarms에서 최근 커스텀 메트릭을 조회하고, 문제 발생 시 알림

즉, 실제 이 시스템의 역할은, Alertmanager가 Dead Man's Switch로 동작하는 것이다.
다음의 문제들은 이 스택에서 잡지 못한다:
- AWS의 리전장애(서울리전 자체 장애 시, 알림이 안될 수 있음)
- 클러스터 내의 다른 문제(단, 이러한 문제들은 모니터링 스택 안에서 별도로 감지할 수 있다.)

장점은 다음과 같다:
- 추가적인 외부 스택 최소화
- 헬스체크의 동작여부 또한 기존 모니러팅 스택에서 확인가능
- 이 헬스체크 스택 또한 IaC로 운영가능

healthchecks.io에 비해, 단점 역시 있다:
- healthchecks.io대비 구성이 복잡하다
- 적지만 비용이 든다. 반면, healthchecks.io는 무료 플랜이면 충분하게 쓸 수 있다.

또 하나의 특징은, 상태 관리로 DynamoDB를 사용하지 않을 수 있다는 것이다. 

## CloudWatch에 커스텀 메트릭을 생성하는 Labmda함수 만들기

```python
# lambda_function.py
import os
import hmac
import hashlib
import boto3


def verify(token):
    actual = hashlib.sha256(token.encode()).hexdigest()
    expected = os.environ["HEARTBEAT_TOKEN_SHA256"]
    return hmac.compare_digest(actual, expected)


def lambda_handler(event, context):
    # auth
    headers = {k.lower(): v for k, v in (event.get("headers") or {}).items()}
    token = headers.get("authorization", "")[7:]  # Trim 'Bearer: '
    if not verify(token):
        return {"statusCode": 401, "body": "Unauthorized"}

    # put metric data
    namespace = os.environ["HEARTBEAT_NAMESPACE"]
    client = boto3.client("cloudwatch")
    response = client.put_metric_data(
        Namespace=namespace,
        MetricData=[{"MetricName": "Heartbeat", "Value": 1, "Unit": "Count"}],
    )

    return {"statusCode": 204}
```

이 코드는 `Authorization: Bearer <token>`에서 토큰 값을 추출하여 저장된 해시와 대조하고, Cloudwatch custom metrics으로 `Heartbeat`라는 이름, 그리고 값은 1로 저장한다. 
단위는 "Count"로 했다. \
Lambda Authorizer는 사용하지 않는데, 어차피 이 함수만 API Gateway에 연결하기 때문에 이게 더 단순하다 생각했다.

토큰은 무작위 랜덤값으로 만들면 된다. \
아래 명령을 실행하면 `original_token.txt`에 실제 토큰값, `hashed_token.txt`에 해시된 토큰값이 저장된다. \
실제 토큰값은 요청할 때 쓰고, 해시된 값은 환경변수에 넣으면 된다.
```bash
TOKEN=$(head -c 32 /dev/urandom | base64)
printf '%s' "$TOKEN" > original_token.txt
printf '%s' "$TOKEN" | sha256sum | awk '{print $1}' > hashed_token.txt
```

환경변수는 다음 값들을 채워주면 된다:
- `HEARTBEAT_TOKEN_SHA256`: 해시된 토큰
- `HEARTBEAT_NAMESPACE`: 커스텀 메트릭의 Namespace

이 함수가 실행되려면, `cloudwatch:PutMetricData` 권한이 필요하다. \
대신, 이 권한을 주면 스코프가 넓어지기에, 위 namespace와 매칭되는 메트릭만 넣을 수 있게 한정하는 것을 권장한다.

terraform코드 예시는 다음과 같다:
```hcl
## lambda에게 씌울 IAM Role을 위한 trust policy
data "aws_iam_policy_document" "heartbeat_assume_role" {
  statement {
    effect = "Allow"

    principals {
      type        = "Service"
      identifiers = ["lambda.amazonaws.com"]
    }

    actions = ["sts:AssumeRole"]
  }
}

## lambda에게 씌울 IAM Role
resource "aws_iam_role" "heartbeat" {
  name               = "${var.heartbeat_func_name}-role"
  assume_role_policy = data.aws_iam_policy_document.heartbeat_assume_role.json
}

## Lambda 아카이빙
data "archive_file" "heartbeat" {
  type        = "zip"
  source_dir  = "${path.module}/lambda/heartbeat/src"
  output_path = "${path.module}/lambda/heartbeat/function.zip"
}

## Lambda 함수 정의
resource "aws_lambda_function" "heartbeat" {
  filename      = data.archive_file.heartbeat.output_path
  function_name = var.heartbeat_func_name
  role          = aws_iam_role.heartbeat.arn
  handler       = "lambda_function.lambda_handler"
  code_sha256   = data.archive_file.heartbeat.output_base64sha256

  runtime = "python3.14"

  environment {
    variables = {
      HEARTBEAT_TOKEN_SHA256 = var.heartbeat_token
      HEARTBEAT_NAMESPACE    = var.cloudwatch_heartbeat_ns
    }
  }
}

## IAM Policy document - PutMetricData를 Allow
## 단, Namespace를 한정지어서 최소범위로 정의
data "aws_iam_policy_document" "heartbeat_put_metric_data" {
  statement {
    effect = "Allow"

    actions = [
      "cloudwatch:PutMetricData"
    ]

    resources = ["*"]

    condition {
      test     = "StringEquals"
      variable = "cloudwatch:namespace"

      values = [
        var.cloudwatch_heartbeat_ns
      ]
    }
  }
}

## 위에서 만든 docuemnt로 실제 policy 정의
resource "aws_iam_policy" "heartbeat_put_metric_data" {
  name   = "${var.heartbeat_func_name}-put-metric-data"
  path   = "/"
  policy = data.aws_iam_policy_document.heartbeat_put_metric_data.json
}

## IAM Role에 위 policy 바인딩
resource "aws_iam_role_policy_attachment" "heartbeat_put_metric_data" {
  role       = aws_iam_role.heartbeat.name
  policy_arn = aws_iam_policy.heartbeat_put_metric_data.arn
}

## CloudWatch Logs에 로그를 저장하기 위한 기본 policy도 IAM Role에 할당
resource "aws_iam_role_policy_attachment" "heartbeat_cloudwatch_logs" {
  role       = aws_iam_role.heartbeat.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

## Log group. 1일간만 보관하도록 설정.
resource "aws_cloudwatch_log_group" "heartbeat" {
  name              = "/aws/lambda/${var.heartbeat_func_name}"
  retention_in_days = 1
}
```

## API Gateway에 연동하기 (w/ Custom Domain)

Lambda Function URL은 사용하지 않았다. \
커스텀 도메인으로 엔드포인트를 관리하길 원했고, 이 경우 구조가 더 직관적이라고 생각하기 때문이다. \
물론 Lambda Function URL도 CloudFront를 이용하면 가능하다지만, API Gateway에 커스텀 도메인을 붙이는 게 더 직관적이다. \
Lambda Function URL을 기반으로 하면, API Gateway를 걷어낼 수 있고, CloudFront의 월 무료 한도 안에서만 사용될테니 비용이 더 싸긴 하다.

추가적인 기능들이 굳이 필요 없이, 단순히 Lambda를 연결하므로 HTTP API를 사용한다.

API Gateway에 lambda_permission으로 `lambda:InvokeFunction`을 할당해야 한다. \
또한, 자신의 개인 도메인을 가지고 있어야 하며, ACM인증서 역시 발급되어있어야 한다.

```hcl
locals {
  route_key = "POST /api/heartbeat"
}

## API Gateway. 기본 엔드포인트는 비활성화
resource "aws_apigatewayv2_api" "heartbeat" {
  name          = "heartbeat"
  protocol_type = "HTTP"
  disable_execute_api_endpoint = true
}

## API Gateway -> Lambda 호출을 위한 권한 할당
resource "aws_lambda_permission" "lambda_permission" {
  statement_id  = "AllowHeartbeatAPIInvoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.heartbeat.function_name
  principal     = "apigateway.amazonaws.com"

  # /*로 하위 모든 구성요소들에게 전파
  source_arn = "${aws_apigatewayv2_api.heartbeat.execution_arn}/*"
}

## Lambda함수와 연결
resource "aws_apigatewayv2_integration" "heartbeat" {
  api_id           = aws_apigatewayv2_api.heartbeat.id
  integration_type = "AWS_PROXY"

  connection_type    = "INTERNET"
  integration_method = "POST"
  integration_uri    = aws_lambda_function.heartbeat.invoke_arn
}

## Route 정의. 경로와 Integration 매핑
resource "aws_apigatewayv2_route" "heartbeat" {
  api_id    = aws_apigatewayv2_api.heartbeat.id
  route_key = local.route_key

  target = "integrations/${aws_apigatewayv2_integration.heartbeat.id}"
}

## stage(배포 환경 및 진입점) 선언
resource "aws_apigatewayv2_stage" "heartbeat" {
  api_id      = aws_apigatewayv2_api.heartbeat.id
  name        = "homelab"
  auto_deploy = true

  # rate limit: 분당 최대 3회만 호출가능
  route_settings {
    route_key              = local.route_key
    throttling_burst_limit = 3
    throttling_rate_limit  = 0.05
  }
}

## 커스텀 도메인 선언
resource "aws_apigatewayv2_domain_name" "heartbeat" {
  domain_name = "heartbeat.chaewoon.work"

  domain_name_configuration {
    certificate_arn = aws_acm_certificate.cert.arn
    endpoint_type   = "REGIONAL"
    security_policy = "TLS_1_2"
  }
}

## 커스텀 도메인과 stage 연결
resource "aws_apigatewayv2_api_mapping" "heartbeat" {
  api_id      = aws_apigatewayv2_api.heartbeat.id
  domain_name = aws_apigatewayv2_domain_name.heartbeat.id
  stage       = aws_apigatewayv2_stage.heartbeat.id
}
```

## kube-prometheus-stack에서 Watchdog 알림 보내기

아래의 helm values를 이용한다:
Alertmanager에서 Watchdog알림을 보내게 한다. \
Alertmanager의 Watchdog은 항상 firing상태의 알람을 보낸다.
```yaml
alertmanager:
  alertmanagerSpec:
    # matcher에서 AlertmanagerConfig CR의 namespace가 Alertmanager와 같으면, Alert 자체의 namespace는 필터하지 않음
    # 즉, 단순하게 alertname=Watchdog만으로 알림을 보낼 수 있음
    alertmanagerConfigMatcherStrategy:
      type: OnNamespaceExceptForAlertmanagerNamespace

  # 기본값은 watchdog -> null 경로를 가지지만, 초기화 (kube-prometheus-stack 차트 92.11.0기준)
  config:
    route:
      routes: []
# ... 볼륨 등은 생략
```

이후, 별도의 `AlertmanagerConfig` CR을 만들어서 `alertname=Watchdog` 알림이 Webhook을 날리도록 한다. \
시크릿도 별도로 미리 저장해주면 된다.
```yaml
apiVersion: monitoring.coreos.com/v1alpha1
kind: AlertmanagerConfig
metadata:
  name: watchdog-heartbeat
  namespace: observability
spec:
  route:
    # 아래 heartbeat receiver로 참조
    receiver: heartbeat
    # alertname으로 묶음
    groupBy:
      - alertname
    # 기다리지 않고 바로 보냄
    groupWait: 0s
    # 그룹 변경 또는
    groupInterval: 1m
    # 계속 발생 중인 알림을 1분 간격으로 재전송. watchdog은 
    # 그러나, 정확히 매 분으로 고정되는건 아님
    repeatInterval: 1m
    # alertname=Watchdog인 경우, heartbeat receiver로 보냄
    matchers:
      - name: alertname
        value: Watchdog
        matchType: "="
  # receiver들 정의
  receivers:
    - name: heartbeat
      webhookConfigs:
        - url: https://heartbeat.chaewoon.work/api/heartbeat
          # 해결 시 알림보내기 비활성화 (어차피 Alertmanager는 항상 firing)
          sendResolved: false
          httpConfig:
            bearerTokenSecret:
              name: heartbeat-token # heartbeat-token 이라는 시크릿으로부터 참조
              key: token
```


## custom metrics기반 Cloudwatch Alarm연동



## 보안?

## 비용

만약, 비용을 더 줄이고 싶다면, API Gateway대신 Lambda Function URL로 전환하는 것을 고려할 수 있겠다. \
현재는 API Gateway 기반을 유지하려고 한다.

## References

- github.com/fudoge/homelab
- https://healthchecks.io
- https://registry.terraform.io/providers/hashicorp/aws/latest
- https://docs.aws.amazon.com/boto3/latest/reference/services/cloudwatch.html
- https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-throttling.html
- https://artifacthub.io/packages/helm/prometheus-community/kube-prometheus-stack
- https://runbooks.prometheus-operator.dev/runbooks/general/watchdog/
- https://prometheus-operator.dev/docs/developer/alerting/
- https://prometheus-operator.dev/docs/api-reference/api/
- https://prometheus.io/docs/alerting/latest/configuration/
