---
title: "Docker Compose - YAML파일 고급 기능"
description: "Docker Compose의 고급 YAML 설정"
date: 2025-07-11T01:09:58+09:00
image: docker-logo.png
math: true
license: 
hidden: false
comments: true
draft: true
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
