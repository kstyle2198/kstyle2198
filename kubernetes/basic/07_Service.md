# Service

Service

pod는 수명이 짧고 언제든 삭제, 재생성이 가능함 (IP가 동적으로 변화함)

이런 pod 들과 통신해야 하는 다른 애플리케이션 또는 사용자는 지속적인 네트워크 주소가 필요함

서비스는 파드의 수명 주기 문제를 해결함, 파드의 생성, 삭제가 빈번하게 일어나서 이름이나 IP가 변경되어도 일관된 네트워크 IP를 제공

밸런싱 - 다수의 파드로 구성된 애플리케이션에 대해 클라이언트 요청을 분배하여 가용성과 성능 보장 

https://kubernetes.io/docs/concepts/services-networking/

# Service Networking

쉽게 말하면 Kubernetes 네트워크는 **“Pod끼리는 직접 통신하고, Service가 Pod들을 묶어서 안정적인 주소를 제공하며, 외부에서는 Ingress/Gateway를 통해 접근한다”**라고 이해하면 됩니다.

### 1. Pod마다 IP가 하나씩 있다

Kubernetes에서는 **Pod 하나마다 고유한 IP 주소**가 할당됩니다.

예를 들어:

```
Pod A → 10.10.1.10
Pod B → 10.10.1.11
Pod C → 10.10.2.10
```

Pod A와 Pod B가 같은 Node에 있든 다른 Node에 있든 기본적으로 서로 통신할 수 있습니다.

그리고 하나의 Pod 안에 컨테이너가 여러 개 있다면:

```
Pod
 ├─ Container A
 └─ Container B

A ↔ B
localhost로 통신 가능
```

즉, **같은 Pod의 컨테이너들은 네트워크 공간을 공유**합니다.

---

### 2. Pod IP를 직접 사용하면 안 되는 이유

여기가 Kubernetes 네트워크에서 가장 중요합니다.

Deployment로 Pod 3개를 만들었다고 해보겠습니다.

```
Deployment
    │
    ├── Pod A  10.10.1.10
    ├── Pod B  10.10.1.11
    └── Pod C  10.10.1.12
```

그런데 Pod A가 삭제되고 새로운 Pod가 생성되면:

```
Pod A 삭제
   ↓
새 Pod 생성
   ↓
10.10.1.20
```

IP가 바뀔 수 있습니다.

따라서 다른 애플리케이션이

```
10.10.1.10
```

을 직접 사용하면 문제가 생깁니다.

---

# 3. 그래서 Service를 사용한다

Service는 **Pod들의 앞에 고정된 주소를 제공하는 역할**을 합니다.

```
                Service
             10.20.0.100
                  │
          ┌───────┼───────┐
          ↓       ↓       ↓
        Pod A   Pod B   Pod C
```

Pod가 삭제되고 새로 만들어져도:

```
Service
10.20.0.100
    │
    ├── 새 Pod
    ├── 새 Pod
    └── 새 Pod
```

Service의 주소는 그대로입니다.

즉,

> **Pod = 실제 서버**
> 
> 
> **Service = 서버들을 대표하는 고정된 주소**
> 

라고 생각하면 쉽습니다.

---

# 4. Service는 Load Balancing도 한다

예를 들어 Pod가 3개 있다면:

```
          Client
             │
             ▼
        Service
             │
       ┌─────┼─────┐
       ↓     ↓     ↓
      Pod1  Pod2  Pod3
```

Service가 요청을 여러 Pod로 전달합니다.

예:

```
요청 1 → Pod1
요청 2 → Pod2
요청 3 → Pod3
요청 4 → Pod1
...
```

따라서 Service는 크게 두 가지 역할을 합니다.

**① 안정적인 접근 주소 제공**

**② 여러 Pod로 트래픽 분산**

---

# 5. Service는 실제 Pod를 어떻게 알까?

Kubernetes가 **EndpointSlice**라는 정보를 관리합니다.

예를 들어 Service가:

```
my-service
```

라고 하면 Kubernetes는 현재 이 Service에 연결된 Pod 정보를 관리합니다.

```
my-service
   │
   ▼
EndpointSlice
   │
   ├── 10.10.1.10
   ├── 10.10.1.11
   └── 10.10.1.12
```

Pod가 죽으면 해당 Pod가 빠지고,

새 Pod가 만들어지면 새 IP가 추가됩니다.

즉:

```
Service
   ↓
EndpointSlice
   ↓
현재 살아있는 Pod 목록
```

이라고 이해하면 됩니다.

---

# 6. Service의 핵심은 Label과 Selector

실제로 Kubernetes에서 Service가 어떤 Pod로 요청을 보낼지는 **label**로 결정합니다.

예를 들어 Pod:

```yaml
metadata:
  labels:
    app: nginx
```

Service:

```yaml
spec:
  selector:
    app: nginx
```

이면:

```
Service
 selector: app=nginx
       │
       ├── Pod A app=nginx
       ├── Pod B app=nginx
       └── Pod C app=nginx
```

이렇게 연결됩니다.

그래서 Kubernetes에서 **Label/Selector가 매우 중요**합니다.

---

# 7. 그런데 외부 사용자는 어떻게 Pod/Service에 접근할까?

여기서 **Ingress 또는 Gateway**가 등장합니다.

전체 구조를 보면:

```
인터넷 사용자
      │
      ▼
Ingress / Gateway
      │
      ▼
   Service
      │
   ┌──┼──┐
   ↓  ↓  ↓
 Pod Pod Pod
```

예를 들어 사용자가:

```
https://myapp.example.com
```

으로 접속하면:

```
사용자
  │
  ▼
Ingress
  │
  ▼
myapp-service
  │
  ├── Pod 1
  ├── Pod 2
  └── Pod 3
```

이런 식으로 전달됩니다.

---

# 8. LoadBalancer Service는 조금 더 단순하다

Ingress를 사용하지 않고 Service 자체를 외부에 공개할 수도 있습니다.

```yaml
kind: Service

spec:
  type: LoadBalancer
```

그러면 클라우드 환경에서는 보통:

```
인터넷
   │
   ▼
Cloud Load Balancer
   │
   ▼
Service
   │
   ├── Pod
   ├── Pod
   └── Pod
```

형태가 됩니다.

따라서:

### LoadBalancer

**Service 자체를 외부에 노출**

### Ingress/Gateway

**여러 Service를 HTTP/HTTPS 기준으로 라우팅**

이라고 생각하면 쉽습니다.

예:

```
                Ingress
                   │
       ┌───────────┼───────────┐
       ↓           ↓           ↓
  /frontend     /api        /admin
       ↓           ↓           ↓
 frontend-svc   api-svc    admin-svc
```

---

# 9. NetworkPolicy는 방화벽이라고 생각하면 쉽다

기본적으로 Pod끼리는 통신할 수 있습니다.

예를 들어:

```
frontend Pod ─────→ backend Pod
                     ↑
                DB Pod
```

그런데 보안을 위해:

```
frontend → backend     허용
backend  → database    허용
frontend → database    차단
```

처럼 만들고 싶을 수 있습니다.

이때 사용하는 것이 **NetworkPolicy**입니다.

쉽게 말하면:

> **NetworkPolicy = Kubernetes Pod 네트워크 방화벽**
> 

입니다.

---

# 10. CNI는 실제 네트워크를 만들어준다

여기서 Kubernetes 자체가 모든 네트워크 기능을 직접 구현하는 것은 아닙니다.

Kubernetes는 주로 **규칙/API를 정의**하고 실제 네트워크 구현은 외부 프로그램이 담당합니다.

대표적으로 **CNI(Container Network Interface)**가 있습니다.

구조를 단순화하면:

```
Kubernetes
    │
    │ 네트워크 규칙
    ▼
   CNI
    │
    ▼
실제 Pod 네트워크
```

대표적인 CNI로는 Calico, Cilium 등이 있습니다.

---

# 11. kube-proxy는 무엇인가?

Service가 있다고 해서 Service가 마법처럼 요청을 Pod로 보내는 것은 아닙니다.

전통적인 Kubernetes 구조에서는 **kube-proxy**가 Service와 Pod 사이의 트래픽 전달을 구성합니다.

```
Client
  │
  ▼
Service
  │
  ▼
kube-proxy가 구성한 네트워크 규칙
  │
  ├── Pod A
  ├── Pod B
  └── Pod C
```

다만 최근에는 **Cilium 같은 네트워크 구현이 kube-proxy 역할을 대신하는 경우도 있습니다.**

---

# 12. 전체를 한 그림으로 이해하면

Kubernetes 네트워크를 처음 공부한다면 아래 구조를 기억하는 것이 가장 좋습니다.

```
                    인터넷
                       │
                       ▼
                Ingress / Gateway
                       │
              ┌────────┴────────┐
              │                 │
              ▼                 ▼
         frontend-svc        backend-svc
              │                 │
          ┌───┼───┐         ┌───┼───┐
          ↓   ↓   ↓         ↓   ↓   ↓
        Pod  Pod  Pod      Pod  Pod  Pod
                              │
                              ▼
                         database-svc
                              │
                           DB Pod
```

그리고 내부적으로는:

```
Pod
 │
 ├─ Pod IP
 │
 └─ CNI가 Pod 네트워크 구성

Service
 │
 ├─ 고정 IP / DNS
 ├─ Selector
 ├─ EndpointSlice
 └─ Pod로 트래픽 분산

Ingress / Gateway
 │
 └─ 외부 → Service

NetworkPolicy
 │
 └─ Pod 간 통신 제어
```

### 핵심만 6개로 암기하면

| 개념 | 쉽게 말하면 |
| --- | --- |
| **Pod IP** | Pod의 실제 IP |
| **Service** | Pod들을 대표하는 고정 주소 |
| **EndpointSlice** | Service 뒤에 있는 현재 Pod 목록 |
| **Ingress / Gateway** | 외부 사용자를 Service로 연결 |
| **NetworkPolicy** | Pod 간 네트워크 방화벽 |
| **CNI** | 실제 Pod 네트워크를 만들어주는 구현 |

특히 Kubernetes 실습에서는 **`Pod → Service → Ingress`** 이 세 가지 관계를 먼저 확실히 이해하는 것이 가장 중요합니다.

그리고 사용자가 최근 실습한 **JupyterHub + OpenLDAP + Ingress + Service** 구조도 정확히 이 네트워크 모델 위에서 동작합니다.

# Service Yaml

앞에서 만든 `nginx-deployment`을 외부에서 접근하기 위한 **Service YAML 샘플**을 보면 이해하기 쉽습니다.

### 1. 가장 기본적인 ClusterIP Service

```yaml
apiVersion: v1
kind: Service

metadata:
  name: nginx-service

spec:
  selector:                 # 특정 라벨을 가진 파드를 선택 
    app: nginx-deployment

  ports:
    - protocol: TCP
      port: 80
      targetPort: 80

  type: ClusterIP
```

적용:

```bash
kubectl apply -f service.yaml
```

확인:

```bash
kubectl get service
```

예상 결과:

```
NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
nginx-service   ClusterIP   10.43.123.45    <none>        80/TCP    10s
```

### 2. 중요한 부분: `selector`

앞에서 만든 Deployment를 보면 Pod에 다음 Label이 있습니다.

```yaml
spec:
  template:
    metadata:
      labels:
        app: nginx-deployment
```

Service에서는:

```yaml
spec:
  selector:
    app: nginx-deployment
```

으로 지정합니다.

즉:

```
Deployment
    │
    ↓
Pod
label:
  app=nginx-deployment
    ↑
    │
Service
selector:
  app=nginx-deployment
```

Service가 **Pod의 이름을 보고 연결하는 것이 아니라 Label을 보고 연결**합니다.

---

### 3. `port`와 `targetPort`

이 부분도 중요합니다.

```yaml
ports:
  - port: 80
    targetPort: 80
```

의 의미는:

```
Client
  │
  │ :80
  ↓
Service
  │
  │ :80
  ↓
Pod
```

입니다.

예를 들어 Service는 8080으로 받고 Pod의 80번 포트로 전달할 수도 있습니다.

```yaml
ports:
  - port: 8080
    targetPort: 80
```

그러면:

```
Client
   │
   │ Service :8080
   ↓
Service
   │
   │ Pod :80
   ↓
Nginx Pod
```

입니다.

---

### 4. NodePort 예제

외부에서 Node의 IP와 포트로 접근하고 싶다면 `NodePort`를 사용할 수 있습니다.

```yaml
apiVersion: v1
kind: Service

metadata:
  name: nginx-service

spec:
  selector:
    app: nginx-deployment

  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30080

  type: NodePort
```

그러면:

```
외부
 │
 │ NodeIP:30080
 ↓
Node
 │
 ↓
Service :80
 │
 ↓
Nginx Pod :80
```

로 연결됩니다.

확인:

```bash
kubectl get svc
```

예:

```
NAME            TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)
nginx-service   NodePort   10.43.12.10    <none>        80:30080/TCP
```

---

### 5. Deployment + Service 전체 구조

지금까지 공부한 내용을 합치면 다음 구조입니다.

```
                 Deployment
               nginx-deployment
                      │
                      ↓
                 ReplicaSet
                      │
             ┌────────┼────────┐
             ↓        ↓        ↓
           Pod      Pod       Pod
       app=nginx  app=nginx  app=nginx
             ↑        ↑        ↑
             └────────┼────────┘
                      │
                 Service
               nginx-service
              selector:
                app=nginx
```

여기서 핵심은 **Deployment의 `name`과 Service의 `selector`를 혼동하지 않는 것**입니다.

예를 들어 Deployment:

```yaml
metadata:
  name: nginx-deployment
```

Service:

```yaml
metadata:
  name: nginx-service
```

둘의 이름은 서로 다릅니다.

대신 Service가 Pod를 찾는 기준은:

```yaml
selector:
  app: nginx-deployment
```

이고, 이 값은 Deployment의 Pod template에 있는:

```yaml
template:
  metadata:
    labels:
      app: nginx-deployment
```

와 일치해야 합니다.

### 6. 실제로 확인해 보기

Service를 만든 후:

```bash
kubectl get endpoints nginx-service
```

또는 최근 Kubernetes에서는:

```bash
kubectl get endpointslice
```

를 확인해보면 Service가 실제로 어떤 Pod를 가리키고 있는지 확인할 수 있습니다.

즉 **Service YAML을 이해하는 핵심은 `selector → Pod label → targetPort` 세 가지**입니다.

# Kubernetes 서비스 타입

Kubernetes의 **Service Type**은 쉽게 말하면 **"Pod에 어떻게 접근할 것인가?"**를 결정하는 설정입니다.

대표적으로 4가지가 있습니다.

```
ClusterIP
NodePort
LoadBalancer
ExternalName
```

가장 중요한 것은 앞의 **3개**입니다.

---

# 1. 전체 그림부터 이해하기

Pod는 직접 접근하는 방식이 좋지 않습니다.

Pod가 재시작되면 IP가 바뀔 수 있기 때문입니다.

```
Pod
10.42.0.10
10.42.0.11
10.42.0.12
```

그래서 Service를 앞에 둡니다.

```
             Service
          nginx-service
                │
       ┌────────┼────────┐
       ↓        ↓        ↓
     Pod      Pod       Pod
  10.42.0.10  .11       .12
```

Service는 **고정된 접근 지점**을 제공합니다.

그리고 Service Type에 따라 **Service를 어디에서 접근할 수 있는지**가 달라집니다.

---

# 2. ClusterIP

가장 기본적인 Service입니다.

```yaml
spec:
  type: ClusterIP
```

또는 `type`을 생략하면 기본값이 `ClusterIP`입니다.

```
Kubernetes Cluster 내부
        │
        ↓
   ClusterIP Service
        │
   ┌────┼────┐
   ↓    ↓    ↓
  Pod  Pod  Pod
```

### 특징

**Cluster 내부에서만 접근 가능**합니다.

예를 들어:

```
Pod A
  │
  │ http://nginx-service
  ↓
Service
  │
  ↓
Nginx Pod
```

다른 Pod에서:

```bash
curl http://nginx-service
```

처럼 접근할 수 있습니다.

하지만 Kubernetes 클러스터 외부의 PC에서는 일반적으로 직접 접근할 수 없습니다.

### 언제 사용?

가장 많이 사용합니다.

예:

```
Frontend Pod
     ↓
Backend Service
     ↓
Backend Pod
```

```
Backend Pod
     ↓
Database Service
     ↓
Database Pod
```

즉 **내부 마이크로서비스 간 통신**에 적합합니다.

---

# 3. NodePort

이번에는 **클러스터 외부에서 Node의 IP를 통해 접근**할 수 있게 합니다.

```yaml
spec:
  type: NodePort
```

구조는:

```
외부 PC
   │
   │ NodeIP:30080
   ↓
Kubernetes Node
   │
   ↓
Service
   │
   ↓
Pod
```

예를 들어:

```yaml
ports:
  - port: 80
    targetPort: 80
    nodePort: 30080    # 가능한 포트범위 (30000~32767)

type: NodePort
```

그러면:

```
Node IP : 30080
       ↓
Service : 80
       ↓
Pod : 80
```

으로 전달됩니다.

### NodePort 번호

일반적으로 Kubernetes의 NodePort 기본 범위는:

```
30000 ~ 32767
```

입니다.

예:

```
192.168.1.10:30080
```

으로 접근할 수 있습니다.

### 언제 사용?

주로:

- 개발/테스트
- 간단한 외부 접근
- 학습용 Kubernetes

에서 많이 사용합니다.

하지만 **실제 운영 환경에서 모든 서비스를 NodePort로 직접 노출하는 방식은 일반적이지 않습니다.**

---

# 4. LoadBalancer

클라우드 환경에서 가장 많이 사용하는 외부 노출 방식 중 하나입니다.

```yaml
spec:
  type: LoadBalancer
```

구조:

```
Internet
    │
    ↓
Cloud Load Balancer
    │
    ↓
Kubernetes Service
    │
    ↓
Pod
```

예를 들어 AWS, GCP, Azure 같은 클라우드 Kubernetes에서:

```yaml
apiVersion: v1
kind: Service

metadata:
  name: nginx-service

spec:
  type: LoadBalancer

  selector:
    app: nginx

  ports:
    - port: 80
      targetPort: 80
```

생성하면 클라우드가 Load Balancer를 만들어 줍니다.

```bash
kubectl get svc
```

예:

```
NAME            TYPE           CLUSTER-IP      EXTERNAL-IP
nginx-service   LoadBalancer   10.43.100.10    34.100.20.30
```

그러면:

```
인터넷
   │
   │ 34.100.20.30:80
   ↓
Load Balancer
   ↓
Service
   ↓
Pod
```

가 됩니다.

### 언제 사용?

예:

```
인터넷
   ↓
LoadBalancer
   ↓
Web Service
   ↓
Web Pods
```

같이 **외부에서 직접 접근해야 하는 서비스**에 사용합니다.

---

# 5. ExternalName

조금 성격이 다릅니다.

`ExternalName`은 Kubernetes 내부 Service 이름을 **외부 DNS 이름에 연결**하는 방식입니다.

예:

```yaml
apiVersion: v1
kind: Service

metadata:
  name: my-database

spec:
  type: ExternalName
  externalName: db.example.com
```

그러면 Kubernetes 내부에서:

```
my-database
```

를 사용하면 실제로는:

```
db.example.com
```

을 가리키도록 DNS CNAME을 제공합니다.

```
Pod
 │
 │ my-database
 ↓
Kubernetes DNS
 │
 ↓
db.example.com
 │
 ↓
외부 Database
```

예를 들어 Kubernetes 밖에 있는 외부 DB를 애플리케이션에서 Service 이름으로 접근하고 싶을 때 사용할 수 있습니다.

---

# 6. 네 가지를 비교하면

| Type | 접근 위치 | 주요 용도 |
| --- | --- | --- |
| **ClusterIP** | Cluster 내부 | 내부 서비스 통신 |
| **NodePort** | 외부 → Node IP | 개발/테스트 |
| **LoadBalancer** | 외부 → Load Balancer | 외부 서비스 |
| **ExternalName** | 외부 DNS 서비스 | 외부 리소스 연결 |

쉽게 기억하면:

```
ClusterIP
    ↓
"클러스터 안에서만"

NodePort
    ↓
"Node의 IP + Port로 외부에서"

LoadBalancer
    ↓
"외부에 Load Balancer를 만들어서"

ExternalName
    ↓
"외부 DNS 이름을 연결"
```

---

# 7. NodePort와 LoadBalancer 관계

여기서 하나 헷갈리기 쉬운 부분이 있습니다.

`LoadBalancer`는 완전히 별개의 네트워크 구조라기보다는 Kubernetes 구현에 따라 **NodePort를 기반으로 동작할 수도 있습니다.**

개념적으로:

```
LoadBalancer
      ↓
NodePort
      ↓
ClusterIP
      ↓
Pod
```

와 같은 구조가 될 수 있습니다.

클라우드 환경에서는 클라우드 Load Balancer가 Kubernetes Service와 연동되어 트래픽을 전달합니다.

---

# 8. 지금 사용 중인 k3d에서는?

현재 사용하시는 **k3d/K3s 환경**에서는 클라우드 Kubernetes와 조금 다르게 생각해야 합니다.

예를 들어:

```yaml
type: LoadBalancer
```

라고 했다고 해서 AWS/GCP처럼 자동으로 실제 클라우드 Load Balancer가 생기는 것은 아닙니다.

K3s에는 기본적으로 **ServiceLB(klipper-lb)**라는 구현이 있어서 `LoadBalancer` Service를 처리합니다.

그리고 k3d에서는 Docker 컨테이너와 포트 매핑 구조까지 함께 고려해야 합니다.

그래서 현재 공부 중인 환경에서는 우선:

```
ClusterIP
   ↓
내부 통신

NodePort
   ↓
Node IP:Port로 접근

LoadBalancer
   ↓
K3s/k3d의 LoadBalancer 구현을 통해 접근
```

정도로 이해하면 충분합니다.

---

## 9. 그런데 실무에서는 Ingress도 중요

실제 Kubernetes 웹 서비스에서는 다음 구조를 많이 사용합니다.

```
                   Internet
                      │
                      ↓
                   Ingress
                      │
          ┌───────────┼───────────┐
          ↓           ↓           ↓
       Service      Service      Service
       frontend     backend       api
          │           │           │
        Pods        Pods         Pods
```

예를 들어:

```
https://example.com
        ↓
Ingress
        ↓
frontend-service
        ↓
Frontend Pods
```

```
https://example.com/api
        ↓
Ingress
        ↓
backend-service
        ↓
Backend Pods
```

이 경우 Service는 보통:

```yaml
type: ClusterIP
```

로 두고 **Ingress가 외부 트래픽을 받아 Service로 전달**하게 합니다.

그래서 Kubernetes 네트워크를 공부할 때는 다음 순서로 이해하면 좋습니다.

```
Pod
 ↓
Service
 ↓
Service Type
 ├── ClusterIP
 ├── NodePort
 └── LoadBalancer
       ↓
Ingress
       ↓
외부 사용자
```

특히 **`Service`와 `Ingress`는 서로 대체 관계가 아니라 역할이 다릅니다.**

Service는 **Pod를 안정적인 네트워크 엔드포인트로 묶는 것**, Ingress는 **HTTP/HTTPS 외부 요청을 여러 Service로 라우팅하는 것**이라고 이해하면 가장 쉽습니다.

# Kubernetes가 자동 생성하는 DNS 이름

쿠버네티스 환경에서는 서비스와 파드가 동적으로 생성, 삭제 되고 이때 IP 주소가 고정되지 않기 때문에 IP 기반 접근 방식은 매우 불안정

그래서 쿠버네티스는 이름기반 통신, 즉 DNS 이름을 통한 접근 방식을 기본으로 제공

동작 흐름

- 사용자가 Service생성
- Kubernetes 가 해당 서비스의 DNS 이름 자동 생성
- 다른 파드에서 서비스명.네임스페이스.svc.cluster.local 형식으로 접근 가능
- (예시 : my-service.default.dvc.cluster.local)
