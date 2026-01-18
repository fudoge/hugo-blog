---
title: "Triton Inference Server Configuration"
description: Triton 서버의 config.pbtxt작성법을 알아보자.  
date: 2026-01-02T23:08:29+09:00
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


Model Repository의 각 모델은 환경 설정이 필요하다.   
`config.pbtxt`로 작성하는데, Modelconfig Protobuf를 의미한다.


---
## ⚙️ 최소한의 Model Configuration

모델에서 platform 과 backend(같이 쓰일수도 있고, 따로 쓰일수도 있다), 최대 배치사이즈, input, output은 반드시 작성되어야 한다. 

아래는 두 개의 입력값 input0과 input1을 받아 output0을 출력하는 모델이다.   
차원은 모두 16엔트리의 float32로 된 텐서이다.

 ```protobuf
 platform: "tensorrt_plan"
 max_batch_size: 8
 
 input [
	 {
		 name: "input0"
		 data_type: TYPE_FP32
		 dims: [16]
	 },
	 {
		 name: "input1"
		 data_type: TYPE_FP32
		 dims: [16]
	 }
 ]
 output [
	 {
		 name: "output0"
		 data_type: TYPE_FP32
		 dims: [16]
	 }
 ]
 ```


--- 
## 🫆 Name, Platform, Backend
Model configuration name은 옵셔널하다.   
명시적으로 모델 이름을 적을 수 있으나, 반드시 폴더의 모델명과 같아야 한다.   
생략가능하다.   

Platform 및 Backend는 각각의 Backend의 구성마다 다르게 적어야 한다.   
예를 들어, Python backend의 경우, 반드시 `backend: "python"`으로 적어야 한다.   

---
## 📐 Model Transaction Policy
`model_transcation_policy`는 모델에서 예상되는 트랙잭션을 기술한다.

### Decoupled
이 불리언값은 모델이 받는 요청수와 응답수가 상이할 수 있음을 나타낸다.   
기본은 `false`이다. 일반적으로는 하나의 요청에 하나의 응답을 답한다는 의미이다.

---
## 🧃 Maximum Batch Size
`max_batch_size` 프로퍼티는 최대 배치 사이즈를 정한다.   
Triton이 dynamic batcher나 sequence batcher를 사용할 경우, 1 이상으로 설정하면 된다.

배치를 지원하지 않는 모델의 경우, 0으로 되어야 한다.

---
## 🛜 Input과 Output

각 모델의 input과 output은 다음을 명시해야 한다:
- `name`
- `datatype`
- `shape`
이들은 모델에서 기대하는 값에 매치해야 한다.

### Pytorch Backend에서의 특별한 이름 관례
TorchScript 모델 파일에서의 불충분한 inputs/outputs의 메타데이터로 인해, `name` 어트리뷰트는 네이밍 컨벤션을 가진다.

1. (Input한정) input이 Tentor의 덱셔너리가 아니면, 모델의 정의의 forward함수의 인자와 같아야 한다.
   예를 들어, Torchscript 모델이ㅡ forward 함수가 `forward(self, input0, input1)`이라면, 첫 번째와 두 번째의 input의 이름은 각각 "input0", "input1"이 되어야 한다.
2. `<name>__<index>`: `<name>`은 아무 string이나 될 수 있고, `<index>`는  대응되는 input/output에 상응하는 위치이ㅡ 인덱스가 될 수 있다.
   두 입력과 출력이 있으면, "INPUT__0", "INPUT__1", "OUTPUT__0", "OUTPUT__1"과 같이 될 수 있다.
3. 만약 모든 입력 또는 출력이 같은 네이밍 컨벤션을 따르지 않는다면, model configurations에서의 엄격한 순서를 따진다.
   즉, configurations에서의 순서가 실제 모델에서의 입력 순서가 된다.

Pytorch backend는 Tensor의 딕셔너리를 입력으로 하는 것 역시 허용한다.  
Dictionary 타입의 단일 입력 모델이 문자열 매핑이 되는 경우에만 지원된다.   
예를 들어, 다음과 같은 형식의 입력을 요구한다.
```protobuf
{'A': tensor1, 'B': tensor2}
```

입력 이름은 위의 네이밍 컨벤션을 따르지 않고, 대신 특정 텐서에 대한 key값을 쓴다.   
여기서는, A는 tensor1을, B는 tensor2를 가리킨다.   


datatype은 모델의 종류에 따라 다르게 할 수 있다.   
input shape은 모델이 기대하는 입력 텐서의 shape을 말한다.   
output shape은 모델이 기대하는 출력 텐서의 shape을 말한다.   
둘다 빈 shape을 허용하지 않는다.   

input과 output shape들은 `max_batch_size`와 input이나 output의 `dims` 프로퍼티를 통해서 특정된다.   
`max_batch_size`가 0 이상이면, 전체 shape은 `[-1] + dims`이다.   
`max_batch_size`가 0이라면, 전체 shape은 `dims`의 꼴일 뿐이다.   

예를 들어, 아래 config에서 "input0"의 shape은 `[-1, 16]`이고, "output0"의 shape은 `[-1, 4]`이다.   
```protobuf
  platform: "tensorrt_plan"
  max_batch_size: 8
  input [
    {
      name: "input0"
      data_type: TYPE_FP32
      dims: [ 16 ]
    }
  ]
  output [
    {
      name: "output0"
      data_type: TYPE_FP32
      dims: [ 4 ]
    }
  ]
```


`max_batch_size`를 0으로 바꾸면, "input0"은 `[16]`이 되고, "output0"은 `[4]`가 된다.
```protobuf
  platform: "tensorrt_plan"
  max_batch_size: 0
  input [
    {
      name: "input0"
      data_type: TYPE_FP32
      dims: [ 16 ]
    }
  ]
  output [
    {
      name: "output0"
      data_type: TYPE_FP32
      dims: [ 4 ]
    }
  ]
```

가변길이의 shape인 input-output을 가진 모델은, input과 output config에서 `-1`을 이용할 수 있다.   
예를 들어, 만약 모델이 2차원 입력 텐서를 원하는데, 첫 차원은 4의 길이지만, 두 번째 차원은 길이가 무한정 늘어날 수 있다면, `dims: [4, -1]`이 되는것이다.  
Triton은 추론 요청을 두 번째의 차원이 0 이상이라면 받을 것이다.   

reshape 프로퍼티는 입력 shape와 실제 모델 내부의 shape이 다른 경우 설정가능하다.   
model output과 원하는 출력이 다른경우에도 reshape를 설정할 수 있다.   


---
## 🤖 Auto-Generated Model Configuration
일부 경우에 `config.pbtxt`를 자동 작성하는것이 제공된다.   
기본적으로, 최소한의 모델 설정을 하려고 시도한다.   
그러나, `--disable-auto-complete-config`으로 실행하면, 자동완성을 비활성화한다.   
그러나, 이와 상관없이 기본적으로 `instance_group`부분이 없으면 완성해준다.   

TensorRT, ONNX, OpenVINO모델에서 자동완성을 지원해준다.   
Python models에서는, `auto_complete_config`함수에서 `set_max_batch_size`, `add_input`, `add_output`등으로 `max_batch_size`, `input`, `output`의 생성을 구현할 수 있다.   
Triton이 Python model에서 최소의 모델 구성을 형성하는데 도움을 준다.   
다른 모델 유형들은 다 직접 작성해야한다.   

커스텀 벡엔드를 개발할 때, 요구된 세팅들을 넣고 `TRITONBACKNED_ModelSetConfig` API를 호출하여 Triton core에서 설정을 업데이트할 수 있다.   
어떻게 하는지는 [OnnxRuntime](https://github.com/triton-inference-server/onnxruntime_backend)을 참조하면 된다.   
현재는 `inputs`, `outpus`, `max_batch_size`, `dynamic_batching`을 구현할 수 있다.   
모델 이름의 `backend`필드는 `<model_name>.<backend_name>`으로 해야한다.   

Triton이 자동생성해준 model configuration을 보려면, `curl`을 이용하면 된다:   
```bash
curl <server-ip>:8000/v2/models/<model name>/config
```

JSON 표현으로 반환하는데, 이를 `config.pbtxt`로 대응되게 작성할 수 있다.   
Triton은 최소한의 부분만 생성해줄 뿐이고, 추가적인 부분들은 `config.pbtxt`를 수정해야 한다.   


---
## 🎨 Custom Model Configuration
가끔은 하나의 모델 레포지토리에서 여러 Triton 인스턴스가 접근할 수 있다.   
최고의 성능을 위해서는, 구성이 다르게 되어야 할 수 있다.   
`--model-config-name`옵션으로 커스텀 모델 config를 쓸 수 있게 할 수 있다.   

예를 들어, `tritonserver --model-repository=</path/to/model/repository> --model-config-name=h100`이면, 이 서버는 커스텀 config인  `h100.pbtxt`를  `/path/to/model/repository/model-name/configs`에서 찾는다.

만약 있다면, 해당 configuration을 쓴다.   
없다면, 기본 `config.pbtxt`를 쓰거나, 자동 생성된 model configuration을 쓴다.   

Custom model configuration은 Explicit과 Poll 방식에서도 모두 잘 동작한다.   
서버가 띄워져있는 동안에도 동적으로 모델을 로드할 수 있다.

> 주의
> 커스텀 config 이름에는 공백이 있으면 안된다.

예시1. `--model-config-name=h100`
```bash
.
└── model_repository/
    ├── model_a/
    │   ├── configs/
    │   │   ├── v100.pbtxt
    │   │   └── **h100.pbtxt**
    │   └── config.pbtxt
    ├── model_b/
    │   ├── configs/
    │   │   └── v100.pbtxt
    │   └── **config.pbtxt**
    └── model_c/
        ├── configs/
        │   └── config.pbtxt
        └── **config.pbtxt**
```


예시2. `--model=config-name=config`
```bash
.
└── model_repository/
    ├── model_a/
    │   ├── configs/
    │   │   ├── v100.pbtxt
    │   │   └── h100.pbtxt
    │   └── **config.pbtxt**
    ├── model_b/
    │   ├── configs/
    │   │   └── v100.pbtxt
    │   └── **config.pbtxt**
    └── model_c/
        ├── configs/
        │   └── **config.pbtxt**
        └── config.pbtxt
```


예시3. `--model-config-name`이 없으면
```bash
.
└── model_repository/
    ├── model_a/
    │   ├── configs/
    │   │   ├── v100.pbtxt
    │   │   └── h100.pbtxt
    │   └── **config.pbtxt**
    ├── model_b/
    │   ├── configs/
    │   │   └── v100.pbtxt
    │   └── **config.pbtxt**
    └── model_c/
        ├── configs/
        │   └── config.pbtxt
        └── **config.pbtxt**
```


---
## 👨‍🚀 기본 Max Batch Size와 Dynamic Batcher

모델이 auto-complete기능을 사용할때, 기본 최대 배치 사이즈는 `--backend-config=default-max-batch-size=<int>`라는 CLI argument로 될 것이다.   
기본적으로 4이다.   
현재, 다음 backend들은 기본 배치값을 사용하면서 동적 배치를 켠다:   
- Onnxruntime backend
- TensorRT backend
	- TensorRT 모델은 이미 자신만의 max batch size를 명시적으로 가지고있다.
	  그래서, `default-max-batch`를 사용하지 않는다.
	  최대 batch size가 1보다 크면, 설정 파일에 scheduler가 없으면 동적 배치가 자동으로 활성화된다.

공통적으로, 최대 배치 사이즈가 1보다 크면, scheduler 설저이 없으면 자동으로 동적 배치가 지원된다.


---
## 🥚 Datatypes
아래의 테이블이 Triton이 지원하는 tensor 데이터타입이다.

| Model Config | TensorRT | ONNX Runtime | PyTorch | API    | NumPy         |
| ------------ | -------- | ------------ | ------- | ------ | ------------- |
| TYPE_BOOL    | kBOOL    | BOOL         | kBool   | BOOL   | bool          |
| TYPE_UINT8   | kUINT8   | UINT8        | kByte   | UINT8  | uint8         |
| TYPE_UINT16  |          | UINT16       |         | UINT16 | uint16        |
| TYPE_UINT32  |          | UINT32       |         | UINT32 | uint32        |
| TYPE_UINT64  |          | UINT64       |         | UINT64 | uint64        |
| TYPE_INT8    | kINT8    | INT8         | kChar   | INT8   | int8          |
| TYPE_INT16   |          | INT16        | kShort  | INT16  | int16         |
| TYPE_INT32   | kINT32   | INT32        | kInt    | INT32  | int32         |
| TYPE_INT64   | kINT64   | INT64        | kLong   | INT64  | int64         |
| TYPE_FP16    | kHALF    | FLOAT16      |         | FP16   | float16       |
| TYPE_FP32    | kFLOAT   | FLOAT        | kFloat  | FP32   | float32       |
| TYPE_FP64    |          | DOUBLE       | kDouble | FP64   | float64       |
| TYPE_STRING  |          | STRING       |         | BYTES  | dtype(object) |
| TYPE_BF16    | kBF16    |              |         | BF16   |               |
|              |          |              |         |        |               |

TensorRT에서는 각 값이 `nvinfer::DataType` namespace에 있다.   
예를 들어, 32비트 부동소수점은 `nvinfer::DataType::kFLOAT`이다.

ONNX Runtime에서는 `ONNX_TENSOR_ELEMENT_DATA_TYPE_.`에 있다.   
예를 들어, `ONNX_TENSOR_ELEMENT_DATA_TYPE_FLOAT`는 32비트 부동소수점 데이터타입이다.

Pytorch에서 각 값은 `torch`네임스페이스에 있다. `torch::kFloat`는 32비트 부동소수점 데이터타입이다.   

Numpy에서는 모두 numpy모듈에 있다. ex: `numpy.float32`


---
## 💠 Reshape

Reshape는 Triton Inference API와 실제 모델이 원하는 shape이 서로 다를 때 사용된다.

Triton에서는 input/output의 dims는 반드시 비어있지 않아야 한다.   
그러나, 현실의 모델은 배치를 지원하여 `[batch_size]`만을 입력으로 받는 경우도 있다.   
실제 모델은 `[batch_size]`를 입력받길 원하지만, Triton API에서는 `[batch_size, 1]`을 요구한다.   

그럴 때에는 아래와 같이 설정해주면 된다.

```protobuf
input [
	{
		name: "in"
		dims: [ 1 ]
		reshape: { shape: [ ] }
	}
]
```


출력에서는 `[batch_size]`를 출력해야 하지만, Triton API에서는 `[batch_size, 1]`로 출력해야 한다.   
아래와 같이 해주면 된다.
```protobuf
  output [
    {
      name: "in"
      dims: [ 1 ]
      reshape: { shape: [ ] }
    }
  ]
```


즉, 
- `dims`: Inference API 기준의 shape
- `reshape.shape`: 실제 모델의 shape
이다.

---
## 🧊 Shape Tensors

shape tensors를 지원하는 모델에서, `is_shape_tensor`프로퍼티가 지정될 수 있다.
```protobuf
  name: "myshapetensormodel"
  platform: "tensorrt_plan"
  max_batch_size: 8
  input [
    {
      name: "input0"
      data_type: TYPE_FP32
      dims: [ 1 , 3]
    },
    {
      name: "input1"
      data_type: TYPE_INT32
      dims: [ 2 ]
      is_shape_tensor: true
    }
  ]
  output [
    {
      name: "output0"
      data_type: TYPE_FP32
      dims: [ 1 , 3]
    }
  ]
```

Triton은 `batch + [dim]`의 꼴로 차원이 형성된다.   
그러나, shape tensor에서는, 첫 번째 shape의 값이 배치이다.   
위의 config로는 추론 요청이 다음과 같은 shape이어야 한다.   
```protobuf
  "input0": [ x, 1, 3]
  "input1": [ 3 ]
  "output0": [ x, 1, 3]
```

"input1"이 `[2]`가 아닌 `[3]`임을 주목하라.   
`myshapetensormodel`은 배치 모델이고, 배치 사이즈는 추가 값으로 제공되어야 한다.   
Triton은 이런 shape 값들을 합산하여 응답 배치차원을 만든다.   

현재는 TensorRT에서만 지원된다.

---
## 👥 Instance Groups
Triton은 여러 추론 요청들이 동시에 처리될 수 있도록 여러 인스턴스를 제공한다.

### Multiple Model Instances
기본적으로, 시스템에서 각 가능한 GPU에서 모델의 인스턴스가 하나 생긴다.   
intance-group을 세팅하여 특정 GPU 또는 모든 GPU에 여러 개의 인스턴스를 띄울 수 있도록 할 수 있다.

예를 들어, 아래의 configuration은 가능한 각 시스템 GPU에 두 개의 모델 인스턴스를 올릴 것이다.   
```protobuf
instance_group [ 
	{
		count: 2
		kind: KIND_GPU
	}
]
```

아래의 configuration은 0번에는 하나, 1,2번에는 2개의 인스턴스를 둔다.
```protobuf
  instance_group [
    {
      count: 1
      kind: KIND_GPU
      gpus: [ 0 ]
    },
    {
      count: 2
      kind: KIND_GPU
      gpus: [ 1, 2 ]
    }
  ]
```

### CPU Model Instance
Instance group은 CPU에서도 올릴 수 있다.   
아래의 설정으로는 CPU환경의 인스턴스가 2개 있는 것이다.
```protobuf
  instance_group [
    {
      count: 2
      kind: KIND_CPU
    }
  ]
```


---
## 🚀 Response Cache
`response_cache`에는 `enable` 불리언값을 받고, 캐시를 설정할 수 있다.   
```protobuf
response_cache {
	enable: true
}
```
추가로, `--cache-config`가 설정되어야 한다.

[여기](https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/user_guide/response_cache.html)에서 응답 캐시에 대한 더 많은 정보를 가질 수 있다.