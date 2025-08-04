---
title: "Terraform: ASG(Auto Scaling Group)과 Elb(Elastic Load Balancer)"
description: 자동으로 인스턴스를 확장하는 ASG와 그에 접근하는 로드밸런서를 제공해보자.
date: 2025-08-04T22:02:09+09:00
image: terraform-logo.png
math: 
license: 
hidden: false
comments: true
draft: false

tags: 
    - Terraform
    - AWS
categories:
    - Terraform
---

## 💼 웹 서버 클러스터 배포하기

단일 서버는 간단하지만, **단일 장애점(Single Point of Failure)** 의 위험이 있다.  
해당 서버가 종료되거나, 너무 많은 트래픽으로 과부하가 발생하는 경우, 사용자는 접속할 수 없다.  

이에 대한 해결책은, 서버 클러스터를 실행하고, 트래픽에 따라 클러스터의 크기를 조정하면 된다.  
이를 수동으로 하면 번거롭지만, **ASG(Auto Scaling Group)** 을 이용하면, AWS가 이를 처리하도록 도울 수 있다.  
ASG는 EC2 클러스터 시작, 각 인스턴스 상태 모니터링, 실패한 인스턴스 교체, 로드에 따른 클러스터 크기 조정 등 많은 작업을 처리한다.  

### 시작 탬플릿 생성

현재 이러한 웹 서버를 가진다고 하자:
```hcl
provider "aws" {
  region = "ap-northeast-2"
}

resource "aws_instance" "example" {
  ami                    = "ami-08943a151bd468f4e"
  instance_type          = "t3.micro"
  vpc_security_group_ids = [aws_security_group.example_sg.id]

  user_data = <<-EOF
              #!/bin/bash
              echo "Hello World!" > index.html
              nohup busybox httpd -f -p ${var.server_port} &
              EOF

  user_data_replace_on_change = true

  tags = {
    Name = "terraform-example"
  }
}

resource "aws_security_group" "example_sg" {
  name = "terraform-example-sg"

  ingress {
    from_port   = var.server_port
    to_port     = var.server_port
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

variable "server_port" {
  type = number
}

output "public_ip" {
  value       = aws_instance.example.public_ip
  description = "THe public IP address of the web server"
}
```


`aws_instance`대신, 아래처럼 시작 탬플릿의 리소스를 선언하자.  
```hcl
resource "aws_launch_template" "example" {
  name_prefix = "example-lt"

  image_id               = "ami-08943a151bd468f4e" # Ubuntu 22.04
  instance_type          = "t3.micro"
  vpc_security_group_ids = [aws_security_group.example_sg.id]

  # aws_instance와는 다르게, base64encode를 진행해야 한다. 
  user_data = base64encode(templatefile("server-init.sh", {
    server_port = var.server_port
  }))

  tags = {
    Name = "terraform-example"
  }
}
```

`server-init.sh`는 아래와 같이 구성한다:
```bash
#!/bin/bash
echo "Hello, World" > index.html
nohup busybox httpd -f -p ${server_port} &  
```

### ASG(Auto Scaling Group) 생성

이제, `aws_autoscaling_group`으로 ASG를 생성할 수 있다.  
```hcl
resource "aws_autoscaling_group" "example" {
  # 시작 탬플릿 선택
  launch_template {
    id      = aws_launch_template.example.id
    version = "$Latest"
  }

  #  최소 ~ 최대 인스턴스 수 선언
  min_size = 2
  max_size = 10

  # 인스턴스에 할당할 tag지정.
  tag {
    key                 = "Name"
    value               = "terraform-asg-example"
    propagate_at_launch = true
  }

  # 먼저 생성 후 제거하여 리소스 교체를 안전하게 진행
  lifecycle {
    create_before_destroy = true
  }
}
```

`lifecycle`의 `create_before_destroy` 옵션을 `true`로 하여, 안정된 교체가 가능하다.  
비활성화 된 경우, 먼저 지우고 새 인스턴스로 교체하지 못하고, 기존 상태의 인스턴스가 부활되어 교체하지 못하고 교착에 걸리게 된다.  


### 서브넷 구성
ASG가 동작하려면, 서브넷을 추가해줘야 한다.  
`data`는 Terraform을 실행할 때 마다 Provider가 가져오는 Read-Only 정보를 가져온다.  
이는 동적이고, 유연하게 리소스의 정보를 가져올 수 있도록 한다.  

Terraform 구성 파일에 `data`를 추가해도, 새로운 리소스가 생기진 않는다.  
각 Provider는 다양한 `data`를 제공한다.  

```hcl
data "<PROVIDER>_<TYPE>" "<NAME>" {
    [CONFIG...]
}
```
- `PROVIDER`: Provider 이름
- `TYPE`: 데이터 소스 유형
- `CONFIG`: 관련된 args


아래는 기본 VPC를 조회한다:
```hcl
data "aws_vpc" "default" {
	default = true
}
```

데이터를 가져오려면, 아래와 같이 참조하면 된다:
```hcl
data.<PROVIDER>_<TYPE>.<NAME>.<ATTR>
```


`aws_subnets`와 결합하여, vpc내의 서브넷을 조회할 수 있다. 아래는 기본 vpc의 서브넷들을 참조한다.  
```hcl
data "aws_subnets" "default" {
	filter {
		name = "vpc-id"
		values = [data.aws_vpc.default.id]
	}
}
```

아래처럼 ASG를 수정하자:
```hcl

resource "aws_autoscaling_group" "example" {
  # 시작 탬플릿 선언
  launch_template {
    id      = aws_launch_template.example.id
    version = "$Latest"
  }

  # 디폴트 서브넷들 참조하기
  vpc_zone_identifier = data.aws_subnets.default.ids

  # 최소 ~ 최대 인스턴스 수 선언
  min_size = 2
  max_size = 10

  # 인스턴스에 할당할 tag지정.
  tag {
    key                 = "Name"
    value               = "terraform-asg-example"
    propagate_at_launch = true
  }

  # 먼저 생성 후 제거하여 리소스 교체를 안전하게 진행
  lifecycle {
    create_before_destroy = true
  }
}
```

설명과 코드가 뒤죽박죽이니, 현재까지의 정리본을 한번 정리하고자 한다.  
시작 탬플릿과 ASG의 구성은 아래와 같다:
```hcl
provider "aws" {
  region = "ap-northeast-2"
}

resource "aws_launch_template" "example" {
  name_prefix = "example-lt"

  image_id               = "ami-08943a151bd468f4e" # Ubuntu 22.04
  instance_type          = "t3.micro"
  vpc_security_group_ids = [aws_security_group.example_sg.id]

  user_data = base64encode(templatefile("server-init.sh", {
    server_port = var.server_port
  }))

  tags = {
    Name = "terraform-example"
  }
}

resource "aws_autoscaling_group" "example" {
  # 시작 탬플릿 선언
  launch_template {
    id      = aws_launch_template.example.id
    version = "$Latest"
  }

  # 디폴트 서브넷들 참조하기
  vpc_zone_identifier = data.aws_subnets.default.ids

  # 최소 ~ 최대 인스턴스 수 선언
  min_size = 2
  max_size = 10

  # 인스턴스에 할당할 tag지정.
  tag {
    key                 = "Name"
    value               = "terraform-asg-example"
    propagate_at_launch = true
  }

  # 먼저 생성 후 제거하여 리소스 교체를 안전하게 진행
  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_security_group" "example_sg" {
  name = "terraform-example-sg"

  ingress {
    from_port   = var.server_port
    to_port     = var.server_port
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

data "aws_vpc" "default" {
  default = true
}

data "aws_subnets" "default" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.default.id]
  }
}

variable "server_port" {
  type    = number
  default = 8080
}
```

---
## ⚖️ 로드밸런서(ALB 배포하기)

이제 여러 서버가 있지만, 사용자에게는 **단일의 엔드포인트만을 제공하는 것이 좋다.**  
로드 밸런서를 이용하여 트래픽을 ASG를 대상으로 보내고, 사용자에게는 하나의 엔드포인트만을 제공할 수 있다.  

AWS에는 세 가지의 로드밸런서가 있다:
- **ALB(Application Load Balancer)**
HTTP 및 HTTPS 트래픽의 로드밸런싱을 제공하며, L7 라우팅을 제공한다.
- **NLB(Network Load Balancer)**
TCP, UDP, TLS수준의 로드밸런싱을 제공, ALB보다 더 빠르게 응답하며, 확장 및 축소할 수 있다.
성능이 더 좋다.
- **CLB(Classic Load Balancer)**
레거시 로드 밸런서로, L7, L4 모두 지원하지만, 기능이 적고 잘 쓰이지 않는다.  

여기서는 ALB를 쓸 것이다.   
ALB는 세 가지의 구성 요소로 구성된다:
- **Listener**
특정 포트 및 프로토콜을 수신한다.
- **Rules**
리스너로 들어오는 요청을 받아 특정 경로 또는 호스트 이름과 일치하는 요청을 특정 대상 그룹에 보낸다.
- **Target Group**
라우팅 대상이 되는 하나 이상의 서버. 
주기적으로 헬스체크를 진행하며, 건강한 노드에만 라우팅한다.


### ALB 및 리스너 생성
아래는 ALB 생성의 예시이다:
```hcl
resource "aws_lb" "example" {
  name               = "terraform-asg-example"
  load_balancer_type = "application" # ALB로 생성
  subnets            = data.aws_subnets.default.ids
}
```

`subnets` 파라미터는 `aws_subnets` 데이터를 사용하여 기본 VPC의 모든 서브넷을 사용하도록 로드 밸런서를 구성한다.  
AWS ALB는 VPC에 종속되어 선택한 서브넷들에 따른 AZ들에 걸쳐 자동으로 확장 및 축소하여, 확정성과 고가용성을 제공한다.  
이것이 추상화 되어있고, 두 개 이상의 서로 다른 AZ의 퍼블릭 서브넷에 걸쳐야 한다.  
기본적으로 디폴트 VPC는 모두 퍼블릭 서브넷이다.  

ALB를 만들었으면, 이제 리스너를 정의하자.  
```hcl
resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.example.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type = "fixed-response"
    fixed_response {
      content_type = "text/plain"
      message_body = "404: Page Not Found"
      status_code  = 404
    }
  }
}
```
이 리스너는 포트 80에서 수신을 대기하고, HTTP 프로토콜을 사용, 리스너 규칙과 일치하지 않는 요청에 대한 기본 응답으로 간단한 404 페이지를 보내도록 구성되어있다.  

### 보안 그룹 생성
이제, ALB를 위한 보안 그룹이 필요하다.  
보안 그룹은 stateful해서, 인바운드 트래픽이 허용되어있다면, 그의 아웃바운드 트래픽도 자동으로 허용되어 별 상관 없지만, 로드 밸런서의 경우, 트래픽이 서버로 가야 하기에, 아웃바운드 트래픽을 허용해주어야 한다.  
```hcl
resource "aws_security_group" "alb" {
  name = "terraform-example-alb"

  # HTTP 요청 허용
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # 서버로 트래픽 전달
  egress {
    from_port   = 0 # 모든 포트
    to_port     = 0
    protocol    = "-1" # 와일드카드: 모든 프로토콜 허용
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

이제, 보안 그룹을 사용하도록 alb에 할당하자. 
```hcl
resource "aws_lb" "example" {
  name               = "terraform-asg-example"
  load_balancer_type = "application"
  subnets            = data.aws_subnets.default.ids
  security_groups    = [aws_security_group.alb.id]
}
```


### Target Group 생성
다음으로, `aws_lb_target_group`을 생성하여, ASG에 대한 타겟 그룹을 생성해야 한다.  
```hcl
resource "aws_lb_target_group" "asg" {
  name     = "terraform-asg-example"
  port     = var.server_port
  protocol = "HTTP"
  vpc_id   = data.aws_vpc.default.id

  # health check
  health_check {
    path                = "/" # 헬스체크 요청 경로
    protocol            = "HTTP"
    matcher             = "200" # 기대하는 응답 코
    interval            = 15    # 체크 간격
    timeout             = 3     # 요청 타임아웃(초)
    healthy_threshold   = 2     # 몇 번 연속 성공적인 응답이어야 healthy한지
    unhealthy_threshold = 2     # 몇 번 연속 실패해야 unhealthy한지
  }
}
```

타겟 그룹은 `aws_lb_target_group_attachment`로 인스턴스의 정적 리스트를 가지킬 수 있지만,   
`aws_autoscaling_group`에서 `target_group_arns`가 새 `target_group`으로 가리키게 하면 된다.

```hcl
resource "aws_autoscaling_group" "example" {
  # 시작 탬플릿 선언
  launch_template {
    id      = aws_launch_template.example.id
    version = "$Latest"
  }

  # 디폴트 서브넷들 참조하기
  vpc_zone_identifier = data.aws_subnets.default.ids

  # 이 ASG와 target group 바인딩
  target_group_arns = [aws_lb_target_group.asg.arn]
  # 타겟 그룹 단위로 헬스체크.
  health_check_type = "ELB"

  # 최소 ~ 최대 인스턴스 수 선언
  min_size = 2
  max_size = 10

  # 인스턴스에 할당할 tag지정.
  tag {
    key                 = "Name"
    value               = "terraform-asg-example"
    propagate_at_launch = true
  }

  # 먼제 생성 후 제거하여 리소스 교체를 안전하게 진행
  lifecycle {
    create_before_destroy = true
  }
}
```

마지막으로, `aws_lb_listener_rule`리소스를 사용하여 묶어주면 된다.  
ASG가 포함된 타겟 그룹에 대한 경로와 일치하는 요청을 보내는 리스너 규칙을 추가한다.  
```hcl
resource "aws_lb_listener_rule" "asg" {
  listener_arn = aws_lb_listener.http.arn
  priority     = 100

  condition {
    path_pattern {
      values = ["*"]
    }
  }

  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.asg.arn
  }
}

```

### 출력 변수 지정
마지막으로, 출력 변수가 나오도록 설정해주자.
```hcl
output "alb_dns_name" {
  value       = aws_lb.example.dns_name
  description = "The domain name of the load balander"
}
```

---
## 🚀 전체 코드 및 테스트

```hcl
`provider "aws" {
  region = "ap-northeast-2"
}

resource "aws_lb" "example" {
  name               = "terraform-asg-example"
  load_balancer_type = "application"
  subnets            = data.aws_subnets.default.ids
  security_groups    = [aws_security_group.alb.id]
}

resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.example.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type = "fixed-response"
    fixed_response {
      content_type = "text/plain"
      message_body = "404: Page Not Found"
      status_code  = 404
    }
  }
}

resource "aws_lb_target_group" "asg" {
  name     = "terraform-asg-example"
  port     = var.server_port
  protocol = "HTTP"
  vpc_id   = data.aws_vpc.default.id

  # health check
  health_check {
    path                = "/" # 헬스체크 요청 경로
    protocol            = "HTTP"
    matcher             = "200" # 기대하는 응답 코
    interval            = 15    # 체크 간격
    timeout             = 3     # 요청 타임아웃(초)
    healthy_threshold   = 2     # 몇 번 연속 성공적인 응답이어야 healthy한지
    unhealthy_threshold = 2     # 몇 번 연속 실패해야 unhealthy한지
  }
}

resource "aws_lb_listener_rule" "asg" {
  listener_arn = aws_lb_listener.http.arn
  priority     = 100

  condition {
    path_pattern {
      values = ["*"]
    }
  }

  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.asg.arn
  }
}

resource "aws_launch_template" "example" {
  name_prefix = "example-lt"

  image_id               = "ami-08943a151bd468f4e" # Ubuntu 22.04
  instance_type          = "t3.micro"
  vpc_security_group_ids = [aws_security_group.example_sg.id]

  user_data = base64encode(templatefile("server-init.sh", {
    server_port = var.server_port
  }))

  tags = {
    Name = "terraform-example"
  }
}

resource "aws_autoscaling_group" "example" {
  # 시작 탬플릿 선언
  launch_template {
    id      = aws_launch_template.example.id
    version = "$Latest"
  }

  # 디폴트 서브넷들 참조하기
  vpc_zone_identifier = data.aws_subnets.default.ids

  # 이 ASG와 target group 바인딩
  target_group_arns = [aws_lb_target_group.asg.arn]
  # 타겟 그룹 단위로 헬스체크.
  health_check_type = "ELB"

  # 최소 ~ 최대 인스턴스 수 선언
  min_size = 2
  max_size = 10

  # 인스턴스에 할당할 tag지정.
  tag {
    key                 = "Name"
    value               = "terraform-asg-example"
    propagate_at_launch = true
  }

  # 먼저 생성 후 제거하여 리소스 교체를 안전하게 진행
  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_security_group" "alb" {
  name = "terraform-example-alb"

  # HTTP 요청 허용
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # 서버로 트래픽 전달
  egress {
    from_port   = 0 # 모든 포트
    to_port     = 0
    protocol    = "-1" # 와일드카드: 모든 프로토콜 허용
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_security_group" "example_sg" {
  name = "terraform-example-sg"

  ingress {
    from_port   = var.server_port
    to_port     = var.server_port
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

data "aws_vpc" "default" {
  default = true
}

data "aws_subnets" "default" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.default.id]
  }
}

variable "server_port" {
  type    = number
  default = 8080
}

output "alb_dns_name" {
  value       = aws_lb.example.dns_name
  description = "The domain name of the load balander"
}
```

이제, 테스트해보자.  
```bash
curl $(terraform output -raw alb_dns_name)
Hello, World
```

---
## 📚 요약
- ASG + LB를 구성하면, 자동으로 확장되면서도 단일 엔드포인트를 제공하는 서비스를 만들 수 있다.  

자원의 의존 관계는 다음과 같다(A -> B: A자원리 B자원을 필요로 함):
1. ALB -> VPC, Security Group for ALB
2. ALB Listener -> ALB
3. ALB Target Group -> VPC
4. Listener Rule -> Listener, Target Group
5. Launch Template -> Security Group for EC2s
6. ASG -> Launch Template, Target Group, Subnets
