---
title: "Kubernetes 리소스: Statefulset"
description: Stateful한 상태의 Pod들은 어떻게 관리되는가
date: 2025-07-23T22:25:40+09:00
image: k8s-logo.webp
math: 
license: 
hidden: false
comments: true
draft: true
---

## Statefulset이 사용되는 이유
웹, WAS등의 서비스들은 무상태(Stateless)의 서비스이다.  
하나의 PV를 여럿이서 공유한다.  
그러고 보통 공통으로 읽기만 하면 된다.  
(Deployment는 RWO를 거의 쓰지 않는다. 대신, ROX, RWX등이 지원되는 파일 스토리지를 많이 사용한다.)

그러나, 데이터베이스, 메시지 브로커 등은 각각의 PV를 가질 필요가 있다.
이렇게 되면,  각 Pod가 동일하지는 않다.
즉, Stateful하다.  
보통 분산 데이터베이스를 위해 사용되는 경우가 대부분이다.  


## StatefulSet
StatefulSet의 Pod들은 동일한 컨테이너 스펙을 가진다.  
그러나, 각 Pod는 고유한 ID를 항상 가진다.  상태를 가지고, 각각의 PVC로 각각의 PV에 접근해야 하기 때문이다.  

StatefulSet의 생성은 항상 작은 번호부터 순차적으로 생긴다.  롤링 업데이트도 작은 번호부터 순차적으로 이루어진다.  
이렇게 되면, Pod - PVC - PV의 매칭에 편리하다.  
제거될 때에는 거꾸로 제거된다.  
분산 시스템의 경우, 보통 Write가 이루어지는 Master가 0번과 같이 낮은 번호를 가지는데, 낮은 번호부터 제거되면 새로운 대표자 선출이 계속되어 비효율적이기 때문이다.  
대신, 높은 숫자부터 제거하면, 새로운 대표자 선출이 최소화될 수 있어 더 성능에 좋다.  

만약 Pod가 죽고 재생성되면 어떻게 될까?  
동일한 Pod의 ID로 생성되기에, 이전에 생성된 PVC-PV와 다시 매칭될 수 있다.  


## 예시: MySQL 복제 클러스터 구축

아래 예시는 1 Master + 2 Slaves의 구조를 가지는 예시이다.  
실제로는 레플리카의 정보를 더 안전하게 주입할 필요가 있다.  

**mysql-svc.yaml**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  clusterIP: None
  selector:
    app: mysql
  ports:
  - port: 3306
```

**mysql-sts.yaml**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: "db"
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:5.7
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-data
          mountPath: /var/lib/mysql
        - name: init-script
          mountPath: /docker-entrypoint-initdb.d/
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: password123
        - name: MYSQL_REPLICATION_USER
          value: repl
        - name: MYSQL_REPLICATION_PASSWORD
          value: replpass
      volumes:
        - name: init-script
          configMap:
            name: mysql-init
  volumeClaimTemplates:
  - metadata:
      name: mysql-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: gp2
      resources:
        requests:
          storage: 1Gi
```

**configmap.yaml**
```yaml
apiVersion: v1
kind: ConfigMap
metadata: 
  name: mysql-init
data: 
  init-replication.sql: |
    -- For Master Only
    CREATE USER IF NOT EXISTS 'repl'@'%' IDENTIFIED BY `replpass`;
    GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
    FLUSH PRIVILEGES;
```


아래의 sql문을 mysql-1, mysql-2에 접속해서 한 번씩만 실행해주자. 

```sql
CHANGE MASTER TO
  MASTER_HOST='mysql-0.mysql.default.svc.cluster.local',
  MASTER_USER='repl',
  MASTER_PASSWORD='replpass',
  MASTER_AUTO_POSITION = 1;
START SLAVE;
```


