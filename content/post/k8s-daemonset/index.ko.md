---
title: "Kubernetes 리소스: DaemonSet"
description: 노드에 상주하는 DaemonSet에 대해 알아보자
date: 2025-07-17T07:16:25+09:00
image: k8s-logo.webp
math: 
license: 
hidden: false
comments: true
draft: false
---

## DaemonSet이란

모든 또는 특정 노드에 상주하는 노드를 위한 Pod가 필요할 수 있다.  
예를 들면, 노드의 메트릭을 추적하기 위해, 각 노드마다 `node-exporter`가 필요할 수 있다.  
또는 ssd를 사용하는 노드에만, 또는 gpu를 사용하는 노드에 노드를 위한 Pod를 사용하고 싶을 수 있다.  

**DaemonSet** 은 모든 또는 선택된 노드에 상주하며 Node 특화된 작업을 주로 하고, 해당되는 노드당 1개가 desired된 특별한 Pod이다.  
DaemonSet 역시 self-healing 메커니즘을 가진다.  

### DaemonSet 예시

아래는 GPU 노드에만 붙는 DaemonSet 예시이다:
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: gpu-daemon
  labels:
    app: gpu-agent
spec:
  selector:
    matchLabels:
      app: gpu-agent
  template:
    metadata:
      labels:
        app: gpu-agent
    spec:
      nodeSelector: # 표현식은 nodeAffinity
        nvidia.com/gpu.present: "true"  # GPU가 있는 노드에만
      containers:
      - name: gpu-monitor
        image: nvidia/cuda:12.2.0-base
        command: ["nvidia-smi"]
```

명령어에서의 축약어는 `ds`이다.  
`kube-system`에서 클러스터 내부에서 돌아가는 ds들을 확인해보자.  
```bash
kubens kube-system

kubectl get ds

kubens default # 다시 돌아오자
```


## 요약
DaemonSet은 노드 단위로 상주시킬 수 있는 특별한 Pod이다. 셀프힐링이 되며, Node를 label 또는 표현식 기반으로 일부 선택도 가능하다.  
