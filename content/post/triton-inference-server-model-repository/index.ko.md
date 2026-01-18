---
title: "Triton Inference Server: Model Repository"
description: 
date: 2026-01-04T15:04:40+09:00
image: tritonserver-logo.png
math: 
license: 
hidden: false
comments: true
draft: false

tags:
    - Conatiner
    - TritonServer
categories:
    - TritonServer
---


---
## 🍰 Repository Layout

레포지토리 경로는 서버 시작 시, `--model-repository`를 통해서 정해진다.
이 옵션은 여러 번 사용되어 여러 모델 레포지토리에 연결되도록 할 수 있다.

모델 레포지토리는 다음과 같은 형식이어야 한다:

```bash
tritonserver --model-repositry=<model-repository-path>
```

```bash
  <model-repository-path>/
    <model-name>/
      [config.pbtxt]
      [<output-labels-file> ...]
      [configs]/
        [<custom-config-file> ...]
      <version>/
        <model-definition-file>
      <version>/
        <model-definition-file>
      ...
    <model-name>/
      [config.pbtxt]
      [<output-labels-file> ...]
      [configs]/
        [<custom-config-file> ...]
      <version>/
        <model-definition-file>
      <version>/
        <model-definition-file>
      ...
    ...
```


모델 레포지토리 디렉토리에는 0개 이상의 서브디렉토리를 가진다.   
서브디렉토리는 상응하는 모델에 대한 레포지토리 정보를 담는다.   
`config.pbtxt` 파일은 모델의 configuration정보를 담는다.   
일부 모델에서는 필수적으로 작성해야하지만, 자동 생성이 지원되는 경우도 있다.   

각 디렉토리는 적어도 1개 이상의 모델 버전을 표현하는 서브디렉토리를 가져야 한다.   
각 모델은 특정 벡엔드에서 실행된다.   
각 서브디렉토리 버전 안에는 벡엔드에 따른 요구되는 파일들이 있다.   
예를 들면, PyTorch Backend를 쓰는 경우, `model.pt`가 요구된다.

---
## 🗺️ Model Repository의 위치

로컬 경로, 또는 클라우드 스토리지에 접근할 수 있다.

### Local Filesystem
모델 경로는 절대경로로 주어져야 한다.
```bash
tritonserver --model-repository=/path/to/model/repository
```


### Cloud Storage

#### Google Cloud
Google Cloud Stroage에 있는 모델 레포지토리에 접근하기 위해서, `gs://` prefix로 시작하는 경로를 이용한다.
```bash
tritonserver --model-repository=gs://bucket/path/to/model/repository ...
```

자격 증명은 다음의 순서로 적용된다:
1. `GOOGLE_APPLICATION_CREDENTIALS` 환경변수
	- 환경 변수가  credential JSON 파일을 포함한 장소를 가져야 한다.
	- Authorized user credential먼저 시도되고, 그 이후 service account credential을 이용한다.
2. Attach된 service account
3. Anonymous Credential(public bucket)
   - 이 경우, 버킷과 그 안의 객체들은 모든 사용자들에게 `get`과 `list`권한이 주어져야 한다.

기본적으로, Triton은 원격 모델 레포지토리로부터 모델을 받아서 로컬의 임시 저장소에 저장한다.   
서버가 종료되면, 지워진다.   
리모트 모델 레포지토리가 로컬에 저장되어있게 하도록 하고 싶으면, `TRITON_GCS_MOUNT_DIRECTORY` 환경변수를 지정해주면 된다.
```bash
export TRITON_GCS_MOUNT_DIRECTORY=/path/to/your/directorty
```

로컬 머신에서 존재하는 경로이며, 비어있도록 보장해줘야 한다.

#### S3
(Cloudflare R2, MinIO도 S3 API를 이용하면 된다)  
`s3://` prefix를 이용하면 S3 API를 이용한다.
```bash
tritonserver --model-repository=s3://bucket/payh/to/model/repository ...
```

로컬이나 프라이빗 인스턴스인 경우(MinIO, Ceph등), host와 port를 따라적으면 된다.
```bash
tritonserver --model-repository=s3://http://host:port/bucket/path/to/model/repository...
```

기본적으로, Triton은 HTTP를 이용한다.   
S3가 HTTPS를 지원하고, HTTPS를 이용하게 하고 싶으면, `https://`로 주면 된다.
```bash
tritonserver --model-repository=s3://https://host:port/bucket/path/to/model/repository...
```

S3를 사용할때, credential과 기본 리전은 aws config 또는 환경변수를 통해 받는다.   
환경변수가 더 높은 우선순위를 가진다.

AWS에서 모델 레포지토리가 카피되길 원한다면, `TRITON_AWS_MOUNT_DIRECTORY`환경변수에 설정해주면 된다.   
이것도 역시 디렉토리가 존재하고 비어있도록 해주자.
```bash
export TRITON_AWS_MOUNT_DIRECTORY=/path/to/your/local/directory
```


### Azure Storage

Azure에서는, `as://` prefix를 쓰면 된다.
```bash
tritonserver --model-repository=as://account_name/container_name/path/to/model/repository ...
```

Azure Storage를 쓸때에는, `AZURE_STORAGE_ACCOUNT`와 `AZURE_STORAGE_KEY` 환경변수를 써서 접근할 수 있도록 해야한다.

Azure Storage에서 모델 레포지토리가 카피되길 원한다면, `TRITON_AZURE_MOUNT_DIRECTORY`환경변수에 설정해주면 된다.   
이것도 역시 디렉토리가 존재하고 비어있도록 해주자.
```bash
export TRITON_AZURE_MOUNT_DIRECTORY=/path/to/your/local/directory
```

### `cloud_credential.json`
Triton에서 단일 파일에 자격증명을 저장하려면, `TRITON_CLOUD_CREDENTIAL_PATH`환경변수값으로 경로를 넣어주면 된다.
```bash
export TRITON_CLOUD_CREDENTIAL_PATH="cloud_credential.json"
```

아래와 같은 형식을 가진다:
```json
{
  "gs": {
    "": "PATH_TO_GOOGLE_APPLICATION_CREDENTIALS",
    "gs://gcs-bucket-002": "PATH_TO_GOOGLE_APPLICATION_CREDENTIALS_2"
  },
  "s3": {
    "": {
      "secret_key": "AWS_SECRET_ACCESS_KEY",
      "key_id": "AWS_ACCESS_KEY_ID",
      "region": "AWS_DEFAULT_REGION",
      "session_token": "",
      "profile": ""
    },
    "s3://s3-bucket-002": {
      "secret_key": "AWS_SECRET_ACCESS_KEY_2",
      "key_id": "AWS_ACCESS_KEY_ID_2",
      "region": "AWS_DEFAULT_REGION_2",
      "session_token": "AWS_SESSION_TOKEN_2",
      "profile": "AWS_PROFILE_2"
    }
  },
  "as": {
    "": {
      "account_str": "AZURE_STORAGE_ACCOUNT",
      "account_key": "AZURE_STORAGE_KEY"
    },
    "as://Account-002/Container": {
      "account_str": "",
      "account_key": ""
    }
  }
}
```

가장 긴 prefix 매칭이 되는대로 자격 증명을 사용한다.
위의 예시에서는, `gs://gcs-bucket-002/model_repositry`인경우는 "gs://gcs-bucket-002"에, 나머지는 ""에 있는 자격증명을 이용할것이다.  
`TRITON_CLOUD_CREDENTIAL_PATH`환경변수가 셋되어있지 않다면, 기존 방법으로 자격증명을 이용할것이다.

### 클라우드 스토리지 캐싱
현재 파일 캐싱을 진행하지는 않지만, 프록시를 주입하여 repository agent API를 통해 구현될 수 있다.  
로컬디렉토리 먼저 체크하고, 클라우드 스토리지로부터 불러오는 전략을 사용할 수 있다.

---
## 🏴󠁩󠁤󠁳󠁬󠁿 모델 버전
각 모델은 하나 이상의 버전을 가져야 한다.  
숫자로 명명되어야 하며, 0으로 시작하거나 숫자가 아닌 서브디렉토리는 무시된다.

---
## 📁 모델 파일
각 backend별로 요구사항이 다르다.


## 📚 References
https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/user_guide/model_repository.html