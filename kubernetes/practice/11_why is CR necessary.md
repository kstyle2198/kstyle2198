# [실습] CR 왜 필요한가?

# Why CR? (CR vs. 통합 yaml 파일 배포)

둘 다 “여러 Kubernetes 리소스를 한 번에 배포한다”는 점에서는 비슷해 보이지만, 목적이 다릅니다.

예를 들어 Deployment, Service, ConfigMap을 하나의 YAML에 `---`로 묶는 방식은 **여러 Kubernetes 리소스를 한 파일에 정리해서 한 번에 apply**하는 것입니다. 반면 CR(Custom Resource)은 “내가 원하는 시스템의 상태”를 하나의 새로운 API 객체로 정의하고, **Controller/Operator가 그 CR을 보고 필요한 Deployment, Service 등을 지속적으로 생성·수정·복구**하도록 만드는 방식입니다.

```yaml
# 단순 multi-resource YAML
apiVersion: apps/v1
kind: Deployment
...
---
apiVersion: v1
kind: Service
...
---
apiVersion: v1
kind: ConfigMap
...
```

이 경우 Kubernetes 입장에서는 세 객체가 서로 별개의 리소스입니다. 사용자가 직접 다음처럼 적용합니다.

```bash
kubectl apply -f app.yaml
```

CR 방식에서는 예를 들어 이런 형태가 됩니다.

```yaml
apiVersion: webapp.example.com/v1
kind: WebApp
metadata:
  name: my-webapp
spec:
  frontend:
    image: frontend:1.0
    replicas: 2
  backend:
    image: backend:1.0
    replicas: 3
```

사용자는 `WebApp`이라는 하나의 객체만 생성하지만 Controller가 내부적으로 Deployment, Service, ConfigMap 등을 생성합니다.

핵심 차이는 다음과 같습니다.

| 구분 | 여러 객체를 하나의 YAML에 묶기 | CR + Controller |
| --- | --- | --- |
| 목적 | 배포 파일 단순화 | 새로운 Kubernetes API/운영 모델 정의 |
| Deployment 생성 | YAML에 직접 작성 | Controller가 생성 |
| Service 생성 | YAML에 직접 작성 | Controller가 생성 |
| 상태 관리 | Kubernetes 기본 Controller 각각 담당 | 사용자 Controller가 전체 상태 조정 |
| 변경 방법 | Deployment/Service YAML 직접 수정 | CR의 `spec`만 수정 |
| 자동 복구 로직 | 기본 Kubernetes 기능 정도 | 원하는 로직 구현 가능 |
| 리소스 간 관계 | 사실상 없음 | Controller가 관계를 관리 |
| 운영 정책 | YAML에 분산 | Controller 코드로 중앙화 |
| 추상화 | 낮음 | 높음 |
| 개발 난이도 | 낮음 | 높음 |

그래서 **단순히 여러 리소스를 한 번에 배포하고 싶은 목적이라면 CR을 만들 필요가 없습니다.** 오히려 Helm이나 Kustomize가 더 적합합니다.

CR이 의미를 갖는 것은 “Deployment + Service를 묶어놓겠다”가 아니라, Kubernetes에게 **새로운 개념을 추가하고 싶을 때**입니다.

예를 들어 `WebApp`이라는 CR을 만들었다고 해보겠습니다.

```yaml
spec:
  frontend:
    replicas: 2
  backend:
    replicas: 3
```

Controller가 이를 보고 다음 구조를 자동으로 만들 수 있습니다.

```
WebApp CR
    │
    ▼
WebApp Controller
    │
    ├── frontend Deployment
    ├── frontend Service
    ├── backend Deployment
    ├── backend Service
    ├── ConfigMap
    ├── Secret 설정
    └── Ingress
```

그리고 사용자가 CR에서 다음 한 줄만 바꿉니다.

```yaml
backend:
  replicas: 5
```

그러면 Controller가 실제 backend Deployment를 찾아 현재 replicas가 3이면 5로 변경합니다.

```
Desired State
CR: replicas = 5

      │
      ▼
Controller reconcile()

      │
      ▼
Actual State
Deployment replicas = 3

      │
      ▼
PATCH Deployment

      │
      ▼
Deployment replicas = 5
```

여기서 CR 방식의 진짜 장점은 **Reconciliation Loop**입니다.

Controller는 일반적으로 계속 다음을 수행합니다.

```
Desired State
      │
      ▼
    비교
      │
Actual State
      │
      ▼
차이가 있으면 수정
```

예를 들어 누군가 실수로 다음을 실행했다고 가정해보겠습니다.

```bash
kubectl scale deployment my-webapp-backend --replicas=1
```

CR은 여전히 다음처럼 되어 있습니다.

```yaml
backend:
  replicas: 5
```

Controller가 이를 발견하면 다시 5로 복구할 수 있습니다.

```
CR desired replicas = 5

실제 Deployment replicas = 1
              │
              ▼
        Controller 감지
              │
              ▼
       replicas = 5 복구
```

단순 multi-document YAML에는 이런 **애플리케이션 수준의 제어 루프가 존재하지 않습니다.**

물론 Deployment 자체에도 Deployment Controller가 있기 때문에 Pod가 죽으면 복구합니다. 하지만 그것은 다음 정도의 역할입니다.

```
Deployment
   ↓
ReplicaSet
   ↓
Pod
```

사용자가 원하는 더 높은 수준의 관계:

```
WebApp
 ├─ Frontend
 ├─ Backend
 ├─ Service
 ├─ Ingress
 ├─ Config
 └─ Health 정책
```

를 Kubernetes 기본 Controller는 모릅니다. 그것을 만들어주는 것이 Custom Controller입니다.

예를 들어 CR이 다음처럼 발전할 수도 있습니다.

```yaml
apiVersion: platform.example.com/v1
kind: WebApp
metadata:
  name: shopping
spec:
  frontend:
    image: frontend:2.1
    replicas: 3

  backend:
    image: backend:4.2
    replicas: 5

  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 10

  ingress:
    enabled: true
    host: shopping.example.com

  database:
    type: postgres

  monitoring:
    enabled: true
```

사용자는 이것만 작성합니다.

Controller는 이를 실제 Kubernetes 리소스로 변환할 수 있습니다.

```
WebApp CR
     │
     ▼
WebApp Operator
     │
     ├── Deployment(frontend)
     ├── Service(frontend)
     ├── Deployment(backend)
     ├── Service(backend)
     ├── HPA
     ├── Ingress
     ├── ConfigMap
     ├── Secret
     ├── ServiceMonitor
     └── NetworkPolicy
```

이렇게 되면 CR이 **플랫폼 사용자를 위한 고수준 API**가 됩니다.

예를 들어 개발자에게 Kubernetes 전체를 알려줄 필요 없이 다음만 알려주면 됩니다.

```bash
kubectl apply -f webapp.yaml
```

그리고 상태도 Kubernetes 방식으로 볼 수 있습니다.

```bash
kubectl get webapps
```

예:

```
NAME        FRONTEND   BACKEND   READY   STATUS
shopping    3          5         True    Running
```

CR의 `status`도 Controller가 업데이트할 수 있습니다.

```yaml
status:
  ready: true
  frontendReady: 3
  backendReady: 5
  endpoint: https://shopping.example.com
```

이 부분은 일반적인 YAML 묶기 방식과 꽤 큰 차이입니다.

---

특히 CR이 유용한 상황은 **생성뿐 아니라 운영 로직까지 필요할 때**입니다.

예를 들어 사용자가 작업하고 있는 Auto-Healer 같은 시스템을 생각해보면, 단순 YAML은 다음 정도를 정의합니다.

```
Deployment
Service
RBAC
PVC
ConfigMap
```

하지만 CR을 도입하면 이런 API를 만들 수도 있습니다.

```yaml
apiVersion: healer.example.com/v1
kind: AutoHealer
metadata:
  name: production-healer
spec:
  namespaces:
    - production
    - backend

  checkInterval: 30

  policies:
    podFailure:
      enabled: true
      maxRestartCount: 5

    pendingPod:
      enabled: true
      timeoutSeconds: 300

  notification:
    enabled: true
```

그러면 Controller가 지속적으로 다음과 같은 운영 정책을 실행할 수 있습니다.

```
AutoHealer CR
       │
       ▼
AutoHealer Controller
       │
       ├─ Pod 상태 감시
       │
       ├─ Failed 감지
       │
       ├─ Pending timeout 감지
       │
       ├─ Recovery 수행
       │
       ├─ Event 기록
       │
       └─ CR status 업데이트
```

이런 경우는 CR을 도입할 가치가 큽니다.

반대로 다음처럼 **그저 리소스 배포 편의를 위한 것**이라면 CR은 과합니다.

```
Deployment
Service
ConfigMap
Ingress
```

이 정도라면 보통:

```
Plain YAML
    ↓
Kustomize
    ↓
Helm
```

중 하나가 적절합니다.

CR은 그 다음 단계입니다.

```
YAML
  │
  ├─ 단순 배포
  ▼
Helm / Kustomize
  │
  ├─ 템플릿 / 환경별 설정
  ▼
CRD + Controller
  │
  └─ 새로운 Kubernetes API + 운영 자동화
```

따라서 판단 기준은 아주 간단하게 볼 수 있습니다.

> **“여러 YAML을 편하게 관리하고 싶은가?” → Helm/Kustomize**
> 

> **“사용자가 선언한 고수준 상태를 보고 시스템이 계속 스스로 관리되게 만들고 싶은가?” → CRD + Controller**
> 

특히 `kubectl apply -f all.yaml`과 CR의 가장 본질적인 차이는 **CR 자체가 중요한 것이 아니라 CR을 감시하는 Controller가 있다는 것**입니다.

```
CRD만 생성
    ↓
거의 데이터 저장소에 가까움

CRD + Controller
    ↓
Kubernetes Operator 패턴
    ↓
실질적인 자동화 가치 발생
```

따라서 지금까지 진행하신 WebApp CR 실습을 기준으로 보면, `WebApp CR → Controller → frontend/backend Deployment/Service 생성 및 reconcile` 구조를 이해하는 것이 핵심입니다. 단순 YAML을 CR로 바꾸는 것 자체가 목적이 아니라, **애플리케이션 운영 규칙 자체를 Kubernetes API로 만드는 것**이 CR 방식을 쓰는 이유입니다.

# 실습환경 세팅

좋습니다. 이번에는 **CRD/CR을 이해하기 위한 최소한의 Frontend + Backend 서비스**부터 만들겠습니다.

목표는 복잡한 기능 없이:

```
Browser
   │
   ▼
Frontend (Nginx)
   │
   │ HTTP
   ▼
Backend (FastAPI)
   │
   ▼
"Hello from Backend"
```

정도만 동작하게 만드는 것입니다.

그리고 **처음에는 CRD를 전혀 사용하지 않습니다.**

먼저 일반적인 Docker → Harbor → Deployment 방식으로 실행한 뒤, 나중에 이것을 CR로 묶어보겠습니다.

---

# 1. 실습 디렉터리 생성

`worker-1`에서 진행하는 것을 추천합니다.

```bash
cd ~
mkdir -p cr-demo
cd cr-demo
```

최종적으로 다음 구조를 만들겠습니다.

```
cr-demo/
├── frontend/
│   ├── index.html
│   └── Dockerfile
│
└── backend/
    ├── main.py
    ├── requirements.txt
    └── Dockerfile
```

---

# 2. Backend 만들기

먼저 FastAPI Backend입니다.

```bash
mkdir -p backend
cd backend
```

## `main.py`

```bash
nano main.py
```

다음 내용을 입력합니다.

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def root():
    return {
        "message": "Hello from Backend"
    }

@app.get("/api/hello")
def hello():
    return {
        "message": "Hello from FastAPI Backend"
    }
```

---

# 3. Backend requirements.txt

```bash
nano requirements.txt
```

```
fastapi
uvicorn
```

---

# 4. Backend Dockerfile

```bash
nano Dockerfile
```

```docker
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY main.py .

EXPOSE 8000

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

---

# 5. Backend 로컬 테스트

먼저 Python으로 직접 실행해도 됩니다.

```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

실행:

```bash
uvicorn main:app --host 0.0.0.0 --port 8000
```

다른 터미널에서:

```bash
curl http://localhost:8000/
```

결과:

```json
{
  "message": "Hello from Backend"
}
```

그리고:

```bash
curl http://localhost:8000/api/hello
```

결과:

```json
{
  "message": "Hello from FastAPI Backend"
}
```

정상이라면 `Ctrl+C`로 종료합니다.

---

# 6. Backend Docker 이미지 만들기

Backend 디렉터리에서:

```bash
docker build -t cr-demo-backend:1.0 .
```

확인:

```bash
docker images | grep cr-demo
```

---

# 7. Backend Docker 테스트

```bash
docker run -d \
  --name cr-demo-backend \
  -p 8000:8000 \
  cr-demo-backend:1.0
```

확인:

```bash
curl http://localhost:8000/api/hello
```

정상적으로:

```json
{
  "message": "Hello from FastAPI Backend"
}
```

가 나오면 됩니다.

테스트가 끝나면:

```bash
docker rm -f cr-demo-backend
```

---

# 8. Frontend 만들기

이제 Frontend를 만듭니다.

```bash
cd ~/cr-demo
mkdir -p frontend
cd frontend
```

이번에는 아주 단순한 HTML을 사용합니다.

## `index.html`

```bash
nano index.html
```

다음 내용을 넣습니다.

```html
<!DOCTYPE html>
<html lang="ko">

<head>
    <meta charset="UTF-8">
    <title>CR Demo</title>
</head>

<body>

    <h1>CR Demo Application</h1>

    <p>Frontend is running.</p>

    <button onclick="callBackend()">
        Call Backend
    </button>

    <pre id="result"></pre>

    <script>
        async function callBackend() {

            const response = await fetch("/api/hello");

            const data = await response.json();

            document.getElementById("result").textContent =
                JSON.stringify(data, null, 2);
        }
    </script>

</body>

</html>
```

---

# 9. Frontend Dockerfile

```bash
nano Dockerfile
```

```docker
FROM nginx:alpine

COPY index.html /usr/share/nginx/html/index.html

EXPOSE 80
```

이렇게 하면 Nginx가 `index.html`을 서비스합니다.

---

# 10. Frontend Docker 이미지 만들기

```bash
docker build -t cr-demo-frontend:1.0 .
```

확인:

```bash
docker images | grep cr-demo
```

결과는 대략:

```
cr-demo-frontend   1.0
cr-demo-backend    1.0
```

---

# 11. Frontend Docker 테스트

```bash
docker run -d \
  --name cr-demo-frontend \
  -p 8080:80 \
  cr-demo-frontend:1.0
```

브라우저에서:

```
http://localhost:8080
```

접속합니다.

다만 **여기서 Backend 연동은 아직 제대로 되지 않습니다.**

왜냐하면 Frontend에서:

```jsx
fetch("/api/hello")
```

를 호출하면 현재 Nginx가 `/api/hello`를 Backend로 전달해주지 않기 때문입니다.

이 부분은 Kubernetes의 **Service**를 배우면서 자연스럽게 연결하겠습니다.

---

# 12. 지금 단계에서 중요한 구조

현재 우리가 만든 것은 단순합니다.

```
cr-demo/
│
├── frontend/
│   ├── index.html
│   └── Dockerfile
│
└── backend/
    ├── main.py
    ├── requirements.txt
    └── Dockerfile
```

Docker 이미지로 보면:

```
Frontend
    │
    ▼
cr-demo-frontend:1.0

Backend
    │
    ▼
cr-demo-backend:1.0
```

---

# 13. 다음 단계에서 Harbor에 Push

현재 사용 중인 Harbor 주소를 그대로 활용하면 됩니다.

```
192.168.56.11:30002
```

예를 들어 프로젝트를:

```
cr-demo
```

라고 한다면 최종 이미지는:

```
192.168.56.11:30002/cr-demo/frontend:1.0
192.168.56.11:30002/cr-demo/backend:1.0
```

형태가 됩니다.

Backend:

```bash
docker tag cr-demo-backend:1.0 \
  192.168.56.11:30002/cr-demo/backend:1.0

docker push \
  192.168.56.11:30002/cr-demo/backend:1.0
```

Frontend:

```bash
docker tag cr-demo-frontend:1.0 \
  192.168.56.11:30002/cr-demo/frontend:1.0

docker push \
  192.168.56.11:30002/cr-demo/frontend:1.0
```

---

# 14. 이번 실습에서는 여기까지가 1단계입니다

전체 실습은 다음 순서로 진행하면 좋습니다.

```
[1단계] ⭐ 지금
Frontend + Backend 소스 작성
        ↓
Docker 이미지 생성
        ↓
Harbor Push

[2단계]
Frontend Deployment
Backend Deployment
        ↓
Service
        ↓
Kubernetes에서 실행

[3단계]
Frontend → Backend 연결
        ↓
실제 WebApp 완성

[4단계]
"이렇게 여러 YAML을 관리하는 게 번거롭네?"
        ↓
MyApp CRD 생성

[5단계]
MyApp CR 생성
        ↓
Controller 작성

[6단계]
MyApp CR 하나
        ↓
Frontend Deployment
Backend Deployment
Frontend Service
Backend Service
자동 생성

[7단계]
CR 수정
        ↓
Controller가 실제 리소스 변경

[8단계]
CR 삭제
        ↓
관련 리소스 자동 정리

[9단계]
일반 YAML 방식 vs CR 방식 비교
```

**특히 2~3단계를 먼저 해보는 것이 중요합니다.**

그래야 나중에 `MyApp CR`을 봤을 때

> "아, 원래 내가 Deployment 2개 + Service 2개를 직접 만들던 것을 `MyApp`이라는 하나의 애플리케이션 객체로 표현하고, Controller가 그 세부 리소스를 관리하도록 만든 것이구나."
> 

라는 차이를 직접 체감할 수 있습니다.

다음 단계에서는 **방금 만든 두 이미지를 Kubernetes에서 `frontend Deployment + backend Deployment + Service`로 실제 실행하는 단계**로 이어가면 됩니다.

# 일반 Deployment 방식과 CR 배포 방식

좋습니다. 이제부터는 **CR을 이해하는 것이 목적**이므로, 처음부터 복잡하게 만들지 않고 다음 순서로 진행하는 것이 가장 좋습니다.

> **일반 Deployment 방식으로 먼저 완성 → 같은 서비스를 CR 하나로 표현 → Controller가 Deployment/Service를 자동 생성**
> 

이번 실습에서는 기존 Auto-Healer 프로젝트와 **분리된 새로운 `cr-demo` 프로젝트**로 진행하겠습니다.

---

# 전체 실습

최종 목표는 다음입니다.

```
                    [처음]
              일반 Kubernetes
                    │
        ┌───────────┴───────────┐
        ▼                       ▼
 frontend Deployment       backend Deployment
        │                       │
 frontend Service          backend Service

                    [최종]
                  MyApp CR
                     │
                     ▼
               MyApp Controller
                 │         │
                 ▼         ▼
             frontend    backend
             Deployment  Deployment
                 │         │
              Service    Service
```

그리고 마지막에는:

```bash
kubectl apply -f myapp.yaml
```

**이 명령 하나로 Frontend + Backend가 생성되는 것**을 확인합니다.

---

# STEP 1. 현재 프로젝트 확인

먼저 기존에 만든 프로젝트가 제대로 있는지 확인합니다.

```bash
cd ~/cr-demo

find . -maxdepth 2 -type f
```

예상:

```
./frontend/index.html
./frontend/Dockerfile
./backend/main.py
./backend/requirements.txt
./backend/Dockerfile
```

---

# STEP 2. Frontend 이미지에 Backend Proxy 추가

앞서 만든 Frontend는:

```jsx
fetch("/api/hello")
```

를 호출하지만, Nginx가 Backend로 전달하지 않았습니다.

따라서 이번에는 Nginx 설정을 추가합니다.

Frontend 디렉터리:

```bash
cd ~/cr-demo/frontend
```

`nginx.conf` 생성:

```bash
nano nginx.conf
```

다음 내용:

```
server {             // NginX 서버 블록(가상 서버) 정의
    listen 80;       // Nginx가 HTTP 80번 포트에서 요청을 받는다. 
                     // http://192.168.56.11로 접속하면 80번 포트로 요청이 들어온다. 
    location / {     // URL이 /로 시작하는 일반적 웹 요청 처리 
        root /usr/share/nginx/html;  // 정적 파일이 저장된 디렉토리 지정 
        index index.html;  // 사용자가 /만 요청했을 때 기본으로 index.html을 찾는다. 
    }                 // 즉 http://localhost/ --> /usr/share/nginx/html/index.html

    location /api/ {   //  /api/로 시작하는 요청은 backend 서버로 전달
        proxy_pass http://backend:8000;
    }
}
```

그리고 Dockerfile을 수정합니다.

```bash
nano Dockerfile
```

```docker
FROM nginx:alpine

COPY index.html /usr/share/nginx/html/index.html
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80
```

여기서 중요한 부분:

```
/api/*
   ↓
http://backend:8000
```

입니다.

즉 Kubernetes에서 `backend`라는 Service가 존재하면:

```
Frontend
    │
    │ /api/hello
    ▼
backend Service
    │
    ▼
Backend Pod
```

구조가 됩니다.

---

# STEP 3. 이미지 다시 빌드

```bash
cd ~/cr-demo/frontend

docker build -t cr-demo-frontend:1.0 .
```

Backend도 확인:

```bash
cd ~/cr-demo/backend

docker build -t cr-demo-backend:1.0 .
```

확인:

```bash
docker images | grep cr-demo
```

---

# STEP 4. Harbor에 Push

현재 사용 중인 Harbor를 그대로 사용합니다.

```
192.168.56.11:30002
```

이미지 이름을 다음처럼 사용하겠습니다.

```
192.168.56.11:30002/cr-demo/frontend:1.0
192.168.56.11:30002/cr-demo/backend:1.0
```

Frontend:

```bash
docker tag cr-demo-frontend:1.0 \
  192.168.56.11:30002/cr-demo/frontend:1.0

docker push \
  192.168.56.11:30002/cr-demo/frontend:1.0
```

Backend:

```bash
docker tag cr-demo-backend:1.0 \
  192.168.56.11:30002/cr-demo/backend:1.0

docker push \
  192.168.56.11:30002/cr-demo/backend:1.0
```

Harbor에서 두 이미지가 올라갔는지 확인합니다.

---

# STEP 5. Kubernetes Namespace 생성

이번 실습을 별도 namespace에서 진행합니다.

```bash
kubectl create namespace cr-demo
```

확인:

```bash
kubectl get ns cr-demo
```

---

# STEP 6. 먼저 일반적인 Deployment 방식으로 배포

**이 단계가 매우 중요합니다.**

아직 CRD를 만들지 않습니다.

먼저 우리가 평소 하던 방식으로 서비스를 완성합니다.

---

## 6-1. Backend Deployment

```bash
cd ~/cr-demo
nano backend-deployment.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: backend
  namespace: cr-demo

spec:
  replicas: 2

  selector:
    matchLabels:
      app: backend

  template:
    metadata:
      labels:
        app: backend

    spec:
      containers:
      - name: backend
        image: 192.168.56.11:30002/cr-demo/backend:1.0

        ports:
        - containerPort: 8000
```

적용:

```bash
kubectl apply -f backend-deployment.yaml
```

확인:

```bash
kubectl get pods -n cr-demo
```

---

# STEP 7. Backend Service

```bash
nano backend-service.yaml
```

```yaml
apiVersion: v1
kind: Service

metadata:
  name: backend
  namespace: cr-demo

spec:
  selector:
    app: backend

  ports:
  - port: 8000
    targetPort: 8000

  type: ClusterIP
```

적용:

```bash
kubectl apply -f backend-service.yaml
```

확인:

```bash
kubectl get svc -n cr-demo
```

예상:

```
NAME       TYPE        CLUSTER-IP
backend   ClusterIP   10.x.x.x
```

여기서 중요한 것은 Service 이름입니다.

```
backend
```

Frontend Nginx에서:

```
http://backend:8000
```

으로 접근할 수 있게 됩니다.

---

# STEP 8. Frontend Deployment

```bash
nano frontend-deployment.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: frontend
  namespace: cr-demo

spec:
  replicas: 2

  selector:
    matchLabels:
      app: frontend

  template:
    metadata:
      labels:
        app: frontend

    spec:
      containers:
      - name: frontend
        image: 192.168.56.11:30002/cr-demo/frontend:1.0

        ports:
        - containerPort: 80
```

적용:

```bash
kubectl apply -f frontend-deployment.yaml
```

확인:

```bash
kubectl get pods -n cr-demo
```

이제:

```
backend   2 Pods
frontend  2 Pods
```

가 실행되어야 합니다.

---

# STEP 9. Frontend Service

```bash
nano frontend-service.yaml
```

```yaml
apiVersion: v1
kind: Service

metadata:
  name: frontend
  namespace: cr-demo

spec:
  selector:
    app: frontend

  ports:
  - port: 80
    targetPort: 80

  type: NodePort
```

적용:

```bash
kubectl apply -f frontend-service.yaml
```

확인:

```bash
kubectl get svc -n cr-demo
```

예:

```
NAME        TYPE       PORT
backend     ClusterIP  8000
frontend    NodePort   80:30xxx
```

---

# STEP 10. 현재 상태 확인

```bash
kubectl get all -n cr-demo
```

대략:

```
NAME                            READY
pod/backend-xxxxx               1/1
pod/backend-yyyyy               1/1
pod/frontend-xxxxx              1/1
pod/frontend-yyyyy              1/1

NAME              TYPE
service/backend   ClusterIP
service/frontend  NodePort

NAME                       READY
deployment.apps/backend   2/2
deployment.apps/frontend  2/2
```

여기까지 성공하면 **일반 Kubernetes 방식의 WebApp이 완성된 것입니다.**

---

# STEP 11. Frontend → Backend 통신 확인

먼저 NodePort 확인:

```bash
kubectl get svc frontend -n cr-demo
```

예를 들어:

```
80:30100/TCP
```

이라면 브라우저에서:

```
http://192.168.56.11:30100
```

접속합니다.

화면:

```
CR Demo Application

Frontend is running.

[ Call Backend ]
```

버튼을 누릅니다.

정상이라면:

```json
{
  "message": "Hello from FastAPI Backend"
}
```

가 나옵니다.

이것이 이번 실습의 **기준점**입니다.

---

# ⭐ STEP 12. 이제 문제를 발견합니다

현재 애플리케이션을 구성하는 Kubernetes YAML을 확인해보세요.

```
backend-deployment.yaml
backend-service.yaml
frontend-deployment.yaml
frontend-service.yaml
```

총 4개입니다.

그리고 실제 서비스가 커지면:

```
Deployment
Service
ConfigMap
Secret
Ingress
HPA
PDB
ServiceMonitor
...
```

점점 늘어납니다.

하지만 사용자 입장에서 이것은:

```
MyApp
```

하나입니다.

그래서 우리가 이런 식으로 표현하고 싶은 겁니다.

```yaml
kind: MyApp

spec:

  frontend:
    image: ...
    replicas: 2

  backend:
    image: ...
    replicas: 2
```

**이제부터 CRD를 도입합니다.**

---

# STEP 13. MyApp CRD 생성

```bash
nano myapp-crd.yaml
```

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition

metadata:
  name: myapps.example.com

spec:
  group: example.com

  names:
    plural: myapps
    singular: myapp
    kind: MyApp

  scope: Namespaced

  versions:
  - name: v1
    served: true
    storage: true

    schema:
      openAPIV3Schema:
        type: object

        properties:
          spec:
            type: object

            properties:

              frontend:
                type: object

                properties:
                  image:
                    type: string

                  replicas:
                    type: integer

              backend:
                type: object

                properties:
                  image:
                    type: string

                  replicas:
                    type: integer
```

적용:

```bash
kubectl apply -f myapp-crd.yaml
```

확인:

```bash
kubectl get crd
```

그리고:

```bash
kubectl api-resources | grep myapp
```

---

# STEP 14. MyApp CR 생성

이제 실제 CR을 만듭니다.

```bash
nano myapp.yaml
```

```yaml
apiVersion: example.com/v1
kind: MyApp

metadata:
  name: myapp
  namespace: cr-demo

spec:

  frontend:
    image: 192.168.56.11:30002/cr-demo/frontend:1.0
    replicas: 2

  backend:
    image: 192.168.56.11:30002/cr-demo/backend:1.0
    replicas: 2
```

적용:

```bash
kubectl apply -f myapp.yaml
```

확인:

```bash
kubectl get myapp -n cr-demo
```

여기서 중요한 사실:

```
MyApp CR
```

은 생성됐지만 **아무 Deployment도 새로 만들어지지 않습니다.**

왜냐하면 아직 Controller가 없기 때문입니다.

이것을 직접 확인합니다.

```bash
kubectl get deployment -n cr-demo
```

### CRD와 CR의 관계

네. 두 파일의 관계를 이해할 때 핵심은 **CRD가 "MyApp이라는 리소스의 설계도/스키마"를 정의하고, CR이 그 설계도에 따라 실제 MyApp 객체를 생성한다**는 것입니다.

먼저 전체 관계를 보면 다음과 같습니다.

```
                 CRD
┌──────────────────────────────────────┐
│ CustomResourceDefinition             │
│                                      │
│ group: example.com                   │
│ version: v1                           │
│ kind: MyApp                          │
│ plural: myapps                       │
│                                      │
│ spec.frontend.image   → string       │
│ spec.frontend.replicas → integer     │
│ spec.backend           → object      │
└──────────────────┬───────────────────┘
                   │
                   │ "이런 형식의 객체를 허용한다"
                   ▼
                  CR
┌──────────────────────────────────────┐
│ apiVersion: example.com/v1           │
│ kind: MyApp                           │
│ metadata.name: myapp                 │
│                                      │
│ spec:                                │
│   frontend:                           │
│     image: ...                        │
│     replicas: 2                       │
│   backend:                            │
│     image: ...                        │
│     replicas: 2                       │
└──────────────────────────────────────┘
                   │
                   │ Controller가 읽음
                   ▼
          Kubernetes 리소스 생성
       ┌────────────┬────────────┐
       ▼            ▼            ▼
   Deployment    Service      ...
   frontend      frontend
   backend       backend
```

## 1. 가장 중요한 연결: `group`

CRD:

```yaml
spec:
  group: example.com
```

CR:

```yaml
apiVersion: example.com/v1
```

여기서

```
example.com
```

이 서로 연결됩니다.

CR의

```yaml
apiVersion: example.com/v1
```

은 다음과 같이 해석됩니다.

```
group   = example.com
version = v1
```

즉,

```yaml
spec:
  group: example.com
  versions:
    - name: v1
```

와 연결됩니다.

---

# 2. `version` 연결

CRD:

```yaml
versions:
  - name: v1
    served: true
    storage: true
```

CR:

```yaml
apiVersion: example.com/v1
```

따라서:

```
CRD
 └── versions
      └── v1
            ↑
            │
CR ─────────┘
apiVersion: example.com/v1
```

입니다.

---

# 3. `kind: MyApp` 연결

CRD:

```yaml
names:
  kind: MyApp
```

CR:

```yaml
kind: MyApp
```

이 둘이 직접 연결됩니다.

즉 CRD가:

> Kubernetes에 `MyApp`이라는 새로운 종류의 리소스를 만들겠다.
> 

라고 선언합니다.

그리고 CR에서는:

> 그 MyApp 리소스 중 하나인 `myapp`을 만들겠다.
> 

라고 하는 것입니다.

---

# 4. `plural: myapps`는 어디에 사용되나?

CRD:

```yaml
names:
  plural: myapps
  singular: myapp
  kind: MyApp
```

여기서:

```
kind     = MyApp
singular = myapp
plural   = myapps
```

입니다.

예를 들어:

```bash
kubectl get myapps -n cr-demo
```

라고 할 수 있습니다.

또는:

```bash
kubectl get myapp -n cr-demo
```

처럼 singular 이름을 사용할 수도 있습니다.

그리고 내부적으로 Kubernetes API 경로는 대략:

```
/apis/example.com/v1/namespaces/cr-demo/myapps
```

가 됩니다.

즉 `plural: myapps`는 **Kubernetes API에서 이 리소스들을 어떻게 부를지**를 정의합니다.

---

# 5. `scope: Namespaced`

CRD:

```yaml
scope: Namespaced
```

CR:

```yaml
metadata:
  name: myapp
  namespace: cr-demo
```

이것도 연결됩니다.

`Namespaced`이므로 MyApp 객체는 특정 Namespace에 속할 수 있습니다.

따라서:

```yaml
metadata:
  name: myapp
  namespace: cr-demo
```

가 가능합니다.

결과적으로:

```
cr-demo Namespace
       │
       └── MyApp/myapp
```

구조가 됩니다.

만약 CRD가:

```yaml
scope: Cluster
```

였다면 CR에서:

```yaml
namespace: cr-demo
```

를 사용할 수 없습니다.

---

# 6. 가장 중요한 `spec` 연결

여기부터가 실제로 **CRD와 CR이 데이터를 주고받는 부분**입니다.

CRD:

```yaml
schema:
  openAPIV3Schema:
    type: object
    properties:
      spec:
        type: object
        properties:
          frontend:
            type: object
            properties:
              image:
                type: string
              replicas:
                type: integer
```

이것은:

> MyApp의 `spec` 안에는 `frontend`라는 객체가 있고, 그 안에는 `image`와 `replicas`가 있어야 한다.
> 

라는 규칙입니다.

그리고 실제 CR에서:

```yaml
spec:
  frontend:
    image: 192.168.56.11:30002/cr-demo/frontend:1.0
    replicas: 2
```

로 값을 넣습니다.

즉 정확하게:

```
CRD                              CR
────────────────────────────────────────────

spec                             spec
 │                                │
 └─ frontend          ←───────────┘
       │                            │
       ├─ image:string  ←────────── image: "...:1.0"
       │
       └─ replicas:int   ←────────── replicas: 2
```

입니다.

---

# 7. `image: type: string`

CRD:

```yaml
image:
  type: string
```

CR:

```yaml
image: 192.168.56.11:30002/cr-demo/frontend:1.0
```

CR의 값이 문자열이므로 정상입니다.

예를 들어 다음도 정상입니다.

```yaml
image: nginx:1.27
```

하지만:

```yaml
image: 12345
```

처럼 숫자를 넣으면 스키마 설정에 따라 validation 문제가 발생할 수 있습니다.

---

# 8. `replicas: type: integer`

CRD:

```yaml
replicas:
  type: integer
```

CR:

```yaml
replicas: 2
```

따라서:

```
CRD                         CR
────────────────────────────────
replicas: integer    ←───  replicas: 2
```

입니다.

이 값은 나중에 **Controller가 실제 Deployment의 replicas로 사용**할 수 있습니다.

예를 들어 Controller가 다음과 같이 구현되어 있다면:

```python
frontend_replicas = myapp.spec["frontend"]["replicas"]
```

여기서:

```
2
```

를 가져옵니다.

그리고:

```python
deployment.spec.replicas = frontend_replicas
```

와 같이 사용해서 Kubernetes Deployment를 생성할 수 있습니다.

---

# 9. Backend도 동일한 구조

현재 CR:

```yaml
spec:
  frontend:
    image: 192.168.56.11:30002/cr-demo/frontend:1.0
    replicas: 2

  backend:
    image: 192.168.56.11:30002/cr-demo/backend:1.0
    replicas: 2
```

CRD도 다음처럼 정의할 수 있습니다.

```yaml
properties:
  frontend:
    type: object
    properties:
      image:
        type: string
      replicas:
        type: integer

  backend:
    type: object
    properties:
      image:
        type: string
      replicas:
        type: integer
```

그러면:

```
                     MyApp
                       │
                      spec
                   ┌───┴───┐
                   │       │
               frontend   backend
                   │       │
              ┌────┴───┐ ┌─┴────────┐
              │         │ │          │
            image    replicas image  replicas
              │         │      │        │
              ▼         ▼      ▼        ▼
           frontend     2    backend    2
            image             image
```

이라는 구조가 됩니다.

---

# 10. 중요한 오해: CRD가 Deployment를 만드는 것은 아닙니다

이 부분이 **CRD 실습에서 가장 중요합니다.**

CRD:

```yaml
kind: CustomResourceDefinition
```

은 단지 **새로운 Kubernetes API 리소스의 형식과 구조를 정의**합니다.

CR:

```yaml
kind: MyApp
```

은 그 리소스를 실제로 생성합니다.

하지만 이것만으로:

```
MyApp
  ↓
Deployment
Service
Pod
```

가 자동으로 만들어지는 것은 아닙니다.

그 역할을 하는 것이 **Controller/Operator**입니다.

즉 전체 구조는:

```
                 ① CRD 설치
                     │
                     ▼
              Kubernetes API
              "MyApp을 알아야 함"
                     │
                     │
                 ② CR 생성
                     │
                     ▼
                MyApp/myapp
                     │
                     │
              ③ Controller 감시
                     │
                     ▼
              Controller 로직
                     │
          ┌──────────┴──────────┐
          ▼                     ▼
     Frontend                  Backend
    Deployment                Deployment
          │                     │
          ▼                     ▼
       Pods                    Pods
```

입니다.

---

# 11. 현재 실습 코드에 대입하면

현재 CR:

```yaml
apiVersion: example.com/v1
kind: MyApp

metadata:
  name: myapp
  namespace: cr-demo

spec:
  frontend:
    image: 192.168.56.11:30002/cr-demo/frontend:1.0
    replicas: 2

  backend:
    image: 192.168.56.11:30002/cr-demo/backend:1.0
    replicas: 2
```

을 Controller가 읽으면 대략 다음과 같이 사용할 수 있습니다.

```python
myapp.spec["frontend"]["image"]
```

결과:

```
192.168.56.11:30002/cr-demo/frontend:1.0
```

그리고:

```python
myapp.spec["frontend"]["replicas"]
```

결과:

```
2
```

Backend는:

```python
myapp.spec["backend"]["image"]
```

→

```
192.168.56.11:30002/cr-demo/backend:1.0
```

```python
myapp.spec["backend"]["replicas"]
```

→

```
2
```

가 됩니다.

Controller는 이 값을 가지고 실제 Kubernetes 리소스를 생성합니다.

---

## 12. 한눈에 보는 "누가 무엇을 정의하는가"

| 항목 | CRD | CR |
| --- | --- | --- |
| API Group | `example.com` 정의 | `example.com/v1` 사용 |
| Version | `v1` 정의 | `example.com/v1` 사용 |
| Kind | `MyApp` 정의 | `MyApp` 사용 |
| 이름 | `myapp`/`myapps` 규칙 정의 | `metadata.name: myapp` |
| Namespace | `Namespaced` 정의 | `namespace: cr-demo` |
| `spec.frontend` | 어떤 구조인지 정의 | 실제 frontend 값 지정 |
| `frontend.image` | `string`이라고 정의 | 실제 이미지 지정 |
| `frontend.replicas` | `integer`라고 정의 | `2` 지정 |
| `backend` | 구조 정의 | 실제 backend 값 지정 |
| Deployment 생성 | ❌ | ❌ |
| Pod 생성 | ❌ | ❌ |
| 실제 동작 | Controller 필요 | Controller 필요 |

### 핵심적으로 기억하면

**CRD = "MyApp이라는 리소스는 이런 모양이다"**

```
MyApp
 └─ spec
     ├─ frontend
     │   ├─ image: string
     │   └─ replicas: integer
     │
     └─ backend
         ├─ image: string
         └─ replicas: integer
```

**CR = "그 MyApp에 실제로 이 값을 넣겠다"**

```
MyApp/myapp
 └─ spec
     ├─ frontend
     │   ├─ image: frontend:1.0
     │   └─ replicas: 2
     │
     └─ backend
         ├─ image: backend:1.0
         └─ replicas: 2
```

**Controller = "CR의 값을 읽어서 실제 Kubernetes 리소스로 만들어 주겠다"**

이 3개를 **"CRD → CR → Controller → Deployment/Service/Pod"** 관계로 이해하시면, 지금 진행하신 `MyApp` 실습과 이후 **Operator/Auto-Healer 구조**까지 자연스럽게 연결됩니다.

---

# ⭐ STEP 15. Controller가 필요한 이유 확인

현재 구조는:

```
myapp.yaml
    │
    ▼
MyApp CR
```

여기서 끝입니다.

Kubernetes는:

> "MyApp이라는 객체가 있구나."
> 

까지만 압니다.

Kubernetes가:

> "그러면 frontend Deployment와 backend Deployment를 만들어야겠네."
> 

라고 판단하지 않습니다.

그 판단을 하는 프로그램이 바로 **Controller**입니다.

---

# STEP 16. 이제 Controller를 만든다

Controller의 역할은 아주 단순하게 시작합니다.

```
MyApp CR 감시
      ↓
frontend 정보 읽기
      ↓
frontend Deployment 생성

backend 정보 읽기
      ↓
backend Deployment 생성
```

처음에는 Service까지 자동 생성하지 않고 **Deployment만 자동 생성**하는 것을 추천합니다.

왜냐하면 CR의 핵심 개념을 이해하는 것이 목적이기 때문입니다.

---

# STEP 17. Controller 구조

새 디렉터리를 만듭니다.

```bash
cd ~/cr-demo

mkdir controller
cd controller
```

구조:

```
controller/
└── controller.py
```

필요한 패키지:

```
kopf
kubernetes
```

`requirements.txt`:

```bash
nano requirements.txt
```

```
kopf
kubernetes
```

---

# STEP 18. Controller의 핵심 동작

Controller는 다음을 수행합니다.

```
MyApp 생성
     ↓
Controller 이벤트 발생
     ↓
spec.frontend 읽기
     ↓
Deployment 생성

spec.backend 읽기
     ↓
Deployment 생성
```

여기서 **기존의 Deployment YAML을 Controller Python 코드로 생성하는 것**이라고 생각하면 됩니다.

---

# STEP 19. Controller 구현

`controller.py`에 다음과 같은 구조를 작성합니다.

```python
import kopf
from kubernetes import client

@kopf.on.create("example.com", "v1", "myapps")
def create_fn(spec, name, namespace, **kwargs):

    apps = client.AppsV1Api()

    frontend = spec["frontend"]
    backend = spec["backend"]

    create_deployment(
        apps,
        name=f"{name}-frontend",
        namespace=namespace,
        image=frontend["image"],
        replicas=frontend["replicas"],
        app="frontend",
    )

    create_deployment(
        apps,
        name=f"{name}-backend",
        namespace=namespace,
        image=backend["image"],
        replicas=backend["replicas"],
        app="backend",
    )

def create_deployment(
    apps,
    name,
    namespace,
    image,
    replicas,
    app,
):

    deployment = client.V1Deployment(
        metadata=client.V1ObjectMeta(
            name=name,
            namespace=namespace,
        ),

        spec=client.V1DeploymentSpec(
            replicas=replicas,

            selector=client.V1LabelSelector(
                match_labels={"app": f"{name}"}
            ),

            template=client.V1PodTemplateSpec(
                metadata=client.V1ObjectMeta(
                    labels={"app": f"{name}"}
                ),

                spec=client.V1PodSpec(
                    containers=[
                        client.V1Container(
                            name=app,
                            image=image,
                        )
                    ]
                )
            )
        )
    )

    apps.create_namespaced_deployment(
        namespace=namespace,
        body=deployment,
    )
```

이 코드의 핵심은 이것입니다.

```
CR의 spec
   ↓
Python Controller
   ↓
Kubernetes API
   ↓
Deployment
```

---

# STEP 20. Controller 실행

환경을 구성하고:

```bash
cd ~/cr-demo/controller

python3 -m venv venv
source venv/bin/activate

pip install -r requirements.txt
```

그 다음:

```bash
kopf run controller.py
```

Controller가 실행된 상태로 둡니다.

---

# STEP 21. CR을 다시 생성

다른 터미널에서:

```bash
kubectl delete -f ~/cr-demo/myapp.yaml
```

그리고:

```bash
kubectl apply -f ~/cr-demo/myapp.yaml
```

Controller가 이벤트를 받아 처리합니다.

확인:

```bash
kubectl get deployment -n cr-demo
```

이제:

```
myapp-frontend
myapp-backend
```

가 생성됩니다.

Pod:

```bash
kubectl get pods -n cr-demo
```

---

# ⭐ 여기서 CR의 의미가 보입니다

사용자가 실행한 것은:

```bash
kubectl apply -f myapp.yaml
```

하나입니다.

그런데 결과는:

```
MyApp
 │
 ├── frontend Deployment
 │
 └── backend Deployment
```

입니다.

즉:

```
사용자
  │
  │ MyApp이라는 애플리케이션을
  │ 이렇게 구성하고 싶다.
  ▼
MyApp CR
  │
  ▼
Controller
  │
  ├── Deployment
  └── Deployment
```

입니다.

---

# STEP 22. CR 수정 실습

이제 `myapp.yaml`을 수정합니다.

기존:

```yaml
frontend:
  replicas: 2
```

변경:

```yaml
frontend:
  replicas: 4
```

적용:

```bash
kubectl apply -f myapp.yaml
```

여기서 중요한 점이 있습니다.

**Controller를 제대로 구현하려면 `on.create`만으로는 부족합니다.**

실제 Operator는:

```
create
update
delete
```

모두 감시해야 합니다.

따라서 다음 단계에서:

```
@kopf.on.create
@kopf.on.update
@kopf.on.delete
```

또는 reconcile 방식으로 발전시킵니다.

---

# STEP 23. 최종적으로 우리가 만들고 싶은 Controller

최종 Controller는:

```
                MyApp CR
                   │
          ┌────────┼────────┐
          │        │        │
          ▼        ▼        ▼
       frontend  backend   policy
          │        │
          ▼        ▼
    Deployment  Deployment
          │        │
          ▼        ▼
       Service   Service
```

그리고:

```
CR 변경
 ↓
Controller
 ↓
실제 리소스 수정
```

CR 삭제:

```
CR 삭제
 ↓
Controller
 ↓
Deployment 삭제
Service 삭제
```

까지 수행하도록 합니다.

---

# 이번 실습에서 가장 중요한 관찰 포인트

단순히 코드를 따라 치는 것보다 아래 **4개의 순간**을 직접 확인하는 것이 중요합니다.

### ① 일반 YAML

```
4개의 YAML
 ↓
4개의 Kubernetes 리소스
```

### ② CRD만 설치

```
CRD
 ↓
Kubernetes가 MyApp을 이해함
```

하지만:

```
Deployment 생성 ❌
```

### ③ CR 생성

```
MyApp CR
 ↓
MyApp 객체 생성
```

하지만:

```
Deployment 생성 ❌
```

### ④ Controller 실행 후 CR 생성

```
MyApp CR
 ↓
Controller
 ↓
Deployment
 ↓
Pod
```

**바로 이 ④번의 차이를 경험하는 것이 이번 실습의 핵심입니다.**

---

## 그리고 다음 단계

여기까지 한 다음에는 Controller를 조금 더 제대로 발전시키는 것이 좋습니다.

```
현재
MyApp CR
   ↓
Controller
   ↓
Deployment 2개
```

에서

```
다음
MyApp CR
   ↓
Controller
   ↓
├── Frontend Deployment
├── Frontend Service
├── Backend Deployment
└── Backend Service
```

로 확장하고,

마지막으로:

```
MyApp CR replicas 변경
        ↓
Controller
        ↓
Deployment replicas 자동 변경
```

을 구현합니다.

**그 단계까지 가면 "CR은 단순히 YAML을 하나로 줄이는 기술이 아니라, Kubernetes에 내가 정의한 애플리케이션 운영 모델을 추가하는 것"이라는 개념이 확실하게 잡힙니다.**

# Operator 기능 확장

좋습니다. 이제 기존 Controller를 **`create / update / delete` 이벤트를 모두 처리하는 형태**로 확장해보겠습니다.

다만 여기서 한 가지 중요한 점이 있습니다.

`@kopf.on.create`, `@kopf.on.update`, `@kopf.on.delete`를 단순히 각각 구현하는 것보다, **이번 실습에서는 이벤트가 발생했을 때 실제 Kubernetes 리소스가 어떻게 변하는지 확인하는 것**이 목적이므로 `Deployment + Service`까지 함께 관리하도록 만드는 것이 좋습니다.

아래 코드는 현재 `~/cr-demo/controller/controller.py`를 대체해서 사용할 수 있는 전체 코드입니다.

### 1. Controller 전체 코드

```python
import kopf
from kubernetes import client

# --------------------------------------------------
# Deployment 생성/수정
# --------------------------------------------------

def create_or_update_deployment(
    apps,
    name,
    namespace,
    image,
    replicas,
    container_port,
    labels,
):
    deployment = client.V1Deployment(
        metadata=client.V1ObjectMeta(
            name=name,
            namespace=namespace,
        ),

        spec=client.V1DeploymentSpec(
            replicas=replicas,

            selector=client.V1LabelSelector(
                match_labels=labels
            ),

            template=client.V1PodTemplateSpec(
                metadata=client.V1ObjectMeta(
                    labels=labels
                ),

                spec=client.V1PodSpec(
                    containers=[
                        client.V1Container(
                            name=labels["app"],
                            image=image,
                            ports=[
                                client.V1ContainerPort(
                                    container_port=container_port
                                )
                            ],
                        )
                    ]
                )
            )
        )
    )

    try:
        apps.read_namespaced_deployment(
            name=name,
            namespace=namespace,
        )

        # 이미 존재하면 수정
        apps.replace_namespaced_deployment(
            name=name,
            namespace=namespace,
            body=deployment,
        )

        print(f"[UPDATE] Deployment: {name}")

    except client.exceptions.ApiException as e:

        if e.status == 404:

            # 없으면 생성
            apps.create_namespaced_deployment(
                namespace=namespace,
                body=deployment,
            )

            print(f"[CREATE] Deployment: {name}")

        else:
            raise

# --------------------------------------------------
# Service 생성/수정
# --------------------------------------------------

def create_or_update_service(
    core,
    name,
    namespace,
    port,
    target_port,
    selector,
):
    service = client.V1Service(
        metadata=client.V1ObjectMeta(
            name=name,
            namespace=namespace,
        ),

        spec=client.V1ServiceSpec(
            selector=selector,

            ports=[
                client.V1ServicePort(
                    port=port,
                    target_port=target_port,
                )
            ],

            type="ClusterIP",
        )
    )

    try:
        core.read_namespaced_service(
            name=name,
            namespace=namespace,
        )

        core.replace_namespaced_service(
            name=name,
            namespace=namespace,
            body=service,
        )

        print(f"[UPDATE] Service: {name}")

    except client.exceptions.ApiException as e:

        if e.status == 404:

            core.create_namespaced_service(
                namespace=namespace,
                body=service,
            )

            print(f"[CREATE] Service: {name}")

        else:
            raise

# --------------------------------------------------
# CREATE
# --------------------------------------------------

@kopf.on.create("example.com", "v1", "myapps")
def create_myapp(spec, name, namespace, **kwargs):

    print(f"[CREATE] MyApp: {name}")

    apps = client.AppsV1Api()
    core = client.CoreV1Api()

    frontend = spec["frontend"]
    backend = spec["backend"]

    # -----------------------------
    # Frontend
    # -----------------------------

    frontend_labels = {
        "app": f"{name}-frontend"
    }

    create_or_update_deployment(
        apps=apps,
        name=f"{name}-frontend",
        namespace=namespace,
        image=frontend["image"],
        replicas=frontend["replicas"],
        container_port=80,
        labels=frontend_labels,
    )

    create_or_update_service(
        core=core,
        name=f"{name}-frontend",
        namespace=namespace,
        port=80,
        target_port=80,
        selector=frontend_labels,
    )

    # -----------------------------
    # Backend
    # -----------------------------

    backend_labels = {
        "app": f"{name}-backend"
    }

    create_or_update_deployment(
        apps=apps,
        name=f"{name}-backend",
        namespace=namespace,
        image=backend["image"],
        replicas=backend["replicas"],
        container_port=8000,
        labels=backend_labels,
    )

    create_or_update_service(
        core=core,
        name=f"{name}-backend",
        namespace=namespace,
        port=8000,
        target_port=8000,
        selector=backend_labels,
    )

    print(f"[CREATE] MyApp {name} completed")

# --------------------------------------------------
# UPDATE
# --------------------------------------------------

@kopf.on.update("example.com", "v1", "myapps")
def update_myapp(spec, name, namespace, **kwargs):

    print(f"[UPDATE] MyApp: {name}")

    apps = client.AppsV1Api()
    core = client.CoreV1Api()

    frontend = spec["frontend"]
    backend = spec["backend"]

    # -----------------------------
    # Frontend
    # -----------------------------

    frontend_labels = {
        "app": f"{name}-frontend"
    }

    create_or_update_deployment(
        apps=apps,
        name=f"{name}-frontend",
        namespace=namespace,
        image=frontend["image"],
        replicas=frontend["replicas"],
        container_port=80,
        labels=frontend_labels,
    )

    create_or_update_service(
        core=core,
        name=f"{name}-frontend",
        namespace=namespace,
        port=80,
        target_port=80,
        selector=frontend_labels,
    )

    # -----------------------------
    # Backend
    # -----------------------------

    backend_labels = {
        "app": f"{name}-backend"
    }

    create_or_update_deployment(
        apps=apps,
        name=f"{name}-backend",
        namespace=namespace,
        image=backend["image"],
        replicas=backend["replicas"],
        container_port=8000,
        labels=backend_labels,
    )

    create_or_update_service(
        core=core,
        name=f"{name}-backend",
        namespace=namespace,
        port=8000,
        target_port=8000,
        selector=backend_labels,
    )

    print(f"[UPDATE] MyApp {name} completed")

# --------------------------------------------------
# DELETE
# --------------------------------------------------

@kopf.on.delete("example.com", "v1", "myapps")
def delete_myapp(name, namespace, **kwargs):

    print(f"[DELETE] MyApp: {name}")

    apps = client.AppsV1Api()
    core = client.CoreV1Api()

    resources = [
        ("deployment", f"{name}-frontend"),
        ("deployment", f"{name}-backend"),
        ("service", f"{name}-frontend"),
        ("service", f"{name}-backend"),
    ]

    for resource_type, resource_name in resources:

        try:

            if resource_type == "deployment":

                apps.delete_namespaced_deployment(
                    name=resource_name,
                    namespace=namespace,
                )

            elif resource_type == "service":

                core.delete_namespaced_service(
                    name=resource_name,
                    namespace=namespace,
                )

            print(
                f"[DELETE] {resource_type}: {resource_name}"
            )

        except client.exceptions.ApiException as e:

            if e.status == 404:

                print(
                    f"[DELETE] Already absent: "
                    f"{resource_type} {resource_name}"
                )

            else:
                raise

    print(f"[DELETE] MyApp {name} completed")
```

이제 중요한 것은 **코드를 바로 실행하는 것보다 이벤트별로 실험하는 것**입니다.

---

# 2. 기존 리소스 정리

현재 이전 실습에서 만들어진 Deployment가 있다면 먼저 정리합니다.

```bash
kubectl get all -n cr-demo
```

기존에 직접 만들었던:

```
frontend
backend
```

Deployment/Service가 있다면 삭제합니다.

```bash
kubectl delete deployment frontend backend -n cr-demo
kubectl delete service frontend backend -n cr-demo
```

그리고 MyApp도 확인합니다.

```bash
kubectl get myapp -n cr-demo
```

---

# 3. Controller 실행

```bash
cd ~/cr-demo/controller

source venv/bin/activate

kopf run controller.py
```

정상적으로 실행되면 Controller가 Kubernetes API를 감시합니다.

---

# 4. CREATE 실습

다른 터미널에서:

```bash
kubectl apply -f ~/cr-demo/myapp.yaml
```

확인:

```bash
kubectl get myapp -n cr-demo
```

그리고:

```bash
kubectl get deployment -n cr-demo
```

예상:

```
NAME              READY
myapp-frontend    2/2
myapp-backend     2/2
```

Service:

```bash
kubectl get svc -n cr-demo
```

예상:

```
NAME              TYPE
myapp-frontend    ClusterIP
myapp-backend     ClusterIP
```

즉:

```
MyApp CR 생성
      ↓
@kopf.on.create
      ↓
Controller
      ↓
┌──────────────────┐
│ myapp-frontend   │
│ Deployment       │
└──────────────────┘

┌──────────────────┐
│ myapp-backend    │
│ Deployment       │
└──────────────────┘

+ Services
```

이것이 `CREATE` 이벤트입니다.

---

# 5. UPDATE 실습 ⭐

이 부분이 CR의 진짜 핵심입니다.

현재 `myapp.yaml`:

```yaml
frontend:
  image: 192.168.56.11:30002/cr-demo/frontend:1.0
  replicas: 2
```

여기서:

```yaml
replicas: 2
```

를:

```yaml
replicas: 4
```

로 변경합니다.

그리고:

```bash
kubectl apply -f ~/cr-demo/myapp.yaml
```

확인:

```bash
kubectl get deployment -n cr-demo
```

예상:

```
NAME              READY
myapp-frontend    4/4
myapp-backend     2/2
```

즉:

```
사용자
 ↓
MyApp CR 수정
 ↓
Kubernetes API
 ↓
@kopf.on.update
 ↓
Controller
 ↓
frontend Deployment
2 → 4
```

입니다.

### 여기서 꼭 기억할 것

사용자가 직접:

```bash
kubectl scale deployment myapp-frontend --replicas=4
```

한 것이 아닙니다.

사용자는:

```yaml
MyApp
  frontend:
    replicas: 4
```

라는 **원하는 상태(desired state)**를 변경했습니다.

Controller가 실제 Deployment를 변경했습니다.

이것이 CR + Controller의 핵심입니다.

---

# 6. UPDATE에서 Image도 변경

이번에는:

```yaml
backend:
  image: 192.168.56.11:30002/cr-demo/backend:1.0
```

를 다른 태그로 바꿔봅니다.

예를 들어 Harbor에 `2.0` 이미지를 준비했다면:

```yaml
backend:
  image: 192.168.56.11:30002/cr-demo/backend:2.0
```

그리고:

```bash
kubectl apply -f ~/cr-demo/myapp.yaml
```

확인:

```bash
kubectl describe deployment myapp-backend -n cr-demo
```

Deployment의 image가 변경된 것을 확인할 수 있습니다.

---

# 7. DELETE 실습 ⭐

이제 CR을 삭제합니다.

```bash
kubectl delete -f ~/cr-demo/myapp.yaml
```

Controller 로그를 확인하면:

```
[DELETE] MyApp: myapp

[DELETE] deployment: myapp-frontend
[DELETE] deployment: myapp-backend
[DELETE] service: myapp-frontend
[DELETE] service: myapp-backend
```

확인:

```bash
kubectl get all -n cr-demo
```

MyApp 관련 리소스가 사라졌는지 확인합니다.

---

# 8. CREATE → UPDATE → DELETE를 한 번에 이해하기

이번 실습의 핵심은 다음 흐름입니다.

```
                  MyApp CR
                     │
          ┌──────────┼──────────┐
          │          │          │
        CREATE     UPDATE     DELETE
          │          │          │
          ▼          ▼          ▼
      생성 처리    변경 처리    삭제 처리
          │          │          │
          ▼          ▼          ▼
     Deployment   Deployment   Deployment
     Service      Service      Service
```

즉:

### CREATE

```
MyApp이 생겼다
→ 필요한 Kubernetes 리소스를 만들어라
```

### UPDATE

```
MyApp의 원하는 상태가 바뀌었다
→ 실제 Kubernetes 리소스를 새로운 상태로 맞춰라
```

### DELETE

```
MyApp이 없어졌다
→ MyApp이 관리하던 리소스를 정리하라
```

---

# 9. 그런데 현재 Controller에는 한 가지 중요한 한계가 있습니다

지금 코드에서는:

```python
@kopf.on.create(...)
@kopf.on.update(...)
@kopf.on.delete(...)
```

를 각각 사용했습니다.

**교육용으로는 아주 좋습니다.**

하지만 실제 Kubernetes Operator에서는 보통 이것보다 한 단계 더 나아가 **Reconciliation(조정)** 개념을 사용합니다.

예를 들어:

```
원하는 상태
   │
   │ MyApp CR
   ▼
┌──────────────┐
│  Controller  │
└──────┬───────┘
       │
       ▼
실제 상태
   │
   ├── Deployment replicas = 2
   ├── Service 존재
   └── Image = 1.0
```

Controller가 계속 비교합니다.

```
Desired State ≠ Actual State
        ↓
     조정 필요
        ↓
      수정
        ↓
Desired State = Actual State
```

이것이 Kubernetes Controller의 본질입니다.

---

## 다음 실습에서 특히 해볼 만한 것

여기까지 성공하면 **일부러 Controller가 없어도 Deployment를 직접 변경해보는 실험**을 해보세요.

예:

```bash
kubectl scale deployment myapp-frontend \
  --replicas=1 \
  -n cr-demo
```

그러면:

```
MyApp CR
frontend replicas = 4

실제 Deployment
replicas = 1
```

가 됩니다.

즉:

```
Desired State = 4
Actual State  = 1
```

입니다.

이것을 Controller가 다시:

```
1 → 4
```

로 되돌리는 **Reconciliation 실습**까지 진행하면, 지금까지의 CRD/CR 개념이 거의 완성됩니다.

그 다음에는 현재 `create/update/delete` 방식의 Controller를 **`@kopf.timer`가 아니라 Kubernetes Operator답게 "desired state ↔ actual state를 지속적으로 맞추는 reconcile 구조"로 개선**하는 것이 다음 단계입니다.

# Reconcile 구조로 개선

좋습니다. **이 단계가 CRD/CR을 제대로 이해하는 데 가장 중요한 단계**입니다.

현재 Controller는:

```
CREATE → 생성
UPDATE → 수정
DELETE → 삭제
```

라는 **이벤트 중심 방식**입니다.

이것을 Kubernetes다운 방식인 **Reconcile(조정) 구조**로 바꿔보겠습니다.

---

# 1. 먼저 Reconcile의 핵심 개념

Reconcile은 아주 간단하게 말하면:

> **"CR이 원하는 상태(Desired State)와 실제 Kubernetes 상태(Actual State)를 비교해서, 실제 상태를 원하는 상태로 계속 맞추는 것"**
> 

입니다.

예를 들어 CR이:

```yaml
spec:
  frontend:
    replicas: 3
```

이라고 되어 있다면:

```
Desired State
frontend replicas = 3

        ↓ 비교

Actual State
frontend Deployment replicas = 1
```

Controller가 발견합니다.

```
3 ≠ 1
```

그러면:

```
Deployment replicas
1 → 3
```

으로 수정합니다.

---

# 2. CREATE/UPDATE 방식의 문제

현재 구조는:

```python
@kopf.on.create(...)
def create_myapp(...):
    ...
```

```python
@kopf.on.update(...)
def update_myapp(...):
    ...
```

입니다.

이 방식은 **이벤트가 발생했을 때만** 동작합니다.

그런데 실제 Kubernetes에서는 이런 일이 발생할 수 있습니다.

```
MyApp CR
  replicas = 3

        ↓

Deployment
  replicas = 3
```

정상입니다.

그런데 누군가 실수로:

```bash
kubectl scale deployment myapp-frontend \
  --replicas=1 \
  -n cr-demo
```

를 실행하면:

```
CR
3 replicas

Deployment
1 replica
```

가 됩니다.

그런데 **UPDATE 이벤트가 발생한 것이 아니므로 기존 `@kopf.on.update`만으로는 자동 복구되지 않습니다.**

---

# 3. Reconcile 구조에서는 다릅니다

Controller가 주기적으로 또는 관련 리소스 변경을 계기로 상태를 확인합니다.

```
             MyApp CR
                │
                │ Desired = 3
                ▼
          ┌─────────────┐
          │ Controller  │
          └──────┬──────┘
                 │
                 ▼
        Deployment 확인
                 │
          Actual = 1
                 │
                 ▼
            3 ≠ 1
                 │
                 ▼
        Deployment 수정
                 │
                 ▼
            Actual = 3
```

이것이 **Reconciliation Loop**입니다.

---

# 4. 이번 실습에서 만들 구조

최종적으로 다음 구조를 만들겠습니다.

```
                MyApp CR
                   │
                   │ Desired State
                   ▼
             ┌────────────┐
             │ Reconcile  │
             └─────┬──────┘
                   │
           ┌───────┴────────┐
           ▼                ▼
      Frontend           Backend
      Deployment         Deployment
           │                │
           ▼                ▼
       Actual State     Actual State
           │                │
           └───────┬────────┘
                   ▼
              비교 / 조정
```

---

# 5. Kopf에서 Reconcile을 구현하는 방법

Kopf에서는 여러 방식으로 구현할 수 있습니다.

교육 목적에서는 우선 **`@kopf.timer`를 이용해 주기적으로 reconcile하는 방식**이 이해하기 쉽습니다.

예:

```python
@kopf.timer("example.com", "v1", "myapps", interval=10)
def reconcile(spec, name, namespace, **kwargs):
    ...
```

의미는:

> **모든 MyApp을 10초마다 확인하고 실제 상태를 원하는 상태에 맞춘다.**
> 

입니다.

다만 이것은 Kubernetes의 일반적인 Controller 설계에서 사용하는 **event-driven reconciliation**보다 단순한 교육용 접근입니다. 이번에는 개념 이해를 위해 사용하겠습니다.

---

# 6. 기존 Controller를 Reconcile 중심으로 변경

`controller.py`를 다음 구조로 바꿉니다.

핵심은:

```
reconcile()
    ↓
ensure_frontend()
    ↓
ensure_backend()
```

입니다.

```python
import kopf
from kubernetes import client

# ==================================================
# Deployment 상태를 Desired State에 맞춘다.
# ==================================================

def ensure_deployment(
    apps,
    name,
    namespace,
    image,
    replicas,
    container_port,
    app_label,
):

    labels = {
        "app": app_label
    }

    desired = client.V1Deployment(
        metadata=client.V1ObjectMeta(
            name=name,
            namespace=namespace,
        ),

        spec=client.V1DeploymentSpec(
            replicas=replicas,

            selector=client.V1LabelSelector(
                match_labels=labels
            ),

            template=client.V1PodTemplateSpec(
                metadata=client.V1ObjectMeta(
                    labels=labels
                ),

                spec=client.V1PodSpec(
                    containers=[
                        client.V1Container(
                            name=app_label,
                            image=image,
                            ports=[
                                client.V1ContainerPort(
                                    container_port=container_port
                                )
                            ],
                        )
                    ]
                )
            )
        )
    )

    try:

        actual = apps.read_namespaced_deployment(
            name=name,
            namespace=namespace,
        )

        # ------------------------------------------
        # Replica 비교
        # ------------------------------------------

        if actual.spec.replicas != replicas:

            print(
                f"[RECONCILE] {name}: "
                f"replicas "
                f"{actual.spec.replicas} -> {replicas}"
            )

            apps.patch_namespaced_deployment(
                name=name,
                namespace=namespace,
                body={
                    "spec": {
                        "replicas": replicas
                    }
                }
            )

        # ------------------------------------------
        # Image 비교
        # ------------------------------------------

        current_image = (
            actual.spec.template.spec.containers[0].image
        )

        if current_image != image:

            print(
                f"[RECONCILE] {name}: "
                f"image "
                f"{current_image} -> {image}"
            )

            apps.patch_namespaced_deployment(
                name=name,
                namespace=namespace,
                body={
                    "spec": {
                        "template": {
                            "spec": {
                                "containers": [
                                    {
                                        "name": app_label,
                                        "image": image,
                                    }
                                ]
                            }
                        }
                    }
                }
            )

        print(
            f"[OK] Deployment {name} "
            f"matches desired state"
        )

    except client.exceptions.ApiException as e:

        if e.status == 404:

            print(
                f"[RECONCILE] "
                f"Creating Deployment: {name}"
            )

            apps.create_namespaced_deployment(
                namespace=namespace,
                body=desired,
            )

        else:
            raise

# ==================================================
# Service가 존재하는지 확인한다.
# ==================================================

def ensure_service(
    core,
    name,
    namespace,
    port,
    target_port,
    selector,
):

    try:

        core.read_namespaced_service(
            name=name,
            namespace=namespace,
        )

        print(
            f"[OK] Service {name} exists"
        )

    except client.exceptions.ApiException as e:

        if e.status == 404:

            print(
                f"[RECONCILE] "
                f"Creating Service: {name}"
            )

            service = client.V1Service(
                metadata=client.V1ObjectMeta(
                    name=name,
                    namespace=namespace,
                ),

                spec=client.V1ServiceSpec(
                    selector=selector,

                    ports=[
                        client.V1ServicePort(
                            port=port,
                            target_port=target_port,
                        )
                    ],

                    type="ClusterIP",
                )
            )

            core.create_namespaced_service(
                namespace=namespace,
                body=service,
            )

        else:
            raise

# ==================================================
# 핵심: Reconcile Loop
# ==================================================

@kopf.timer(
    "example.com",
    "v1",
    "myapps",
    interval=10,
)
def reconcile(
    spec,
    name,
    namespace,
    **kwargs,
):

    print(
        f"[RECONCILE] MyApp: {name}"
    )

    apps = client.AppsV1Api()
    core = client.CoreV1Api()

    frontend = spec["frontend"]
    backend = spec["backend"]

    # ----------------------------------------------
    # Frontend
    # ----------------------------------------------

    frontend_name = f"{name}-frontend"

    ensure_deployment(
        apps=apps,
        name=frontend_name,
        namespace=namespace,
        image=frontend["image"],
        replicas=frontend["replicas"],
        container_port=80,
        app_label=frontend_name,
    )

    ensure_service(
        core=core,
        name=frontend_name,
        namespace=namespace,
        port=80,
        target_port=80,
        selector={
            "app": frontend_name
        },
    )

    # ----------------------------------------------
    # Backend
    # ----------------------------------------------

    backend_name = f"{name}-backend"

    ensure_deployment(
        apps=apps,
        name=backend_name,
        namespace=namespace,
        image=backend["image"],
        replicas=backend["replicas"],
        container_port=8000,
        app_label=backend_name,
    )

    ensure_service(
        core=core,
        name=backend_name,
        namespace=namespace,
        port=8000,
        target_port=8000,
        selector={
            "app": backend_name
        },
    )

    print(
        f"[RECONCILE] MyApp {name} completed"
    )
```

이 코드에서는 기존의:

```
@kopf.on.create
@kopf.on.update
```

를 제거하고 **Reconcile이라는 하나의 함수가 전체 상태를 관리**합니다.

---

# 7. Controller 실행

기존 Controller를 종료하고:

```bash
cd ~/cr-demo/controller

source venv/bin/activate

kopf run controller.py
```

로그에 약 10초마다:

```
[RECONCILE] MyApp: myapp
[OK] Deployment myapp-frontend matches desired state
[OK] Service myapp-frontend exists
[OK] Deployment myapp-backend matches desired state
[OK] Service myapp-backend exists
```

같은 메시지가 나옵니다.

---

# 8. 가장 중요한 실습 ⭐⭐⭐⭐⭐

이제 일부러 실제 상태를 깨뜨립니다.

먼저 CR을 확인합니다.

```bash
kubectl get myapp myapp -n cr-demo -o yaml
```

예:

```yaml
spec:
  frontend:
    replicas: 2
```

그러면 Deployment도:

```bash
kubectl get deployment -n cr-demo
```

```
myapp-frontend   2/2
```

입니다.

---

# 9. 실제 Deployment를 강제로 변경

이번에는 CR을 건드리지 않고 Deployment만 변경합니다.

```bash
kubectl scale deployment myapp-frontend \
  --replicas=1 \
  -n cr-demo
```

확인:

```bash
kubectl get deployment -n cr-demo
```

잠시 동안:

```
myapp-frontend   1/1
```

이 됩니다.

그런데 약 10초 후:

```
myapp-frontend   2/2
```

로 돌아옵니다.

Controller 로그:

```
[RECONCILE] myapp-frontend:
replicas 1 -> 2
```

---

# ⭐ 10. 이것이 CR의 진짜 의미입니다

지금 발생한 상황을 그림으로 보면:

```
MyApp CR
replicas = 2
     │
     │ Desired State
     ▼
 Controller
     │
     │ 비교
     ▼
Deployment
replicas = 1
```

Controller가:

```
2 ≠ 1
```

을 발견합니다.

그래서:

```
Deployment
1 → 2
```

로 수정합니다.

최종:

```
Desired State = 2
Actual State  = 2
```

가 됩니다.

---

# 11. Image도 같은 방식입니다

CR:

```yaml
backend:
  image: backend:1.0
```

Actual Deployment:

```
backend:2.0
```

이라면:

```
Desired
backend:1.0

Actual
backend:2.0

        ↓

Controller

        ↓

backend:1.0
```

으로 되돌립니다.

---

# 12. CREATE/UPDATE와 Reconcile 차이

이제 둘을 비교해보면 명확합니다.

### 이벤트 방식

```
CREATE
  ↓
처리

UPDATE
  ↓
처리

DELETE
  ↓
처리
```

즉:

> **"무슨 일이 발생했을 때 처리한다."**
> 

---

### Reconcile 방식

```
Desired State
      ↓
Actual State 확인
      ↓
차이가 있는가?
      │
   ┌──┴──┐
   │     │
  YES    NO
   │     │
   ▼     ▼
 수정    종료
   │
   └──────→ 다시 확인
```

즉:

> **"현재 상태가 원하는 상태와 같은지 계속 확인하고 다르면 고친다."**
> 

---

# 13. 그런데 DELETE는 어떻게 하나?

여기서 아주 중요한 문제가 하나 생깁니다.

현재 Reconcile은:

```
MyApp 존재
 ↓
Deployment 존재 여부 확인
 ↓
없으면 생성
```

입니다.

그런데 MyApp이 삭제되면 Reconcile 대상 자체가 사라집니다.

그래서 삭제 처리는 별도로 필요합니다.

교육용으로는:

```python
@kopf.on.delete(...)
```

를 유지해도 됩니다.

즉 최종 구조를:

```
CREATE
   ↓
Reconcile

UPDATE
   ↓
Reconcile

주기적 확인
   ↓
Reconcile

DELETE
   ↓
Cleanup
```

으로 만들 수 있습니다.

---

# 14. 실제 Operator에서는 더 발전합니다

Kubernetes의 전형적인 Controller는 단순 timer보다:

```
Watch Event
    ↓
Reconcile Request
    ↓
Reconcile
    ↓
Actual State 확인
    ↓
Desired State로 변경
```

방식을 사용합니다.

즉:

```
Deployment 변경
      ↓
Controller가 감지
      ↓
Reconcile
```

이 가능합니다.

그래서 다음 단계에서는 `@kopf.timer` 방식에서 더 나아가 **MyApp CR뿐 아니라 MyApp이 만든 Deployment/Service의 변경도 감지하여 자동으로 Reconcile하는 구조**를 만들어보는 것이 좋습니다.

---

# 15. 그리고 한 가지 더 중요한 개념

현재 코드의 핵심 함수는:

```python
ensure_deployment()
```

입니다.

이름 그대로:

> **"Deployment가 원하는 상태인지 보장한다."**
> 

입니다.

이것이 Operator 코드에서 자주 보는:

```
ensure
reconcile
desired
actual
```

이라는 용어의 의미입니다.

결국 Controller는 다음 일을 반복합니다.

```
           Desired
              │
              ▼
         ┌──────────┐
         │ Reconcile│
         └─────┬────┘
               │
               ▼
            Actual
               │
               ▼
          차이가 있는가?
           /          \
         YES           NO
          │             │
          ▼             ▼
       수정한다        끝
          │
          └──────→ 다시 확인
```

**이 구조를 이해하면 CRD → CR → Controller → Operator의 관계가 거의 완성됩니다.**

그리고 지금 하신 실습에서 가장 중요한 테스트는 반드시 **`kubectl scale`로 Deployment를 CR과 다르게 만든 후, Controller가 원래 CR의 값으로 자동 복구하는 것**입니다. 이것을 직접 확인하면 "CR은 단순히 여러 이미지를 묶는 YAML"이라는 생각에서 벗어나 **"CR이 선언한 Desired State를 Controller가 계속 보장한다"**는 개념으로 넘어가게 됩니다.

# Controller를 Pod로 실행

네. **오히려 실제 운영에서는 Controller/Operator를 수동으로 `kopf run controller.py` 하는 것이 아니라 Kubernetes Pod로 배포하는 방식이 일반적**입니다.

구조는 이렇게 됩니다.

```
MyApp CR
   │
   ▼
Kubernetes API
   │
   ▼
Controller Pod
   │
   ├── Deployment 생성/수정
   ├── Service 생성/수정
   └── 상태 Reconcile
```

즉 지금 수동으로 실행하던:

```bash
kopf run controller.py
```

를 **Docker 이미지로 만들고 Deployment로 실행**하면 됩니다.

### 1. Controller용 Dockerfile

`~/cr-demo/controller/Dockerfile`

```docker
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY controller.py .

CMD ["kopf", "run", "/app/controller.py", "--verbose"]
```

`requirements.txt`는 그대로:

```
kopf
kubernetes
```

입니다.

이미지 빌드:

```bash
cd ~/cr-demo/controller

docker build -t cr-demo-controller:1.0 .
```

Harbor 태그:

```bash
docker tag cr-demo-controller:1.0 \
  192.168.56.11:30002/cr-demo/controller:1.0
```

Push:

```bash
docker push \
  192.168.56.11:30002/cr-demo/controller:1.0
```

---

### 2. 중요한 문제: Controller Pod의 권한

Pod 안에서 Controller가 다음 작업을 해야 합니다.

```
MyApp 조회/watch

Deployment
  get
  list
  watch
  create
  patch
  update
  delete

Service
  get
  list
  watch
  create
  patch
  update
  delete
```

따라서 **ServiceAccount + RBAC**가 필요합니다.

`controller-rbac.yaml`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-controller
  namespace: cr-demo

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: myapp-controller
  namespace: cr-demo

rules:

- apiGroups:
  - example.com
  resources:
  - myapps
  - myapps/status
  - myapps/finalizers
  verbs:
  - get
  - list
  - watch
  - patch
  - update

- apiGroups:
  - apps
  resources:
  - deployments
  verbs:
  - get
  - list
  - watch
  - create
  - update
  - patch
  - delete

- apiGroups:
  - ""
  resources:
  - services
  - pods
  - events
  verbs:
  - get
  - list
  - watch
  - create
  - update
  - patch
  - delete

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: myapp-controller
  namespace: cr-demo

subjects:
- kind: ServiceAccount
  name: myapp-controller
  namespace: cr-demo

roleRef:
  kind: Role
  name: myapp-controller
  apiGroup: rbac.authorization.k8s.io
```

적용:

```bash
kubectl apply -f controller-rbac.yaml
```

---

### 3. Controller Deployment 생성

`controller-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: myapp-controller
  namespace: cr-demo

spec:
  replicas: 1

  selector:
    matchLabels:
      app: myapp-controller

  template:
    metadata:
      labels:
        app: myapp-controller

    spec:
      serviceAccountName: myapp-controller

      containers:
      - name: controller
        image: 192.168.56.11:30002/cr-demo/controller:1.0
        imagePullPolicy: Always
```

적용:

```bash
kubectl apply -f controller-deployment.yaml
```

확인:

```bash
kubectl get pods -n cr-demo
```

예:

```
NAME                                READY   STATUS
myapp-controller-6c7c9d8f75-x8abc   1/1     Running
```

---

### 4. 이제 수동 `kopf run`이 필요 없습니다

Controller 로그는:

```bash
kubectl logs -f deployment/myapp-controller -n cr-demo
```

로 확인합니다.

이제:

```bash
kubectl apply -f myapp.yaml
```

하면

```
MyApp CR
   ↓
Controller Pod
   ↓
myapp-frontend Deployment
myapp-backend Deployment
myapp-frontend Service
myapp-backend Service
```

가 자동으로 생성됩니다.

---

### 5. Reconcile 테스트도 그대로 가능합니다

예를 들어 CR은:

```yaml
frontend:
  replicas: 2
```

인데 Deployment를 강제로:

```bash
kubectl scale deployment myapp-frontend \
  --replicas=1 \
  -n cr-demo
```

로 바꿉니다.

Controller가 주기적으로 reconcile하도록 되어 있다면 잠시 후 다시:

```bash
kubectl get deploy myapp-frontend -n cr-demo
```

```
READY
2/2
```

로 돌아옵니다.

이제 중요한 차이가 생깁니다.

```
이전

터미널
  │
  └── kopf run controller.py
            ↓
         Controller

변경 후

Kubernetes
   │
   └── Controller Deployment
            ↓
       Controller Pod
            ↓
        MyApp 관리
```

즉 **Controller 자체도 Kubernetes가 관리하는 애플리케이션**이 됩니다.

Pod가 죽어도 Deployment가 다시 만들어주므로 훨씬 안정적입니다.

그리고 여기서 아주 재미있는 점이 하나 있습니다.

> **우리가 만든 Controller도 결국 Deployment로 실행됩니다.**
> 

즉 CRD/Operator가 Deployment를 없애는 것이 아닙니다.

```
Controller 자체
    → Deployment로 실행

Controller가 관리하는 애플리케이션
    → Deployment/Service 등으로 생성
```

이 관계를 이해하면 CRD가 훨씬 명확해집니다.

다음 단계로는 이 Controller Pod에 **OwnerReference를 추가해서 `MyApp CR` 삭제 시 frontend/backend 리소스가 Kubernetes Garbage Collector에 의해 자동 삭제되도록 만드는 실습**이 좋습니다. 이 단계부터 Operator다운 구조가 됩니다.

# RBAC 권한 오류 해결

네, 원인이 정확히 나왔습니다. **Controller 코드 문제가 아니라 RBAC 권한 문제입니다.**

에러의 핵심은 이것입니다.

```
User "system:serviceaccount:cr-demo:myapp-controller"
cannot list resource "myapps"
in API group "example.com"
at the cluster scope
```

즉 Controller Pod가 `MyApp`을 감시하려고 하는데 Kubernetes가:

> `myapp-controller` ServiceAccount에게 `myapps`를 볼 권한이 없다.
> 

라고 거부하고 있습니다.

특히 중요한 부분은:

```
at the cluster scope
```

입니다.

---

# 1. 왜 Role인데 cluster scope라고 나올까?

현재 우리가 만든 RBAC에는:

```yaml
kind: Role
metadata:
  namespace: cr-demo
```

가 있고:

```yaml
resources:
- myapps
verbs:
- get
- list
- watch
```

를 줬습니다.

그런데 Kopf가 `MyApp`을 watch하면서 **cluster-level list**를 먼저 시도하고 있습니다.

따라서 현재 `Role`만으로는 Kopf의 초기 discovery/watch 동작을 만족시키지 못합니다.

이번 실습에서는 `MyApp`이 `Namespaced` CR이므로 **Controller가 클러스터 전체의 MyApp을 감시할 수 있도록 ClusterRole + ClusterRoleBinding을 사용하는 것이 가장 간단합니다.**

---

# 2. 기존 RoleBinding 삭제

먼저 기존 것을 삭제합니다.

```bash
kubectl delete rolebinding myapp-controller -n cr-demo
kubectl delete role myapp-controller -n cr-demo
```

ServiceAccount는 그대로 둡니다.

확인:

```bash
kubectl get sa myapp-controller -n cr-demo
```

---

# 3. ClusterRole 생성

`controller-rbac.yaml`을 다음처럼 수정합니다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole

metadata:
  name: myapp-controller

rules:

# ----------------------------------------
# MyApp CR
# ----------------------------------------

- apiGroups:
  - example.com
  resources:
  - myapps
  - myapps/status
  - myapps/finalizers
  verbs:
  - get
  - list
  - watch
  - patch
  - update

# ----------------------------------------
# Deployment
# ----------------------------------------

- apiGroups:
  - apps
  resources:
  - deployments
  verbs:
  - get
  - list
  - watch
  - create
  - update
  - patch
  - delete

# ----------------------------------------
# Service
# ----------------------------------------

- apiGroups:
  - ""
  resources:
  - services
  verbs:
  - get
  - list
  - watch
  - create
  - update
  - patch
  - delete

# ----------------------------------------
# Pod
# ----------------------------------------

- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - get
  - list
  - watch

# ----------------------------------------
# Event
# ----------------------------------------

- apiGroups:
  - ""
  resources:
  - events
  verbs:
  - create
  - patch

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding

metadata:
  name: myapp-controller

subjects:
- kind: ServiceAccount
  name: myapp-controller
  namespace: cr-demo

roleRef:
  kind: ClusterRole
  name: myapp-controller
  apiGroup: rbac.authorization.k8s.io
```

여기서 **중요한 변화**는:

```
Role
 ↓
ClusterRole
```

그리고:

```
RoleBinding
 ↓
ClusterRoleBinding
```

입니다.

---

# 4. 적용

```bash
kubectl apply -f ~/cr-demo/controller-rbac.yaml
```

확인:

```bash
kubectl get clusterrole myapp-controller
kubectl get clusterrolebinding myapp-controller
```

---

# 5. 권한 확인

이제 실제 Controller ServiceAccount로 테스트합니다.

### MyApp

```bash
kubectl auth can-i list myapps \
  --as=system:serviceaccount:cr-demo:myapp-controller
```

반드시:

```
yes
```

가 나와야 합니다.

그리고:

```bash
kubectl auth can-i watch myapps \
  --as=system:serviceaccount:cr-demo:myapp-controller
```

```
yes
```

---

### Deployment

```bash
kubectl auth can-i create deployments \
  --as=system:serviceaccount:cr-demo:myapp-controller \
  -n cr-demo
```

```
yes
```

---

### Service

```bash
kubectl auth can-i create services \
  --as=system:serviceaccount:cr-demo:myapp-controller \
  -n cr-demo
```

```
yes
```

---

# 6. Controller Pod 재시작

RBAC 변경 후 Controller를 재시작합니다.

```bash
kubectl rollout restart deployment myapp-controller -n cr-demo
```

확인:

```bash
kubectl get pods -n cr-demo
```

예:

```
myapp-controller-xxxxx   1/1   Running
```

---

# 7. Controller 로그 확인

```bash
kubectl logs -f deployment/myapp-controller -n cr-demo
```

이제 더 이상:

```
403 Forbidden
```

이 나오면 안 됩니다.

대신:

```
[RECONCILE] MyApp: myapp
```

같은 로그가 나와야 합니다.

---

# 8. 기존 Deployment를 다시 깨뜨려 봅니다 ⭐

현재 상황이:

```
CR
frontend replicas = 2

Deployment
frontend replicas = 1
```

이라면:

```bash
kubectl get deploy myapp-frontend -n cr-demo
```

```
myapp-frontend   1/1
```

입니다.

Controller가 정상적으로 reconcile하면 잠시 후:

```
myapp-frontend   2/2
```

가 되어야 합니다.

로그에서는:

```
[RECONCILE] MyApp: myapp
[RECONCILE] myapp-frontend: replicas 1 -> 2
```

가 나오는지 확인하세요.

---

# 9. 이번 문제에서 꼭 이해해야 하는 것

이번 에러는 오히려 **아주 좋은 실습 경험**입니다.

구조를 보면:

```
                  Kubernetes API
                       │
                       │
                ┌──────▼──────┐
                │    RBAC     │
                └──────┬──────┘
                       │
             허용된 작업만 가능
                       │
                       ▼
              Controller Pod
                       │
                       │ list/watch
                       ▼
                   MyApp CR
```

Controller가 아무리 Python 코드를 잘 작성해도:

```
Kubernetes API
       │
       X
   RBAC Forbidden
       │
       ▼
Controller 동작 실패
```

가 됩니다.

즉 **Operator는 크게 3개의 요소가 같이 있어야 합니다.**

```
┌──────────────────────────────┐
│          Operator            │
│                              │
│  CRD + CR                    │
│       │                      │
│       ▼                      │
│  Controller                  │
│       │                      │
│       ▼                      │
│  RBAC                        │
│       │                      │
│       ▼                      │
│ Kubernetes API               │
└──────────────────────────────┘
```

이번에는 그중 **RBAC 때문에 Controller가 MyApp을 보지 못했던 것**입니다.

---

## 10. 그리고 한 가지 수정할 부분

앞서 제가 `@kopf.timer` 방식으로 Reconcile을 설명했는데, **지금 단계에서는 이 구조를 조금 더 제대로 개선하는 것이 좋습니다.**

현재:

```
@kopf.timer
    ↓
10초마다 reconcile
```

보다 다음 구조가 Operator에 훨씬 가깝습니다.

```
MyApp 변경
   │
   ▼
Kopf event handler
   │
   ▼
reconcile()
   │
   ├── Deployment 확인
   ├── Service 확인
   └── 필요한 경우 수정
```

그리고 **Deployment가 누군가에 의해 변경됐을 때도 Controller가 다시 reconcile**하도록 만들 수 있습니다.

최종 구조는:

```
                 MyApp CR
                    │
                    ▼
              ┌───────────┐
              │ Reconcile │
              └─────┬─────┘
                    │
        ┌───────────┴───────────┐
        ▼                       ▼
Frontend Deployment       Backend Deployment
        │                       │
        └───────────┬───────────┘
                    │
                 실제 상태
                    │
                    ▼
              Desired와 비교
                    │
             다르면 자동 수정
```

여기까지 구현하면 **"CR은 여러 이미지를 묶어놓은 YAML"이 아니라 "애플리케이션의 Desired State를 선언하고 Operator가 실제 상태를 계속 보장하는 Kubernetes 확장 모델"**이라는 개념을 제대로 체감할 수 있습니다.

# 

좋습니다. 이 단계에서는 단순히 이벤트마다 Deployment를 생성하는 방식이 아니라, **Desired State(`spec.components`)와 Actual State(실제 Deployment)를 비교해서 맞추는 Reconciliation Loop**로 변경하는 것이 핵심입니다.

아래 코드는 다음을 지원하는 `operator.py` 전체 코드입니다.

- `@kopf.on.create`
- `@kopf.on.update`
- `@kopf.on.delete`
- `@kopf.on.resume`
- `spec.components` 기반 Generic Application
- Deployment 생성
- Deployment 변경 시 update
- CR에서 제거된 component의 Deployment 삭제
- Operator 재시작 후 기존 CR 재조정
- Deployment의 `image`, `replicas`, `containerPort` 변경 감지
- CR의 `ownerReferences` 설정
- 실제 Deployment와 Desired State 비교
- 불필요한 Deployment 자동 삭제

## `operator.py`

```python
import kopf
from kubernetes import client
from kubernetes.client.rest import ApiException

GROUP = "example.com"
VERSION = "v1"
PLURAL = "myapps"

LABEL_APP = "app.kubernetes.io/name"
LABEL_INSTANCE = "app.kubernetes.io/instance"
LABEL_COMPONENT = "app.kubernetes.io/component"

def get_components(spec):
    return spec.get("components", [])

def get_component_map(spec):
    return {
        component["name"]: component
        for component in get_components(spec)
    }

def get_deployment_name(app_name, component_name):
    return f"{app_name}-{component_name}"

def build_deployment(
    app_name,
    component,
    namespace,
    owner_references=None,
):
    component_name = component["name"]
    image = component["image"]
    replicas = component.get("replicas", 1)
    container_port = component.get("containerPort", 80)

    deployment_name = get_deployment_name(
        app_name,
        component_name,
    )

    labels = {
        LABEL_APP: app_name,
        LABEL_INSTANCE: app_name,
        LABEL_COMPONENT: component_name,
    }

    metadata = client.V1ObjectMeta(
        name=deployment_name,
        namespace=namespace,
        labels=labels,
        owner_references=owner_references,
    )

    container = client.V1Container(
        name=component_name,
        image=image,
        ports=[
            client.V1ContainerPort(
                container_port=container_port,
            )
        ],
    )

    pod_template = client.V1PodTemplateSpec(
        metadata=client.V1ObjectMeta(
            labels=labels,
        ),
        spec=client.V1PodSpec(
            containers=[container],
        ),
    )

    deployment_spec = client.V1DeploymentSpec(
        replicas=replicas,
        selector=client.V1LabelSelector(
            match_labels={
                LABEL_APP: app_name,
                LABEL_COMPONENT: component_name,
            }
        ),
        template=pod_template,
    )

    return client.V1Deployment(
        metadata=metadata,
        spec=deployment_spec,
    )

def deployment_needs_update(
    deployment,
    component,
):
    desired_image = component["image"]
    desired_replicas = component.get("replicas", 1)
    desired_port = component.get("containerPort", 80)

    current_replicas = deployment.spec.replicas

    if current_replicas != desired_replicas:
        return True

    containers = deployment.spec.template.spec.containers

    if not containers:
        return True

    current_container = containers[0]

    if current_container.image != desired_image:
        return True

    current_ports = current_container.ports or []

    if not current_ports:
        return True

    current_port = current_ports[0].container_port

    if current_port != desired_port:
        return True

    return False

def create_or_update_deployment(
    apps,
    app_name,
    component,
    namespace,
    owner_references,
):
    component_name = component["name"]

    deployment_name = get_deployment_name(
        app_name,
        component_name,
    )

    desired = build_deployment(
        app_name=app_name,
        component=component,
        namespace=namespace,
        owner_references=owner_references,
    )

    try:
        current = apps.read_namespaced_deployment(
            name=deployment_name,
            namespace=namespace,
        )

        if deployment_needs_update(
            current,
            component,
        ):
            apps.replace_namespaced_deployment(
                name=deployment_name,
                namespace=namespace,
                body=desired,
            )

            return "updated"

        return "unchanged"

    except ApiException as e:
        if e.status != 404:
            raise

        apps.create_namespaced_deployment(
            namespace=namespace,
            body=desired,
        )

        return "created"

def delete_removed_deployments(
    apps,
    app_name,
    namespace,
    desired_components,
):
    desired_names = {
        get_deployment_name(
            app_name,
            component_name,
        )
        for component_name in desired_components
    }

    selector = f"{LABEL_APP}={app_name}"

    deployments = apps.list_namespaced_deployment(
        namespace=namespace,
        label_selector=selector,
    )

    deleted = []

    for deployment in deployments.items:
        deployment_name = deployment.metadata.name

        if deployment_name not in desired_names:
            apps.delete_namespaced_deployment(
                name=deployment_name,
                namespace=namespace,
                body=client.V1DeleteOptions(),
            )

            deleted.append(deployment_name)

    return deleted

def reconcile(
    spec,
    name,
    namespace,
    body,
):
    apps = client.AppsV1Api()

    components = get_components(spec)

    owner_references = [
        client.V1OwnerReference(
            api_version=f"{GROUP}/{VERSION}",
            kind="MyApp",
            name=name,
            uid=body["metadata"]["uid"],
            controller=True,
            block_owner_deletion=True,
        )
    ]

    result = {
        "created": [],
        "updated": [],
        "unchanged": [],
        "deleted": [],
    }

    component_map = get_component_map(spec)

    for component_name, component in component_map.items():
        action = create_or_update_deployment(
            apps=apps,
            app_name=name,
            component=component,
            namespace=namespace,
            owner_references=owner_references,
        )

        result[action].append(component_name)

    deleted = delete_removed_deployments(
        apps=apps,
        app_name=name,
        namespace=namespace,
        desired_components=component_map.keys(),
    )

    result["deleted"].extend(deleted)

    return result

@kopf.on.create(GROUP, VERSION, PLURAL)
def on_create(spec, name, namespace, body, **kwargs):
    return reconcile(
        spec=spec,
        name=name,
        namespace=namespace,
        body=body,
    )

@kopf.on.update(GROUP, VERSION, PLURAL)
def on_update(spec, name, namespace, body, **kwargs):
    return reconcile(
        spec=spec,
        name=name,
        namespace=namespace,
        body=body,
    )

@kopf.on.resume(GROUP, VERSION, PLURAL)
def on_resume(spec, name, namespace, body, **kwargs):
    return reconcile(
        spec=spec,
        name=name,
        namespace=namespace,
        body=body,
    )

@kopf.on.delete(GROUP, VERSION, PLURAL)
def on_delete(name, namespace, **kwargs):
    apps = client.AppsV1Api()

    selector = f"{LABEL_APP}={name}"

    deployments = apps.list_namespaced_deployment(
        namespace=namespace,
        label_selector=selector,
    )

    deleted = []

    for deployment in deployments.items:
        deployment_name = deployment.metadata.name

        apps.delete_namespaced_deployment(
            name=deployment_name,
            namespace=namespace,
            body=client.V1DeleteOptions(),
        )

        deleted.append(deployment_name)

    return {
        "deleted": deleted,
    }
```

## 1. 기존 구조와 가장 큰 차이

기존에는:

```
CR 생성
  ↓
create_fn()
  ↓
Deployment 생성
```

이었습니다.

개선된 구조는:

```
             MyApp CR
                │
                ▼
           reconcile()
                │
       ┌────────┴────────┐
       │                 │
 Desired State       Actual State
 spec.components     Deployments
       │                 │
       └────────┬────────┘
                ▼
             비교
                │
       ┌────────┼────────┐
       ▼        ▼        ▼
     Create   Update    Delete
```

즉 **이벤트 핸들러 자체가 핵심 로직이 아니라 `reconcile()`가 핵심**입니다.

---

## 2. Create

다음 CR을 생성합니다.

```yaml
apiVersion: example.com/v1
kind: MyApp
metadata:
  name: myapp
  namespace: default
spec:
  components:
    - name: frontend
      image: nginx:1.27
      replicas: 2
      containerPort: 80

    - name: backend
      image: python:3.12
      replicas: 1
      containerPort: 8000
```

Operator는:

```
myapp-frontend
myapp-backend
```

Deployment를 생성합니다.

`reconcile()` 결과는 개념적으로:

```
created:
  frontend
  backend

updated:
unchanged:
deleted:
```

가 됩니다.

---

## 3. Update

예를 들어 CR을:

```yaml
spec:
  components:
    - name: frontend
      image: nginx:1.28
      replicas: 3
      containerPort: 80

    - name: backend
      image: python:3.12
      replicas: 1
      containerPort: 8000
```

로 변경하면:

```
myapp-frontend
```

의

```
image
nginx:1.27 → nginx:1.28

replicas
2 → 3
```

를 감지해서 Deployment를 업데이트합니다.

반면 backend는 변경사항이 없으므로:

```
backend → unchanged
```

가 됩니다.

---

## 4. Component 삭제

기존 CR이:

```yaml
spec:
  components:
    - name: frontend
    - name: backend
    - name: worker
```

였다가:

```yaml
spec:
  components:
    - name: frontend
    - name: backend
```

로 변경되면 실제 클러스터에는:

```
myapp-frontend
myapp-backend
myapp-worker
```

가 존재하지만 Desired State에는 `worker`가 없습니다.

따라서:

```
Desired:
frontend
backend

Actual:
frontend
backend
worker

             ↓

삭제:
worker
```

가 됩니다.

이 부분이 기존 단순 `create` 방식과 **Reconciliation 방식의 중요한 차이**입니다.

---

## 5. `@kopf.on.resume`

```python
@kopf.on.resume(GROUP, VERSION, PLURAL)
def on_resume(...):
    return reconcile(...)
```

는 Operator가 재시작됐을 때 중요합니다.

예를 들어:

```
09:00  MyApp 생성
      ↓
      Deployment 생성

09:10  Operator Pod 장애
      ↓
      Operator 재시작

09:11  @kopf.on.resume
      ↓
      reconcile()
      ↓
      현재 Deployment 상태 확인
```

따라서 Operator가 재시작되어도 현재 상태를 다시 확인할 수 있습니다.

---

## 6. OwnerReference

이 코드에서는 Deployment에:

```python
owner_references = [
    client.V1OwnerReference(
        api_version=f"{GROUP}/{VERSION}",
        kind="MyApp",
        name=name,
        uid=body["metadata"]["uid"],
        controller=True,
        block_owner_deletion=True,
    )
]
```

를 지정합니다.

따라서 Kubernetes 입장에서:

```
MyApp
  │
  ├── myapp-frontend Deployment
  │
  ├── myapp-backend Deployment
  │
  └── myapp-worker Deployment
```

라는 **소유 관계**가 만들어집니다.

이것은 Generic Operator를 만들 때 상당히 중요한 부분입니다.

다만 위 코드에서는 `@kopf.on.delete`에서도 명시적으로 Deployment를 삭제합니다. 즉 **Operator가 삭제 이벤트를 직접 처리**하면서 동시에 OwnerReference를 사용하는 구조입니다.

---

## 7. 현재 구조에서 한 가지 더 개선할 부분

현재 코드는 Generic Application Operator의 기본 구조로는 적절하지만, 실제 운영 수준으로 발전시키려면 다음 단계가 좋습니다.

```
MyApp
 │
 ├── components
 │      │
 │      ├── Deployment
 │      ├── Service
 │      ├── ConfigMap
 │      └── Secret
 │
 └── status
        ├── phase
        ├── componentStatuses
        ├── readyReplicas
        └── conditions
```

특히 현재 코드는 **Deployment가 존재하는지만 확인**합니다.

다음 단계에서는 Deployment의:

```
desired replicas
        vs
available replicas
```

를 비교해서 CR의 `status`에:

```yaml
status:
  phase: Running
  components:
    frontend:
      desiredReplicas: 3
      readyReplicas: 3
      status: Ready
    backend:
      desiredReplicas: 1
      readyReplicas: 0
      status: Pending
```

처럼 기록하도록 만들면, 지금 구축하고 있는 **Kubernetes Operator → CRD → Deployment → 상태 관리** 구조를 훨씬 제대로 경험할 수 있습니다.
