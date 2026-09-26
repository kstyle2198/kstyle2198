# Ingress

쿠버네티스에서 파드는 동적으로 생성되고 사라지며 IP도 자주 변경됨

외부에서 특정 파드에 접근하거나 클러스터 내에서 안정적으로 통신하려면 고정된 네트워크 접근 방법이 필요 

이를 해결하기 위해 Service가 등장했고, 외부 HTTP/HTTPS 트래픽을 세밀하게 제어하기 위해 Ingress가 사용됨 

즉 Service는 내부/외부 통신을 연결하는 기본통로, Ingress는 HTTP/HTTPS 중심의 트래픽을 라우팅하는 고급 제어 도구 

# 샘플코드

네. **Ingress는 외부의 HTTP/HTTPS 요청을 내부의 여러 Service로 라우팅**할 때 사용합니다.

현재 학습 중인 `nginx Deployment + Service`를 기준으로 가장 간단한 예제를 보면 다음과 같습니다.

## 1. Deployment

`deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
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
          ports:
            - containerPort: 80
```

## 2. Service

`service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
```

## 3. Ingress

`ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress

metadata:
  name: nginx-ingress

spec:
  ingressClassName: nginx

  rules:
    - host: nginx.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nginx-service
                port:
                  number: 80
```

적용:

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f ingress.yaml
```

확인:

```bash
kubectl get ingress
```

---

## 전체 구조

```
브라우저
   │
   │ http://nginx.example.com
   ↓
┌──────────────┐
│   Ingress    │
└──────┬───────┘
       │
       │ /
       ↓
┌──────────────┐
│   Service    │
│ nginx-service│
└──────┬───────┘
       │
    ┌──┴──┐
    ↓     ↓
   Pod   Pod
 nginx   nginx
```

### 핵심 필드

```yaml
ingressClassName: nginx
```

→ **어떤 Ingress Controller가 이 Ingress를 처리할지** 지정합니다.

```yaml
host: nginx.example.com
```

→ 이 도메인으로 들어온 요청을 처리합니다.

```yaml
path: /
```

→ `/` 경로의 요청을 처리합니다.

```yaml
service:
  name: nginx-service
  port:
    number: 80
```

→ 요청을 `nginx-service:80`으로 전달합니다.

---

### 경로에 따라 여러 Service로 보내기

Ingress의 가장 큰 장점 중 하나입니다.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress

spec:
  ingressClassName: nginx

  rules:
    - host: example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80

          - path: /api
            pathType: Prefix
            backend:
              service:
                name: backend-service
                port:
                  number: 8080
```

그러면:

```
https://example.com/
        ↓
frontend-service

https://example.com/api
        ↓
backend-service
```

가 됩니다.

즉 지금까지 배운 것을 연결하면:

```
                 Ingress
                    │
          HTTP/HTTPS Routing
                    │
          ┌─────────┴─────────┐
          ↓                   ↓
 frontend-service       backend-service
          │                   │
       Deployment          Deployment
          │                   │
        Pods                Pods
```

**Service는 Pod를 연결하고, Ingress는 외부 HTTP/HTTPS 요청을 Service로 연결한다**고 기억하면 가장 쉽습니다.

> 
>
