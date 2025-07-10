---
title: "Docker Compose - 기본 YAML파일 작성법"
description: "Docker Compose의 기본적인 YAML 파일 작성 방법"
date: 2025-07-10T22:45:20+09:00
image: "docker-logo.png"
math: 
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

## 🏷️ Version
type: `string`

- docker-compose의 버전을 명시한다.
- V1(docker-compose)에서는 사용되지만, V2(docker compose)에서는 사용되지 않는다.

---
## 🪪 Name
type: `string`
- 프로젝트의 이름을 명시한다.

---
## 🧩 Service
type: `map`
프로젝트의 서비스들을 명시한다.
- 자식들로는 각 서비스들이 `map`으로 입력된다.

### image
type: `string`
- 컨테이너에서 쓰일 이미지이다.
- 이미지를 `build`하는 경우 생성된 이미지의 이름으로 사용된다.

### container_name
type: `string`
- 값을 명시하지 않은 경우, 프로젝트, 서비스, 컨테이너 번호를 기반으로 생성된다. 
- 알파벳 대소문자, 숫자, 및 일부 특수기호(., _, -)만 사용 가능하다.
- `container_name`이 설정된 경우, 해당 서비스를 위해 컨테이너를 1개만 생성 가능하다.

### expose
type: `list`

```yaml
expose:
  - 80 # NUMBER
  - "8080" # STRING
  - "1000-1010" # RANGE
```

- 같은 네트워크를 사용하는 서비스에서 접근 가능하지만, host로 publish되지 않는다.
- `-`를 이용하여 노출할 포트의 범위를 지정할 수 있다.

### ports
type: `list`
```yaml
ports:
    # Short Syntax
  - [HOST:]CONTAINER[/PROTOCOL]
  # Long Syntax
  - target: 80
    host_ip: HOST
    published: "8080"
    protocol: tcp
```

- 외부에서 접근할 수 있도록 host로 publish할 포트 정의한다.
- `-`를 이용하여 노출할 포트 범위를 지정한다.

### entrypoint
type: `string | list`
```yaml
# String
entrypoint: String

#List
entrypoint: 
  - String
  - String
```
- `Dockerfile`에서 정의된 `ENTRYPOINT`를 오버라이드한다.
- `ENTRYPOINT`가 선언된 경우, `Dockerfile`에서 정의된 `CMD`는 무시된다.

### command
type: `string | list`
```yaml
# String
command: bundle exec thin -o 3000
# List
command: ["bundle", "exec", "thin", "-p", "3000"]
# List
command:
  - String
  - String
```
- `Dockerfile`의 `CMD`를 오버라이드 한다.
- 값이 `null`인 경우, `Image`의 `command`를 사용한다.
- `[]` 또는 `''` 인 경우, `command`는 `""`로 인정, 즉 `Image`의 `CMD`를 사용하지 않는다.

### environment
type: `map | list`
```yaml
# Map Syntax
environment:
  KEY1: value
  KEY2: "value"
  KEY3:

# List Syntax
environment:
  - KEY1=value
  - KEY2="value"
  - KEY3
```

- `map`또는 `list`형식으로 작성가능하다.
- 값이 정의되지 않은 경우, `env_file`로 제공되지 않으면 `unset`된다.
- `environment`는 `env_file`보다 우선으로 적용된다.

### env_file
type: `string | list`
```yaml
# Single
env_file: .env

# Multiple
env_file:
  - ./a.env
  - ./b.env

```
- 여러 개의 파일에 동일한 변수가 선언된 경우, 가장 **마지막 파일에 있는 값으로 설정된다.**

### restart
type: `string`
- 컨테이너가 종료된 경우 어떻게 처리할 것이지 정의한다.

**Options:**
- `no`: 재실행을 하지 않음
- `always`: 제거되지 않는 한, 컨테이너를 항상 재실행
- `on-failure`: `exit code`가 `0`이 아닌 경우에 재실행
- `unless-stopped`: 종료 또는 삭제를 제외하고 항상 제실행

### build
type: `map`
```yaml
services:
# Short
  frontend:
    image: example/webapp
    build: ./webapp
# Long - dockerfile
  backend:
    image: example/database
    build:
      context: backend
      dockerfile: ../backend.Dockerfile
      args:
        GIT_COMMIT: cdc3b19

build:
  context: .
  dockerfile_inline: |
  FROM baseimage
  RUN some command
```
- 서비스의 컨테이너를 `build`할 때 사용하며, 각 서비스별로 context경로를 설정해준다.
- `dockerfile`대신 `dockerfile_inline`을 이용하여 사용할 수도 있다.

### pull_policy
type: `string`
- 이미지를 어떻게 가져올 것인지 설정한다.

**Options:**
- `always`: 매번 `registry`에서 이미지 다운로드
- `never`: 저장된 이미지만을 사용
- `missing`: 저장된 이미지가 없는 경우에만 레지스트리에서 다운로드
- `build`: 매번 이미지를 `build`

> **주의할 점**  
`scale-out`을 하는 상황에서, `always`나 `missing`을 이용하다가, 같은 tag 이미지이나 다른 digest로 바뀌었다고 하자.  
이러면 레플리카들이 동일하지 않은 컨테이너로 돌아가서, 로드밸런서가 어떻게 분산을 하느냐에 따라 구버전/신버전으로 바뀔 수 있다.
> 

### platform
type: `string`
```yaml
platform: linux/arm64
platform: linux/amd64
```
- 컨테이너의 platform을 명시한다.

---
## 🏃 실습
### ubuntu 서버 실행

```yaml
version: '3.8'

services:
  ubuntu:
    image: cloudwave:example.1
    restart: no
```

```bash
docker compose up -d
```

```bash
root@820fc32a9f60:/code# docker compose ps -a
WARN[0000] /code/docker-compose.yaml: the attribute `version` is obsolete, it will be ignored, please remove it to avoid potential confusion 
NAME            IMAGE                 COMMAND       SERVICE   CREATED          STATUS                      PORTS
code-ubuntu-1   cloudwave:example.1   "/bin/bash"   ubuntu    49 seconds ago   Exited (0) 49 seconds ago   
root@820fc32a9f60:/code# 
```

계속 살리려면 compose파일을 아래와 같이 하여 터미널에 attach 시키거나,
```yaml
version: '3.8'

services:
  ubuntu:
    image: cloudwave:example.1
    tty: true
    stdin_open: true
    restart: no
```

아래와 같이 sleep시키면 된다.

```yaml
version: '3.8'

services:
  ubuntu:
    image: cloudwave:example.1
    entrypoint: "/bin/bash"
    command: 
      - -c
      - sleep 3600
    restart: no
```

### 환경변수를 이용하여 compose file제어
`.env`파일을 아래와 같이 작성한다.

```bash
FROM=".env file"
```

`docker-compose.yml`을 아래와 같이 작성한다.

```yaml
version: '3.8'

services:
  ubuntu:
    image: ubuntu:22.04
    entrypoint: /bin/bash
    command: 
      - -c
      - echo 'env form "$FROM"'
    restart: no
```


`.env`파일은 `compose up`되면서 호스트의 환경변수로 적용된다.  
즉, compose 파일을 도커 엔진이 평문으로 변환해주었다는 뜻이다.
```bash
root@820fc32a9f60:/code# docker compose up
WARN[0000] /code/docker-compose.yaml: the attribute `version` is obsolete, it will be ignored, please remove it to avoid potential confusion 
[+] Running 1/1
 ✔ Container code-ubuntu-1  Re...                           0.1s 
Attaching to ubuntu-1
ubuntu-1  | env form ".env file"
ubuntu-1 exited with code 0
root@820fc32a9f60:/code# 
```

호스트에서 `FROM`의 환경변수를 `host`로 바꾸고, 다시 `up`을 해보자.

```bash
root@820fc32a9f60:/code# export FROM="host"
root@820fc32a9f60:/code# docker compose up
WARN[0000] /code/docker-compose.yaml: the attribute `version` is obsolete, it will be ignored, please remove it to avoid potential confusion 
[+] Running 1/1
 ✔ Container code-ubuntu-1  Re...                           0.1s 
Attaching to ubuntu-1
ubuntu-1  | env form "host"
ubuntu-1 exited with code 0
root@820fc32a9f60:/code# 
```

`${KEY:-DEFAULT_VALUE}`를 붙이면, 기본 환경변수를 설정 가능하다.

`${BY:-default}`를 붙이고 한번 더 돌려보자.

```yaml
version: '3.8'

services:
  ubuntu:
    image: ubuntu:22.04
    entrypoint: /bin/bash
    command: 
      - -c
      - echo 'env from "$FROM"' && echo 'env from ${BY:-default}' 
    restart: no
```



### command에서 컨테이너 환경변수 사용

아래의 compose파일을 보자.
```yaml
version: '3.8'

services:
  ubuntu:
    image: ubuntu:22.04
    environment:
      - FROM="env definition"
    entrypoint: /bin/bash
    command: 
      - -c
      - echo 'env from ${FROM}' && echo env from $${FROM}
    restart: no
```

```bash
root@820fc32a9f60:/code# docker compose up
WARN[0000] /code/docker-compose.yaml: the attribute `version` is obsolete, it will be ignored, please remove it to avoid potential confusion 
[+] Running 1/1
 ✔ Container code-ubuntu-1  Re...                           0.1s 
Attaching to ubuntu-1
ubuntu-1  | env from host
ubuntu-1  | env from "env definition"
ubuntu-1 exited with code 0
root@820fc32a9f60:/code# 
```

docker inspect에서도 확인할 수 있다:

```bash
root@820fc32a9f60:/code# docker inspect code-ubuntu-1 -f "{{ .Args }}"ble Watch
[-c echo 'env from host' && echo env from ${FROM}]
```

우리는 아래와 같은 결론을 지을 수 있다:
- `${ENV}`로 되어있는 환경 변수는 호스트의 환경 변수
- `$${ENV}`로 되어있는 환경 변수는 컨테이너의 환경 변수

호스트 환경변수 우선순위는 다음과 같다:
1. `export`로 선언한 환경변수
2. `.env`파일의 환경변수

컨테이너 환경변수 우선순위는 다음과 같다:
1. `compose`파일의 `environment`
2. `env_file`의 리스트에서 가장 마지막 선언

### docker compose에서 build 사용
아래의 compose파일을 확인해보자.
```yaml
version: '3.8'

services:
  ubuntu:
    container_name: server
    build:
      context: .
      dockerfile_inline: |
        FROM ubuntu:22.04
        RUN apt-get update
        RUN apt-get upgrade -y
        RUN apt-get install -y curl
    image: cloudwave/compose:inline_build.v1
    restart: no
  
  nginx:
    image: nginx:latest
    expose:
      - 80
    restart: always
```

`--build`옵션을 넣어서 빌드하면서 프로젝트를 시작하자.
```bash
root@820fc32a9f60:/code# docker compose -p ex1 up -d --build
WARN[0000] /code/docker-compose.yaml: the attribute `version` is obsolete, it will be ignored, please remove it to avoid potential confusion 
[+] Building 22.5s (10/10) FINISHED                              
 => [internal] load local bake definitions                  0.0s
 => => reading from stdin 415B                              0.0s
 => [internal] load build definition from Dockerfile        0.0s
 => => transferring dockerfile: 125B                        0.0s
 => [internal] load metadata for docker.io/library/ubuntu:  0.0s
 => [internal] load .dockerignore                           0.0s
 => => transferring context: 2B                             0.0s
 => CACHED [1/4] FROM docker.io/library/ubuntu:22.04        0.0s
 => [2/4] RUN apt-get update                                8.3s
 => [3/4] RUN apt-get upgrade -y                            3.5s
 => [4/4] RUN apt-get install -y curl                      10.3s
 => exporting to image                                      0.2s
 => => exporting layers                                     0.2s
 => => writing image sha256:b09f8cea39d2204e166b0fa1856a98  0.0s
 => => naming to docker.io/cloudwave/compose:inline_build.  0.0s
 => resolving provenance for metadata file                  0.0s
[+] Running 3/3
 ✔ ubuntu                 Built                             0.0s 
 ✔ Container ex1-nginx-1  Starte...                         0.3s 
 ✔ Container server       Started                           0.4s 
root@820fc32a9f60:/code# 
```


---
## 📦 Volumes

```yaml
volumes:
  volume_key:
```

### 볼륨 정의

볼륨들은 `volumes`라는 최상위 프로퍼티의 자식들로 정의디된다.

```yaml
volumes:
  db-data: #volume_key
    name: "db-volume"
    external: true
    labels: 
      com.example.description: "Database volume"
```

**Name**  
type: `string`
- 볼륨의 이름이다.
- 설정되지 않은 경우, `{project_name}_{volume_key}`

**External**  
type: `bool`
- 생성되어 있는 기존 볼륨 사용 여부이다.
- 해당 볼륨이 생성되어 있지 않은 경우, 프로젝트가 실행될 수 없다.
- `name`이 지정되어 있지 않은 경우, `volume`에 정의된 key가 사용된다.
    - `db-volume`이 명시되지 않으면, `db_data`란 이름의 `volume` 탐색한다.

**Labels**
- 관리를 위한 라벨 설정이다.

> **주의 사항**
> - `volume`을 정의하더라도, `service`에서 사용하지 않는 경우, 생성되지 않는다.
> 

### 볼륨 사용

```yaml
services:
  backend:
    image: example/database
    volumes:
    - db-data:/etc/data
		
  backup:
    image: backup-service
    volumes:
	## long syntax
      - type: volume
      source: mydata
      target: /data
      volume:
      nocopy: true
      reda_only: true
	## short syntax
      - db-data:var/lib/backup/data:rw
```

**Short Syntax:**
```yaml
volumes:
  - VOLUME:CONTAINER_PATH:ACCESS_MODE
```

- `VOLUME`: host의 경로 또는 volume이름이다.
- `CONTAINER_PATH`: 볼륨이 마운트될 컨테이너의 경로이다.
- `ACCESS_MODE`: 해당볼륨의 Access mode 설정이다.
    - `rw`: 읽기쓰기 모두 가능
    - `ro`: 읽기전용

**Long Syntax:**
```yaml
volumes:
  - type: volume
    source: mydata
    target: /data
    read_only: true
```
- `type`: mount type 설정
    - `volume`: source로 volume 사용
    - `bind`: source로 host directory 사용
- `source`: volume 또는 directory설정
- `target`: mount될 directory 설정
- `read_only`: 읽기전용 설정


---
## 🏃 실습

### External volume 사용

미리 볼륨이 생성되어 있어야 한다.

```bash
docker volume create vault
```

```yaml
version: '3.8'
name: 'volume-external'

services:
  master:
    image: ubuntu:22.04
    entrypoint: /bin/bash
    command:
      - -c
      - sleep infinity

    volumes:
      - vault:/root/vault

volumes:
  vault:
    external: true
    name: 'vault'
```

### read_only로 설정하여 사용

`master` 서비스와 `slave`서비스를 하나씩 만들자.
```yaml
# docker-compose.yaml
version: '3.8'
name: 'volume-external'

services:
  master:
    image: ubuntu:22.04
    entrypoint: /bin/bash
    command:
      - -c
      - sleep infinity
    volumes:
      - vault:/root/vault:rw
  slave:
    image: ubuntu:22.04
    entrypoint: /bin/bash
    command:
      - -c
      - sleep infinity
    volumes:
      - vault:/root/vault:ro

volumes:
  vault:
    external: true
    name: 'vault'

```

`master`에서 파일을 만들어보자

```yaml
root@820fc32a9f60:/code# docker exec volume-external-master-1 /bin/bash -c "echo master > /root/vault/temp.txt"
root@820fc32a9f60:/code# docker exec volume-external-master-1 /bin/bash -c "cat /root/vault/temp.txt"
master
```

`slave`에서는 파일을 읽을 수는 있지만, 파일을 생성할 수는 없다.

```yaml
root@820fc32a9f60:/code# docker exec volume-external-slave-1 /bin/bash -c "cat /root/vault/temp.txt"
master
root@820fc32a9f60:/code# docker exec volume-external-slave-1 /bin/bash -c "echo slave > /root/vault/temp.txt"
/bin/bash: line 1: /root/vault/temp.txt: Read-only file system
root@820fc32a9f60:/code# 
```

### volumes_from으로 volume사용하기

이전 예제의 프로젝트를 실행하는 채로, 아래 yaml대로 새로운 프로젝트를 열어 적용한다.

```yaml
# docker-compose.yaml
version: '3.8'
name: 'volume-external2'

services:
  other:
    image: ubuntu:22.04
    entrypoint: /bin/bash
    command:
      - -c
      - sleep infinity
    volumes_from:
      - container:volume-external-slave-1:ro
```

`up`으로 적용한다.
```yaml
root@820fc32a9f60:/code# docker compose -f ccompose/docker-compose.yaml up -d
WARN[0000] /code/ccompose/docker-compose.yaml: the attribute `version` is obsolete, it will be ignored, please remove it to avoid potential confusion 
[+] Running 2/2
 ✔ Network volume-external2_default    Created              0.0s 
 ✔ Container volume-external2-other-1  Started              0.3s 
root@820fc32a9f60:/code# 
```

볼륨 정의 없이 다른 프로젝트의 볼륨에 접속하여 사용하는것을 볼 수 있다.
```yaml
root@820fc32a9f60:/code# docker exec volume-external2-other-1 /bin/bash -c "cat /root/vault/temp.txt"
master
root@820fc32a9f60:/code# docker exec volume-externa2-other-1 /bin/bash -c "echo other > /root/vault/temp.txt"
Error response from daemon: No such container: volume-externa2-other-1
root@820fc32a9f60:/code# docker exec volume-external2-other-1 /bin/bash -c "echo other > /root/vault/temp.txt"
/bin/bash: line 1: /root/vault/temp.txt: Read-only file system
root@820fc32a9f60:/code# 
```

---
## 🌐 Networks

최상위 프로퍼티에 네트워크들을 정의한다.

```yaml
networks:
  network1:
  network2: 
```

### 네트워크 정의하기

```yaml
networks:
  private: #network_key
    name: "private_net"
    external: true
    labels:
      com.example.description: "private network"
```

**Name**
- 네트워크의 이름이다.
- 설정되지 않은 경우 `{project_name}_{network_key}`이다.

**External**
- 생성되어 있는 기존 네트워크 사용 여부를 명시한다.
- 해당 네트워크가 생성되어 있지 않은 경우, 프로젝트가 실행될 수 없다.
- `name`이 지정되어 있지 않은 경우, networks에 정의된 key가 사용된다.
    - `private_net`이 명시되지 않으면, `private`란 이름을 가진 `network` 를 탐색한다.

> **주의사항**  
> - network를 정의하였더라도 `service`에서 사용하지 않는 경우, 생성되지 않는다.
> 

### 네트워크 사용
```yaml
services:
  backend:
    image: example/database
    # List Syntax
    networks:
      - private
  backup:
    image: backup-service
    # Map Syntax
    networks:
      private:
        aliases:
          - alias1
  bastion:
    image: bastion
    network_mode: host
```

- 정의된 네트워크는 서비스의 `networks`에서 사용가능하다.
- `host`또는 `none`을 사용하려는 경우, `network_mode`를 이용하여 설정한다.

> **주의사항**  
> - 1개 이상의 컨테이너에서 동일한 Alias를 사용할 경우, 응답할 컨테이너를 특정지을 수 없다.  
> 즉, 로드밸런싱 된다.
> 

---
## 🏃 실습

### host네트워크를 사용하는 컨테이너 만들기

우분투를 하나 띄운다.
```yaml
# docker-compose.yaml
version: '3.8'
name: 'network-host'
services:
  ubuntu:
    image: ubuntu:22.04
    entrypoint: /bin/bash
    command:
      - -c
      - sleep infinity
    network_mode: host

```

```bash
root@820fc32a9f60:/code# docker compose up -d
WARN[0000] /code/docker-compose.yaml: the attribute `version` is obsolete, it will be ignored, please remove it to avoid potential confusion 
[+] Running 1/1
 ✔ Container network-host-ubuntu-1  Started                 0.2s 
root@820fc32a9f60:/code# 
```

`nginx`를 별도 실행한다.

```bash
 docker run -d -p 80:80 --name nginx nginx:latest
```

우분투에서 `curl`을 설치한다.

```bash
docker exec network-host-ubuntu-1 /bin/bash -c "apt-get update && apt-get upgrade -y && apt-get install -y curl"
```

`nginx`는 포트포워딩 되어있고, 우분투는 호스트 네트워크를 사용하므로, 우분투는 nginx에 요청을 보낼 수 있다.  

```bash
root@820fc32a9f60:/code# docker exec network-host-ubuntu-1 /bin/bash -c "curl localhost:80"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- -100   615  100   615    0     0   283k      0 --:--:-- --:--:-- --:--:--  600k
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
root@820fc32a9f60:/code# 
```

### 이미 생성된 네트워크 사용하기

`bridge`드라이버를 사용하는 네트워크 생성한다.

```bash
docker network create private -d bridge
```

아래와 같은 프로젝트 생성한다.

```yaml
# docker-compose.yaml
version: '3.8'
name: 'network-external'

services:
  ubuntu:
    image: ubuntu:22.04
    entrypoint: /bin/bash
    command:
      - -c
      - sleep infinity
    networks:
      private:

networks:
  private:
    name: "private"
    external: true
```

이미 존재하는 네트워크가 있어서 잘 실행된다.

```yaml
root@820fc32a9f60:/code# docker compose up -d
WARN[0000] /code/docker-compose.yaml: the attribute `version` is obsolete, it will be ignored, please remove it to avoid potential confusion 
[+] Running 1/1
 ✔ Container network-external-ubuntu-1  Started             0.4s 
root@820fc32a9f60:/code# 
```

`name`을 제거하여도, `key`기반으로 서치가 되어 프로젝트가 정상 진행된다.

```yaml
# docker-compose.yaml
version: '3.8'
name: 'network-external'

services:
  ubuntu:
    image: ubuntu:22.04
    entrypoint: /bin/bash
    command:
      - -c
      - sleep infinity
    networks:
      private:

networks:
  private:
    external: true
```

```bash
root@820fc32a9f60:/code# docker compose up -d
WARN[0000] /code/docker-compose.yaml: the attribute `version` is obsolete, it will be ignored, please remove it to avoid potential confusion 
[+] Running 1/1
 ✔ Container network-external-ubuntu-1  Running             0.0s 
root@820fc32a9f60:/code# 
```

그러나, 이름을 바꾸면, 오류가 발생한다.

```yaml
# docker-compose.yaml
version: '3.8'
name: 'network-external'

services:
  ubuntu:
    image: ubuntu:22.04
    entrypoint: /bin/bash
    command:
      - -c
      - sleep infinity
    networks:
      private:

networks:
  private:
    name: "my-private"
    external: true
```

```bash
root@820fc32a9f60:/code# docker compose up -d
WARN[0000] /code/docker-compose.yaml: the attribute `version` is obsolete, it will be ignored, please remove it to avoid potential confusion 
network my-private declared as external, but could not be found
root@820fc32a9f60:/code# 
```

### alias 설정하기

아래와 같은 compose를 작성한다.

```yaml
# docker-compose.yaml
version: '3.8'
name: 'network-alias'

services:
  ubuntu:
    image: ubuntu:22.04
    entrypoint: /bin/bash
    command:
      - -c
      - sleep infinity
    networks:
      private:

networks:
  private:
```

실행하고 확인한다.

```bash
root@820fc32a9f60:/code# docker compose up -d
WARN[0000] /code/docker-compose.yaml: the attribute `version` is obsolete, it will be ignored, please remove it to avoid potential confusion 
[+] Running 2/2
 ✔ Network network-alias_private     Created                0.1s 
 ✔ Container network-alias-ubuntu-1  Started                0.4s 
root@820fc32a9f60:/code# docker inspect network-alias-ubuntu-1 | jq ".[0].NetworkSettings.Networks" | jq -r '.
[].DNSNames'
[
  "network-alias-ubuntu-1",
  "ubuntu",
  "cfb812c501f9"
]
root@820fc32a9f60:/code# 
```

alias로 server를 추가한다.

```yaml
# docker-compose.yaml
version: '3.8'
name: 'network-alias'

services:
  ubuntu:
    image: ubuntu:22.04
    entrypoint: /bin/bash
    command:
      - -c
      - sleep infinity
    networks:
      private:
        aliases:
          - server

networks:
  private:
```

다시 확인한다.

```bash
root@820fc32a9f60:/code# docker compose up -d
WARN[0000] /code/docker-compose.yaml: the attribute `version` is obsolete, it will be ignored, please remove it to avoid potential confusion 
[+] Running 1/1
 ✔ Container network-alias-ubuntu-1  Started               10.6s 
root@820fc32a9f60:/code# docker inspect network-alias-ubuntu-1 | jq ".[0].NetworkSettings.Networks" | jq -r '.
[].DNSNames'
[
  "network-alias-ubuntu-1",
  "ubuntu",
  "server",
  "0fe78f331322"
]
root@820fc32a9f60:/code# 
```

### 생성된 network를 이용하여 서로 다른 프로젝트의 서비스와 연결

- `across_project` 네트워크 생성
- `network-across-project-1` 프로젝트 생성
    - `ubuntu:22.04`
    - `curl` 설치
    - `across_project`네트워크의 alias를 `main`으로
- `network-across-project-2` 프로젝트 생성
    - `nginx:latest`
    - `80`을 `expose`
    - `across_project`의 alias를 `web`으로 설정
- `docker network inspect`를 이용하여 각 서비스의 ip확인
- `ubuntu`에서 curl로 `web`호출
    - IP로
    - DNS로

1. 네트워크를 생성한다.

```bash
docker network create -d bridge across_project

root@820fc32a9f60:/code# docker network ls -f name=across
NETWORK ID     NAME             DRIVER    SCOPE
05ce4356725f   across_project   bridge    local
```

`network-across-project-1`를 구성한다.

```yaml
version: '3.8'

name: network-across-project-1

services:
  main:
    image: "ubuntu:22.04"
    command:
      - "/bin/bash"
      - "-c"
      - "apt-get update && apt-get upgrade -y && apt-get install -y curl && sleep 3600" # All commands in one string
    networks:
      across_project:
        aliases:
          - main
networks:
  across_project:
    name: "across_project"
    external: true
```

`network-across-project-2`를 구성한다.

```yaml
version: '3.8'

name: network-across-project-2

services:
  main:
    image: "nginx:latest"
    expose:
      - 80
    networks:
      across_project:
        aliases:
          - web
  
networks:
  across_project:
    name: "across_project"
    external: true
```

둘다 실행해본다.

```bash
root@820fc32a9f60:/code# docker compose up -d
WARN[0000] /code/docker-compose.yaml: the attribute `version` is obsolete, it will be ignored, please remove it to avoid potential confusion 
[+] Running 1/1
 ✔ Container network-across-project-1-main-1  Started       0.3s 
root@820fc32a9f60:/code# docker compose -f ccompose/docker-compose.yaml up -d
WARN[0000] /code/ccompose/docker-compose.yaml: the attribute `version` is obsolete, it will be ignored, please remove it to avoid potential confusion 
[+] Running 1/1
 ✔ Container network-across-project-2-main-1  Started       0.3s 
root@820fc32a9f60:/code# 
```

IP를 확인해보자.

```bash
root@820fc32a9f60:/code# docker network inspect across_project | jq '.[].Containers'
{
  "9cf2b9e8f369be15c54b2deda383fa60427375bee605e4fe668de768e0e73eb9": {
    "Name": "network-across-project-1-main-1",
    "EndpointID": "0fd80eed36449a5e903b7ab36e8ddacb163b245507f7c5815cd8c27ee13875d9",
    "MacAddress": "22:55:18:c6:38:b3",
    "IPv4Address": "172.24.0.2/16",
    "IPv6Address": ""
  },
  "ee149bbe733a59c3fae3ec9aa86c46e272d4f35e600cccfd150df24aaeab7326": {
    "Name": "network-across-project-2-main-1",
    "EndpointID": "bcfcc6a9636b3699084bbf731a8d38f5e9829fb894fedfb51e8dc0e51c5245f4",
    "MacAddress": "ba:78:6b:d1:c8:d9",
    "IPv4Address": "172.24.0.3/16",
    "IPv6Address": ""
  }
}
root@820fc32a9f60:/code# 
```

IP, hostname으로 모두 요청할 수 있다, 둘은 같은 네트워크에 속해 있다.


## ⚙️ Config & Secret

|  | **Config** | **Secret** |
| --- | --- | --- |
| **목적** | 외부에서 서버 설정을 위한 변수 제공 | 비밀번호와 같은 민감한 데이터 제공 |
| **기본 Mount위치** | `/<config-name>` | `/run/secrets/<secret_name>` |
| **암호화 여부** | X | Swarm의 경우 O, 그 외 X |

**Config vs Mount**

- 컨테이너 외부에서 파일을 제공한다는 점에서 유사하다.
- Volume과는 달리, Config는 Mode를 이용하여 파일의 권한을 세부적으로 관리한다.
