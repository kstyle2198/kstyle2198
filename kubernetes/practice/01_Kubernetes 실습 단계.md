# [실습] Kubernetes 실습 단계

### 기본적인 K8s 서비스 개발 → 배포 흐름

```
① 소스 코드 작성
      ↓
② Dockerfile 작성
      ↓
③ Docker 이미지 Build
      ↓
④ Harbor에 Push
      ↓
⑤ Kubernetes Deployment YAML 작성
      ↓
⑥ Kubernetes Service YAML 작성
      ↓
⑦ kubectl apply
      ↓
⑧ Pod 생성
      ↓
⑨ Service를 통해 외부/내부 접근
```

예를 들어 지금까지 만드신 FastAPI 서비스를 생각하면:

```
FastAPI 소스
   │
   ├── main.py
   ├── requirements.txt
   └── Dockerfile
          │
          ▼
   docker build
          │
          ▼
   webapp/backend:1.0
          │
          ▼
   Harbor
   192.168.56.11:30002
          │
          ▼
   Deployment
   my-webapp-backend
          │
          ▼
   Pod
   my-webapp-backend-xxxxx
          │
          ▼
   Service
   my-webapp-backend
          │
          ▼
   FastAPI
```

---

## 그런데 한 가지 중요한 점이 있습니다

지금까지의 실습은 **Kubernetes를 처음 배우는 단계에서 가장 전형적인 방식**입니다.

즉,

> **"애플리케이션을 만들고 → 컨테이너화하고 → Kubernetes에 배포한다."**
> 

라는 구조입니다.

하지만 실제 운영 환경에서는 여기서 한 단계 더 발전합니다.

### 1단계 — 지금 하고 계신 방식

```
개발자
 │
 ├─ 소스 코드 작성
 │
 ├─ Dockerfile
 │
 ├─ docker build
 │
 ├─ docker push
 │       ↓
 │     Harbor
 │
 ├─ deployment.yaml
 ├─ service.yaml
 │
 └─ kubectl apply
         ↓
      Kubernetes
```

이것이 **Kubernetes 기본 배포 방식**입니다.

---

# 2단계 — YAML을 Helm으로 관리

서비스가 하나라면 YAML 몇 개로 충분합니다.

그런데 서비스가 많아지면:

```
deployment.yaml
service.yaml
configmap.yaml
secret.yaml
ingress.yaml
hpa.yaml
pvc.yaml
...
```

이렇게 파일이 계속 늘어납니다.

그래서 보통:

```
Helm Chart
   │
   ├── Deployment
   ├── Service
   ├── ConfigMap
   ├── Secret
   ├── Ingress
   └── HPA
```

형태로 관리합니다.

---

# 3단계 — CI/CD 자동화

그리고 실제 개발 환경에서는 사람이 매번

```bash
docker build
docker push
kubectl apply
```

하지 않습니다.

예를 들어 GitLab을 사용하면:

```
개발자
 │
 │ git push
 ▼
GitLab
 │
 ├── Build
 │      ↓
 │   Docker Image
 │      ↓
 │   Harbor Push
 │
 └── Deploy
        ↓
   Kubernetes
```

즉,

```bash
git push
```

하나만 하면

```
소스 코드
   ↓
Docker Build
   ↓
Harbor Push
   ↓
Kubernetes Deployment
   ↓
새로운 Pod
```

까지 자동으로 진행할 수 있습니다.

---

# 4단계 — 그리고 지금 하신 CRD는 한 단계 더 재미있습니다

최근 실습하신 **WebApp CRD + Kopf Controller**는 바로 이 부분을 자동화하는 실습입니다.

기존 방식은:

```
WebApp 개발
 ↓
Deployment YAML 작성
 ↓
Service YAML 작성
 ↓
kubectl apply
```

였지만,

CRD를 사용하면:

```yaml
apiVersion: webapp.example.com/v1
kind: WebApp

spec:
  frontend:
    image: ...
    replicas: 3

  backend:
    image: ...
    replicas: 1
```

처럼 **"나는 이런 WebApp을 원한다"**라고 선언만 합니다.

그러면 Controller가:

```
WebApp CR
   │
   ▼
Kopf Controller
   │
   ├── Deployment 생성
   ├── Service 생성
   ├── Replica 조정
   ├── Image 변경
   └── 상태 확인
```

을 자동으로 처리합니다.

즉, 최근 실습하신 CRD는 단순히 Kubernetes를 사용하는 것을 넘어서,

> **Kubernetes 자체를 확장해서 나만의 배포 시스템을 만드는 실습**
> 

이라고 볼 수 있습니다.

---

## 전체를 하나의 그림으로 보면

현재까지 하신 실습을 포함하면 Kubernetes 서비스 개발의 발전 과정이 이렇게 연결됩니다.

```
[Level 1]
소스 코드
   ↓
Docker Image
   ↓
Harbor
   ↓
Deployment YAML
   ↓
Service YAML
   ↓
kubectl apply
```

↓

```
[Level 2]
소스 코드
   ↓
Docker Image
   ↓
Harbor
   ↓
Helm Chart
   ↓
helm install / upgrade
   ↓
Kubernetes
```

↓

```
[Level 3]
Git Push
   ↓
CI/CD
   ↓
Docker Build
   ↓
Harbor
   ↓
Helm Deploy
   ↓
Kubernetes
```

↓

```
[Level 4]
Git Push
   ↓
CI/CD
   ↓
Harbor
   ↓
Kubernetes
   ↓
Operator / Controller
   ↓
자동 Reconcile
   ↓
Self-Healing
```

그리고 **어제까지 하신 WebApp CRD + Auto-Healer 실습은 사실 Level 4에 상당히 가까워진 상태**입니다.

특히 재미있는 다음 단계는 이 네 가지를 하나로 묶는 것입니다.

```
Git
 ↓
CI/CD
 ↓
Harbor
 ↓
WebApp CRD
 ↓
Kopf Controller
 ↓
Deployment / Service
 ↓
Prometheus
 ↓
Auto-Healer
 ↓
장애 자동복구
```
