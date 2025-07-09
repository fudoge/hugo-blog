---
title: "Docker Compose 기초"
description: "Docker Compose 기본 CLI 가이드"
date: 2025-07-10T00:09:40+09:00
image: docker-logo.png
math: true
license: 
hidden: false
comments: true
draft: true

tags:
    - Docker
    - Container
categories:
    - Docker
---

컨테이너는 **서비스의 추상화** 를 제공한다.  
파일 시스템, 네트워크, 의존성 및 라이브러리, 환경 변수 등의 환경들이 격리되었다.  
이렇게 환경이 격리되면서, 개발이 편해졌다.  

그러나, 이게 전부일까? 
실제 운영단계를 고려해보자.  

하나의 프로덕션에서는 여러 서비스로 나뉘고,  
각 서비스는:
- 서버 컨테이너
- 로그 포워딩 컨테이너
- node exporter 컨테이너
- 이들을 묶는 네트워크 및 볼륨들
등을 가지는데, 이러한 서비스들이 수백, 수천 개씩 있다고 해보자.

이런 수많은 컨테이너의 바다 속에서 장애가 나서 디버깅이나 로그를 확인해야 하거나, 업데이트를 해야 한다고 해보자.  
컨테이너들이 많아질수록, 관리가 매우 어려워진다.  

하나의 `yaml`파일로 이러한 복잡한 구성을 해결하기 위해, `docker compose`가 등장했다.  


---
## YAML
**YAML(YAML Ain't Markup Language)**은 사람이 읽기 좋은 계층적 구조의 데이터 표현 포맷이다.  
**JSON**에 비해서 매우 가독성이 좋다.

`키: 값`의 형식을 기본적으로 따르고, CI/CD, DevOps, k8s, 설정 파일 등에서 유용하에 쓰일 수 있는 문법이다.  

### 문법 기본

기본적으로 `키: 값`의 형태를 가진다.
```yaml
name: "chaewoon"
age: 23
running: true
```

리스트는 `-` 또는 대괄호`[]`를 이용할 수 있다.
```yaml
todo:
    - 과제
    - 수업 내용 정리
    - 운동

fruits: [apple, banana]
```

`키: 값`의 계층구조가 연쇄적으로 이루어진다.
```yaml
person: 
    name: "chaewoon"
    age: 23
    courses:
        docker:
            - what is container
            - docker-cli
            - docker-volume
            - docker-network
            - docker-compose
        k8s:
```

주석은 `#`을 이용한다.
```yaml
# 이름은 chaewoon 입니다.
name: "chaewoon"
```

> **주의사항**  
> YAML은 공백만 허용한다. Tab을 사용하지 않는다.  
> 들여쓰기를 기반으로 스코프가 정해지기에, 엄격한 들여쓰기를 준수해야 한다.
> `key: value`와 같이 `:`이후에는 공백이 있어야 한다.


---
## Compose CLI
```bash
docker compose [OPTIONS] COMMAND
```

`docker compose`는 프로젝트 단위이다.  
즉, 프로젝트의 이름을 설정하는 것이 중요하다.  
프로젝트의 명을 정하는 세 가지 방식이 있다. 아래는 우선순위 순서로 나열되었다:    
1. `docker compose -p <name> COMMAND`
2. `docker-compose.yaml`에 있는 파일
3. 현재 working directory

> **주의사항**
> `-f`를 사용할 compose파일을 지정하지 않는 경우, `docker-compose.yml` 또는 `docker-compose.yaml`이 사용된다.
> - `Docker compose` 파일의 `version`에 따라서 사용할 수 있는 기능과 변수명이 달라질 수 있다.   
> 따라서, 서버에 설치된 Docker Engine의 버전과 호환되는지 체크하는 것이 좋다.
> - 프로젝트 이름이 설정되지 않은 경우, 현재 디렉토리 이름이 프로젝트 이름으로 설정된다.

---
## Docker compose 버전 확인
```bash
docker compose version [OPTIONS]
```

---
## 프로젝트 실행
```bash
docker compose up [OPTIONS] [SERVICE...]
```

**Options:**
- `--abort-on-container-exit`: 1개 이상의 컨테이너가 멈춘(stopped) 경우, 모든 컨테이너 종료
- `--build`: 실행 전에 이미지 빌드
- `--detach(-d)`: 백그라운드에서 실행(하지 않으면 모든 stdout, stderr들이 모임)
- `--force-recreate`: 변경 여부와 관계없이 모든 컨테이너 재생성
- `--remove-orphans`: compose파일에 정의되지 않은 서비스를 위한 컨테이너 삭제

> **주의사항**
> - 서로 다른 `docker-compose.yaml`의 프로젝트 명이 중복된 경우,   
> 각 파일에 명시된 모든 `component`들 적용
