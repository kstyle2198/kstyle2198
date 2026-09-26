# Service Account

쿠버네티스에서 애플리케이션이 API 서버와 상호 작용할 수 있도록 인증을 제공하는 계정

여기서 API 서버는 쿠버네티스 컨트롤 플레인의 API 서버를 의미

User Account vs. Service Accout

일반 사용자는 사람이 Kubectl을 이용하여 클러스터를 조작할 때 로그인하는 계정 

서비스 어카운트는 사람이 아니라 파드와 같은 애플리케이션이 API를 호출할 수 있게 해주는 계정 

이게 왜 필요할까?

쿠버네티스는 클러스터 안에 여러 리소를 가지고 있는데.. 이 리소스들은 필요할 때마나 쿠버네티스 API 서버와 통신하여 정보를 가져오거나 변경한다… 아무 리소스나 API 서버에 접근하면 보안 문제가 발생할 수 있기 때문에.. 쿠버네티스는 모든 리소스가 서비스 어카운트를 통해 API와 통신하도록 강제

기본 서비스 어카운트..

쿠버네티스 클러스터는 네임스페이스마다 default 서비스 어카운트가 존재… 

특별히 지정하지 않으면.. 자동으로 default 서비스 어카운트 사용됨… 

단, default 서비스 어카운트는 기본적으로 아무런 RBAC 권한이 없기 때문에.. API 서버에서 데이터를 가져오거나 수정하려면 별도의 권한 부여해야 함 

# 샘플 코드

네. Kubernetes **ServiceAccount**는 쉽게 말하면 **Pod가 Kubernetes API를 사용할 때 사용하는 "Kubernetes용 계정"**입니다.

가장 기본적인 예제를 보면:

### 1. `serviceaccount.yaml`

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
```

### 2. Pod에서 사용

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod

spec:
  serviceAccountName: my-app-sa

  containers:
    - name: nginx
      image: nginx
```

적용:

```bash
kubectl apply -f serviceaccount.yaml
kubectl apply -f pod.yaml
```

확인:

```bash
kubectl get serviceaccount
```

결과:

```
NAME        SECRETS   AGE
default     0         10s
my-app-sa   0         10s
```

---

### 3. ServiceAccount의 핵심 구조

```
ServiceAccount
    │
    │ serviceAccountName
    ↓
   Pod
    │
    │ Kubernetes API 요청
    ↓
Kubernetes API Server
```

예를 들어 Pod가 Kubernetes API에:

```
"Pod 목록을 조회하고 싶습니다."
```

라고 요청할 때 **ServiceAccount의 권한(RBAC)**에 따라 허용/거부됩니다.

---

### 4. 실제 권한까지 연결한 예제

ServiceAccount만 만든다고 권한이 생기는 것은 아닙니다.

예를 들어 Pod가 `Pod` 목록을 조회할 수 있도록 하려면:

#### `serviceaccount.yaml`

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
```

#### `role.yaml`

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader

rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list"]
```

#### `rolebinding.yaml`

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding

subjects:
  - kind: ServiceAccount
    name: my-app-sa

roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

그리고 Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod

spec:
  serviceAccountName: my-app-sa

  containers:
    - name: nginx
      image: nginx
```

전체 관계는:

```
ServiceAccount
    │
    │ 연결
    ↓
RoleBinding
    │
    ↓
Role
    │
    │ pods: get, list
    ↓
Kubernetes API
```

즉 핵심은:

```
ServiceAccount = 누구인가?
Role           = 무엇을 할 수 있는가?
RoleBinding    = 누구에게 그 권한을 줄 것인가?
```

입니다.

**실무에서는 `default` ServiceAccount를 그대로 사용하기보다, 애플리케이션별 ServiceAccount를 만들고 필요한 최소 권한만 `Role/RoleBinding`으로 부여하는 방식이 권장됩니다.**

# Cluster Role

네. 앞의 **ServiceAccount + Role** 예제를 **ClusterRole**까지 확장하면 다음과 같습니다.

`Role`과 `ClusterRole`의 가장 큰 차이는 **권한의 범위**입니다.

```
Role
  → 특정 Namespace에서만 권한

ClusterRole
  → Cluster 전체 범위에서 권한
```

## 1. ServiceAccount

`serviceaccount.yaml`

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: default
```

## 2. ClusterRole

`clusterrole.yaml`

예를 들어 모든 Namespace의 Pod를 조회할 수 있는 권한입니다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pod-reader

rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
```

## 3. ClusterRoleBinding

`clusterrolebinding.yaml`

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: my-app-pod-reader

subjects:
  - kind: ServiceAccount
    name: my-app-sa
    namespace: default

roleRef:
  kind: ClusterRole
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

## 4. Pod

`pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod

spec:
  serviceAccountName: my-app-sa

  containers:
    - name: nginx
      image: nginx
```

## 전체 관계

```
ServiceAccount
   my-app-sa
       │
       ↓
ClusterRoleBinding
       │
       ↓
ClusterRole
   pod-reader
       │
       │ get/list/watch pods
       ↓
Kubernetes API
```

### Role과 비교

```
Role
 │
 └── default Namespace의 Pod 조회
       ↓
   RoleBinding

ClusterRole
 │
 └── Cluster 전체의 Pod 조회
       ↓
   ClusterRoleBinding
```

예를 들어 `ClusterRole`을 사용하면 `my-app-sa`가 **다른 Namespace의 Pod도 조회**할 수 있습니다.

권한 확인은:

```bash
kubectl auth can-i get pods \
  --as=system:serviceaccount:default:my-app-sa
```

결과:

```
yes
```

특정 Namespace:

```bash
kubectl auth can-i get pods \
  -n kube-system \
  --as=system:serviceaccount:default:my-app-sa
```

이렇게 확인할 수 있습니다.

**기억할 핵심은 `ClusterRole` 자체가 반드시 Cluster 전체에서만 사용되는 것은 아니라는 점입니다.** `ClusterRoleBinding`으로 연결하면 Cluster 전체 권한이 되고, `RoleBinding`으로 연결하면 특정 Namespace 범위에서 ClusterRole의 권한을 사용할 수도 있습니다.
