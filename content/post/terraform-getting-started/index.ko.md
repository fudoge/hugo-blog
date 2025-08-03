---
title: "Terraform: 기본적인 EC2 생성하기"
description: Terraform에서 AWS EC2를 생성해보자
date: 2025-08-03T16:41:07+09:00
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

## 🐣 Provider 선언
```hcl
provider "aws" {
  region = "ap-northeast-2"
}
```

Terraform에 AWS Provider를 사용할 것이며, 인프라를 인프라를 `ap-northeast-2`지역에 배포할 것임을 알린다.  
`access_key`도 프로퍼티에 선언할 수 있으나, **여기에서 선언하는 것은 권장되지 않는다.**  


---
## 💼 리소스 선언

```hcl
resource "<PROVIDER>_<TYPE>:" "NAME" {
  [CONFIG...]
}
```

- `PROVIDER`: Provider의 이름
- `TYPE`: Provider에서 생성할 리소스 타입
- `NAME`: Terraform 내부에서 리소스를 참조하는 데 사용할 수 있는 식별자.  
즉, **실제 자원의 이름이 아니다.**   


아래는 AWS EC2 인스턴스를 만드는 예시이다:
```hcl
resource "aws_instance" "example" {
  ami = "ami-08943a151bd468f4e" // Ubuntu 22.04의 AMI(Amazon Machine Image) id
  instance_type = "t2.micro"
}
```

Ubuntu 22.04버전의 t2.micro인스턴스이고, 코드베이스 내부에서 example이란 고유 이름을 가진다.  

---
## 💻 CLI 명령

### init
`terraform init`명령으로 Provider를 설치할 수 있다.  
Provider 코드는 `.terraform`폴더에 생기고,  
Provider들의 목록 및 버전들에 대한 스펙은 `.terraform.lock.hcl`에 기록된다.  

`init`으로 현재 코드베이스에서 필요한 Provider를 설치하고, `module`을 설치할 수도 있다.  
상태를 저장하는 벡엔드의 초기화를 하는 역할에도 필요하다.  
**멱등한 명령이기에, 몇 번이고 실행해도 안전하다.**  



### plan
`terraform plan`으로 실제로 리소스들을 변경하기 전에 Terraform이 수행할 작업을 확인할 수 있다.  
Linux의 `diff`명령과 유사하며, 
- `+`는 추가되는 내용
- `-`는 제거되는 내용
- `~`는 수정되는 내용
을 말한다.  

### apply
`plan`과 같이 계획을 출력하고, 실제 진행할 것인지 사용자에게 물어본다.  
`yes`를 입력하면 **실제 작업을 수행** 한다.  
```bash
terraform apply

terraform apply -auto-approve # 자동 yes
```

### destroy
선언된 **자원들을 제거** 한다.  
```bash
terraform destroy

terraform destroy -auto-approve
```

### 예시

위에서의 HCL대로 작성하고, `terraform apply`명령을 실행해보자.  EC2 인스턴스가 하나 생성되었다. 
![EC2 인스턴스 생성됨](tf-first-ec2)


---
## 🏷️ 리소스에 tag달기

AWS의 리소스에는 tag를 달아주는 것이 식별에 용이하고, 규모가 커질수록 필수이다.  

`tags`프로퍼티에 `key = value`꼴로 아래와 같이 작성해주면 된다.  
```hcl
provider "aws" {
  region = "ap-northeast-2"
}

resource "aws_instance" "example" {
  ami           = "ami-08943a151bd468f4e"
  instance_type = "t2.micro"

  tags = {
    Name = "terraform-example"
  }
}
```

다시 적용 해보자:
```bash
➜ terraform apply -auto-approve
aws_instance.example: Refreshing state... [id=i-0ce952075771e9cc1]

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  ~ update in-place

Terraform will perform the following actions:

  # aws_instance.example will be updated in-place
  ~ resource "aws_instance" "example" {
        id                                   = "i-0ce952075771e9cc1"
      ~ tags                                 = {
          + "Name" = "terraform-example"
        }
      ~ tags_all                             = {
          + "Name" = "terraform-example"
        }
        # (37 unchanged attributes hidden)

        # (8 unchanged blocks hidden)
    }

Plan: 0 to add, 1 to change, 0 to destroy.
aws_instance.example: Modifying... [id=i-0ce952075771e9cc1]
aws_instance.example: Modifications complete after 2s [id=i-0ce952075771e9cc1]

Apply complete! Resources: 0 added, 1 changed, 0 destroyed.
```

`changed`에 주목하자. 인스턴스는 제거되지 않고, 태그만 변경되었다.  
인스턴스의 ID가 같은 것에 주목하자.  
![Tag 생성됨](tf-tagged-ec2)

`terraform.tfstate`를 확인해보자.  
Terraform이 **마지막으로 적용(apply)한 인프라의 실제 상태** 를 저장하고 있다.  


---
## 🙅 .gitignore

Terraform은 인프라를 코드로 관리하는 도구이므로, Git을 통해 관리될 수 있는데, Git이 추적하지 않아야 하는 일부 파일들이 있다.  
아래와 같이 `.gitignore`파일을 프로젝트 최상위 디렉토리에 위치시키자: 
```plaintext
.vscode # vscode 설정
.DS_Store # MacOS에서 자동 생성됨
.terraform 
*.tfstate
*.tfstate.backup
```

- `.terraform`폴더에는 provider 플러그인과 모듈 캐시 등이 저장되어있는데, 이들을 Git으로 관리할 필요가 없다.   
언제든 다운로드받을 수 있기 때문이다.  
불필요한 용량을 줄이기 위해, 제외시킨다.  
- `*.tfstate`는 상태 파일로, 민감한 정보가 들어있을 수 있다.  
- `*.tfstate.backup`역시 같은 이유로, Git으로 관리되면 안된다.  

---
## 🌭 단일 웹 서버 배포해보기

```hcl
provider "aws" {
  region = "ap-northeast-2"
}

resource "aws_instance" "example" {
  ami           = "ami-08943a151bd468f4e"
  instance_type = "t2.micro"

  user_data = <<-EOF
              #!/bin/bash
              echo "Hello World!" > index.html
              nohup busybox httpd -f -p 8080 &
              EOF

  user_data_replace_on_change = true

  tags = {
    Name = "terraform-example"
  }
}

```

- `<<-EOF`는 Terraform 자체의 heredoc구문으로, 여러 줄로 된 문자를 escape문자 없이 구문을 생성 가능하게 한다.  
- `user_data`는 인스턴스의 startup script를 제공한다.  
- `user_data_replace_on_change`는 `true`인 경우, `user_data`가 변경된 후 `apply`가 되면, 기존 인스턴스를 새로운 인스턴스로 교체시키도록 한다.  

이러면 `httpd 기반` 간단한 웹 서버가 완성되었다!   
하지만, 아직 접속할 수는 없다.  
**보안 그룹(Security Group)** 이 적용되지 않았기 때문이다.  
보안 그룹에서 8080포트를 허용하는 인바운드 규칙을 생성해야 한다.  

---
## 🔐 보안 그룹 설정

아래는 위의 웹 서버에게 모든 IP 주소로부터 tcp 8080연결을 허용하는 보안 그룹 선언이다:  
```hcl
resource "aws_security_group" "example_sg" {
  name = "terraform-example-sg"

  ingress {
    from_port = 8080
    to_port = 8080
    protocol = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

규칙은 여러 개 만들 수 있다:  
```hcl
resource "aws_security_group" "example_sg" {
  ingress {
  ..........
  }
  ingress {
  ..........
  }
  ingress {
  ..........
  }
}
```

그러나, 생성한다고 끝이 아니다. 인스턴스에서 보안 그룹에 연결해야 한다.  

리소스의 속성에 참조하려면, 아래와 같이 할 수 있다:
```hcl
<PROVIDER>_<TYPE>.<NAME>.<attribute>
```


아래는 EC2 인스턴스와 보안 그룹을 연결한 예시이다: 
```hcl
provider "aws" {
  region = "ap-northeast-2"
}

resource "aws_instance" "example" {
  ami                    = "ami-08943a151bd468f4e"
  instance_type          = "t2.micro"
  vpc_security_group_ids = [aws_security_group.example_sg.id]

  user_data = <<-EOF
              #!/bin/bash
              echo "Hello World!" > index.html
              nohup busybox httpd -f -p 8080 &
              EOF

  user_data_replace_on_change = true

  tags = {
    Name = "terraform-example"
  }
}

resource "aws_security_group" "example_sg" {
  name = "terraform-example-sg"

  ingress {
    from_port   = 8080
    to_port     = 8080
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

이제 적용 후, EC2 인스턴스의 퍼블릭 IP에 접속해보자. 잘 연결된다!  
![Public IP:8080으로 연결](tf-web-ec2)

---
## 🔗 Graph

리소스 간에는 종속성이 존재한다.  
사용자는 상관없이 작성하면 되지만, Terraform은 구문을 정상적으로 실행하기 위해, `DAG`로 구성한다.  
이것이 선언형의 장점이다.   
**사용자는 원하는 상태를 정의하고, 순서와는 무관하게 작성하면 된다.**  

```bash
➜ terraform graph              
digraph G {
  rankdir = "RL";
  node [shape = rect, fontname = "sans-serif"];
  "aws_instance.example" [label="aws_instance.example"];
  "aws_security_group.example_sg" [label="aws_security_group.example_sg"];
  "aws_instance.example" -> "aws_security_group.example_sg";
}
```

---
## 📥 변수 사용하기

변수를 사용하여 반복적인 표현에서의 변경을 편하게 할 수 있고, 실수를 줄여주고, 코드를 간결하게 만들 수 있다.  
```hcl
variable "NAME" {
    [CONFIG...]
}
```

다음의 선택적 매개변수가 포함된다:  
- **설명(description):** 변수가 어떻게 쓰이는지에 대한 설명(주석 기능)  
- **기본(default)**: 값이 전달되지 않을 때의 기본적으로 사용되는 값
    - 기본값도 없으면 대화형으로 값을 물어봄
- **유형(type):**
    - string
    - number
    - bool
    - list
    - map
    - set
    - object
    - tuple
    - any(기본)
- **검증(validation)**
    - 최소값이나 최대값 등의 유효성 검사 규칙
- **중요(sensitive)**
    - apply할 때 기록하지 않음(비밀번호, API키 등)


아래는 전달한 값이 숫자인지 확인하는 변수의 예시이다: 
```hcl
variable "number_example" {
  description = "An example of a number variable in Terraform"
  type = number
  default = 42
}
```

값이 리스트인지 확인하는 변수의 예시이다:  
```hcl
variable "list_example" {
 description = "An example of a list in Terraform"
 type = list
 default = ["a", "b", "c"]
}
```

타입 제약 조건을 결합할 수도 있다.  
아래는 리스트의 값들이 모두 숫자인지 확인한다: 
```hcl
variable "list_numeric_example" {
 description = "An example of a numeric list in Terraform"
 type = list(number)
 default = [1, 2, 3]
}
```

아래는 모든 값이 문자열이어야 하는 map을 요구한다:
```hcl
variable "map_example" {
 description = "An example of a map in Terraform"
 type = map(string)
 default = {
   key1 = "value1"
   key2 = "value2"
   key3 = "value3"
 }
}
```


객체 타입 제약을 사용하여 더 복잡한 구조를 만들 수 있다:  
```hcl
variable "object_example" {
 description = "An example of a structural type in Terraform"
 type = object({
   name = string
   age = number
   tags = list(string)
   enabled = bool
 })
 default = {
   name = "value1"
   age = 42
   tags = ["a", "b", "c"]
   enabled = true
 }
}
```


아까 웹 서버의 예시에서, 포트 번호를 변수화 해보자.  
아래처럼 할 수 있다: 
```hcl
provider "aws" {
  region = "ap-northeast-2"
}

resource "aws_instance" "example" {
  ami                    = "ami-08943a151bd468f4e"
  instance_type          = "t2.micro"
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
```

사용할 때에는 `var.변수명`으로 사용하면 되고,  
문자열 사이에서는 `${변수}`로 사용할 수 있다. 이를 interpolation이라고 한다.  


아래와 같이 하면, 8081번 포트로 동작한다:  
```bash
terraform apply -var "server_port=8081"
```

또는, `TF_VAR_변수명`으로 변수의 기본값을 정할 수 있다.  
```bash
export TF_VAR_server_port=8081
```

또는 `default`를 정해줄 수 있다:
```hcl
variable "server_port" {
  type = number
  default = 8080
}
```


입력 변수들의 우선순위는 다음과 같다: 
1. CLI 직접 입력(`-var="..." 또는 -var-file="..."`)
2. `*.auto.tfvars` 파일
3. `terraform.tfvars` 파일
4. 호스트 환경 변수(`TF_VAR_<variable_name>`)


---
## 📤 출력 변수

변수는 입력할 때에만 있는 것이 아니다.  출력되는 변수들 역시 존재한다.  실행 이후, 기본적으로 결과에 출력된다.  
```hcl
output "<NAME>" {
	value = <VALUE>
	[CONFIG...]
}
```

`NAME`은 출력 변수의 이름,   
`VALUE`는 출력하려는 Terraform표현식이다.  
`CONFIG`에는 다음의 매개변수가 포함될 수 있다:
- **설명(description):** 설명
- **중요(sensitive)**: 계획 또는 적용 종료 시 출력을 기록하지 않음(credential 에서 유용)
- **의존성**
    - 기본적으로 종속관계를 만들 수 있지만, 일부 경우 추가 참고정보 제공 필요.
    - ex. 서버의 IP주소를 반환하는 출력 변수가 있지만 해당 서버에 대해 보안 그룹이 올바르게 구성될 때까지 해당 IP에 액세스할 수 없다.
    이 경우, `dependency_on`을 사용하여 종속성을 명시적으로 알릴 수 있다.


아래는 서버의 IP주소를 찾기 위해 IP주소를 출력 변수로 제공하는 예시이다: 
```hcl
output "public_ip" {
  value       = aws_instance.example.public_ip
  description = "THe public IP address of the web server"
}
```

실행 이후, 터미널의 출력으로 나오고,  
출력 변수는 `tfstate`에 저장되어 있기에, `terraform outpu`으로 출력결과를 다시 가져올 수 있다. 
```bash
terraform output public_ip

terraform output -raw public_ip
```

응용하면, 아래와 같이 curl도 바로 날릴 수 있다.  
`-raw` 옵션은 특히 자동화 작업에 유용할 수 있다.  
```bash
➜ curl $(terraform output -raw public_ip):8080
Hello World!
```


---
## 📚 요약
- `provider`로 리소스의 Provider를 선언할 수 있다.  
- `resource`로 자원을 생성할 수 있으며, 이때 선언하는 이름은 Terraform 내부에서만 참조를 위해 사용된다.  
- 입력변수가 존재하며, 범위가 좁은 변수가 주로 더 우선이 되는 경향이 있다.  
- 출력변수는 결과 출력에 나오며, `tfstate`에도 저장되어 다시 출력시킬 수도 있다.  
