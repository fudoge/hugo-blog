---
title: "Configuring CiliumNetworkPolicy(CNP) FQDN egress rules for AWS EKS Pod Identity Association"
description: 
date: 2026-06-07T22:39:39+09:00
image: hubble-ui.png
math: 
license: 
hidden: false
comments: true
draft: false

tags:
    - AWS
    - EKS
    - Kubernetes
    - Cilium
    - CiliumNetworkPolicy

categories:
    - AWS
    - Cilium
    - Kubernetes
---

## 🚨 Problem

A server Pod needed to access AWS S3, but the connection kept failing. \
The logs looked like this:

```plaintext
Traceback (most recent call last):
TimeoutError: timed out
...
urllib3.exceptions.ConnectTimeoutError: 
 (<AWSHTTPConnection(host='169.254.170.23', port=80) at 0xffffabb35d00>, 'Connection to 169.254.170.23 timed out. (connect timeout=2)')
...
botocore.exceptions.ConnectTimeoutError: 
 Connect timeout on endpoint URL: \"http://169.254.170.23/v1/credentials\"
...
```

The Pod was trying to request IAM credentials from the EKS Pod Identity Agent through `http://169.254.170.23/v1/credentials`, but the connection timed out. \
Even after allowing egress traffic to `169.254.170.23` on port `80`, the issue continued. \
The traffic was not visible in Hubble UI either.


---
## 🔐 What is EKS Pod Identity?

EKS Pod Identity is a newer way to assign IAM roles to Pods in EKS. \
The Pod Identity Agent runs as a daemonset, one per node, and Pods request credentials from the node-local link-local address `169.254.170.23`. \

The flow works like this:
1. A Pod is created using a ServiceAccount connected through an EKS Pod Identity association.
2. EKS automatically injects environment variables and a token volume into the Pod manifest.
Environment variables:
```yaml
env:
- name: AWS_CONTAINER_AUTHORIZATION_TOKEN_FILE
  value: "/var/run/secrets/pods.eks.amazonaws.com/serviceaccount/eks-pod-identity-token"
- name: AWS_CONTAINER_CREDENTIALS_FULL_URI
  value: "http://169.254.170.23/v1/credentials"
```
Token:
```yaml
volumes:
- name: eks-pod-identity-token
  projected:
    sources:
    - serviceAccountToken:
        audience: pods.eks.amazonaws.com
        expirationSeconds: 86400
        path: eks-pod-identity-token
```
3. A token is created at `/var/run/secrets/pods.eks.amazonaws.com/serviceaccount/eks-pod-identity-token`.
4. The Pod is scheduled onto a node.
5. The SDK requests credentials from the Pod Identity Agent.
    - If the default credential provider chain is used, it uses Pod Identity.
    - It calls the node-local link-local endpoint `http://169.254.170.23/v1/credentials`.
6. The Pod Identity Agent calls the EKS Auth API using the `AssumeRoleForPodIdentity` action.
    - The EKS Auth API issues credentials for the IAM role.
7. The Agent returns the credentials to the SDK.

> **Comparison with IRSA(IAM Role for ServiceAccount)** \
> IRSA is the traditional way to provide credentials to Pods in EKS. \
> With IRSA, the Pod accesses STS directly.
>
> **EKS Pod Identity** has these advantages:
> - Simpler architecture
> - Better reusability
> - No need to manage an OIDC provider
> - IaC-friendly, for example managing associations with Terraform
> 
> **IRSA** has these advantages:
> - Existing system compatibility
> - Cross-account compatibility
> - Support for non-EC2 Linux environments, such as Fargate, Outposts, and EKS Anywhere
> If you only use EC2 Linux nodes, Pod Identity is currently the recommended option.

EKS Pod Identity is installed automatically in EKS Auto Mode, and it can also be installed manually as an add-on.

---
## 🛡️ Configuring CiliumNetworkPolicy Egress FQDN Rules

Traffic targeting the node-local link-local address `169.254.170.23` is identified as traffic to the `host` destination. \
That means it must be allowed with the `host` entity in CNP.

Assuming the Pod accesses S3, you can create a CNP like this:
```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: aws-pod
  namespace: default
spec:
  endpointSelector:
    matchLabels:
      env: test
      cloud: aws
  egress:
    # Pod Identity Agent
    - toEntities:
      - host
      toPorts:
      - ports:
        - port: "80"
        - port: "2703"
    # S3 Bucket
    - toFQDNs:
      - matchName: s3.ap-northeast-2.amazonaws.com
      - matchPattern: "*.s3.ap-northeast-2.amazonaws.com"
      toPorts:
      - ports:
        - port: "443"
          protocol: TCP
    # Kube DNS
    - toEndpoints:
        - matchLabels:
            "k8s:io.kubernetes.pod.namespace": kube-system
            "k8s:k8s-app": kube-dns
      toPorts:
        - ports:
            - port: "53"
              protocol: ANY
          rules:
            dns:
              - matchPattern: "*"
```

In Hubble UI, the traffic is allowed like this.
![Hubble UI](hubble-ui.png)

---
## 📚 Reference

- [AWS Docs - How Pod Identity Works](https://docs.aws.amazon.com/eks/latest/userguide/pod-id-how-it-works.html)
- [AWS Docs - EKS Pod identity vs IRSA](https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html)
- [GitHub Issue - cilium/ciium #34387](https://github.com/cilium/cilium/issues/34387)
