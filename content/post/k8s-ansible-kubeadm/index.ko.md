---
title: "Kubernetes - Ansible을 이용한 클러스터 구성"
description: 
date: 2025-09-07T22:50:05+09:00
image: k8s-logo.webp
math: 
license: 
hidden: false
comments: true
draft: false

categories:
    - Kubernetes
tags:
    - Kubernetes
    - Cilium
    - Ansible
---


[플레이북 GitHub Repository](https://github.com/fudoge/ansible-kubeadm)
이 게시글은 다음의 작업에 대해 설명한다:

- **Ansible** 을 활용하여 쿠버네티스 클러스터의 필수 요소 설치를 자동화
    - `containerd`
    - `kubelet`
    - `kubeadm`
    - `kubectl`
- kubeadm으로 Control Plane 구성
- Cilium을 **CNI(Container Network Interface)** 플러그인으로 설치
- 마스터 자격 증명 가져오기

Ansible은 Red Hat의 오픈 소스 IaC도구로, 서버 자원의 구성 관리를 자동화하여 휴먼 에러를 줄이고, 생산성을 높여준다.

Ansible은 다음의 장점을 가진다:

- YAML기반으로 **IaC(Infrastructure as Code)** 실현
- **멱등성(Idempotence)** - 여러 번 실행해도 크래시가 생기지 않는다
    - 그러나, 이미 생성된 클러스터의 노드에 대해서 다시 실행하면, 오류가 발생할 수 있다
    - 기본 필요사항들의 설치까지만 멱등성이 보장된다.
    - 노드를 추가하고 싶을 시, 인벤토리에 기존 노드들은 제거하고 돌리시는게 좋다

---
## 🎬 Ansible 시작을 위한 기본 세팅
Ansible의 구성 요소는 다음과 같다:

- `controller`
    - 구성 대상이 되는 서버들에게 명령을 내리는 호스트
- `inventory`
    - 대상 서버들에 대한 정보
- `playbook`
    - 대상 서버들에 대한 명령집 파일
    - playbook은 여러 개의 play들로 구성
    - **각 play에는 1개 이상의 task로 구성됨**
    - **각 play마다 대상 host가 있음**

Ansible은 대상 서버에서 작업을 취하기 위해, ssh로 접속하여 동작한다.

즉, Ansible이 접속할 수 있는 사용자를 대상 서버에서 만들어줘야 한다.

각 서버에 아래의 작업들을 해야 한다:

### 각 서버에 ansible 유저 생성

`adduser`로 사용자를 생성한다.

`useradd` 를 내부적으로 사용하며, 최초 비밀번호 초기화 등의 더 많은 기능들을 제공한다.

```bash
adduser ansible
```

쿠버네티스 설치를 위해, 권한이 필요하다.

또한, 완전한 자동화를 위해 패스워드 입력을 생략시킬 것이다

```bash
#/etc/sudoers.d/ansible에서 아래와 같이 작성
ansible ALL=(ALL) NOPASSWD:ALL

# ansible 사용자는
# 모든 명령어를 sudo권한으로 사용가능하며
# 모든 명령어를 비밀번호 없이 사용가능하다
# 실제로는 최소권한을 주는 것이 낫다.
```

### ssh 키 생성

(로컬, 즉 컨트롤러에서) ssh key를 생성한다.

```bash
ssh-keygen -N '' -f ~/.ssh/id_rsa -b 2048 -t rsa

# no passphrase
# save private key to ~/.ssh/id_rsa
# 2048 b
# type rsa
```

### 로컬에서 각 서버로 키 배포

```bash
ssh-copy-id -i <key-file> -p <port> ansible@<server-ip>
```

---
## ⚙️ ansible.cfg 및 inventory.ini 작성
Ansible은 `/etc/ansible/ansible.cfg`의 값을 기본적으로 참조하지만,

현재 작업 디렉토리에서의 `ansible.cfg`가 있다면, 그 설정 파일을 우선으로 읽는다.

- `ansible.cfg`는 ansible명령을 실행할 때의 환경 설정 파일이다.
- `inventory.ini`는  대상 호스트들의 정보가 담겨있는 파일이다.

### ansible.cfg 예시

```ini
[defaults]
inventory = ./inventory.ini # inventory.ini가 쓸 인벤토리 파일임을 명시
remote_user = ansible # ssh 유저는 ansible
aks_pass = false # 패스워드 묻지 않음(SSH키 이용)

[privilege_escalation]
become = true # 권한 상승 허용
become_method = sudo # 권한 상승 방법은 sudo(sudo > su)
become_user = root # 권한 상승되어 root로서 동작
become_ask_pass = false # 권한 상승 비밀번호를 묻지 않음. true로 두면 과정에서 비밀번호 물음
```

### inventory.ini 예시

```ini
[masters]
master ansible_host=192.168.64.10 ansible_port=22

[workers]
worker1 ansible_host=192.168.64.12 ansible_port=22

[k8s-nodes:children] # 호스트 그룹 상속
masters
workers
```

또는, `~/.ssh/config` 파일에서 정리해두면, 해당 hostname을 간단하게 쓸 수 있다

---
## 🚀 Ansible을 이용한 모든 노드들에 대해 필수 구성요소 설치
본격적으로 쿠버네티스를 설치한다.

각 노드들은 ubuntu 24.04를 기준으로 한다.

### 준비사항

사전 확인 할것들은 다음과 같다:

- 2GB이상 RAM
- 2코어 이상의 CPU/vCPU
- 클러스터 전체는 네트워크에 연결되어야 함(퍼블릭/프라이빗 상관없음)
- MAC주소 및  `product_uuid`가 모두 고유해야 함
    - `ip link` 또는 `ifconfig -a`로 MAC주소 확인
        - `sudo cat/sys/class/dmi/id/product_uuid`로 `product_uuid`확인 가능
- `hostname`이 모두 고유해야 함
- 시간이 알맞게 동기화되어있어야 함
- 네트워크 어댑터가 2개 이상인 경우, 신경써야 함
    - 기본 라우트 설정 등
- 스왑 비활성화(kubelet이 제대로 작동하려면 스왑은 꺼야 함) → 플레이북에서 자동화한다
- 6443포트가 방화벽에 막히지는 않는지 확인해야 함(kube-apiserver 위해)
    - `iptables/firewalld`, 보안 그룹 등
    - cni에 따라 추가로 필요한 게 있을 수 있음

### 순서

1. 시스템 업데이트 및 업그레이드
2. NTP서버 동기화(현재 플레이북에 없음)
3. ufw비활성화(현재 플레이북에 없음)
4. 스왑 제거
5. 커널 설정 변경(`br_netfilter`, `overlay`, `iptables`, `forwarding`)
    - `br_netfilter`는 브릿지 네트워크 패킷을 `iptables`에서 볼 수 있게 해준다. 
    - `overlay`는 컨테이너의 파일 시스템인 overlayfs를 사용할 수 있도록 해준다
    - `net.ipv4.ip_forward`로 ipv4 패킷 포워딩을 활성화한다.
    - `net.bridge-nf-call-iptables`, `net.bridge-nf-call-ip6tables`을 활성화하여 리눅스 브릿지를 통과하는 IPv4 및 IPv6 트래픽이 `iptables`의 규칙을 통과하도록 해준다
6. 컨테이너 런타임 설치(containerd)
    - `containerd`는 Docker로부터 제공받을 수 있다
7. `containerd` 초기 설정 + cgroup설정(SystemdCgroup=true) + 재시동
    - `systemd`와 통합하여 리소스 제한 및 격리를 관리시킨다
8. `kubelet`, `kubeadm`, `kubectl` 설치
9. 버전 고정
10. `kubelet` 실행

```yaml
---
- name: Install Kubernetes essentials on all nodes
  hosts: k8s_nodes
  become: true
  gather_facts: true
  vars:
    docker_repo_codename: '{{ ansible_lsb.codename | default(''noble'') }}' # noble: 24.04
    k8s_minor: v1.34
    k8s_keyring: /etc/apt/keyrings/kubernetes-apt-keyring.gpg
    k8s_list: /etc/apt/sources.list.d/kubernetes.list
    containerd_pause_image: registry.k8s.io/pause:3.10.1
  tasks:
    - name: Update apt cache
      apt:
        update_cache: yes
        cache_valid_time: 3600
    - name: Upgrade all packages to the latest version
      apt:
        upgrade: dist
    - name: Disable swap temporarily
      command: swapoff -a
      when: ansible_swaptotal_mb | default(0) > 0
    - name: Disable swap permanently
      ansible.posix.mount:
        name: none
        fstype: swap
        state: absent
    - name: Ensure kernel modules are enabled (br_netfilter, overlay)
      modprobe:
        name: '{{ item }}'
        state: present
        persistent: present
      loop:
        - br_netfilter
        - overlay
    - name: Enable iptables and forwarding via sysctl config
      sysctl:
        name: '{{ item }}'
        value: "1"
        reload: yes
        sysctl_file: /etc/sysctl.d/k8s.conf
        state: present
        sysctl_set: yes
      loop:
        - net.bridge.bridge-nf-call-ip6tables
        - net.bridge.bridge-nf-call-iptables
        - net.ipv4.ip_forward
    - name: Install required packages
      apt:
        name:
          - ca-certificates
          - curl
          - gnupg
          - lsb-release
        state: present
        update_cache: yes
    - name: Ensure keyrings directory exists
      file:
        path: /etc/apt/keyrings
        state: directory
        mode: "0755"
    - name: Download Docker GPG key
      get_url:
        url: https://download.docker.com/linux/ubuntu/gpg
        dest: /etc/apt/keyrings/docker.asc
        mode: "0644"
    - name: Convert Docker GPG key to dearmored keyring
      command: /usr/bin/gpg --dearmor -o /etc/apt/keyrings/docker.gpg /etc/apt/keyrings/docker.asc
      args:
        creates: /etc/apt/keyrings/docker.gpg
    - name: Add Docker APT repository
      apt_repository:
        repo: deb [arch={{ (ansible_architecture == 'aarch64') | ternary('arm64', 'amd64' )}} signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu {{ docker_repo_codename }} stable
        state: present
        filename: docker
    - name: Update apt and install containerd.io
      apt:
        name: containerd.io
        state: present
        update_cache: yes
    ## cgroup
    - name: Setup containerd config directory
      ansible.builtin.file:
        path: /etc/containerd
        state: directory
    - name: Generate default containerd config
      command: containerd config default
      register: containerd_config
    - name: Write containerd config.toml
      copy:
        dest: /etc/containerd/config.toml
        content: '{{ containerd_config.stdout }}'
    - name: Set SystemdCgroup=true in containerd
      replace:
        path: /etc/containerd/config.toml
        regexp: SystemdCgroup = false
        replace: SystemdCgroup = true
    - name: Ensure CRI sandbox_image matches kubeadm expected pause image
      replace:
        path: /etc/containerd/config.toml
        regexp: ^\s*sandbox_image\s*=\s*".*"\s*$
        replace: '    sandbox_image = "{{ containerd_pause_image }}"'
    - name: Restart and enable containerd
      systemd:
        name: containerd
        state: restarted
        enabled: true
    - name: state the kubernetes keyring file
      stat:
        path: '{{ k8s_keyring }}'
      register: k8s_keyring_stat
    - name: Download Kubernetes apt signing key
      get_url:
        url: https://pkgs.k8s.io/core:/stable:/{{ k8s_minor }}/deb/Release.key
        dest: /tmp/k8s_key.gpg
        mode: "0644"
        timeout: 30
    - name: Ensure temporary GNUPG home for dearmor
      file:
        path: /etc/apt/keyrings/gnupg
        state: directory
        mode: "0700"
        owner: root
        group: root
    - name: Dearmor Kubernetes Release.key to keyring (idempotent)
      command: |
        /usr/bin/gpg --dearmor --yes --batch --no-tty --output {{ k8s_keyring }} /tmp/k8s_key.gpg
      environment:
        GNUPGHOME: /etc/apt/keyrings/gnupg
      args:
        creates: '{{ k8s_keyring }}'
    - name: Ensure permissions on Kubernetes keyring
      file:
        path: '{{ k8s_keyring }}'
        mode: "0644"
    - name: Configure Kubernetes apt repository
      copy:
        dest: '{{ k8s_list }}'
        mode: "0644"
        content: |
          deb [signed-by={{ k8s_keyring }}] https://pkgs.k8s.io/core:/stable:/{{ k8s_minor }}/deb/ /
    - name: apt-get update for Kubernetes repo
      apt:
        update_cache: yes
    - name: Install kubelet, kubeadm, kubectl for {{ k8s_minor }}
      apt:
        name:
          - kubelet
          - kubeadm
          - kubectl
        state: present
    - name: Hold kube packages (no unintended upgrades)
      command: apt-mark hold kubelet kubeadm kubectl
      changed_when: false
    - name: Enable kubelet
      systemd:
        name: kubelet
        enabled: true
        state: started
```

### 설치하기
`k8s-install.yml`을 실행하여 플레이북을 실행한다
```bash
ansible-playbook k8s-install.yml
```

---
## 🕹️ Control Plane 초기화

### 단일 Control Plane 구성
`kubeadm init`으로 Control Plane을 초기화한다
```bash
kubeadm init --pod-network-cidr=10.43.0.0/16
```

조금만 기다리면, 아래와 같은 결과가 나온다:
```bash
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.0.4:6443 --token <token> \
        --discovery-token-ca-cert-hash sha256:<hash>
```

### HA구성
(미완, 추후 추가예정)

---
## 🛜 네트워크 플러그인 설치
쿠버네티스에는 여러 네트워크 플러그인이 존재합다

- Flannel
- Calico
- Cilium
- 기타(클라우트 환경 CNI 등)

여기서는 떠오르고 있는 CNI인 **Cilium**을 설치합다.

Cilium은 L7정책, 네트워크 가시성, 서비스 메시 등의 기능들을 강력히 제공합다

### 기존 kube-proxy제거

```bash
kubectl delete kube-proxy -n kube-system
```

### cilium CLI를 이용한 설치

Linux 기준

```bash
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}

```

```bash
cilium install --version 1.18.1
```

### helm을 이용한 설치

cilium CLI 대신, helm으로 설치도 가능하다

```bash
helm install cilium cilium/cilium --version 1.18.1 \
    --namespace kube-system \
    --set kubeProxyReplacement=true # kube-proxy를 대체하기
```

### 설치 확인

```bash
root@ubuntu:~# cilium status --wait
    /¯¯\
 /¯¯\__/¯¯\    Cilium:             OK
 \__/¯¯\__/    Operator:           OK
 /¯¯\__/¯¯\    Envoy DaemonSet:    OK
 \__/¯¯\__/    Hubble Relay:       disabled
    \__/       ClusterMesh:        disabled

DaemonSet              cilium                   Desired: 1, Ready: 1/1, Available: 1/1
DaemonSet              cilium-envoy             Desired: 1, Ready: 1/1, Available: 1/1
Deployment             cilium-operator          Desired: 1, Ready: 1/1, Available: 1/1
Containers:            cilium                   Running: 1
                       cilium-envoy             Running: 1
                       cilium-operator          Running: 1
                       clustermesh-apiserver
                       hubble-relay
Cluster Pods:          2/2 managed by Cilium
Helm chart version:    1.18.1
Image versions         cilium             quay.io/cilium/cilium:v1.18.1@sha256:65ab17c052d8758b2ad157ce766285e04173722df59bdee1ea6d5fda7149f0e9: 1
                       cilium-envoy       quay.io/cilium/cilium-envoy:v1.34.4-1754895458-68cffdfa568b6b226d70a7ef81fc65dda3b890bf@sha256:247e908700012f7ef56f75908f8c965215c26a27762f296068645eb55450bda2: 1
                       cilium-operator    quay.io/cilium/operator-generic:v1.18.1@sha256:97f4553afa443465bdfbc1cc4927c93f16ac5d78e4dd2706736e7395382201bc: 1
root@ubuntu:~#
```

### 연결 테스트하기
```bash
cilium connectivity test
ℹ️  Monitor aggregation detected, will skip some flow validation steps
✨ [kubernetes] Creating namespace cilium-test-1 for connectivity check...
✨ [kubernetes] Deploying echo-same-node service...
✨ [kubernetes] Deploying DNS test server configmap...
✨ [kubernetes] Deploying same-node deployment...
✨ [kubernetes] Deploying client deployment...
✨ [kubernetes] Deploying client2 deployment...
⌛ [kubernetes] Waiting for deployment cilium-test-1/client to become ready...
⌛ [kubernetes] Waiting for deployment cilium-test-1/client2 to become ready...
⌛ [kubernetes] Waiting for deployment cilium-test-1/echo-same-node to become ready...
⌛ [kubernetes] Waiting for pod cilium-test-1/client-64d966fcbd-72mrh to reach DNS server on cilium-test-1/echo-same-node-65dd6bdb5c-dl4g8 pod...
⌛ [kubernetes] Waiting for pod cilium-test-1/client2-5f6d9498c7-4cdpn to reach DNS server on cilium-test-1/echo-same-node-65dd6bdb5c-dl4g8 pod...
⌛ [kubernetes] Waiting for pod cilium-test-1/client-64d966fcbd-72mrh to reach default/kubernetes service...
⌛ [kubernetes] Waiting for pod cilium-test-1/client2-5f6d9498c7-4cdpn to reach default/kubernetes service...
⌛ [kubernetes] Waiting for Service cilium-test-1/echo-same-node to become ready...
⌛ [kubernetes] Waiting for Service cilium-test-1/echo-same-node to be synchronized by Cilium pod kube-system/cilium-rl58t
⌛ [kubernetes] Waiting for NodePort 192.168.0.4:32364 (cilium-test-1/echo-same-node) to become ready...
ℹ️  Skipping IPCache check
🔭 Enabling Hubble telescope...
⚠️  Unable to contact Hubble Relay, disabling Hubble telescope and flow validation: rpc error: code = Unavailable desc = connection error: desc = "transport: Error while dialing: dial tcp 127.0.0.1:4245: connect: connection refused"
ℹ️  Expose Relay locally with:
   cilium hubble enable
   cilium hubble port-forward&
ℹ️  Cilium version: 1.18.1
🏃[cilium-test-1] Running 123 tests ...

(중략)
.........

✅ [cilium-test-1] All 74 tests (295 actions) successful, 49 tests skipped, 0 scenarios skipped.
```

---
## 👬 Worker Node 조인시키기
`kubeadm init`의 결과로, 아래와 같은 부분이 있는데, 이를 Worker Node에서 실행하면 된다
```bash
kubeadm join <Control-Plane>:6443 --token <token> \
        --discovery-token-ca-cert-hash sha256:<hash>
```

---
## 🔑 관리자 자격 증명 가져오기
context를 내 로컬(개인 PC 및 노트북)에 가져오면, 원격으로 `kubectl`명령어를 사용할 수 있다.  
context는 **클러스터 정보 및 자격 증명의 조합** 을 말합다.  

### 처음 kubectl을 사용하는 경우
`kubeadm init`의 결과로, 아래와 같은 부분이 나왔을 겁니다.   
즉, Control Plane의 `/etc/kubernetes/admin.conf`를 자신의 `~/.kube/config`로 가져오면 된다.  
`scp`등을 이용할 수 있다.  
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 기존 config에 context를 추가하는 경우
우선, admin의 설정 파일을 로컬로 가져온다
```bash
scp <remote-user>:<control-plane>:/etc/kubernetes/admin.conf ~/admin.conf
```

그 뒤, 기존 설정 파일과 가져온 설정파일을 병합한다
```bash
KUBECONFIG=~/.kube/config:~/admin.conf kubectl config view --flatten > ~/merged; mv ~/merged ~/.kube/config
```

(Optional)컨텍스트의 이름을 변경할 수 있다.   
```bash
kubectl config rename-context kubernetes-admin@kubernetes <새 컨텍스트 이름>
```

---
## 💻 CLI 도구 모음

- [kubectl 설치](https://kubernetes.io/ko/docs/tasks/tools/#kubectl)
- [kubectx 설치](https://github.com/ahmetb/kubectx)
- [helm 설치](https://helm.sh/docs/intro/install/)
- [argocd cli 설치](https://argo-cd.readthedocs.io/en/stable/cli_installation/)


