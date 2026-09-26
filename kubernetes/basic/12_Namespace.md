# Namespace

여러 팀이 같은 클러스터를 공유할 경우 자원 관리 복잡성 증가 

이를 해결하기 위해 쿠버네티스는 네임스페이스라는 논리적 분리 기능을 제공 

네임스페이스는 클러스터 내에서 **독립적인 환경**을 생성하여 워크로드를 격리하고 자원관리 및 접근 제어를 효율적으로 수행하도록 지원 

네임스페이스별로 RBAC 적용 가능 , 리소스 할당 관리 등 

# 샘플 코드

네. Kubernetes `Namespace`는 **하나의 Kubernetes 클러스터를 논리적으로 여러 공간으로 나누는 것**입니다.

### 1. Namespace 생성

`namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: myapp
```

적용:

```bash
kubectl apply -f namespace.yaml
```

확인:

```bash
kubectl get namespaces
```

---

### 2. Namespace에 Deployment 생성

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: nginx
  namespace: myapp

spec:
  replicas: 2

  selector:
    matchLabels:
      app: nginx

  template:
    metadata:
      labels:
        app: nginx

    spec:
      containers:
        - name: nginx
          image: nginx
```

여기서 중요한 부분은:

```yaml
metadata:
  namespace: myapp
```

입니다.

즉 이 Deployment는 `myapp` Namespace에 생성됩니다.

```bash
kubectl apply -f deployment.yaml
```

확인:

```bash
kubectl get pods -n myapp
```

---

### 3. Namespace별 리소스 확인

```bash
kubectl get all -n myapp
```

현재 Namespace를 지정하지 않으면 기본적으로 `default` Namespace를 조회합니다.

```bash
kubectl get pods
```

`myapp`의 Pod를 보려면:

```bash
kubectl get pods -n myapp
```

---

### 4. 전체 구조

```
Kubernetes Cluster
│
├── default
│   └── Pod
│
├── myapp
│   ├── Deployment
│   ├── ReplicaSet
│   ├── Pod
│   └── Service
│
└── dev
    ├── Deployment
    └── Pod
```

예를 들어:

```
myapp Namespace
    │
    ├── frontend
    │    └── Pods
    │
    ├── backend
    │    └── Pods
    │
    └── database
         └── Pods
```

처럼 **환경이나 애플리케이션별로 리소스를 분리**할 수 있습니다.

### 5. 지금까지 배운 것과 연결

```
Namespace
    │
    ├── Deployment
    │      ↓
    │   ReplicaSet
    │      ↓
    │     Pod
    │
    ├── Service
    │
    ├── ConfigMap
    │
    ├── Secret
    │
    └── ServiceAccount
```

그리고 RBAC에서는 Namespace가 특히 중요합니다.

```
Role
 ↓
특정 Namespace 권한

ClusterRole
 ↓
Cluster 수준 권한

RoleBinding
 ↓
Namespace 범위로 권한 연결

ClusterRoleBinding
 ↓
Cluster 전체 범위로 권한 연결
```
