---
title: "Docker Compose - YAML파일 고급 기능"
description: "Docker Compose의 고급 YAML 설정"
date: 2025-07-11T01:09:58+09:00
image: docker-logo.png
math: true
license: 
hidden: false
comments: true
draft: false
---

## 📎 Anchor & Alias
- Anchor와 Alias를 사용하면, **반복적으로 사용되는 요소들을 모듈화** 하여 사용할 수 있다.  
- 재사용을 위한 모듈은 `x-`를 Prefix로 사용하여야 한다.
- `<<:`를 사용하면 yaml을 병합하는 형태이다.

Anchor들은 자신의 형제 프로퍼티들을 모두 포함한다.
Anchor는 항상 프로퍼티들 중 맨 위에 선언되어야 한다.

이 Anchor와 Alias이 처리되는 방식은 다음과 같다:
1. 순차적으로 읽는다.
2. Alias(*)가 보이면, Anchor(&)가 포함하는 범위의 YAML을 가져와 모두 병합한다.  
그러나, 병합할 때, 중복되는 프로퍼티가 있을 수 있다.  
이 경우, Anchor로부터 가져온 프로퍼티가 아닌, 직접 명시한 기존의 프로퍼티가 생존된다.
3. 불러온 뒤, 하위 Alias가 있을 수 있다. 그 부분들 역시 재귀적으로 반복되면 된다.  
4. 이를 모든 Alias가 풀어질 때 까지 반복한다.

아래의 compose 파일이 있다고 하자.
```yaml
version: '3.8'

x-common:
  &common
  restart: always
  volumes:
    - source:/code
  environment:
    &default-env
    BY: "x-common"

x-value: &v1 x

services:
  ubuntu:
    <<: *common
    image: ubuntu:22.04
    environment:
      <<: *default-env
      FROM: "env definition"
      X: *v1
    entrypoint: /bin/bash
    command:
      - -c
      - echo 'env from ${FROM}' && echo env from $${BY}
    restart: no

volumes:
  source:

```

이를 `up`해보자.
```bash
docker compose up -d
```

어떻게 풀어져있는지는 `config`를 보면 알 수 있다.  
`config`는 도커 엔진이 최종적으로 구성한 compose를 보여 주기 때문이다.
```yaml
root@820fc32a9f60:/code# docker compose config
WARN[0000] /code/docker-compose.yaml: the attribute `version` is obsolete, it will be ignored, please remove it to avoid potential confusion 
name: code
services:
  ubuntu:
    command:
      - -c
      - echo 'env from host' && echo env from $${BY}
    entrypoint:
      - /bin/bash
    environment:
      BY: x-common
      FROM: env definition
      X: x
    image: ubuntu:22.04
    networks:
      default: null
    restart: "no"
    volumes:
      - type: volume
        source: source
        target: /code
        volume: {}
networks:
  default:
    name: code_default
volumes:
  source:
    name: code_source
x-common:
  environment:
    BY: x-common
  restart: always
  volumes:
    - source:/code
x-value: x
root@820fc32a9f60:/code#
```

## 🪪 profile
type: `list`
profile을 이용하면, 테스트/스테이지/배포 등의 환경을 쉽게 전환할 수 있다.  
`service`의 프로퍼티로 사용된다.  
`--profile`옵션으로 `up`이 되면, 리스트에 일치하는 `profile`이 있을 때 적용된다.  
이 프로퍼티가 없으면, 포든 프로파일에서 실행된다.

> **주의사항**  
> `down`을 시킬 때에도 `--profile`을 명시해서 컨테이너를 정리해야 한다.  
> 그렇지 않으면, 중요한 상황에서 원치 않는 서비스가 중단되지 않는 문제가 생길 수도 있다.

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:16.1-bullseye
    environment:
      - POSTGRES_PASSWORD=mysecretpassword
  server:
    image: ubuntu:22.04
    stdin_open: true # docker run -i
    tty: true # docker run -t
    depends_on:
      - postgres
  pgadmin:
    image: dpage/pgadmin4:7.4
    environment:
      - PGADMIN_DEFAULT_EMAIL=user@sample.com
      - PGADMIN_DEFAULT_PASSWORD=SuperSecret
    depends_on:
      - postgres
      - server
    profiles:
      - debug
```

`--profile`을 사용하여 프로파일을 명시할 수 있다.

```bash
root@820fc32a9f60:/code# docker compose --profile debug up -d
[+] Running 3/3
 ✔ Container code-postgres-1  Started                       0.2s 
 ✔ Container code-server-1    Started                       0.2s 
 ✔ Container code-pgadmin-1   Started                       0.2s 
root@820fc32a9f60:/code# 
```

`down`을 할 때에도 명시해주자.

```bash
root@820fc32a9f60:/code# docker compose --profile debug down
[+] Running 4/4
 ✔ Container code-pgadmin-1   Removed                       1.2s 
 ✔ Container code-server-1    Removed                      10.3s 
 ✔ Container code-postgres-1  Removed                       0.3s 
 ✔ Network code_default       Rem...                        0.4s 
root@820fc32a9f60:/code# 
```

프로파일을 명시하지 않으면, 기본 서비스들만 실행됨을 볼 수 있다.

`pgadmin`은 이번에 실행되지 않음을 볼 수 있다.

```bash
root@820fc32a9f60:/code# docker compose up -d
[+] Running 3/3
 ✔ Network code_default       Cre...                        0.0s 
 ✔ Container code-postgres-1  Started                       0.3s 
 ✔ Container code-server-1    Started                       0.5s 
root@820fc32a9f60:/code#
```

---
## 🐳 deploy
type: `map`  
컨테이너 운영 환경에서의 정책들을 명시할 수 있다.    
아래는 동일한 Nginx 컨테이너를 3개 만든다.
```yaml
# docker-compose.yaml
name: deploy-replica
services:
  web:
    image: nginx:latest
    expose:
      - 80
    deploy:
      replicas: 3
```

이렇게 `replicas`를 정해주는 경우, `container_name`을 정할 수 없다.  
`container_name`을 쓰면 레플리카들의 이름이 충돌되어 오류이기 때문이다.  

---
## 🫂 depends_on
type: `list`  

선행으로 실행되어야 하는 서비스를 명시한다.  
**Short Syntax** 의 경우, 선행 서비스의 condition이 `running`일때부터 실행 가능하다는 의미이다.  

**Long Syntax:**
- `condition`: 상위 서비스의 상태 조건 정의
    - `service_started`: 서비스가 실행된 상태**(running)**
    - `service_healthy`: 서비스가 healthy인 상태**(실제 사용가능)**
    - `service_completed_successfully`: 서비스가 실행 완료된 상태**(일회성 작업을 기다릴때 주로 사용)**
- `restart`: 상위 서비스가 업데이트 된 경우, 서비스 재실행
- `required`: `false`인 경우 상위 서비스가 실행되지 않더라도 서비스 실행

---
## 💉 healthcheck
type: `map`  
```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost"]
  interval: 1m30s
  timeout: 10s
  retries: 3
  start_period: 40s
  start_interval: 5s
```

서비스의 상태를 확인할 수 있다.
시간은 `{value}{unit}`형태로 작성하고, 다음의 단위를 사용한다:  

- `us(microseconds)`
- `ms(milliseconds)`
- `s(seconds)`
- `m(minutes)`
- `h(hours)`

**Attr:**
- `test`: 컨테이너 상태 체크를 위한 명령어
- `interval`: 상태 체크 간격
- `timeout`: 응답까지 걸리는 최대 시간
- `retries`: 특정 횟수 초과 시 장애로 봄
- `start_period`: 컨테이너 실행까지 소요되는 시간
- `start_interval`: `start_period`내에서의 체크 간격
- `disable`: `HEALTHCHECK`비활성화. `test: [”NONE”]`과 동일

---
## 🏋️ 실습

### Anchor & Alias를 이용하여 yaml작성
```yaml
x-common:
  &common
  restart: always
  volumes:
    - source:/code
  environment:
    &default-env
    BY: "x-common"

services:
  ubuntu:
    <<: *common
    image: ubuntu:22.04
    environment:
      <<: *default-env
      FROM: "env definition"
    entrypoint: /bin/bash
    command:
      - -c
      - echo 'env from ${FROM}' && echo env from $${FROM}
    restart: no

volumes:
  source:

```

`up`하고 `config`를 확인하면 아래와 같다:
```bash
root@820fc32a9f60:/code# docker compose config
name: code
services:
  ubuntu:
    command:
      - -c
      - echo 'env from host' && echo env from $${FROM}
    entrypoint:
      - /bin/bash
    environment:
      BY: x-common
      FROM: env definition
    image: ubuntu:22.04
    networks:
      default: null
    restart: "no"
    volumes:
      - type: volume
        source: source
        target: /code
        volume: {}
networks:
  default:
    name: code_default
volumes:
  source:
    name: code_source
x-common:
  environment:
    BY: x-common
  restart: always
  volumes:
    - source:/code
root@820fc32a9f60:/code#
```

### depends_on을 이용하여 DB생성 후 application 실행
```yaml
services:
  postgres:
    image: postgres:16.1-bullseye
    environment:
      - POSTGRES_PASSWORD=1234
  
  server:
    image: ubuntu:22.04
    stdin_open: true
    tty: true
    depends_on:
      - postgres
  
  pgadmin:
    image: dpage/pgadmin4:7.4
    environment:
      - PGADMIN_DEFAULT_EMAIL=user@sample.com
      - PGADMIN_DEFAULT_PASSWORD=1234
    depends_on:
      - postgres
      - server
    profiles:
      - debug

```



```bash
root@820fc32a9f60:/code# docker compose --profile debug up -d
WARN[0000] Found orphan containers ([code-ubuntu-1]) for this project. If you removed or renamed this service in your compose file, you can run this command with the --remove-orphans flag to clean it up. 
[+] Running 3/3
 ✔ Container code-postgres-1  Running                       0.0s 
 ✔ Container code-server-1    Running                       0.0s 
 ✔ Container code-pgadmin-1   Started                       0.3s 
root@820fc32a9f60:/code# docker compose --profile debug ^Cd
root@820fc32a9f60:/code#
```

### build를 이용한 docker compose 생성 및 레플리카 생성
아래의 `Dockerfile`을 생성한다.
```Dockerfile
FROM ubuntu:22.04

RUN apt-get update
RUN apt-get upgrade -y
RUN apt install -y dnsutils wget
```

아래와 같이 `docker-compose.yaml`을 작성한다.
```yaml
services:
  server:
    image: cloudwave:ubuntu.dig.v1
    build:
      dockerfile: ./Dockerfile
    command: sleep infinity


  web-app:
    image: nginx:latest
    deploy:
      replicas: 3
    expose: 
      - 80
```

실행 후 우분투에서 DNS를 조회해본다.

```bash
docker exec -it server /bin/bash
Error response from daemon: No such container: server

~/build-compose runs 🐙 BBBB …
➜ docker compose exec -it server /bin/bash
root@682c36b0ae3d:/# dig web-app
                                                                                           ; <<>> DiG 9.18.30-0ubuntu0.22.04.2-Ubuntu <<>> web-app
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 45092
;; flags: qr rd ra; QUERY: 1, ANSWER: 3, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;web-app.                       IN      A

;; ANSWER SECTION:
web-app.                600     IN      A       172.27.0.3
web-app.                600     IN      A       172.27.0.5
web-app.                600     IN      A       172.27.0.4

;; Query time: 0 msec
;; SERVER: 127.0.0.11#53(127.0.0.11) (UDP)
;; WHEN: Fri Jul 11 06:08:20 UTC 2025
;; MSG SIZE  rcvd: 94

root@682c36b0ae3d:/#
```
