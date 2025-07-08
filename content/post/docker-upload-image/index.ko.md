---
title: "Docker 이미지 업로드하기"
description: Docker CLI에서 이미지를 업로드하는 법
date: 2025-07-08T21:40:04+09:00
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

이전까지 우리는 이미지를 내려받고, 컨테이너를 실행하고, 이미지를 빌드하는 법을 배웠다.  
이제, 이미지를 업로드하는 방법을 알아보자.  

## 🏷️ 이미지 태그 변경
이미지의 태그를 변경한다. 
```bash
docker tag SOURCE_IMAGE[:TAG] TARGET_IMAGE[:TAG]
```

---
## ⬆️ 이미지 업로드
```bash
docker push [OPTIONS] NAME[:TAG]
```

**Options:**
- `-all-tags(-a)`: 레포지토리에 이미지의 모든 태그를 푸시한다.

> **주의사항**
> - Docker hub가 아닌, ECR이나 자체 구축한 Hub와 같이 별도의 레지스트리에 업로드하려는 경우, 이름은 **full image name**형식으로 작성되어야 한다.  
> - Push 이전에 `docker login`을 통해서 사용할 registry에 인증해야 한다.

### Full image name 포맷
```bash
[HOST[:PORT_NUMBER]/]PATH
# namespace를 명시하는 경우
[HOST[:PORT_NUMBER]/][NAMESPACE/]REPOSITORY
```
- `HOST`: registry의 Host
- `PORT_NUMBER`: registry서버의 포트
- `NAMESPACE`: 논리적 구분 단위. Docker Hub에서는 Namespace가 필수임.
    - 없으면 library
- `REPOSITORY`: 이미지의 이름

ECR 예시는 다음과 같다:  
```plaintext
<aws_account_id>.dkr.ecr.ap-northeast-2.amazonaws.com/myapp:v1
```
ECR과 같은 경우는, URL에 이미 논리적인 구분이 적용되어있어서 `NAMESPACE`를 사용하지 않음. 

Private Registry 예시는 다음과 같다:
```plaintext
my.registry.com:5000/mynamespace/myapp:v1
```

---

## 🔑 Docker Hub 레지스트리에 로그인
```bash
docker login [OPTIONS] [SERVER[:PORT]]
```

**Options:**
- `--password(-p)`: password
- `--password-stdin`: `STDIN`으로 password를 입력
- `--username(-u)`: username


GitHub Container Registry에 로그인하기:
```bash
docker login ghcr.io
Username: <your_username>
Password: <Fine-grained personal access token>
Login Succeeded
```

Docker Hub에 업로드하기:  
```bash
docker push riveroverflow/cloudwave:base.v1
The push refers to repository [docker.io/riveroverflow/cloudwave]
8016e9d19af3: Pushed
882955112ee7: Pushed
3fd98b9cc2bb: Pushed
5f70bf18a086: Pushed
7922986690e5: Pushed
98eb01e614a6: Pushed
8d6b7eb76b62: Mounted from library/ubuntu
base.v1: digest: sha256:a71d873faaedddf9b1dd5d71bbe941b19fc0293f04ebb370e5075f861c1a3cd5 size: 1780
```

`docker search`로 조회: 
```bash
docker search riveroverflow/cloudwave
NAME                                    DESCRIPTION   STARS     OFFICIAL
riveroverflow/cloudwave                               0
```
