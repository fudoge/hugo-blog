---
title: "Docker Compose 기초"
description: "Docker Compose 기본 CLI 가이드"
date: 2025-07-10T00:09:40+09:00
image: docker-logo.png
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
## ⚙️ YAML
**YAML(YAML Ain't Markup Language)** 은 사람이 읽기 좋은 계층적 구조의 데이터 표현 포맷이다.  
**JSON**에 비해서 매우 가독성이 좋다.

`키: 값`의 형식을 기본적으로 따르고, CI/CD, DevOps, k8s, 설정 파일 등에서 유용하게 쓰일 수 있는 문법이다.  

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
## 🖥️ Compose CLI
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
> - `-f`를 사용할 compose파일을 지정하지 않는 경우, `docker-compose.yml` 또는 `docker-compose.yaml`이 사용된다.
> - `Docker compose` 파일의 `version`에 따라서 사용할 수 있는 기능과 변수명이 달라질 수 있다.   
> 따라서, 서버에 설치된 Docker Engine의 버전과 호환되는지 체크하는 것이 좋다.
> - 프로젝트 이름이 설정되지 않은 경우, 현재 디렉토리 이름이 프로젝트 이름으로 설정된다.

---
## ☑️ Docker compose 버전 확인
```bash
docker compose version [OPTIONS]
```

---
## 🏁 프로젝트 실행
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

---
## 📋 프로젝트 목록
```bash
docker compose ls [OPTIONS]
```

**Options:**
- `--all(-a)`: 종된 프로젝트를 포함한 모든 프로젝트를 표시
- `--filter`: 필터 조건 추가
- `--quiet(-q)`: 프로젝트 이름만 표기

---
## 📋 컨테이너 목록
```bash
docker compose ps [OPTIONS] [SERVICE...]
```

**Options**
- `--all(-a)`: 종료된 컨테이너를 포함한 모든 컨테이너 표시
- `--quiet(-a)`: 컨테이너 ID
- `--services`: 서비스만 표기
- `--status`: 상태값을 기반으로 서비스 필터링
- `--project(-p)`: 프로젝트 명시

> **주의사항**
> - CLI를 통해 전달된 `project-name`또는 `compose file`에 명시된 프로젝트명을 기반으로 조회하므로, 유의하자.  
> 즉, `docker compose`에서는 -p의 사용을 습관화하는게 중요하다.

---
## 📋 이미지 목록
```bash
docker compose images [OPTIONS] [SERVICE...]
```

---
## ❌ 프로젝트 삭제
```bash
docker compose down [OPTIONS] [SERVICES]
```

**Options:**
- `--remove-orphans`: `compose` 파일에 정의되지 않은 서비스를 구성하는 컨테이너도 삭제
- `--volumes(-v)`: `compose`파일에 정의된 `anonymous` 볼륨을 같이 삭제

> **주의 사항**
> - 현재 디렉토리에 `docker-compose.yaml`파일이 존재하지 않는 경우, `-p` 또는 `-f`를 이용하여야 한다.
> - `--remove-orphans`를 사용하지 않을 경우, `compose` 파일에 정의된 서비스를 구성하는 컨테이너만 삭제
> - `external`로 정의된 `network`와 `volume`은 삭제되지 않는다.
> - 다른 프로젝트에서 사용중인 리소스(`volume`, `network`)는 삭제되지 않는다.


예제: 삭제 시 volume도 함께 제거하기
```yaml
# docker-compose.yaml
version: '3.8'
name: 'myproject'

services:
  success:
    image: ubuntu:22.04
    entrypoint: /bin/bash
    command:
      - -c
      - sleep infinity
    networks:
      private: {}
    volumes:
      - anonymous:/anonymous
      - named:/named

networks:
  private:

volumes:
  anonymous:
  named:
    name: "named_volume"

```


```bash
docker compose -p myproject down --volumes
[+] Running 4/4                                                                                                                                                                            
 ✔ Container myproject-success-1  Removed 10.3s  
 ✔ Volume myproject_anonymous     Removed 0.0s  
 ✔ Volume named_volume            Removed 0.0s  
 ✔ Network myproject_private      Removed
```




---
## 🔁 재실행
```bash
docker compose restart [OPTIONS] [SERVICE...]
```

> **주의사항**
> - 서비스를 명시하지 않은 경우, 모든 서비스에 대해서 종료된 컨테이너를 포함하여 재실행  
> - 반영되지 않은 변경사항이 `compose` 파일에 존재하더라도, 해당 변경사항은 반영되지 않는다.  

---
## 🚥 종료 및 시작
```bash
# 종료
docker compose stop [OPTIONS] [SERVICE...]
# 시작
docker compose start [SERVICE...]
```

이렇게 되면, 서브프로세스들은 다시 살아나지 않는다.  

___
## ⏯️ 중지 및 실행
```bash
# 중지
docker compose pause [OPTIONS] [SERVICE...]
# 실행
docker compose unpause [SERVICE...]
```

절전모드를 생각하면 된다.  
다시 꺠우면 서브프로세스들도 깨어난다.  

---
## 🏗️ 서비스 빌드하기
```bash
docker compose build [OPTIONS] [SERVICE...]
```

`yaml`에 명시된 서비스의 image의 이름으로 하여 이미지가 생성된다.  


**Options:**
- `--build-arg`: 빌드시에 사용할 `ARG`를 설정
- `--builder`: 사용할 builder 설정
- `--no-cache`: 이미지 빌드시 캐시 사용하지 않음
- `--pull`: 항상 이미지를 새로 다운로드
- `--push`: 서비스의 이미지 `push`


예시: `docker compose`로 빌드하기
```yaml
# docker-compose.yaml
version: "3.8"

services:
  server:
    image: compose:build.v1
    build:
      dockerfile_inline: |
        FROM ubuntu:22.04
        RUN apt-get update && apt-get -y upgrade
        # Or Use `dockerfile`
        # dockerfile: "server.dockerfile"
    entrypoint: /bin/bash
    command:
      - -c
      - "sleep 3600"
    restart: no

```

```bash
docker compose build
```


---
## 📝 프로젝트 로그 확인
```bash
docker compose logs [OPTIONS] [SERVICE...]
```

**Options:**
- `--follow(-f)`: 계속 로그를 표시
- `--tail(-n)`: 마지막 n개의 로그 표시 **(컨테이너 당)**
    - 기본값: `all`
- `--timestamps(-t)`: timestamp표기
- `--since`: 특정시간 이후 발생한 로그만 표시
- `--until`: 특정시간 이전 발생한 로그만 표시


---
## 📊 프로세스 확인
```bash
docker compose top [SERVICES...]
```

---
## 🔍 Config 확인
```bash
docker compose config [OPTIONS] [SERVICE...]
```

실제 프로젝트의 `config`가 보인다.
즉, 기본 설정들과 같은 명세들도 작성되어 있다.
```bash
root@c56b293b8e91:/code# docker compose -p myproject config
WARN[0000] /code/docker-compose.yaml: the attribute `version` is obsolete, it will be ignored, please remove it to avoid potential confusion 
name: myproject
services:
  server:
    build:
      context: /code
      dockerfile_inline: |
        FROM ubuntu:22.04
        RUN apt-get update && apt-get -y upgrade
        # Or Use `dockerfile`
        # dockerfile: "server.dockerfile"
    command:
      - -c
      - sleep 3600
    entrypoint:
      - /bin/bash
    image: compose:build.v1
    networks:
      default: null
    restart: "no"
networks:
  default:
    name: myproject_default
```

---
## 😵 종료된 서비스 삭제
```bash
docker compose rm [OPTIONS] [SERVICE...]
```

**Options:**
- `--force(-f)`: 물어보지 않고 종료
- `--stop(-s)`:
- `--volumes(-v)`

```bash
root@c56b293b8e91:/code# docker compose up -d
[+] Running 2/2
 ✔ Container cloudwave-fail-1     Started                  0.3s 
 ✔ Container cloudwave-success-1  Started                  0.2s 
root@c56b293b8e91:/code# docker compose ps -a
WARN[0000] /code/docker-compose.yaml: the attribute `version` is obsolete, it will be ignored, please remove it to avoid potential confusion 
NAME                  IMAGE          COMMAND                  SERVICE   CREATED              STATUS                      PORTS
cloudwave-fail-1      ubuntu:22.04   "/bin/bash -c 'sleep…"   fail      About a minute ago   Exited (1) 18 seconds ago   
cloudwave-success-1   ubuntu:22.04   "/bin/bash -c 'sleep…"   success   About a minute ago   Up 18 seconds
```

```bash
root@c56b293b8e91:/code# docker compose -p cloudwave rm 
? Going to remove cloudwave-fail-1 Yes
[+] Removing 1/1
 ✔ Container cloudwave-fail-1  Removed  0.0s
```

---
## 🏋️ 실습

### 프로젝트 실행 및 Network 확인

아래 파일을 `example.yaml`로 저장하자.

```yaml
version: '3.8'

services:
  ubuntu:
    image: ubuntu:22.04
    entrypoint: /bin/bash
    command:
      - -c
      - "apt-get update && apt-get -y upgrade && apt-get install -y curl && sleep 3600"
    restart: no

  nginx:
    image: nginx:latest
    expose:
      - 80
    restart: always
```

명령어로 프로젝트를 실행해보자.
```bash
root@c56b293b8e91:/code# docker compose -f example.yaml -p ex1 up -d
[+] Running 3/3
 ✔ Network ex1_default     Create...                       0.0s 
 ✔ Container ex1-ubuntu-1  Sta...                          0.3s 
 ✔ Container ex1-nginx-1   Star...                         0.4s
```

보면, `ex1_default`라는 네트워크가 새로 생성된 것을 확인할 수 있다.  
또한, `service`의 이름과 컨테이너의 이름이 alias로 등록되어 있는 것을 볼 수 있다.  
```bash
root@c56b293b8e91:/code# docker inspect ex1-ubuntu-1 -f "{{ println .NetworkSettings.Networks }}"
map[ex1_default:0xc000000000]

root@c56b293b8e91:/code# docker inspect ex1-ubuntu-1 -f "Alias:{{ println
.NetworkSettings.Networks.ex1_default.Aliases }}IP:{{ println
.NetworkSettings.Networks.ex1_default.IPAddress }}"
Alias:[ex1-ubuntu-1 ubuntu]
IP:172.19.0.2
```

`ubuntu-1`로 접속해서, `nginx`에 `curl`을 날려보자.
```bash
root@c56b293b8e91:/code# docker exec -it ex1-ubuntu-1 /bin/bash
root@23b9a83c0bba:/# curl nginx
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

### 프로젝트 업데이트: 서비스 포트 추가
위에서의 nginx는 외부 포트가 연결되지 않아 host에서 접근할 수 없다.
```bash
curl localhost:80                                                                                                                                                                        
curl: (7) Failed to connect to localhost port 80 after 0 ms: Connection refused
```

host에서 접근할 수 있도록 `example.yaml`을 수정한다.
```yaml
version: '3.8'

services:
  ubuntu:
    image: ubuntu:22.04
    entrypoint: /bin/bash
    command:
      - -c
      - "apt-get update && apt-get -y upgrade && apt-get install -y curl && sleep 3600"
    restart: no

  nginx:
    image: nginx:latest
    ports:
      - 80:80
    restart: always
```

다시 up을 해서 업데이트한다.
```bash
docker compose -f example.yaml -p ex1 up -d
```

호스트에서 실행해보자.
```bash
curl localhost:80
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

```

### 이미지 빌드 후 프로젝트 실행
`server.dockerfile`을 아래와 같이 작성하자:
```dockerfile
FROM ubuntu:22.04
RUN apt-get update && apt-get -y upgrade
```

`docker-compose.yaml`을 다음과 같이 작성해보자:
```yaml
version: '3.8'

services:
  ubuntu:
    image: cloudwave:example.1
    build:
      dockerfile: "server.dockerfile"
    entrypoint: /bin/bash
    command:
      - -c
      - "sleep 3600"
    restart: no

  nginx:
    image: nginx:latest
    ports:
      - "80:80"
    restart: always
```

`--build`옵션으로 프로젝트를 실행하자.
```bash
root@820fc32a9f60:/code# docker compose -p cloud_wave up -d --build
WARN[0000] /code/docker-compose.yaml: the attribute `version` is obsolete, it will be ignored, please remove it to avoid potential confusion 
[+] Building 0.4s (8/8) FINISHED                                 
 => [internal] load local bake definitions                  0.0s
 => => reading from stdin 325B                              0.0s
 => [internal] load build definition from server.dockerfil  0.0s
 => => transferring dockerfile: 102B                        0.0s
 => [internal] load metadata for docker.io/library/ubuntu:  0.0s
 => [internal] load .dockerignore                           0.0s
 => => transferring context: 2B                             0.0s
 => [1/2] FROM docker.io/library/ubuntu:22.04               0.0s
 => CACHED [2/2] RUN apt-get update && apt-get -y upgrade   0.0s
 => exporting to image                                      0.0s
 => => exporting layers                                     0.0s
 => => writing image sha256:a211fa2032c9e4d25754c88a87e737  0.0s
 => => naming to docker.io/library/cloudwave:example.1      0.0s
 => resolving provenance for metadata file                  0.0s
[+] Running 3/3
 ✔ ubuntu                         Built                     0.0s 
 ✔ Container cloud_wave-nginx-1   Started                   0.4s 
 ✔ Container cloud_wave-ubuntu-1  Started                   0.4s 
root@820fc32a9f60:/code#
```


### 프로젝트 삭제

현재 `compose`파일이 다음과 같다고 해보자:
```yaml
version: '3.8'

services:
  ubuntu:
    image: cloudwave:example.1
    build:
      dockerfile: "server.dockerfile"
    entrypoint: /bin/bash
    command:
      - -c
      - "sleep 3600"
    restart: no

  nginx:
    image: nginx:latest
    ports:
      - "80:80"
    restart: always
```

여기서, `nginx`를 제외해보자.
```yaml
version: '3.8'

services:
  ubuntu:
    image: cloudwave:example.1
    build:
      dockerfile: "server.dockerfile"
    entrypoint: /bin/bash
    command:
      - -c
      - "sleep 3600"
    restart: no
```

이 상태로 `down`을 해보자.
```bash
root@820fc32a9f60:/code# docker compose down
WARN[0000] /code/docker-compose.yaml: the attribute `version` is obsolete, it will be ignored, please remove it to avoid potential confusion 
[+] Running 2/2
 ✔ Container code-ubuntu-1  Re...                          10.3s 
 ! Network code_default     Resou...                        0.0s 
root@820fc32a9f60:/code#
```

`nginx`는 남아있음을 볼 수 있다.
```bash
root@820fc32a9f60:/code# docker compose ps
WARN[0000] /code/docker-compose.yaml: the attribute `version` is obsolete, it will be ignored, please remove it to avoid potential confusion 
NAME           IMAGE          COMMAND                  SERVICE   CREATED              STATUS          PORTS
code-nginx-1   nginx:latest   "/docker-entrypoint.…"   nginx     About a minute ago   Up 41 seconds   0.0.0.0:80->80/tcp, [::]:80->80/tcp
root@820fc32a9f60:/code#
```

`--volumes`와 `--remove-orphans` 옵션을 이용하면, 남아있는 컨테이너와 볼륨을 삭제할 수 있다.
