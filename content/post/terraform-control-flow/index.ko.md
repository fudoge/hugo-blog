---
title: "Terraform: 반복문과 제어문"
description: Terraform에서의 반복문과 제어문 사용
date: 2025-08-07T21:34:46+09:00
image: terraform-logo.png
math: 
license: 
hidden: false
comments: true
draft: false
---

## 🔁 반복문

### Count를 이용한 반복
테라폼 리소스에는 `count`라는 `meta-argument`가 있다.  
`count`는 생성하고자 하는 리소스 사본 수만 정의하면 된다.  

3명의 IAM 사용자는 다음과 같이 생성할 수 있다:
```hcl
resource "aws_iam_user" "example" {
  count = 3
  name  = "riveroverflow"
}
```

그러나, 이는 오류가 발생한다.  
각 사용자의 이름이 유일해야 하는데, 같은 이름을 가지기 때문이다.  

대신, `count.index`를 쓰면 각각의 반복을 가리키는 인덱스를 얻을 수 있다:
```hcl
resource "aws_iam_user" "example" {
  count = 3
  name  = "riveroverflow.${count.index}"
}
```

`terraform plan`을 해보자.
```bash
Terraform will perform the following actions:

  # aws_iam_user.example[0] will be created
  + resource "aws_iam_user" "example" {
      + arn           = (known after apply)
      + force_destroy = false
      + id            = (known after apply)
      + name          = "riveroverflow.0"
      + path          = "/"
      + tags_all      = (known after apply)
      + unique_id     = (known after apply)
    }

  # aws_iam_user.example[1] will be created
  + resource "aws_iam_user" "example" {
      + arn           = (known after apply)
      + force_destroy = false
      + id            = (known after apply)
      + name          = "riveroverflow.1"
      + path          = "/"
      + tags_all      = (known after apply)
      + unique_id     = (known after apply)
    }

  # aws_iam_user.example[2] will be created
  + resource "aws_iam_user" "example" {
      + arn           = (known after apply)
      + force_destroy = false
      + id            = (known after apply)
      + name          = "riveroverflow.2"
      + path          = "/"
      + tags_all      = (known after apply)
      + unique_id     = (known after apply)
    }

Plan: 3 to add, 0 to change, 0 to destroy.
```

이름이 `riveroverflow.[0~2]`를 갖는 것을 볼 수 있다.  

그러나, 일반적으로 이런 이름을 사용하지는 않는다.  
대신, 아래와 같이 배열을 이용하면 여러 고유한 이름을 가진 3개의 IAM 사용자를 생성할 수 있다:
```hcl
provider "aws" {
  region = "ap-northeast-2"
}

resource "aws_iam_user" "example" {
  count = length(var.names) # 길이를 구하는 내장 함수
  name  = var.names[count.index] # 인덱스 참조
}

variable "names" {
  type    = list(string)
  default = ["UserA", "UserB", "UserC"]
}
```

```bash
Terraform will perform the following actions:

  # aws_iam_user.example[0] will be created
  + resource "aws_iam_user" "example" {
      + arn           = (known after apply)
      + force_destroy = false
      + id            = (known after apply)
      + name          = "UserA"
      + path          = "/"
      + tags_all      = (known after apply)
      + unique_id     = (known after apply)
    }

  # aws_iam_user.example[1] will be created
  + resource "aws_iam_user" "example" {
      + arn           = (known after apply)
      + force_destroy = false
      + id            = (known after apply)
      + name          = "UserB"
      + path          = "/"
      + tags_all      = (known after apply)
      + unique_id     = (known after apply)
    }

  # aws_iam_user.example[2] will be created
  + resource "aws_iam_user" "example" {
      + arn           = (known after apply)
      + force_destroy = false
      + id            = (known after apply)
      + name          = "UserC"
      + path          = "/"
      + tags_all      = (known after apply)
      + unique_id     = (known after apply)
    }

Plan: 3 to add, 0 to change, 0 to destroy.
```

리소스에 `count`를 쓰면, 하나의 리소스가 아니라 리소스의 배열이 된다.  
이제, 배열 조회 구문과 결합하여, 아래와 같이 사용되어야 한다:
```hcl
<PROVIDER>_<TYPE>.<NAME>[INDEX].ATTR
```

예를 들어, 유저 하나의 `arn`을 원한다면, 아래와 같이 할 수 있다:
```hcl
output "user_a_arn" {
  value = aws_iam_user.example[0].arn # 0번 인덱스 참조
}
```

모든 유저의 `arn`을 원한다면, 아래와 같이 쓸 수 있다:

```hcl
output "user_all_arn" {
  value = aws_iam_user.example[*].arn # 모든 인덱스 참조
}
```

그러나, `count`는 전체 리소스를 반복할 수는 있지만, 리소스 내의 인라인 블록을 반복할 수는 없다.  
```hcl
resource "aws_security_group" "example" {
  count       = 2
  name_prefix = my-sg

  ingress {
    from_port   = 8080
    to_port     = 8080
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```
이 선언은 보안 그룹을 두개만들 뿐, `ingress`를 여러 개 만들지 못한다.

또한, 위의 3개의 IAM계정을 생성하는 것에서, `UserB`를 없앤다고 해보자.  
```hcl
variable "names" {
  type    = list(string)
  default = ["UserA", "UserC"] # UserB 제거
}
```

이 경우, 인덱스가 당겨져서 `names[1] = "UserC"`가 된다.  

### for_each 표현식

`for_each`를 사용하면, 리스트, 집합, 맵을 사용하여 전체 리소스의 여러 복사도 가능하고, 리소스 내 인라인 블록의 여러 복사를 생성할 수 있다.  
```hcl
resource "<PROVIDER>_<TYPE>" "<NAME>" {
    for_each <COLLECTION>

    [CONFIG...]
}
```
- `PROVIDER`: aws와 같은 공급자
- `TYPE`: `instance`와 같은 리소스 유형
- `NAME`: 테라폼 코드 내부에서의 리소스 식별자
- `COLLECTION`: 루프를 처리할 집합 또는 맵  
(리스트는 지원하지 않는다)

```hcl
resource "aws_iam_user" "example" {
  for_each = toset(var.names)
  name     = each.value
}
```

이렇게 하면, `names`의 요소들로 IAM 유저들을 만든다.  
리스트는 지원하지 않기에, 집합 또는 맵으로 바꿔야 한다.  

`for_each`로 생성된 요소는 리소스 맵이 된다.  
모든 요소의 arn을 출력하고 싶다면, 아래처럼 해야 한다:
```hcl
output "all_arns" {
    value = values(aws_iam_user.example)[*].arn
}
```

`for_each`를 사용해서 리소스를 맵으로 처리하면, 컬렉션 중간의 항목도 안전하게 제거할 수 있는 이점이 있다.  
리소스가 아닌 맵으로 결과가 저장되기 때문이다.   

`for_each`는 여러 개의 인라인 블록을 만들 수 있다는 것이 장점이다.  

모듈에서 아래와 같이 변수 추가를 한다고 해보자:
```hcl
variable "custom_tags" {
  type    = map(string)
  default = {}
}
```

루트 모듈에서는 이렇게 사용가능하다:
```hcl
module "mymodule" {
    source      = "";

    custom_tags = {
        Name = "foo"
    }
}
```

모듈 안에서는 이렇게 동적인 `tag`를 생성할 수 있다:
```hcl
resource "aws_instance" "myec2" {
  ami           = var.ami
  instance_type = var.instance_type

  dynamic "tag" {
    for_each = var.custom_tags
    content {
      key                 = tag.key
      value               = tag.value
      propagate_at_launch = true
    }
  }
}
```


### for 표현식
단일 값의 반복문은 어떨까?  
`for`의 기본 구문을 다음과 같다:
`for <ITEM> in <LIST> : <OUTPUT>`

```hcl
variable "names" {
  type    = list(string)
  default = ["UserA", "UserB", "UserC"]
}

output "upper_names" {
  value = [for name in var.names : upper(name)]
}
```

`upper_names`으로 이름들이 전부 대문자로 출력된다.    
(USERA, USERB, USERC)



아래는 길이가 5 이하인 이름만 출력한다:
```hcl
variable "names" {
  type    = list(string)
  default = ["Ronaldo", "Messi", "Neymar"]
}

output "upper_names" {
  value = [for name in var.names : upper(name) if length(name) <= 5] # [MESSI] 만 출력됨
}
```


`Map`의 반복은 다음과 같다:
`[ for <KEY>, <VALUE> in <MAP> : <OUTPUT> ]`  
이는 리스트가 반환된다.  

다음과 같이 `Map`으로의 출력도 가능하다:
```hcl
[for <ITEM> in <LIST> : <OUTPUT_KEY> => <OUTPUT_VALUE>] # List to Map
{for <KEY>, <VALUE> in <MAP> : <OUTPUT_KEY> => <OUTPUT_VALUE>} # Map to Map
```



### for 문자열 지시자

문자열 지시자를 사용하면 문자열 내에서 for 반복문, if와 같은 제어문을 사용할 수 있다.  
사용법은 아래와 같다:  
```hcl
%{ for <ITEM> in <COLLECTION> }<BODY>%{ endfor }
```
- `COLLECTION`: 반복할 리스트 또는 맵
- `ITEM`: `COLLECTION`의 각 항목에 할당할 로컬 변수의 이름
- `BODY`: `ITEM`을 참조할 수 있는 각각의 반복을 렌더링할 대상

```hcl
variable "names" {
  type    = list(string)
  default = ["Ronaldo", "Messi", "Neymar"]
}

output "for_directive" {
  value = <<EOF
  %{for name in var.names}
  ${name}
  %{endfor}
  EOF
}
```

`terraform apply`를 실행해보자.  

```bash
terraform apply

Apply complete! Resources: 0 added, 0 changed, 0 destroyed.

Outputs:

for_directive = <<EOT

  Ronaldo

  Messi

  Neymar
```


---
## 🟰 조건문

반복문이 있듯이, 조건문도 존재한다.  

### Count를 이용한 조건문
`count = <EXPRESSION> ? 1 : 0>`을 이용하면, 조건식에 따라서 해당 리소스를 생성할지 말지에 대한 여부를 정할 수 있다.  

아래 예시에서, `make_ec2`가 `true`라면 인스턴스를 생성하고, `false`라면 생성하지 않는다.  
```hcl
resource "aws_instance" "ec2" {
  count = var.make_ec2 ? 1 : 0

  ami = var.ami
  instance_type = var.instance_type
}

variable "make_ec2" {
  type = bool 
}
```


이를 응용하면, 범용 프로그래밍 언어에서의 `if-else`를 사용할 수 있다.    
아래는 `isadmin`이 `true`이면 `admin`사용자를 만들고, 아니라면 `user`를 만든다.  
```hcl
resource "aws_iam_user" "user" {
  name  = "user"
  count = var.isadmin ? 1 : 0
}

resource "aws_iam_user" "admin" {
  name  = "admin"
  count = var.isadmin ? 0 : 1
}

variable "isadmin" {
  type = bool
}
```

### for_each와 for를 이용한 조건문
`for_each`로도 조건 논리가 가능하다.  
빈 컬렉션이라면, 0개의 리소스 또는 인라인 블록이 생성되고,  
비지 않은 컬렉션이라면, 1개 이상의 리소스 또는 인라인 블록이 생성될 것이다.  
비어 있는지에 대한 여부는 `for`를 이용하면 된다.  


아래 예제는 `enabled`된 사용자만 생성한다: 
```hcl
variable "users" {
  default = [
    { name = "alice", enabled = true },
    { name = "bob", enabled = false },
    { name = "carol", enabled = true }
  ]
}

resource "null_resource" "enabled_users" {
  for_each = {
    for user in var.users : user.name => user
    if user.enabled
  } 
}
```

아래 예제는 `prod`환경만 필터링한다.  
```hcl
variable "servers" {
  default = {
    server1 = { env = "prod" }
    server2 = { env = "dev" }
    server3 = { env = "prod" }
  }

}

locals {
  prod_servers = {
    for name, config in var.servers : name => config
    if config.env == "prod"
  }
}
```


### if 문자열 지시자 조건문

`CONDITION`은 불리언으로 평가되는 조건식이고, `TRUEVAL`은 `CONDITION`이 `true`로 평가되면 렌더링할 표현식이다.  
`else`절도 포함 가능하다.
```hcl
% {if <CONDITION> }<TRUEVAL>%{ endif }

% {if <CONDITION> }<TRUEVAL>%{ else }<FALSEVAL>%{ endif }
```


---
## ❗️ 테라폼을 사용할 때의 주의 사항

### count와 for_each의 제한 사항
리소스 출력을 `count`또는 `for_each`에서 참조할 수 없음  
`plan`단계에서 `count`와 `for_each`를 계산할 수 있어야 한다.  
즉, 하드코딩된 값인 경우 참조 가능하지만, 이후에 계산되는 리소스 출력은 쓸 수 없다는 뜻이다.  


### plan이 유효함에도 실패
`plan`이 성공적이라고 해도 실제 적용이 오류가 나는경우가 많다.  

대부분의 이유는, Terraform만을 사용하지 않고 일부를 콘솔에서 직접 건드렸다가 충돌이 나는 경우가 대부분이다.  
`tfstate`는 테라폼이 관리하는 해당 루트 모듈에서의 상태만을 담기 때문이다.  

- 테라폼을 사용하면 되도록 테라폼만 쓰자..
- 기존 인프라가 있다면 `import`를 쓰자..

### 리팩토링의 까다로움
코드에서의 식별자를 바꾸는 상황에서, 시스템 중단이 생길 수 있다. **같은 자원임에도 다르게 식별되어 삭제 후 재생성되기 때문** 이다.  
이러한 사소한 변화에도 민감하게 반응하기 때문에, 신중한 조정이 필요하다.  

적용하기 이전에 참사를 줄이기 위해서..
- 항상 `plan`명령 사용
- 파기하기 전에 생성하기(`create_before_destroy`또는 수작업)
- 식별자 변경을 위한 상태 변경: `terraform state mv`를 이용
- 일부 매개변수를 변경할 수 없음  
리소스의 공식 문서 참조 필요

### 최종 일관성
일부 클라우드 공급자의 API는 비동기적이며, 결국 일관성을 관리한다.  
**변경사항의 적용이 전체 시스템에 전파되는 데에는 시간이 걸리므로, 일정 기간 일관성 없는 응답을 받을 수 있다.**   

예를 들어, EC2의 생성이 되었다고 뜨지만, 아직 프로비저닝중인 경우가 있다.  

VPC가 생성었는데 상태 동기화가 늦어서 리소스 생성의 오류가 날 수도 있다.  
그러나, 이러한 문제는 대부분 `terraform apply`를 한번 더 선언해주면 해결된다.  


---
## 📚 요약
- `count`, `for_each`, `for`, `if`는 반복문 및 조건문, 그리고 필터링에 쓰인다.  
- 테라폼의 구성을 리팩토링하는 것은 까다로우므로, 신중한 선택이 필요하다.  
