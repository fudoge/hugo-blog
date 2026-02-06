---
title: "Triton Inference Server: Optimization"
description: Triton Inference Server에서 Perf Analyzer로 성능을 튜닝해보자
date: 2026-01-10T22:48:09+09:00
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

Triton Inference Server에서는 지연시간을 줄이고, 처리량을 늘리는 여러 기능이 있다.  
우선은, 단일 모델에서의 지연과 처리량 트레이드오프를 이해하는데 집중한다.  
- [Model Analyzer](https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/user_guide/model_analyzer.html)는 단일 GPU에서 여러 모델들을 최적으로 올릴지에 대한 고민으로 GPU 메모리 사용량의 이해를 돕는다.  

이미 클라이언트 애플리케이션이 있다고 해도, [Performance Analyzer](https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/perf_analyzer/README.html)에 익숙해질 필요가 있다.    
Performance Analyzer는 모델의 성능을 최적화하는데 기본적인 도구이다.  
예시에서는 [QuickStart](https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/getting_started/quickstart.html)를 따라해서도 쉽게 시작할 수 있는 ONNX Inception 모델을 쓴다.


---
## 🎠 기본적인 실행 해보기

`perf-analyzer`를 실행해본다.
```bash
# 모델: inception_onnx
# url: 클러스터 내 tritonserver svc(현재 replica 1개임)
# percentile: 95(95%의 요청이 최소 몇 초 내로 끝나는지)
# in-flight동시요청: 1~4개까지 늘려보기
root@perf-analyzer-6bdc45d864-w4ttw:/workspace# perf_analyzer \
	-m inception_onnx \
	-u <triton_server_ip>:8000 \
	--percentile=95 \
	--concurrency-range=1:4
Inferences/Second vs. Client p95 Batch Latency
Concurrency: 1, throughput: 205.285 infer/sec, latency 4910 usec
Concurrency: 2, throughput: 229.494 infer/sec, latency 8644 usec
Concurrency: 3, throughput: 229.91 infer/sec, latency 12978 usec
Concurrency: 4, throughput: 229.833 infer/sec, latency 17323 usec
```

참고로, 공식 문서에서의 예시는 아래와 같은 수치가 나온다:
```bash
$ perf_analyzer -m inception_graphdef --percentile=95 --concurrency-range 1:4
...
Inferences/Second vs. Client p95 Batch Latency
Concurrency: 1, throughput: 62.6 infer/sec, latency 21371 usec
Concurrency: 2, throughput: 73.2 infer/sec, latency 34381 usec
Concurrency: 3, throughput: 73.2 infer/sec, latency 50298 usec
Concurrency: 4, throughput: 73.4 infer/sec, latency 65569 usec
```

현재 초당 230개정도의 추론을 처리한다.   
1개의 동시요청에서는 처리량이 약간 적은데, 다음 요청을 받기까지의 유휴가 생기기 때문이다.  
2개의 동시요청부터는 처리량이 오르는데, 다른 통신동안 작업이 겹치기 때문이다.

---
## 🚀 최적화 세팅

대부분의 모델에서 Triton 성능 개선에 큰 도움을 주는 기능은 동적 배칭이다.   
그러나, 모델이 지원하지 못할 수도 있다.

### Dynamic Batcher
Dynamic Batcher는 단일의 추론 요청들을 큰 배치로 묶어서 효율적으로 추론한다.
동적 배치를 추가해주면 된다.

```protobuf
dynamic_batching { }
```

dynamic batcher는 많은 양의 동시 요청을 견디도록 해준다
```protobuf
dynamic_batching {
  perferred_batch_size: [4, 8, 16]
  max_queue_delay_microseconds: 100 # 100ms
}
```

성공적으로 batching이 된다면, latency가 늘어나지 않으면서도 처리량이 늘어나는것을 볼 수 있다.   
아래는 공식문서에서의 예제이다. 공식문서에서는 quickstart 모델들로 할 수 있다고 하는데, 정작 `inception_graphdef`는 not found여서 따라할 수는 없었다.

```bash
$ perf_analyzer -m inception_graphdef --percentile=95 --concurrency-range 1:8
...
Inferences/Second vs. Client p95 Batch Latency
Concurrency: 1, throughput: 66.8 infer/sec, latency 19785 usec
Concurrency: 2, throughput: 80.8 infer/sec, latency 30732 usec
Concurrency: 3, throughput: 118 infer/sec, latency 32968 usec
Concurrency: 4, throughput: 165.2 infer/sec, latency 32974 usec
Concurrency: 5, throughput: 194.4 infer/sec, latency 33035 usec
Concurrency: 6, throughput: 217.6 infer/sec, latency 34258 usec
Concurrency: 7, throughput: 249.8 infer/sec, latency 34522 usec
Concurrency: 8, throughput: 272 infer/sec, latency 35988 usec
```

`perf_analyzer`와 서버를 같은 시스템에서 띄울 때, 다음의 휴리스틱 기반의 간단한 룰을 이용하면, 쉽게 목표값을 구할 수 있다고 한다.
- 최소 지연을 구하려면, `concurrency=1`, dynamic batching 비활성화, 모델 instance 1개로만 세팅해서 보기
- 최대 처리량을 보려면, dynamic batching을 가능하다면 활성화하고, `instance_count`를 정한 뒤, `concurrency = 2 * max_batch_size * instance_count`

```bash
$ perf_analyzer -m inception_graphdef --percentile=95 --concurrency-range 8
...
Inferences/Second vs. Client p95 Batch Latency
Concurrency: 8, throughput: 267.8 infer/sec, latency 35590 usec
```


### Model Instances
Triton은 각 모델이 추론 인스턴스를 몇 개를 띄울지를 정하게 할 수 있다.   
기본적으로는 1개를 가짖지만, `instance group`을 이용해서 여러 개의 인스턴스를 만들 수 있다.   
일반적으로 두 개의 인스턴스를 쓸 때 처리량이 늘어날 수 있다.   
CPU 와 GPU의 전환작업 및 겹치는 연산, 또는 작은 모델이 GPU 자원을 전부 쓰지 못하는 경우에 효과적이다.

아래의 규칙을 `config.pbtxt`에 넣고 서버를 재시동한다.
```protobuf
instance group [ { count: 2 } ]
```

다시 `perf_analyzer`를 돌렸더니, 아래와 같다고 한다.(이번에도 File Not Found로 직접 따라할 순 없었다)
```bash
$ perf_analyzer -m inception_graphdef --percentile=95 --concurrency-range 1:4
...
Inferences/Second vs. Client p95 Batch Latency
Concurrency: 1, throughput: 70.6 infer/sec, latency 19547 usec
Concurrency: 2, throughput: 106.6 infer/sec, latency 23532 usec
Concurrency: 3, throughput: 110.2 infer/sec, latency 36649 usec
Concurrency: 4, throughput: 108.6 infer/sec, latency 43588 usec
```

1개의 인스턴스일때 대비 처리량이 늘어난 것을 볼 수 있다.   


Dynamic Batching과 instance group 복제를 둘 다 적용시킬 수도 있다.
```protobuf
dynamic_batching { }
instance_group [ { count: 2 }]
```

아래 수치도 공식 문서에서의 예제이다.
```bash
$ perf_analyzer -m inception_graphdef --percentile=95 --concurrency-range 16
...
Inferences/Second vs. Client p95 Batch Latency
Concurrency: 16, throughput: 289.6 infer/sec, latency 59817 usec
```
dynamic batcher + 1개 인스턴스의 경우보다 더 별로인 것으로 보인다.   
왜냐하면 이 모델이 dynamic batcher를 쓰는 것만으로 자원을 충분히 많이 쓰는 탓에 성능적인 이점을 보지 못하는 것이다.   
인스턴스를 늘려서 더 얻는 것은 없고, 오히려 스케줄링 및 경합으로 더 느려질 수도 있다.

성능 개선으로 dynamic batcher이냐 model instance 복제이냐는 모델에 따라 다르다.   
sweet-spot을 찾는 것은 많은 경험 및 실험이 필요하다.


---
## 🎢 추가 최적화

NUMA노드, ONNX with TensorRT, ONNX with OpenVIVO에서의 최적화 기법들이 있다고 하는데, 이는 당장 지금 보진 않았다.   
보고싶다면 [여기](https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/user_guide/optimization.html#framework-specific-optimization)에서 참조..


---
## 💨 Hands-on

간단하지만 좀 무거운 선형회귀 모델을 만들어줄 것이다.  
아래의 구조를 만들어주자.
```bash
model_repository
└── linear_model
    ├── 1
    │   └── model.pt
    └── config.pbtxt
```

### 모델 레포지토리 셋업
이 코드를 실행했을때의 `model.pt`를 레포지토리에 넣는다.
```python
import torch
import torch.nn as nn

class LinearModel(torch.nn.Module):
    def __init__(self):
        super().__init__()
        self.layers = torch.nn.Sequential(
            *[torch.nn.Linear(1, 1) for _ in range(1000)]
        )

    def forward(self, x):
        for layer in self.layers:
            x = layer(x)
        return x


model = LinearModel()
model.eval()

example_input = torch.randn(1, 1)
scripted_model = torch.jit.trace(model, example_input)

scripted_model.save("model.pt")
```

`config.pbtxt`를 다음과 같이 저장한다:
```protobuf
name: "linear_model"
platform: "pytorch_libtorch"

input [
  {
    name: "INPUT"
    data_type: TYPE_FP32
    dims: [1]
  }
]

output [
  {
    name: "OUTPUT"
    data_type: TYPE_FP32
    dims: [1]
  }
]
```

### `docker-compose.yaml`
```yaml
services:
  triton:
    container_name: triton
    image: nvcr.io/nvidia/tritonserver:25.09-py3
    volumes:
      - ./model_repository:/models
    command:
      - tritonserver
      - --model-repository=/models
    ports:
      - 8000:8000
      - 8001:8001
      - 8002:8002
  perf_analyzer:
    container_name: perf_analyzer
    image: nvcr.io/nvidia/tritonserver:25.09-py3-sdk
    command: 
      - sleep
      - infinity
```

이후, 컨테이너들을 실행한다.
```bash
docker compose up -d
```

### 동적 배치 없이 테스트

`perf_analyzer`를 실행한다.
```bash
docker exec -it perf_analyzer /bin/bash
```

1~4로 concurrency를 올려보아도, 처리량이 늘어나는 데에는 한계가 있다.   
오히려 latency만 선형적으로 늘어나고 있다.
```bash
perf_analyzer -m linear_model -u triton:8000 --percentile=95 --concurrency-range=1:4 --measurement-interval 15000
(...)
Inferences/Second vs. Client p95 Batch Latency
Concurrency: 1, throughput: 465.277 infer/sec, latency 2548 usec
Concurrency: 2, throughput: 541.307 infer/sec, latency 4175 usec
Concurrency: 3, throughput: 535.462 infer/sec, latency 6284 usec
Concurrency: 4, throughput: 519.308 infer/sec, latency 8732 usec
```

concurrency를 16까지 올려보자.
```bash
perf_analyzer -m linear_model -u triton:8000 --percentile=95 --concurrency-range=16 --measurement-interval 15000
(...)
Inferences/Second vs. Client p95 Batch Latency
Concurrency: 16, throughput: 506.553 infer/sec, latency 35164 usec
```

### 동적 배치를 포함한 테스트

이제, 동적 배치를 추가해보자. 기존 컨테이너들을 내린다.
```bash
docker compose down
```

`config.pbtxt`를 아래와 같이 바꾼다:
```protobuf
name: "linear_model"
platform: "pytorch_libtorch"

max_batch_size: 32 

input [
  {
    name: "INPUT"
    data_type: TYPE_FP32
    dims: [1]
  }
]

output [
  {
    name: "OUTPUT"
    data_type: TYPE_FP32
    dims: [1]
  }
]

dynamic_batching {
  preferred_batch_size: [4, 8, 16]
  max_queue_delay_microseconds: 100
}
```

다시 컨테이너들을 올려준다.
```bash
docker compose up -d
```

다시 `perf_analyzer`에 접속한다.
```bash
docker exec -it perf_analyzer /bin/bash
```

concurrency를 1~4까지 늘려보자.   
latency가 크게 늘어나지 않으면서도 처리량이 늘어나는 것을 볼 수 있다.
```bash
perf_analyzer -m linear_model -u triton:8000 --percentile=95 --concurrency-range=1:4 --measurement-interval 15000
(...)
Inferences/Second vs. Client p95 Batch Latency
Concurrency: 1, throughput: 518.028 infer/sec, latency 2287 usec
Concurrency: 2, throughput: 655.595 infer/sec, latency 3397 usec
Concurrency: 3, throughput: 945.922 infer/sec, latency 3525 usec
Concurrency: 4, throughput: 1200.72 infer/sec, latency 3694 usec
```

concurrency를 16까지 올려보자.   
```bash
perf_analyzer -m linear_model -u triton:8000 --percentile=95 --concurrency-range=16 --measurement-interval 15000
(...)
Inferences/Second vs. Client p95 Batch Latency
Concurrency: 16, throughput: 4141.76 infer/sec, latency 5251 usec
```

---
## 📚 References
- [Optimization](https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/user_guide/optimization.html)
- [Perf Analyzer](https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/perf_analyzer/docs/cli.html)