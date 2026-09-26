# Deployment

리플리카셋을 관리하며, 애플리케이션 배포, 업데이트, 롤백 등 고급 기능을 제공 

아래는 **Nginx 3개 Pod를 생성하는 가장 기본적인 Deployment 샘플**입니다. `Deployment → ReplicaSet → Pod` 관계를 이해하기에도 좋은 예제입니다.

```
apiVersion: apps/v1
kind: Deployment

metadata:
  name: nginx-deployment
  labels:
    app: nginx

spec:
  replicas: 3

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
          ports:
            - containerPort: 80
```

### 1. YAML 적용

파일을 `deployment.yaml`로 저장합니다.

```
kubectl apply -f deployment.yaml
```

### 2. Deployment 확인

```
kubectl get deployments
```

예상 결과:

```
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           10s
```

### 3. ReplicaSet 확인

Deployment가 자동으로 ReplicaSet을 생성합니다.

```
kubectl get replicasets
```

예:

```
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-7c79c4bf97   3         3         3       20s
```

### 4. Pod 확인

```
kubectl get pods
```

예:

```
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-7c79c4bf97-abc12   1/1     Running   0          30s
nginx-deployment-7c79c4bf97-def34   1/1     Running   0          30s
nginx-deployment-7c79c4bf97-ghi56   1/1     Running   0          30s
```

전체 관계는 다음과 같습니다.

```
Deployment
nginx-deployment
        │
        │ creates/manages
        ↓
ReplicaSet
nginx-deployment-7c79c4bf97
        │
        │ creates/manages
        ↓
┌───────────────┬───────────────┬───────────────┐
│               │               │
Pod             Pod             Pod
abc12           def34           ghi56
```

여기서 특히 기억해야 할 부분은 다음입니다.

```
spec:
  replicas: 3
```

→ Pod를 **3개 유지**

```
selector:
  matchLabels:
    app: nginx
```

→ `app=nginx` Label을 가진 Pod를 관리 대상으로 선택

```
template:
  metadata:
    labels:
      app: nginx
```

→ 생성되는 Pod에 `app=nginx` Label 부여

즉,

```
Deployment
   │
   ├── replicas: 3
   │
   └── selector: app=nginx
                    │
                    ↓
              Pod template
              labels: app=nginx
```

라는 연결 구조입니다.

# Rollback

네. Kubernetes `Deployment`의 **롤백(rollback)**은 실습할 때 `image` 버전을 변경한 후 이전 버전으로 되돌려보는 방식으로 이해하면 가장 쉽습니다.

앞에서 만든 `nginx-deployment`을 기준으로 예를 들어보겠습니다.

## 1. 현재 Deployment

처음에는:

```yaml
spec:
  replicas: 3

  template:
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
```

적용:

```bash
kubectl apply -f deployment.yaml
```

확인:

```bash
kubectl get pods
```

---

## 2. 새로운 버전으로 업데이트

예를 들어 `nginx:1.28`로 변경합니다.

```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.28
```

그러면 Deployment가 새로운 ReplicaSet을 생성하고 Pod를 교체합니다.

```bash
kubectl rollout status deployment/nginx-deployment
```

확인:

```bash
kubectl get pods
```

그리고:

```bash
kubectl get replicasets
```

하면 대략:

```
NAME                          DESIRED   CURRENT   READY
nginx-deployment-7c79c4bf97   0         0         0
nginx-deployment-5f6d7c8b9a   3         3         3
```

처럼 **기존 ReplicaSet과 새로운 ReplicaSet이 존재**하게 됩니다.

---

## 3. Deployment의 변경 이력 확인

```bash
kubectl rollout history deployment/nginx-deployment
```

예:

```
deployment.apps/nginx-deployment
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
```

여기서:

```
REVISION 1
    ↓
nginx:1.27

REVISION 2
    ↓
nginx:1.28
```

라고 생각하면 됩니다.

---

## 4. 이전 버전으로 롤백

가장 간단한 방법은:

```bash
kubectl rollout undo deployment/nginx-deployment
```

입니다.

그러면:

```
현재
nginx:1.28
    ↓
rollout undo
    ↓
이전 버전
nginx:1.27
```

으로 돌아갑니다.

롤백 상태 확인:

```bash
kubectl rollout status deployment/nginx-deployment
```

그리고:

```bash
kubectl get pods
```

---

## 5. 특정 Revision으로 롤백

Revision이 여러 개 있다면 특정 버전을 선택할 수도 있습니다.

```bash
kubectl rollout history deployment/nginx-deployment
```

예:

```
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
3         <none>
```

Revision 1로 돌아가려면:

```bash
kubectl rollout undo deployment/nginx-deployment --to-revision=1
```

---

## 6. 실습 전체 흐름

실제로는 다음 순서로 실습하면 좋습니다.

### 최초 배포

```bash
kubectl apply -f deployment.yaml
```

```bash
kubectl get deployment
```

```
NAME               READY
nginx-deployment   3/3
```

현재 이미지 확인:

```bash
kubectl get deployment nginx-deployment \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```

결과:

```
nginx:1.27
```

### 버전 변경

```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.28
```

확인:

```bash
kubectl rollout status deployment/nginx-deployment
```

이미지 확인:

```bash
kubectl get deployment nginx-deployment \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```

결과:

```
nginx:1.28
```

### 롤백

```bash
kubectl rollout undo deployment/nginx-deployment
```

확인:

```bash
kubectl rollout status deployment/nginx-deployment
```

다시 이미지 확인:

```bash
kubectl get deployment nginx-deployment \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```

결과:

```
nginx:1.27
```

---

## 7. Deployment 롤백에서 중요한 개념

앞에서 배운 구조와 연결하면 이해하기 쉽습니다.

```
Deployment
nginx-deployment
       │
       ├── ReplicaSet #1
       │      └── nginx:1.27
       │
       └── ReplicaSet #2
              └── nginx:1.28
```

`nginx:1.28`로 업데이트하면 새로운 ReplicaSet이 만들어집니다.

```
Deployment
       │
       ├── RS #1 ── nginx:1.27
       │
       └── RS #2 ── nginx:1.28
                       ↑
                    현재 사용
```

롤백하면:

```
Deployment
       │
       ├── RS #1 ── nginx:1.27
       │                ↑
       │             다시 사용
       │
       └── RS #2 ── nginx:1.28
```

즉 **Deployment가 이전 ReplicaSet의 버전으로 되돌리는 것**이라고 이해하면 됩니다.

### 실무에서 자주 사용하는 명령어

```bash
# 배포 상태
kubectl rollout status deployment/nginx-deployment

# 변경 이력
kubectl rollout history deployment/nginx-deployment

# 이전 버전으로 롤백
kubectl rollout undo deployment/nginx-deployment

# 특정 버전으로 롤백
kubectl rollout undo deployment/nginx-deployment --to-revision=1

# 롤아웃 일시 중지
kubectl rollout pause deployment/nginx-deployment

# 롤아웃 재개
kubectl rollout resume deployment/nginx-deployment
```
