---
title: "Tekton: Git 레포지토리로부터 Pull 해오기"
description: Tekton에서 Git 레포지토리로부터 Pull 해오기
date: 2025-08-18T01:15:38+09:00
image: tekton-logo.png
math: 
license: 
hidden: false
comments: true
draft: false

tags:
    - Kubernetes
    - CI/CD 
    - Tekton

categories:
    - Tekton
---


## 🔖 Pipeline 정의

[Community Hub](https://hub.tekton.dev/)에서 재사용 가능한 Task와 Pipeline들이 공유되고 있다.  
여기서는 하나의 Task를 써볼 것이다.  `pipeline.yaml`을 작성한다.  

### Pipeline 선언
```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: clone-read
```

`clone-read`이름은 `PipelineRun`에서 참조할 Run이 된다.  
CLI에서 로그 확인 및 파이프라인 제거 등에도 쓰일 수 있다.  

### 파라미터 입력
`spec`에서 파라미터 리스트를 입력할 수 있다.
```yaml
spec:
  description: | 
    This pipeline clones a git repo, then echoes the README file to the stdout.
  params:
  - name: repo-url
    type: string
    description: The git repo URL to clone from.
```

여기서는 `repo-url`이란 이름의 파라미터를 하나 쓰고 있다.

### Workspace 정의
```yaml
workspaces:
- name: shared-data
  description: | 
    This workspace contains the cloned repo files, so they can be read by the
    next task.
```

Task가 코드를 받아올 `shared-data`을 정의한다.  

### Task 정의
```yaml
tasks:
- name: fetch-source
  taskRef:
    name: git-clone
  workspaces:
  - name: output
    workspace: shared-data
  params:
  - name: url
    value: $(params.repo-url)
```

`fetch-source`라는 이 task는 `git-clone`이라는 다른 Task를 참조하는데, community hub에서 받아와야 한다.   
Task에는 사용할 params와 workspaces를 정의가능하다.  

---
## 🏃 PipelineRun 정의

실제 인스턴스가 될 PipelineRun을 만들어주자.  

### PipelineRun 선언
`pipelinerun.yaml`을 만들어주자.  
```yaml
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  generateName: clone-read-run-
spec:
  pipelineRef:
    name: clone-read
  podTemplate:
    securityContext:
      fsGroup: 65532
```

이 PipelineRun은 `pipelineRef`를 이용하여 `clone-read`로부터 인스턴스를 만든다.  

### Workspace 실제 사용

```yaml
workspaces:
- name: shared-data
  volumeClaimTemplate:
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
```

이 PVC정의는 `shared-data` 워크스페이스와 매칭된다.  

### 파라미터 입력
```yaml
params:
- name: repo-url
  value: https://github.com/tektoncd/website
```

Tekton에서 제공되는 공식 저장소 주소를 입력한다.

---
## 📜 두 번째 Task 정의

### 리스트에 새로운 Task 추가하기
`pipeline.yaml`에 새로운 Task를 리스트에 추가해보자:
```yaml
- name: show-readme
  runAfter: ["fetch-source"]
  taskRef:
    name: show-readme
  workspaces:
  - name: source
    workspace: shared-data
```

### 커스텀 Task 정의

`show-readme.yaml`을 아래와 같이 작성하자:

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: show-readme spec:
  description: Read and display README file.
  workspaces:
  - name: source
  steps:
  - name: read
    image: alpine:latest
    script: | 
      #!/usr/bin/env sh
      cat $(workspaces.source.path)/README.md
```

workspace에서 이전 작업에서 받은 레포지토리에서 `cat`명령을 수행한다.  

---
## 🚀 파이프라인 실행하기

### 외부 Task 설치

`tkn` 명령으로 `git-clone` Task를 설치한다.  
```yaml
tkn hub install task git-clone
```

또는 `kubectl`로도 설치 가능하다.  
```yaml
kubectl apply -f \
https://raw.githubusercontent.com/tektoncd/catalog/main/task/git-clone/0.6/git-clone.yaml
```

### 파일 적용하기

우선 정의한 Task부터 설치한다
```yaml
kubectl apply -f show-readme.yaml
```

Pipeline을 적용시킨다:
```yaml
kubectl apply -f pipeline.yaml
```

PipelineRun을 적용시킨다:
```yaml
kubectl create -f pipelinerun.yaml
```

아래와 같은 로그가 뜬다:
```bash
➜ tkn pipelinerun logs clone-read-run-01 -f
[fetch-source : clone] + '[' false '=' true ]
[fetch-source : clone] + '[' false '=' true ]
[fetch-source : clone] + '[' false '=' true ]
[fetch-source : clone] + CHECKOUT_DIR=/workspace/output/
[fetch-source : clone] + '[' true '=' true ]
[fetch-source : clone] + cleandir
[fetch-source : clone] + '[' -d /workspace/output/ ]
[fetch-source : clone] + rm -rf /workspace/output//lost+found
[fetch-source : clone] + rm -rf '/workspace/output//.[!.]*'
[fetch-source : clone] + rm -rf '/workspace/output//..?*'
[fetch-source : clone] + test -z
[fetch-source : clone] + test -z
[fetch-source : clone] + test -z
[fetch-source : clone] + git config --global --add safe.directory /workspace/output
[fetch-source : clone] + /ko-app/git-init '-url=https://github.com/tektoncd/website' '-revision=main' '-refspec=' '-path=/workspace/output/' '-sslVerify=true' '-submodules=true' '-depth=1' '-sparseCheckoutDirectories='
[fetch-source : clone] {"level":"info","ts":1755359404.1654475,"caller":"git/git.go:176","msg":"Successfully cloned https://github.com/tektoncd/website @ a5a5f1de2ff535ff82f63830fb3f9d03773eb71c (grafted, HEAD, origin/main) in path /workspace/output/"}
[fetch-source : clone] {"level":"info","ts":1755359404.190729,"caller":"git/git.go:215","msg":"Successfully initialized and updated submodules in path /workspace/output/"}
[fetch-source : clone] + cd /workspace/output/
[fetch-source : clone] + git rev-parse HEAD
[fetch-source : clone] + RESULT_SHA=a5a5f1de2ff535ff82f63830fb3f9d03773eb71c
[fetch-source : clone] + EXIT_CODE=0
[fetch-source : clone] + '[' 0 '!=' 0 ]
[fetch-source : clone] + git log -1 '--pretty=%ct'
[fetch-source : clone] + RESULT_COMMITTER_DATE=1754472027
[fetch-source : clone] + printf '%s' 1754472027
[fetch-source : clone] + printf '%s' a5a5f1de2ff535ff82f63830fb3f9d03773eb71c
[fetch-source : clone] + printf '%s' https://github.com/tektoncd/website

[show-readme : read] # TektonCD Website
[show-readme : read]
[show-readme : read] This repo contains the code behind [the Tekton org's](https://github.com/tektoncd)
[show-readme : read] website at [tekton.dev](https://tekton.dev).
[show-readme : read]
[show-readme : read] For more information on the Tekton Project, see
[show-readme : read] [the community repo](https://github.com/tektoncd/community).
[show-readme : read]
[show-readme : read] For more information on contributing to the website see:
[show-readme : read]
[show-readme : read] * [CONTRIBUTING.md](CONTRIBUTING.md)
[show-readme : read] * [DEVELOPMENT.md](DEVELOPMENT.md)
```

## 전체 마니페스트

`show-readme.yaml`

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: show-readme
spec:
  description: Read and display README file.
  workspaces:
  - name: source
  steps:
  - name: read
    image: alpine:latest
    script: | 
      #!/usr/bin/env sh
      cat $(workspaces.source.path)/README.md
```

`pipeline.yaml`
```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: clone-read
spec:
  description: | 
    This pipeline clones a git repo, then echoes the README file to the stout.
  params:
  - name: repo-url
    type: string
    description: The git repo URL to clone from.
  workspaces:
  - name: shared-data
    description: | 
      This workspace contains the cloned repo files, so they can be read by the
      next task.
  - name: git-credentials
    description: My ssh credentials
  tasks:
  - name: fetch-source
    taskRef:
      name: git-clone
    workspaces:
    - name: output
      workspace: shared-data
    - name: ssh-directory
      workspace: git-credentials
    params:
    - name: url
      value: $(params.repo-url)
  - name: show-readme
    runAfter: ["fetch-source"]
    taskRef:
      name: show-readme
    workspaces:
    - name: source
      workspace: shared-data
```

`pipelinerun.yaml`
```yaml
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  generateName: clone-read-run-
spec:
  pipelineRef:
    name: clone-read
  podTemplate:
    securityContext:
      fsGroup: 65532
  workspaces:
  - name: shared-data
    volumeClaimTemplate:
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 1Gi
  - name: git-credentials
    secret:
      secretName: git-credentials
  params:
  - name: repo-url
    value: git@github.com:tektoncd/website.git
```

---
## 📚 결론
- Task는 직접 정의할 수 있고, 커뮤니티에서 만들어진 Task를 사용할 수 있다.  
- Params로 파라미터를 넣을 수 있고, Workspace로 작업 공간을 공유할 수 있다.  
- 적용 순서는 다음과 같다:
    - Task 설치 및 적용
    - Pipeline 적용
    - PipelineRun 적용
