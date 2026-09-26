# Replicaset

## Replicaset

파드의 복제본을 관리하고 장애 발생시 자동 복구 수행 

동일한 파드를 설정한 개수 만큼 실행 

파드가 삭제되거나 비정상 동작하는 경우 자동으로 새 파드 생성 

파드는 단일 실행 단위로, 장애 발생시 복구 되지 않음 

replicaset은 여러 파드 복제본을 관리하고 항상 원하는 개수를 유지 

Replicaset의 한계

파드 복제와 복구에는 적합하지만

업데이트와 롤백 기능은 없다. .. 그래서 deployment가 필요

### 기본 ReplicaSet YAML

```
apiVersion: apps/v1
kind: ReplicaSet

metadata:
  name: nginx-replicaset  # 이 replicaset의 이름 
  labels:
    app: nginx

spec:
  replicas: 3         # 실행 유지할 파드의 개수

  selector:           # 관리 대상 파드의 labels가 뭐여야 한다..
    matchLabels:
      app: nginx

  template:           # 생성할 파드의 템플릿 
    metadata:
      labels:
        app: nginx

    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
```

파일을 예를 들어 `replicaset.yaml`로 저장한 뒤:

```
kubectl apply -f replicaset.yaml
```

확인:

```
kubectl get replicaset
```

결과:

```
NAME               DESIRED   CURRENT   READY   AGE
nginx-replicaset   3         3         3       10s
```

Pod도 확인할 수 있습니다.

```
kubectl get pods
```

예:

```
NAME                     READY   STATUS    RESTARTS   AGE
nginx-replicaset-abc12   1/1     Running   0          10s
nginx-replicaset-def34   1/1     Running   0          10s
nginx-replicaset-ghi56   1/1     Running   0          10s
```

### 여기서 중요한 부분

ReplicaSet의 핵심 구조는 다음과 같습니다.

```
ReplicaSet
    │
    │ replicas: 3
    │
    ├── Pod 1
    ├── Pod 2
    └── Pod 3

selector
    │
    └── app=nginx
            ↑
            │
Pod template
metadata:
  labels:
    app=nginx
```

특히 이 두 부분이 반드시 연결되어야 합니다.

```
selector:
  matchLabels:
    app: nginx
```

그리고:

```
template:
  metadata:
    labels:
      app: nginx
```

즉,

```
selector의 app=nginx
             ↓
template의 app=nginx
```
