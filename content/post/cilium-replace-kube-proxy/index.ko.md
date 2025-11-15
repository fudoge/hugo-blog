---
title: "Cilium Replace Kube Proxy"
description: Kube-Proxy없이 Cilium을 이용해보자
date: 2025-10-30T14:39:44+09:00
image: cilium-logo.png
math: 
license: 
hidden: false
comments: true
draft: false

tags:
    - Kubernetes
    - Cilium
categories:
    - Cilium
---

## ✅ Kube-Proxy를 대체했을 때의 장점
Cilium과 같은 CNI플러그인만이 컨테이너 네트워킹 설정을 하는 곳이 아니다.  
**kube-proxy** 데몬셋은 Kubernetes의 서비스 모델에서의 중요한 부분을 맡는데, 리눅스 컨테이너의 네트워킹 스택과 상호작용을 한다.  
kube-proxy는 iptables 규칙을 조작하여 service에서 Pod로 트래픽이 흐르도록 포워딩규칙을 조작한다.  
Kube-proxy는 여러 iptables 규칙을 설치한다.  
Service가 추가되면, iptables의 리스트는 지수적으로 증가한다!  
이는 대규모 클러스터에서 성능 저하가 온다.  

eBPF로, kube-proxy의 기능을 Cilium에서 대체하는 것이 가능하다.  
iptables규칙의 빈번한 수정을 줄이고, 리소스 오버헤드를 줄여 클러스터의 스케일링에서의 속도를 더 빠르게 한다.  

---
## 🍳 Kube-proxy의 기능
기본적으로, Cilium은 ClusterIP Service만 eBPF로 처리하지만, kube-proxy replacement를 켜면, NodePort, LoadBalancer, ExternalIP, HostPort 모두 전부 eBPF로 처리한다.  

---
## 🚀 Kube-Proxy Replacement 활성화하기
helm upgrade로 `kubeProxyReplacement` 옵션을 켤 수 있다.  

```Bash
API_SERVER_IP=<your_api_server_ip>
API_sERVER_PORT=<your_api_server_port>
helm upgrade cilium cilium/cilium \
	-n kube-system \
	--reuse-values \
	--set kubeProxyReplacement=true \
	--set k8sServiceHost=API_SERVER_IP \
	-- set k8sServicePort=6443
```

또는, values.yaml에 아래와 같이 추가한다:
```YAML
# values.yaml
kubeProxyReplacement: true
k8sServiceHost: "192.168.0.4" # API Server IP
k8sServicePort: 6443
```

그 뒤, 적용한다.
```Bash
helm upgrade cilium cilium/cilium \
	-n kube-system
	-f values.yaml
```

이제, kube-proxy를 제거한다.
```Bash
kubectl delete -n kube-system ds kube-proxy

16:56:42 in ~/cilium-lab/helm_install …
➜ kubectl delete -n kube-system ds kube-proxy
daemonset.apps "kube-proxy" deleted
```

---
## 🐣 처음부터 kube-proxy없는 클러스터 만들기
kube-proxy를 없애고 kind 클러스터를 만들려면, 아래와 같이 해주면 된다:
```YAML
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
networking:
  disableDefaultCNI: true
  kubeProxyMode: none
```

```Bash
kind create cluster --config kind-no-kp-config.yaml
```
  
`kubeadm`에서 `kube-proxy`없는 클러스터를 만들려면, 아래와 같이 해주면 된다:
```Bash
kubeadm init --skip-phases=addon/kube-proxy
```

---
## 👀 kube-proxy가 대체되었는지 확인하기
Cilium Agent에서 검증할 수 있다:

```Bash
kubectl -n kube-system exec ds/cilium -- cilium status | grep KubeProxyReplacement

#####
17:10:34 in ~
➜ kubectl -n kube-system exec ds/cilium -- cilium status | grep KubeProxyReplacement
KubeProxyReplacement:    True   [enp6s0   192.168.0.10 fe80::3256:fff:fe01:8c89 (Direct Routing)]
#####
```
  
NodePort Service를 만들어서 잘 되는지 보는 방법도 있다.  
아래의 yaml을 작성하자.
```YAML
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
        - name: my-nginx
          image: nginx
          ports:
            - containerPort: 80
```
  
적용해주자.
```Bash
kubectl apply -f my-nginx.yaml
```

Pod들이 띄워져있는지 확인한다:
```Bash
kubectl get po -l run=my-nginx -o wide
#####
17:15:17 in ~/kube-practice …
➜ kubectl get po -l run=my-nginx -o wide
NAME                       READY   STATUS    RESTARTS   AGE   IP           NODE      NOMINATED NODE   READINESS GATES
my-nginx-7d9b55c9f-8jwm7   1/1     Running   0          21s   10.0.1.154   worker1   <none>           <none>
my-nginx-7d9b55c9f-wtl64   1/1     Running   0          21s   10.0.0.179   master    <none>           <none>
#####
```
NodePort Service를 만들어주자.
```Bash
kubectl expose deploy my-nginx --type=NodePort --port=80
```

Service가 있는지 조회하자.
```Bash
kubectl get svc my-nginx

#####
17:16:19 in ~/kube-practice …
➜ kubectl get svc my-nginx
NAME       TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
my-nginx   NodePort   10.102.47.19   <none>        80:31825/TCP   12s
#####
```

Cilium Client에서 `service list`라는 커맨드를 이용하면, Cilium의 eBPF가 kube-proxy를 대체하고 새로운 NodePort를 쓰고 있는 것을 볼 수 있다.
```Bash
17:16:31 in ~/kube-practice …
➜ kubectl -n kube-system exec ds/cilium -- cilium service list
ID   Frontend                  Service Type   Backend
3    10.96.0.10:53/TCP         ClusterIP      1 => 10.0.0.54:53/TCP (active)
                                              2 => 10.0.0.236:53/TCP (active)
4    10.96.0.10:53/UDP         ClusterIP      1 => 10.0.0.54:53/UDP (active)
                                              2 => 10.0.0.236:53/UDP (active)
5    10.96.0.10:9153/TCP       ClusterIP      1 => 10.0.0.54:9153/TCP (active)
                                              2 => 10.0.0.236:9153/TCP (active)
6    10.104.90.115:8080/TCP    ClusterIP      1 => 10.0.1.161:8080/TCP (active)
7    10.96.0.1:443/TCP         ClusterIP      1 => 192.168.0.4:6443/TCP (active)
9    10.98.77.75:8080/TCP      ClusterIP      1 => 10.0.0.138:8080/TCP (active)
10   10.100.242.234:80/TCP     ClusterIP      1 => 10.0.0.41:80/TCP (active)
                                              2 => 10.0.1.81:80/TCP (active)
11   10.100.44.200:443/TCP     ClusterIP      1 => 192.168.0.10:4244/TCP (active)
14   10.99.76.169:3000/TCP     ClusterIP      1 => 10.0.1.13:3000/TCP (active)
15   10.100.106.101:9090/TCP   ClusterIP      1 => 10.0.1.139:9090/TCP (active)
16   10.109.184.54:80/TCP      ClusterIP      1 => 10.0.1.225:4245/TCP (active)
17   10.103.253.41:80/TCP      ClusterIP      1 => 10.0.1.110:8081/TCP (active)
18   0.0.0.0:30209/TCP         NodePort       1 => 10.0.1.161:8080/TCP (active)
20   0.0.0.0:31872/TCP         NodePort       1 => 10.0.0.138:8080/TCP (active)
22   0.0.0.0:31825/TCP         NodePort       1 => 10.0.0.179:80/TCP (active)
                                              2 => 10.0.1.154:80/TCP (active)
24   10.102.47.19:80/TCP       ClusterIP      1 => 10.0.0.179:80/TCP (active)
 
```

0.0.0.0:31825가 Pod IP:80으로 바인딩된 것을 볼 수 있다.  
(어느 노드포트로 되었는지는 각자 다를 수 있다.)  
또한, Curl로도 검증이 가능하다. Cilium Agent에 접속해보자.  
```Bash
kubectl -n kube-system exec -ti ds/cilium -- /bin/bash
apt upgrade
apt install curl -y
```

`localhost:Nodeport`로 요청을 날려보자. Pod로 잘 서비스가 되는 것을 볼 수 있다.
```Bash
root@worker1:/home/cilium# curl localhost:31825
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

추가로, iptables의 사용이 없어야 한다.  
Cilium Agent에서 이 명령을 해보자:

```Bash
iptables-save | grep KUBE-SVC
```

아무 출력이 없어야 정상이다.