# Daemonset

왜?

데몬셋은 쿠버네티스 클러스터내 각 노드마다 특정 파드를 하나씩 자동으로 배포하고 유지하는 리소스 

운영하다보면.. 모든 노드에 공통적으로 실행되어야 하는 파드가 있다. (로그수집, 모니터링, 보안관리 등) 이러한 파드는 모든 노드에 항상 하나씩 실행되어야 올바르게 기능..

일반적인 방식으로 배포할 경우, 노드가 추가, 삭제될 때마다 수동으로 관리해야 하므로 운영 부담 

이를 자동화 해주는 리소스가 데몬셋 이다. 

# 샘플코드

네. `DaemonSet`은 **각 Kubernetes Node마다 Pod를 하나씩 실행**하고 싶을 때 사용하는 리소스입니다.

예를 들어 모든 Node에서 로그 수집기나 모니터링 Agent를 실행할 때 사용합니다.

### 1. 가장 간단한 DaemonSet

`daemonset.yaml`

```yaml
apiVersion: apps/v1
kind: DaemonSet

metadata:
  name: nginx-daemonset

spec:
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
          image: nginx:1.27
```

적용:

```bash
kubectl apply -f daemonset.yaml
```

확인:

```bash
kubectl get daemonset
```

```
NAME              DESIRED   CURRENT   READY
nginx-daemonset   2         2         2
```

현재 Node가 2개라면 Pod도 2개가 생성됩니다.

```bash
kubectl get pods -o wide
```

예:

```
NAME                     NODE
nginx-daemonset-abc12   k3d-j-cluster-server-0
nginx-daemonset-def34   k3d-j-cluster-agent-0
```

---

### 2. Deployment와 비교

지금까지 배운 `Deployment`와 가장 큰 차이는 이것입니다.

```
Deployment
replicas: 3
     ↓
어떤 Node에 배치될지는 Scheduler가 결정
     ↓
Pod 3개
```

반면:

```
DaemonSet
     ↓
Node마다 1개
     ↓
Node 1 → Pod
Node 2 → Pod
Node 3 → Pod
```

즉 **Deployment는 "Pod 개수"를 관리하고, DaemonSet은 "Node당 Pod"를 관리**한다고 기억하면 쉽습니다.

### 3. 현재 사용 중인 k3d에서 확인

현재 클러스터가:

```
k3d-j-cluster-server-0
k3d-j-cluster-agent-0
```

2개의 Node라면:

```bash
kubectl get nodes
```

```
NAME
k3d-j-cluster-agent-0
k3d-j-cluster-server-0
```

DaemonSet을 생성하면 기본적으로:

```
k3d-j-cluster-agent-0
    └── nginx-daemonset Pod

k3d-j-cluster-server-0
    └── nginx-daemonset Pod
```

처럼 실행됩니다.

### 4. 중요한 특징

Node를 하나 추가하면:

```
기존
Node 1 → Pod
Node 2 → Pod

Node 3 추가
       ↓

Node 1 → Pod
Node 2 → Pod
Node 3 → Pod ← 자동 생성
```

Node가 제거되면 해당 Node의 DaemonSet Pod도 함께 사라집니다.

**대표적인 사용 사례:** `Fluent Bit` 같은 로그 수집 Agent, `node-exporter` 같은 모니터링 Agent, 각 Node에서 동작해야 하는 네트워크/스토리지 Agent입니다.

# nodeSelector, nodeAffinity 샘플

네. `DaemonSet`에 **`nodeSelector`와 `nodeAffinity`를 함께 넣는 샘플**을 만들어보겠습니다.

다만 먼저 차이를 간단히 보면:

```
nodeSelector
    → "이 Label을 가진 Node에서만 실행"

nodeAffinity
    → Node Label을 조건식으로 더 유연하게 선택
```

## 1. Node에 Label 지정

먼저 특정 Node에 Label을 붙입니다.

```bash
kubectl label node k3d-j-cluster-agent-0 disktype=ssd
```

확인:

```bash
kubectl get nodes --show-labels
```

---

## 2. DaemonSet YAML

`daemonset.yaml`

```yaml
apiVersion: apps/v1
kind: DaemonSet

metadata:
  name: nginx-daemonset

spec:
  selector:
    matchLabels:
      app: nginx

  template:
    metadata:
      labels:
        app: nginx

    spec:
      # ① nodeSelector
      nodeSelector:
        disktype: ssd

      # ② nodeAffinity
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: kubernetes.io/os
                    operator: In
                    values:
                      - linux

      containers:
        - name: nginx
          image: nginx:1.27
```

적용:

```bash
kubectl apply -f daemonset.yaml
```

확인:

```bash
kubectl get pods -o wide
```

---

## 3. 어떻게 동작하는가?

위 YAML에서는 **두 조건을 모두 만족하는 Node**에만 Pod가 생성됩니다.

```
Node
 │
 ├── disktype=ssd ?
 │       ↓
 │      YES
 │
 └── kubernetes.io/os=linux ?
         ↓
        YES
         ↓
       Pod 생성
```

즉:

```
nodeSelector
disktype=ssd
        AND
nodeAffinity
kubernetes.io/os=linux
        ↓
     실행 가능
```

---

## 4. nodeSelector와 nodeAffinity 차이

### nodeSelector

간단한 조건입니다.

```yaml
nodeSelector:
  disktype: ssd
```

의미:

```
disktype=ssd인 Node에서만 실행
```

---

### nodeAffinity

좀 더 복잡한 조건을 사용할 수 있습니다.

```yaml
nodeAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
    nodeSelectorTerms:
      - matchExpressions:
          - key: disktype
            operator: In
            values:
              - ssd
              - nvme
```

의미:

```
disktype=ssd
       OR
disktype=nvme
```

인 Node를 선택합니다.

---

## 5. DaemonSet에서 특히 주의

일반적으로 DaemonSet은:

```
모든 Node
   ↓
각 Node에 Pod 1개
```

이지만 `nodeSelector` 또는 `nodeAffinity`를 사용하면:

```
Node 1  disktype=ssd
   ↓
  Pod ✓

Node 2  disktype=hdd
   ↓
  Pod ✗

Node 3  disktype=ssd
   ↓
  Pod ✓
```

처럼 **조건을 만족하는 Node에만 DaemonSet Pod가 생성**됩니다.

현재처럼 k3d에서 실습한다면:

```bash
kubectl get nodes --show-labels
```
