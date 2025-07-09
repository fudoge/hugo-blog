---
title: "Docker 이미지 고급 기능들: Commit과 Buildx"
description: Docker의 Commit과 Buildx에 대해 알아보자
date: 2025-07-09T00:10:37+09:00
image: "docker-logo.png"
math: true
license: 
hidden: false
comments: true
draft: false

tags:
    - Docker
    - Container

categories:
    - Docker
---

## ✅ Docker commit

`commit`은 자주 쓰이지는 않는다.  
보통은 `Dockerfile`로 이미지를 제작하지만, 긴급한 상황이나 디버깅의 경우에서 사용된다.  

```bash
docker commit [OPTIONS] CONTAINER [REPOSITORY[:TAG]]
```

**Options:**
- `--author(-a)`: author를 설정
- `--change(-c)`: 추가로 적용할 `Dockerfile` 명령어 설정
- `--message(-m)`: Commit message
- `--pause(-p):` : Commit 동안 컨테이너 중지(기본값 true)

> **주의사항**
> 
> - Production 에서 사용할 이미지라면 `commit` 대신 `Dockerfile`을 기반으로 제작하는 것이 좋다.  
> - 볼륨( volume )에 저장된 데이터는 포함되지 않는다.  
> - --pause 옵션을 설정하지 않은 경우, `commit` 하는 동안 컨테이너를 중지( pause )된다.  
> - `--change` 에서 지원하는 명령어는 다음과 같다:
>     - `CMD`
>     - `ENTRYPOINT`
>     - `ENV`
>     - `EXPOSE`
>     - `LABEL`
>     - `ONBUILD`
>     - `USER`
>     - `VOLUME`
>     - `WORKDIR`

### 연습: Commit을 이용하여 패키지가 추가로 설치된 이미지 생성

`ubuntu:22.04`이미지로부터 base라는 이름의 컨테이너 생성
```bash
docker run -itd --name base ubuntu:22.04
```

`curl``이 있는지 확인
```bash
docker exec base apt list --installed "curl"
                                                                                                                                                                                           WARNING: apt does not have a stable CLI interface. Use with caution in scripts.

Listing...
```

`curl`설치
```bash
docker exec base /bin/bash -c "apt-get update && apt-get upgrade -y && apt-get install -y curl"
Get:1 http://security.ubuntu.com/ubuntu jammy-security InRelease [129 kB]
Get:2 http://archive.ubuntu.com/ubuntu jammy InRelease [270 kB]
Get:3 http://security.ubuntu.com/ubuntu jammy-security/multiverse amd64 Packages [48.5 kB]
Get:4 http://security.ubuntu.com/ubuntu jammy-security/restricted amd64 Packages [4763 kB]
Get:5 http://archive.ubuntu.com/ubuntu jammy-updates InRelease [128 kB]
Get:6 http://archive.ubuntu.com/ubuntu jammy-backports InRelease [127 kB]
Get:7 http://archive.ubuntu.com/ubuntu jammy/multiverse amd64 Packages [266 kB]
Get:8 http://archive.ubuntu.com/ubuntu jammy/main amd64 Packages [1792 kB]
```

`curl`이 있는지 확인
```bash
docker exec base apt list --installed "curl"
                                                                                                                                                                                           WARNING: apt does not have a stable CLI interface. Use with caution in scripts.

Listing...                                                                                                                                                                                 
curl/jammy-updates,jammy-security,now 7.81.0-1ubuntu1.20 amd64 [installed]
```

base 컨테이너는 commit:v1으로 저장
```bash
docker commit base commit:v1                                                                                                                                                             
sha256:3a9f99c95d9812a5be56da9cad3c3950e2d20ada22eb8ab6b85efe4735e53ddb
```

`commit:v1`으로부터 새로운 컨테이너를 생성하여 `curl`이 있는지 확인
```bash
docker run --name restore commit:v1 apt list --installed "curl"

Listing...                                                                                                                                                                                 
curl/jammy-updates,jammy-security,now 7.81.0-1ubuntu1.20 amd64 [installed]
```



### 연습: 실행중인 컨테이너의 Port를 추가로 Expose하기

컨테이너를 하나 생성하고, 포트가 닫힌걸 확인
```bash
➜ docker run -it -d --name base ubuntu:22.04
016c47b24b5c553f4031380ca474918db296f58f5235589a5777e9dbc0200e7b

~ …
➜ docker ps
CONTAINER ID   IMAGE          COMMAND       CREATED         STATUS         PORTS     NAMES
016c47b24b5c   ubuntu:22.04   "/bin/bash"   4 seconds ago   Up 3 seconds             base
```

포트를 개방하며 무중단으로 커밋
```bash
~ …
➜ docker commit --change="EXPOSE 80" --pause=false base commit:v1
sha256:5fd59b5ef350c53be96c55f3a2eab5667b6d1fccc80172337d40aaa02c6fce3f

```

새로운 컨테이너를 열고, 포트가 열려있는지 확인
```bash
~ …
➜ docker run -itd commit:v1
490f7f67188e8a96ffdd67109fb8d389a8d47d590906801f6111150bcb0fcae4

~ …
➜ docker ps
CONTAINER ID   IMAGE          COMMAND       CREATED          STATUS          PORTS     NAMES
490f7f67188e   commit:v1      "/bin/bash"   3 seconds ago    Up 3 seconds    80/tcp    epic_brahmagupta
016c47b24b5c   ubuntu:22.04   "/bin/bash"   57 seconds ago   Up 56 seconds             base

```

`ubuntu:22.04`의 inspect
![Inspect of ubuntu:22.04](inspect-ubuntu.png)

`commit:v1`의 inspcet
![Inspect of commit:v1](inspect-commit.png)

---
## 🚀 Docker buildx
최근에는 arm아키텍처가 랩탑뿐만 아니라, 서버에서도 점차 쓰여가고 있다.  
멀티플랫폼 빌드에 대한 수요가 점점 늘어나고 있다.  

그러나, 기존의 `docker build`는:
- 단일 플랫폼만 지원
- 빌드 캐시 공유 어려움
- 병렬 빌드 지원 안함
- 클라우드 환경에서 최적화되지 않음
이라는 문제점들을 가지고 있다.  

그래서, 멀티플랫폼 빌드와 캐시 최적화 등을 지원하는 고급 빌드 도구로 `docker buildx`가 등장했다.  

buildx는 여러 builder를 제공하는데, 다음과 같다:
- `docker`: 기존 방식. 로컬에서 이미지를 만든다.
위에서의 문제점들을 다 가지고 있다.
- `docker-container`: `BuildKit`이 별도 컨테이너로 실행된다.  
병렬 빌드 및 멀티 플랫폼이 지원된다. 
- `kubernetes`: 쿠버네티스 클러스터에서 실행된다.  
클러스터의 `Pod`에서 동작한다.
- `remote`: `BuildKit`이 원격에 있고, 여러 개발자가 같은 빌드 머신을 쓸 수 있다.  

---
## 📋 builder 목록 확인
```bash
docker buildx ls
```


---
## 🐣 builder 생성
```bash
docker buildx create [OPTIONS] [CONTEXT|ENDPOINT]
```

**Options:**
- `--driver`: 사용할 드라이버 설정
- `--driver-opt`: 드라이버에 따른 옵션 설정
- `--name`: builder 이름 지정
- `--platform`: platform지정
- `--use`: 생성 후 해당 builder사용

### Driver별 특징

|Feature\Driver | `docker` | `docker-container` | `kubernetes` | `remote` |
| --------------- | --------------- | --------------- | --------------- | --------------- |
|이미지 자동 로드 | O | X | X | X |
|멀티 플랫폼 | X | O | O | O |
|캐시 공유 | X | O | O | O |
|CI/CD적합성 | X | O | O | O |
|사용 환경 | 로컬 개발, 테스트 | 실무 빌드, 로컬 CI | K8s기반 환경 | 원격 빌드 머신 |


---
## 🔍 builder 정보 조회
```bash
docker buildx inspect [NAME]
```

`--bootstrap`을 붙이면 현재 builder를 볼 수 있다.

---
## 🫵 builder 선택
```bash
docker buildx use [OPTIONS] NAME
```
**Options:**
- `--default`: default로 설정

---
## 🏗️ 이미지 빌드
```bash
docker buildx build [OPTIONS] PATH | URL | -

**Options:**
- `--build-arg`: `ARG`를 설정
- `--file(-f)`: `Dockerfile`의 경로 지정
- `--label`: 라벨 추가
- `--no-cache`: 캐시 사용 안함
- `--platform`: `platform` 지정
- `--pull`: 관련된 이미지를 반드시 `pull`
- `--load`: 이미지를 `host`에 저장
- `--push`: 이미지를 `registry`에 저장
- `--tag(-t)`: 이름과 `Tag`설정
```

> **주의사항**
> - `docker` 드라이버를 쓰는 builder는 단일 `platform`이미지만 가능  
> - `docker` 드라이버가 아닌 다른 드라이버를 쓰는 경우, ``--load``를 써야 이미지를 가져올 수 있음  
> - 2개 이상의 `platform`을 지원하는 이미지를 제작하는 경우, `--load`가 불가능, `--push`를 이용해 `registry`로 업로드 해야 함  

---
## 💪 실습

### linux/arm/v7용 이미지 제작

우선, 현재 builder에서 지원하는 platform을 확인해보자.
```bash
docker buildx inspect --bootstrap
Name:          default
Driver:        docker
Last Activity: 2025-07-08 05:44:24 +0000 UTC

Nodes:
Name:             default
Endpoint:         default
Status:           running
BuildKit version: v0.23.2
Platforms:        linux/amd64, linux/amd64/v2, linux/amd64/v3, linux/arm64, linux/riscv64, linux/ppc64le, linux/s390x, linux/386, linux/arm/v7, linux/arm/v6
Labels:
 org.mobyproject.buildkit.worker.moby.host-gateway-ip: 172.17.0.1
```

`--platform` 옵션으로 `linux/arm/v7`이미지를 제작하자.  
그 후, 확인해보자. 
```bash
docker buildx build --platform linux/arm/v7 -t cloudwave:arm

docker image inspect cloudwave:arm -f "arch: {{ .Architecture }}/{{ .Variant }}"
➜ docker image inspect cloudwave:arm -f "arch: {{ .Architecture }}/{{ .Variant }}"
arch: arm/v7
```

### buildx를 이용한 multi-platform 이미지 제작

`docker-container`드라이버를 이용하는 builder를 만들어보자.  
이름은 `dockerbuilder`라고 했다.  
```bash
docker buildx create --driver=docker-container --name dockerbuilder                                                                                                                      
dockerbuilder
```

목록을 조회해보자.  
```bash
docker buildx ls
NAME/NODE             DRIVER/ENDPOINT                   STATUS     BUILDKIT   PLATFORMS
dockerbuilder         docker-container
 \_ dockerbuilder0     \_ unix:///var/run/docker.sock   inactive
laughing_allen        docker-container
 \_ laughing_allen0    \_ unix:///var/run/docker.sock   inactive
default*              docker
 \_ default            \_ default                       running    v0.23.2    linux/amd64 (+3), linux/arm64, linux/arm (+2), linux/ppc64le, (3 more)
```

`dockerbuilder`를 사용하도록 빌더를 바꿔보자.   
```bash
docker buildx use dockerbuilder 
```


우선, `Docker Hub`에 로그인한다.
```bash
docker login
```

`Docker Hub`로 업로드 시키면서, `linux/amd64`, `linux/arm/v7`을 지원하게 해보자.
```bash
docker buildx build --platform linux/arm/v7,linux/amd64 -t riveroverflow/cloudwave:multiplatform.v1 --push .
```

Docker Hub에서 Pull해오거나, `Docker Hub`에서 확인해보자.
```bash
docker pull riveroverflow/cloudwave:multiplatform.v1
```

