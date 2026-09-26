# Gateway API

# 기본 샘플

Kubernetes **Gateway API**는 기존 `Ingress`를 더 세분화하고 확장한 Kubernetes의 최신 네트워킹 API입니다.

핵심 구조는 아주 간단합니다.

```
Client
  │
  ▼
Gateway
  │
  ├── HTTPRoute ──► Service ──► Pod
  │
  └── HTTPRoute ──► Service ──► Pod
```

### 1. GatewayClass

어떤 Gateway Controller를 사용할지 정의합니다.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: nginx
spec:
  controllerName: gateway.nginx.org/gateway-controller
```

`IngressClass`와 비슷한 역할입니다.

---

### 2. Gateway

실제 외부에서 들어오는 **IP / Port / TLS** 등의 진입점을 정의합니다.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-gateway
spec:
  gatewayClassName: nginx

  listeners:
    - name: http
      protocol: HTTP
      port: 80
```

즉,

> "80번 포트로 들어오는 트래픽을 받겠다."
> 

라는 의미입니다.

---

### 3. HTTPRoute

어떤 URL을 어떤 Service로 보낼지 정의합니다.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-route
spec:
  parentRefs:
    - name: my-gateway

  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /api

      backendRefs:
        - name: backend
          port: 8080
```

그러면 다음과 같이 동작합니다.

```
http://example.com/api/users
             │
             ▼
        my-gateway
             │
             ▼
       HTTPRoute
       /api
             │
             ▼
    backend:8080
             │
             ▼
          Pod
```

---

## Ingress와 비교

기존 Ingress에서는 보통 하나의 YAML에 많은 설정이 들어갑니다.

```yaml
Ingress
 ├── host
 ├── path
 ├── service
 └── TLS
```

Gateway API에서는 역할을 분리합니다.

```
GatewayClass
    │
    ▼
 Gateway
    │
    ▼
HTTPRoute
    │
    ▼
 Service
    │
    ▼
   Pod
```

### 가장 중요한 차이

**Ingress**

```
Ingress
   └── Routing + Entry Point
```

**Gateway API**

```
GatewayClass → 누가 Gateway를 구현하는가
Gateway      → 어디서 트래픽을 받을 것인가
HTTPRoute    → 트래픽을 어디로 보낼 것인가
Service      → 어떤 Pod로 전달할 것인가
```

그래서 Gateway API는 **인프라 담당자와 애플리케이션 담당자의 설정을 분리하기 좋습니다.**

예를 들어:

```
인프라 담당자
    │
    └── Gateway
          │
          │
개발자      ▼
        HTTPRoute
          │
          ▼
        Service
```

특히 **Traefik, NGINX Gateway Fabric, Envoy Gateway, Cilium** 같은 Gateway API 구현체를 사용할 때 이 구조를 이해하는 것이 중요합니다.

**한 줄로 기억하면:**

> `Gateway = 입구`, `HTTPRoute = URL 라우팅 규칙`, `Service = 실제 애플리케이션` 입니다.
> 

# DNS URL 사례

Gateway API를 처음부터 구성한다면 보통 **`GatewayClass → Gateway → HTTPRoute → Service → Pod`** 순서로 보는 게 맞습니다.

다만 중요한 점이 하나 있습니다.

### 1. GatewayClass

`GatewayClass`는 **Gateway Controller가 이미 설치되어 있다는 전제**에서 그 Controller를 Kubernetes에 연결하는 리소스입니다.

예를 들어 **NGINX Gateway Fabric**을 사용한다면:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: nginx
spec:
  controllerName: gateway.nginx.org/gateway-controller
```

구조는:

```
GatewayClass
     │
     │ "NGINX Gateway Controller를 사용"
     ▼
Gateway
     │
     │ "80번 포트에서 app.example.com을 받음"
     ▼
HTTPRoute
     │
     │ "/ → app-service:8080"
     ▼
Service
     │
     ▼
Pod
```

### 2. 전체 YAML을 한 번에 보면

```yaml
# 1. GatewayClass
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: nginx
spec:
  controllerName: gateway.nginx.org/gateway-controller

---
# 2. Gateway
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: app-gateway
spec:
  gatewayClassName: nginx

  listeners:
    - name: http
      protocol: HTTP
      port: 80
      hostname: app.example.com

---
# 3. HTTPRoute
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: app-route
spec:
  parentRefs:
    - name: app-gateway

  hostnames:
    - app.example.com

  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /

      backendRefs:
        - name: app-service
          port: 8080

---
# 4. Service
apiVersion: v1
kind: Service
metadata:
  name: app-service
spec:
  selector:
    app: myapp

  ports:
    - port: 8080
      targetPort: 8080
```

그리고 별도로 DNS에서:

```
app.example.com
       ↓
Gateway External IP
```

를 설정합니다.

### 그런데 `GatewayClass`를 직접 만들지 않는 경우도 있습니다

이게 혼동하기 쉬운 부분입니다.

Gateway Controller를 설치하면 **GatewayClass가 자동으로 생성되는 경우가 많습니다.**

예를 들어:

```bash
kubectl get gatewayclass
```

결과가:

```
NAME    CONTROLLER
nginx   gateway.nginx.org/gateway-controller
```

처럼 이미 존재한다면 **직접 `GatewayClass` YAML을 만들 필요가 없습니다.**

따라서 실제 작업 순서는:

```
① Gateway Controller 설치
        ↓
② GatewayClass 확인
        ↓
③ Gateway 생성
        ↓
④ HTTPRoute 생성
        ↓
⑤ Service
        ↓
⑥ Pod
        ↓
⑦ DNS → Gateway External IP
```

즉, **GatewayClass는 Gateway API의 최상위 개념이지만, Controller 설치 과정에서 자동 생성되는 경우가 있어서 예제에서 생략되기도 합니다.**
