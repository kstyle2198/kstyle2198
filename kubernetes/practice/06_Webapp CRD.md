# [실습] Webapp CRD

# 기본 과정

좋습니다. 지금까지의 실습을 **복습/실습자료로 바로 사용할 수 있도록 핵심 명령과 구조 중심으로 간명하게** 정리하면 아래와 같습니다.

# WebApp CRD 실습 전체 과정

## 1. 실습 환경

```
Kubernetes Cluster

control-plane
    └─ kubectl, CRD, RBAC, 배포 관리

worker-1
    ├─ Python Controller
    ├─ nginx Frontend
    └─ FastAPI Backend
```

**원칙**

- 코드 작성 → `worker-1`
- Docker build/push → `worker-1`
- Kubernetes 리소스 적용/확인 → `control-plane`

---

# 2. 최종 구조

```
WebApp CR
   │
   ▼
Python Controller (Kopf)
   │
   ├── Frontend Deployment
   │       └── nginx
   │
   ├── Frontend Service
   │
   ├── Backend Deployment
   │       └── FastAPI
   │
   └── Backend Service
```

실제 리소스:

```
my-webapp
├── my-webapp-frontend Deployment
├── my-webapp-frontend Service
├── my-webapp-backend Deployment
└── my-webapp-backend Service
```

---

# 3. Backend 작성

### worker-1

```
~/webapp-crd/backend/
├── Dockerfile
├── requirements.txt
└── main.py
```

FastAPI:

```
Port: 8000
```

이미지:

```
192.168.56.11:30002/webapp/backend:1.0
```

빌드/Push:

```bash
docker build -t 192.168.56.11:30002/webapp/backend:1.0 .
docker push 192.168.56.11:30002/webapp/backend:1.0
```

---

# 4. Frontend 작성

### worker-1

```
~/webapp-crd/frontend/
├── Dockerfile
├── default.conf
└── index.html
```

nginx:

```
Port: 80
```

Backend 연결:

```
proxy_pass http://my-webapp-backend:8000;
```

이미지:

```
192.168.56.11:30002/webapp/frontend:1.1
```

---

# 5. Frontend/Backend 직접 배포 테스트

### control-plane

먼저 CRD 없이 일반 Kubernetes 리소스로 테스트했습니다.

확인:

```bash
kubectl get deployments
kubectl get pods
kubectl get svc
```

Frontend에서 발생했던 문제:

```
host not found in upstream
```

원인은 nginx가 잘못된 Service 이름을 사용했기 때문입니다.

최종:

```
nginx
  ↓
my-webapp-backend:8000
  ↓
FastAPI
```

---

# 6. WebApp CRD 생성

### worker-1에서 YAML 작성

CRD 핵심:

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition

metadata:
  name: webapps.webapp.example.com

spec:
  group: webapp.example.com

  names:
    plural: webapps
    singular: webapp
    kind: WebApp

  scope: Namespaced

  versions:
    - name: v1
      served: true
      storage: true
```

### control-plane에서 적용

```bash
kubectl apply -f webapp-crd.yaml
```

확인:

```bash
kubectl get crd
```

결과:

```
webapps.webapp.example.com
```

---

# 7. WebApp Custom Resource 생성

### worker-1에서 작성

```yaml
apiVersion: webapp.example.com/v1
kind: WebApp

metadata:
  name: my-webapp

spec:
  frontend:
    image: 192.168.56.11:30002/webapp/frontend:1.1
    replicas: 1

  backend:
    image: 192.168.56.11:30002/webapp/backend:1.0
    replicas: 1
```

### control-plane

```bash
kubectl apply -f webapp.yaml
```

확인:

```bash
kubectl get webapp
```

---

# 8. Python Controller 작성

### worker-1

```
~/webapp-crd/controller/
├── Dockerfile
├── requirements.txt
└── controller.py
```

사용 기술:

```
Python
 ├─ Kopf
 └─ Kubernetes Python Client
```

Controller 핵심 역할:

```
WebApp
  ↓
reconcile_webapp()
  ↓
Deployment/Service 생성 또는 수정
```

---

# 9. Reconcile 구현

핵심 함수:

```python
reconcile_webapp()
reconcile_deployment()
reconcile_service()
```

Deployment:

```
없음
 ↓
CREATE

존재
 ↓
replicas 비교
 ↓
image 비교
 ↓
필요할 때 PATCH
```

즉:

```
Desired State
      ≠
Actual State
      ↓
   PATCH
```

---

# 10. OwnerReference 구현

WebApp을 부모로 설정합니다.

```
WebApp
 │
 ├── Frontend Deployment
 ├── Frontend Service
 ├── Backend Deployment
 └── Backend Service
```

WebApp 삭제 시 Kubernetes Garbage Collector가 하위 리소스를 정리할 수 있습니다.

주의:

```python
body["apiVersion"]
```

사용.

잘못된 방식:

```python
meta["apiVersion"]
```

---

# 11. Controller 이벤트

Controller는 다음 이벤트에서 reconcile합니다.

```python
@kopf.on.create(...)
@kopf.on.update(...)
@kopf.on.resume(...)
```

모두:

```
        ↓
reconcile_webapp()
```

으로 연결됩니다.

---

# 12. RBAC 구성

Controller ServiceAccount:

```
webapp-system/webapp-controller
```

필요한 주요 권한:

### WebApp

```yaml
apiGroups:
  - webapp.example.com
resources:
  - webapps
```

### Deployment

```yaml
apiGroups:
  - apps
resources:
  - deployments
```

### Service

```yaml
apiGroups:
  - ""
resources:
  - services
```

### CRD

```yaml
apiGroups:
  - apiextensions.k8s.io
resources:
  - customresourcedefinitions
```

---

# 13. RBAC에서 발생했던 오류

잘못된 API Group:

```
apps.example.com
```

정상:

```
webapp.example.com
```

구분:

```
WebApp CRD       → webapp.example.com
Deployment       → apps
Service          → core API
CRD              → apiextensions.k8s.io
```

권한 확인:

```bash
kubectl auth can-i \
  list webapps.webapp.example.com \
  --as=system:serviceaccount:webapp-system:webapp-controller
```

```
yes
```

---

# 14. Controller Docker 이미지

### worker-1

예:

```bash
docker build \
  -t 192.168.56.11:30002/webapp/controller:2.4 .
```

```bash
docker push \
  192.168.56.11:30002/webapp/controller:2.4
```

---

# 15. Controller 배포

### control-plane

```bash
kubectl set image deployment/webapp-controller \
  controller=192.168.56.11:30002/webapp/controller:2.4 \
  -n webapp-system
```

확인:

```bash
kubectl rollout status \
  deployment/webapp-controller \
  -n webapp-system
```

```bash
kubectl get pods \
  -n webapp-system \
  -o wide
```

---

# 16. 최종 Reconcile 테스트

WebApp:

```yaml
frontend:
  replicas: 3
```

확인:

```bash
kubectl get webapp my-webapp \
  -o jsonpath='{.spec.frontend.replicas}{"\n"}'
```

결과:

```
3
```

Deployment가:

```
1/1
```

이라면 Controller가:

```
Desired = 3
Actual  = 1
       ↓
    Reconcile
       ↓
Actual = 3
```

으로 변경해야 합니다.

확인:

```bash
kubectl get deployment my-webapp-frontend -w
```

최종:

```
3/3
```

---

# 17. Controller 로그 확인

```bash
kubectl logs \
  -n webapp-system \
  deployment/webapp-controller \
  --tail=100
```

정상적인 로그 흐름:

```
[EVENT] UPDATE
        ↓
[RECONCILE] WebApp default/my-webapp
        ↓
[UPDATE] my-webapp-frontend replicas: 1 -> 3
        ↓
[RECONCILED] Deployment
```

---

# 18. 지금까지 배운 핵심 개념

```
① CRD
   → 새로운 Kubernetes API 생성

② CR
   → WebApp이라는 사용자 리소스 생성

③ Controller
   → CR을 감시

④ Reconcile
   → Desired State와 Actual State를 일치시킴

⑤ Deployment
   → Frontend/Backend Pod 관리

⑥ Service
   → Pod 접근 및 DNS 제공

⑦ OwnerReference
   → 부모-자식 리소스 관계

⑧ RBAC
   → Controller의 Kubernetes API 접근 권한

⑨ Kopf
   → Python으로 Kubernetes Controller 구현
```

---

# 19. 전체 실습 흐름

가장 간단하게 요약하면:

```
[1] FastAPI 작성
        ↓
[2] nginx 작성
        ↓
[3] Docker 이미지 생성
        ↓
[4] Registry Push
        ↓
[5] 일반 Deployment/Service로 동작 확인
        ↓
[6] WebApp CRD 작성
        ↓
[7] WebApp CR 생성
        ↓
[8] Python Controller 작성
        ↓
[9] RBAC 구성
        ↓
[10] Controller 배포
        ↓
[11] reconcile 구현
        ↓
[12] replicas/image 변경 테스트
        ↓
[13] Desired State = Actual State 확인
```

---

## 현재 실습의 핵심 결과

결국 사용자는 앞으로 다음 YAML 하나만 관리하면 됩니다.

```yaml
apiVersion: webapp.example.com/v1
kind: WebApp
metadata:
  name: my-webapp

spec:
  frontend:
    image: ...
    replicas: 3

  backend:
    image: ...
    replicas: 1
```

그리고 Python Controller가 자동으로:

```
WebApp
  ↓
Frontend Deployment + Service
Backend Deployment + Service
```

를 생성하고, **replicas/image 등의 변경도 자동으로 Kubernetes 실제 상태에 반영**합니다.

이것이 이번 실습에서 구현한 **기본적인 Kubernetes Operator/Controller 패턴**입니다.

# Testing 및 구조적 문제

맞습니다. **그렇게 매번 control-plane과 worker-1에서 각각 수동으로 수정하는 방식은 비효율적입니다.** 그리고 현재 실습 구조에서 혼동이 생긴 이유는 **Docker 이미지 관리와 Kubernetes YAML 관리가 분리되어 있기 때문**입니다.

다만 중요한 점이 하나 있습니다.

> **`my-webapp.yaml`은 control-plane에만 있으면 됩니다.**
> 
> 
> **Docker 이미지는 worker-1에 수동으로 빌드해서 넣는 방식으로 운영할 필요가 없습니다.**
> 

현재 구조에서는 **Private Registry를 중심으로 관리하는 방식**으로 바꾸는 것이 좋습니다.

---

# 1. 현재 구조의 문제

현재는 대략 이렇게 되어 있습니다.

```
control-plane
└── ~/webapp-crd/
    └── my-webapp.yaml
          │
          │ kubectl apply
          ▼
     Kubernetes API
          │
          ▼
     worker-1
     └── Docker image
         webapp/frontend:1.1
```

그리고 지금은 worker-1에서:

```bash
docker build ...
docker push ...
```

를 하고 있습니다.

그러다 보니 사용자가 느끼기에:

```
worker-1
  └── 이미지 변경

control-plane
  └── YAML 변경
```

을 각각 해야 하는 것처럼 보입니다.

**맞습니다. 이 방식은 장기적으로 좋지 않습니다.**

---

# 2. Kubernetes에서는 Node별로 이미지를 관리하지 않습니다

여기가 핵심입니다.

Kubernetes에서는 다음과 같은 개념으로 생각하는 것이 좋습니다.

```
              Private Registry
                    │
       ┌────────────┼────────────┐
       │            │            │
       ▼            ▼            ▼
   control-plane  worker-1    worker-2
                    │
                    ▼
                  Pod
```

예를 들어:

```
192.168.56.11:30002/webapp/frontend:1.1
```

이라는 이미지가 Registry에 존재하면,

Kubernetes는 Pod가 실행되는 노드에서 Registry로부터 이미지를 가져옵니다.

따라서 **worker-1에 미리 Docker image를 수동으로 만들어 놓을 필요가 없습니다.**

---

# 3. 가장 권장하는 구조

현재 환경에서는 다음 구조가 가장 깔끔합니다.

```
             개발 작업
                │
                ▼
       ~/webapp-crd/frontend
                │
                │ docker build
                ▼
        Private Registry
     192.168.56.11:30002
                │
        ┌───────┴───────┐
        │               │
        ▼               ▼
 frontend:1.1       backend:1.0
        │
        ▼
     Kubernetes
        │
        ▼
     worker-1
        │
        ▼
 Frontend Pod
```

그리고 Kubernetes 설정은:

```
control-plane
└── my-webapp.yaml
```

하나만 관리합니다.

---

# 4. 그런데 Docker build는 어디에서 해야 하나?

여기에는 여러 방법이 있습니다.

현재 실습에서는 가장 간단하게:

> **control-plane에서 소스 코드 → Docker build → Registry push**
> 

를 추천합니다.

즉 앞으로는 worker-1에서 Docker 이미지를 만들지 않습니다.

예를 들어 control-plane에서:

```bash
cd ~/webapp-crd/frontend

docker build \
  -t 192.168.56.11:30002/webapp/frontend:1.1 \
  .

docker push \
  192.168.56.11:30002/webapp/frontend:1.1
```

하면 됩니다.

그러면:

```
control-plane
      │
      │ docker push
      ▼
192.168.56.11:30002
      │
      │ image pull
      ▼
worker-1
```

입니다.

---

# 5. 사실 `tag`를 매번 바꾸는 것도 필수는 아닙니다

현재 실습에서:

```
frontend:1.0
frontend:1.1
frontend:1.2
frontend:1.3
...
```

식으로 계속 만드는 것은 **버전 관리 측면에서는 좋은 방식**이지만, 실습 단계에서는 조금 번거로울 수 있습니다.

두 가지 방법이 있습니다.

### 방법 A — 버전 tag

```
frontend:1.0
frontend:1.1
frontend:1.2
```

추천도: ★★★★★

장점:

- 어떤 이미지인지 명확
- rollback 쉬움
- 운영환경에서 권장

---

### 방법 B — `latest`

```
frontend:latest
```

개발 실습에서는 편합니다.

하지만 Kubernetes에서:

```yaml
image: .../frontend:latest
```

를 사용하면 **이미지가 변경되어도 Pod가 자동으로 새 이미지를 가져온다는 보장이 없습니다.**

그래서 실습에서도 저는 **버전 tag를 유지하는 것을 추천**합니다.

---

# 6. 더 좋은 방법: `imagePullPolicy`

예를 들어:

```yaml
image: 192.168.56.11:30002/webapp/frontend:latest
imagePullPolicy: Always
```

로 하면 Pod가 시작될 때 Registry에서 최신 이미지를 확인합니다.

하지만 이것도:

```
코드 수정
   ↓
docker build
   ↓
docker push
   ↓
Pod 재시작
```

과정이 필요합니다.

따라서 현재 단계에서는 **버전 tag 방식이 더 이해하기 쉽습니다.**

---

# 7. `my-webapp.yaml`도 매번 직접 수정하지 않는 방법

여기서 한 단계 더 나아갈 수 있습니다.

현재:

```yaml
frontend:
  image: 192.168.56.11:30002/webapp/frontend:1.1
```

을 매번 수정하는 대신 CI/CD를 사용할 수 있습니다.

예:

```
Git
 │
 │ push
 ▼
CI/CD
 │
 ├── docker build
 │
 ├── docker push
 │
 └── Kubernetes deploy
```

최종적으로는:

```
개발자
  │
  │ git push
  ▼
GitLab
  │
  ▼
GitLab CI
  │
  ├── Frontend image build
  ├── Backend image build
  ├── Registry push
  └── kubectl apply
          │
          ▼
      Kubernetes
```

가 됩니다.

이것이 실제 운영환경에서 사용하는 일반적인 형태입니다.

---

# 8. 그런데 지금 CRD 실습에서는 여기까지 갈 필요는 없습니다

현재 목표가 **Kubernetes CRD + Python Controller 학습**이므로 저는 다음 정도를 추천합니다.

```
control-plane
~/webapp-crd/
│
├── my-webapp.yaml
│
├── frontend/
│   ├── Dockerfile
│   ├── index.html
│   └── nginx.conf
│
├── backend/
│   ├── Dockerfile
│   └── ...
│
└── controller/
    ├── controller.py
    └── Dockerfile
```

그리고 **모든 작업을 control-plane에서 수행**합니다.

---

# 9. 앞으로의 작업 흐름

### Frontend 수정

```bash
control-plane$ vi ~/webapp-crd/frontend/index.html
```

### 이미지 빌드

```bash
control-plane$ cd ~/webapp-crd/frontend

control-plane$ docker build \
  -t 192.168.56.11:30002/webapp/frontend:1.1 .
```

### Registry Push

```bash
control-plane$ docker push \
  192.168.56.11:30002/webapp/frontend:1.1
```

### CR 변경

```yaml
frontend:
  image: 192.168.56.11:30002/webapp/frontend:1.1
```

### 적용

```bash
control-plane$ kubectl apply -f my-webapp.yaml
```

### Controller가 자동 처리

```
WebApp CR
   ↓
Python Controller
   ↓
Deployment 변경
   ↓
worker-1
   ↓
Registry에서 frontend:1.1 pull
   ↓
새 Frontend Pod
```

**worker-1에서는 아무것도 수정하지 않습니다.**

---

# 10. 그리고 한 가지 중요한 구조 개선

현재 Docker Registry 주소가:

```
192.168.56.11:30002
```

입니다.

이것은 worker-1의 IP와 Registry NodePort를 합친 주소로 보입니다.

즉 Registry 자체도 Kubernetes NodePort를 통해 접근하고 있을 가능성이 높습니다.

이 구조에서는 worker-1이 Registry를 제공하면서 동시에 그 Registry에서 이미지를 pull하게 됩니다.

실습으로는 가능하지만, 조금 더 깔끔하게 하려면:

```
control-plane
       │
       │
       ▼
Private Registry
192.168.56.11:30002
       │
       ├──────────► worker-1
       │
       └──────────► worker-2
```

처럼 **Registry를 Kubernetes 외부의 독립적인 서비스로 보는 것**이 좋습니다.

---

# 결론

질문하신 부분에 대한 답은:

> **아니요. control-plane의 YAML과 worker-1의 Docker 이미지를 각각 수동 관리할 필요가 없습니다.**
> 

현재 실습을 다음 방식으로 바꾸는 것이 좋습니다.

```
                 control-plane
                      │
          ┌───────────┼───────────┐
          │           │           │
       source      Docker      kubectl
          │         build         │
          │           │           │
          │           ▼           ▼
          │       Registry     WebApp CR
          │           │           │
          │           │           ▼
          │           │       Controller
          │           │           │
          │           └──────► worker-1
          │                       │
          │                       ▼
          │                 Frontend Pod
          │
          └── 모든 소스/설정 관리
```

즉 **control-plane을 실습용 작업 노드로 사용하고, Registry를 이미지의 단일 저장소로 사용**하면 됩니다.

그리고 현재 프로젝트에서는 다음 단계로 **`frontend:1.1`을 control-plane에서 Registry에 push하고, `my-webapp.yaml`만 `1.1`로 변경한 다음 worker-1에서 이미지가 자동 pull되는 것**까지 확인해 보는 것이 좋습니다.

그렇게 한 번 성공시키면 이후에는 **worker-1에서 Docker build를 할 필요가 없어집니다.**

# Kopf(Kubernetes Operator Pythonic Framework)

**Kopf**는 Kubernetes에서 **Python으로 Operator/Controller를 쉽게 개발할 수 있게 해주는 프레임워크**입니다.

지금까지 실습하신 **`WebApp CRD + Python Controller`**와 아주 밀접한 기술입니다.

### 1. 한마디로

> **Kopf = Python으로 Kubernetes Operator를 만드는 프레임워크**
> 

일반적인 Kubernetes Controller를 직접 만들려면 Kubernetes API를 감시하고 이벤트를 처리하는 코드를 상당히 많이 작성해야 합니다.

Kopf를 사용하면 이런 부분을 상당히 단순화할 수 있습니다.

```
Kubernetes
    │
    │ WebApp CR 생성
    ▼
┌─────────────────┐
│ WebApp CRD      │
│ kind: WebApp    │
└────────┬────────┘
         │
         │ 이벤트 감지
         ▼
┌─────────────────┐
│ Kopf Controller │
│   Python        │
└────────┬────────┘
         │
         ├── Deployment 생성
         ├── Service 생성
         └── ConfigMap 생성
```

---

## 2. 기존에 작성하신 Controller와 비교

앞서 실습하신 방식은 대략 이런 구조였습니다.

```python
from kubernetes import client, config, watch

config.load_incluster_config()

while True:
    for event in watch.Watch().stream(...):
        if event["type"] == "ADDED":
            # WebApp 생성
            ...
```

즉, 직접 다음과 같은 작업을 구현해야 합니다.

- Kubernetes API 연결
- CRD watch
- 이벤트 처리
- ADD / UPDATE / DELETE 구분
- Deployment 생성
- Service 생성
- 오류 처리
- 재시도
- 상태 관리

Kopf를 사용하면 이런 구조를 훨씬 간단하게 만들 수 있습니다.

```python
import kopf

@kopf.on.create("webapps")
def create_fn(spec, name, **kwargs):

    print(f"WebApp 생성: {name}")

    # Deployment 생성
    # Service 생성

    return {"message": "WebApp created"}
```

즉,

```
직접 Controller 구현
        ↓
Kubernetes Python Client
        +
Watch
        +
Event 처리
        +
Reconciliation
        +
Error handling
        +
Retry
        ↓
상당히 많은 코드
```

대신

```
Kopf
  ↓
@kopf.on.create()
@kopf.on.update()
@kopf.on.delete()
@kopf.timer()
...
  ↓
Python 함수 작성
```

형태로 만들 수 있습니다.

---

# 3. Kopf의 핵심 개념

가장 중요한 것은 **Handler**입니다.

예를 들어:

```python
@kopf.on.create("webapps")
def create_webapp(spec, name, **kwargs):
    print(f"{name} 생성")
```

이것은 다음 의미입니다.

> `webapps` 리소스가 생성되면 `create_webapp()` 함수를 실행하라.
> 

### 생성

```python
@kopf.on.create("webapps")
def create_webapp(spec, name, **kwargs):
    ...
```

### 수정

```python
@kopf.on.update("webapps")
def update_webapp(spec, name, **kwargs):
    ...
```

### 삭제

```python
@kopf.on.delete("webapps")
def delete_webapp(spec, name, **kwargs):
    ...
```

### 주기적인 작업

```python
@kopf.timer("webapps", interval=30)
def check_webapp(spec, name, **kwargs):
    ...
```

이런 식으로 Kubernetes 리소스의 lifecycle에 Python 함수를 연결할 수 있습니다.

---

# 4. 지금 하신 WebApp CRD에 적용하면

현재 실습의 CRD가 예를 들어:

```yaml
apiVersion: webapp.example.com/v1
kind: WebApp

metadata:
  name: my-webapp

spec:
  frontend:
    image: nginx:latest

  backend:
    image: my-fastapi:latest
```

라면 Kopf Controller는:

```python
import kopf

@kopf.on.create("webapps")
def create_webapp(spec, name, **kwargs):

    frontend_image = spec["frontend"]["image"]
    backend_image = spec["backend"]["image"]

    print(f"WebApp: {name}")
    print(f"Frontend: {frontend_image}")
    print(f"Backend: {backend_image}")

    # Kubernetes Deployment 생성
    # Kubernetes Service 생성

    return {
        "phase": "Ready"
    }
```

처럼 만들 수 있습니다.

그러면 사용자는:

```bash
kubectl apply -f webapp.yaml
```

만 하면 됩니다.

Kopf가:

```
WebApp 생성
   ↓
Kopf 이벤트 감지
   ↓
Python handler 실행
   ↓
Frontend Deployment 생성
   ↓
Backend Deployment 생성
   ↓
Frontend Service 생성
   ↓
Backend Service 생성
   ↓
WebApp Status 업데이트
```

를 담당하도록 만들 수 있습니다.

---

# 5. Kopf와 Operator의 관계

여기서 중요한 구분이 있습니다.

**Kopf 자체가 Operator는 아닙니다.**

Kopf는 **Operator를 만들기 위한 프레임워크**입니다.

```
             Kubernetes
                  │
                  ▼
             CRD / CR
                  │
                  ▼
          ┌───────────────┐
          │    Operator   │
          │               │
          │  Python code  │
          └───────────────┘
                  ▲
                  │
                Kopf
        Python Operator Framework
```

즉:

| 구성요소 | 역할 |
| --- | --- |
| CRD | 새로운 Kubernetes 리소스 정의 |
| CR | 실제 사용자 요청 |
| Controller | CR을 보고 실제 상태를 변경 |
| Operator | CRD + Controller를 이용한 애플리케이션 관리 |
| **Kopf** | Python으로 Controller/Operator를 쉽게 만드는 프레임워크 |

---

# 6. 왜 Kopf를 사용하는가?

특히 **Python 개발자에게 편합니다.**

현재 작성하신 Controller는 Kubernetes Python Client를 직접 사용했는데, Kopf를 사용하면 다음과 같은 장점이 있습니다.

### 직접 구현

```
Kubernetes API
     ↓
watch()
     ↓
event parsing
     ↓
handler
     ↓
reconciliation
     ↓
exception handling
     ↓
retry
```

### Kopf

```python
@kopf.on.create("webapps")
def create(...):
    ...

@kopf.on.update("webapps")
def update(...):
    ...

@kopf.on.delete("webapps")
def delete(...):
    ...
```

Kopf가 Kubernetes 이벤트 처리와 Operator 실행에 필요한 많은 기반 기능을 제공하기 때문입니다.

---

# 7. 그렇다면 현재 실습에서 Kopf를 쓰는 것이 좋을까?

**네. 특히 지금 단계에서는 매우 좋은 다음 실습 주제입니다.**

지금까지 하신 실습을 발전시키면:

```
1단계
CRD 작성
   ↓
2단계
Python Kubernetes Client로 Controller 직접 구현
   ↓
3단계
WebApp 생성
   ↓
4단계
Frontend + Backend Deployment 생성
   ↓
5단계
Service 생성
   ↓
──────────────
다음 단계
──────────────
6단계
Kopf 설치
   ↓
7단계
Kopf 기반 Controller 작성
   ↓
8단계
기존 Controller와 비교
   ↓
9단계
WebApp Status 구현
   ↓
10단계
Reconciliation 구현
```

이렇게 학습하면 **"Kubernetes Controller가 내부적으로 어떻게 동작하는가"**와 **"실제 Operator는 어떻게 편하게 만드는가"**를 둘 다 이해할 수 있습니다.

특히 앞에서 직접 작성했던 **`webapp-controller.py`를 Kopf 버전으로 다시 작성해 보는 실습**을 추천합니다. 그러면 `Kubernetes Python Client 직접 구현 방식 ↔ Kopf 방식`의 차이가 아주 명확해집니다.

# Webapp 과 Ldap 사용자 인증 연결

아래는 지금까지 구축한 **`WebApp CRD + Nginx Frontend + FastAPI Backend + OpenLDAP`** 환경을 기준으로, **LDAP 사용자 인증을 WebApp에 연결하는 전체 실습 절차**를 처음부터 순서대로 간명하게 정리한 것입니다.

핵심 구조는 다음과 같습니다.

```
Browser
   │
   ▼
Nginx Frontend
   │
   │ /api/*
   ▼
FastAPI Backend
   │
   │ LDAP Bind
   ▼
OpenLDAP
   │
   └── uid=jongkim,ou=users,dc=hdaic,dc=com
```

---

# 1. 현재 Kubernetes 환경 확인

## 실행 위치: control-plane

LDAP Service 확인:

```bash
kubectl get svc -A | grep -i ldap
```

현재 환경:

```
ldap    openldap    ClusterIP    10.104.20.241    389/TCP,636/TCP
```

따라서 Backend에서 LDAP 접속 주소는:

```
ldap://openldap.ldap.svc.cluster.local:389
```

입니다.

> `openldap`만 사용하면 Backend와 LDAP가 서로 다른 namespace일 경우 문제가 발생할 수 있으므로 FQDN을 사용하는 것을 권장합니다.
> 

---

# 2. LDAP 사용자 구조 확인

현재 LDAP 사용자 구조를 다음과 같이 가정합니다.

```
dc=hdaic,dc=com
└── ou=users
      ├── uid=alice
      ├── uid=bob
      ├── uid=jongkim
      └── uid=honglee
```

로그인할 사용자 DN:

```
uid=jongkim,ou=users,dc=hdaic,dc=com
```

LDAP 사용자 확인:

```bash
ldapsearch \
  -x \
  -H ldap://10.104.20.241:389 \
  -b "ou=users,dc=hdaic,dc=com" \
  "(uid=jongkim)"
```

---

# 3. LDAP 직접 인증 테스트

WebApp에 연결하기 전에 LDAP 인증 자체가 정상인지 확인합니다.

```bash
ldapwhoami \
  -x \
  -H ldap://10.104.20.241:389 \
  -D "uid=jongkim,ou=users,dc=hdaic,dc=com" \
  -W
```

성공:

```
dn:uid=jongkim,ou=users,dc=hdaic,dc=com
```

여기까지 성공해야 합니다.

---

# 4. FastAPI Backend 구성

## 실행 위치: worker-1

```bash
cd ~/webapp-crd/backend
```

구성:

```
backend/
├── main.py
├── requirements.txt
└── Dockerfile
```

---

# 5. `requirements.txt`

```
fastapi
uvicorn
ldap3
python-jose[cryptography]
```

---

# 6. FastAPI `main.py`

```python
import os

from fastapi import FastAPI, HTTPException, Depends
from fastapi.middleware.cors import CORSMiddleware
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials

from pydantic import BaseModel
from ldap3 import Server, Connection, ALL
from jose import jwt, JWTError

LDAP_SERVER = os.getenv(
    "LDAP_SERVER",
    "ldap://openldap.ldap.svc.cluster.local:389"
)

LDAP_BASE_DN = os.getenv(
    "LDAP_BASE_DN",
    "ou=users,dc=hdaic,dc=com"
)

JWT_SECRET = os.getenv(
    "JWT_SECRET",
    "change-this-secret"
)

JWT_ALGORITHM = "HS256"

app = FastAPI(
    title="WebApp Backend",
    version="2.0.0"
)

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=False,
    allow_methods=["*"],
    allow_headers=["*"],
)

security = HTTPBearer()

class LoginRequest(BaseModel):
    username: str
    password: str

def authenticate_ldap(username, password):

    user_dn = (
        f"uid={username},"
        f"{LDAP_BASE_DN}"
    )

    print(f"LDAP server: {LDAP_SERVER}")
    print(f"LDAP user DN: {user_dn}")

    try:

        server = Server(
            LDAP_SERVER,
            get_info=ALL
        )

        conn = Connection(
            server,
            user=user_dn,
            password=password,
            auto_bind=True
        )

        conn.unbind()

        return True

    except Exception as e:

        print(
            f"LDAP authentication failed: "
            f"{repr(e)}"
        )

        return False

def create_access_token(username):

    payload = {
        "sub": username
    }

    return jwt.encode(
        payload,
        JWT_SECRET,
        algorithm=JWT_ALGORITHM
    )

def get_current_user(
    credentials: HTTPAuthorizationCredentials =
        Depends(security)
):

    token = credentials.credentials

    try:

        payload = jwt.decode(
            token,
            JWT_SECRET,
            algorithms=[JWT_ALGORITHM]
        )

        username = payload.get("sub")

        if not username:

            raise HTTPException(
                status_code=401,
                detail="Invalid token"
            )

        return username

    except JWTError:

        raise HTTPException(
            status_code=401,
            detail="Invalid token"
        )

@app.get("/")
def root():

    return {
        "message": "Hello from FastAPI backend"
    }

@app.get("/api/hello")
def hello():

    return {
        "message": "Hello from WebApp backend"
    }

@app.post("/api/login")
def login(request: LoginRequest):

    if not authenticate_ldap(
        request.username,
        request.password
    ):

        raise HTTPException(
            status_code=401,
            detail="Invalid username or password"
        )

    token = create_access_token(
        request.username
    )

    return {
        "access_token": token,
        "token_type": "bearer",
        "username": request.username
    }

@app.get("/api/me")
def me(
    username: str = Depends(get_current_user)
):

    return {
        "username": username,
        "message": f"Hello {username}"
    }

@app.get("/api/protected")
def protected(
    username: str = Depends(get_current_user)
):

    return {
        "message": "Protected API access granted",
        "username": username
    }
```

---

# 7. Backend Dockerfile

```docker
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY main.py .

EXPOSE 8000

CMD [
    "uvicorn",
    "main:app",
    "--host",
    "0.0.0.0",
    "--port",
    "8000"
]
```

---

# 8. Frontend Nginx 구성

Frontend가 Backend NodePort를 직접 호출하지 않도록 합니다.

## `frontend/nginx.conf`

```
server {

    listen 80;

    server_name _;

    location / {

        root /usr/share/nginx/html;

        index index.html;

        try_files $uri $uri/ /index.html;
    }

    location /api/ {

        proxy_pass http://my-webapp-backend:8000;

        proxy_http_version 1.1;

        proxy_set_header Host $host;

        proxy_set_header X-Real-IP $remote_addr;

        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

여기서:

```
my-webapp-backend
```

는 실제 Kubernetes Backend Service 이름이어야 합니다.

확인:

```bash
kubectl get svc
```

---

# 9. Frontend JavaScript

기존 Backend 호출:

```jsx
fetch(
    "http://192.168.56.11:32277/api/hello"
)
```

를 제거합니다.

대신:

```jsx
fetch("/api/hello")
```

를 사용합니다.

---

## LDAP Login

```jsx
async function login() {

    const username =
        document.getElementById("username")
        .value.trim();

    const password =
        document.getElementById("password")
        .value;

    const response =
        await fetch(
            "/api/login",
            {
                method: "POST",

                headers: {
                    "Content-Type":
                        "application/json"
                },

                body: JSON.stringify({
                    username: username,
                    password: password
                })
            }
        );

    const data =
        await response.json();

    if (!response.ok) {

        throw new Error(
            data.detail
        );
    }

    localStorage.setItem(
        "access_token",
        data.access_token
    );

    localStorage.setItem(
        "username",
        data.username
    );
}
```

---

# 10. 인증된 API 호출

```jsx
async function testProtectedAPI() {

    const token =
        localStorage.getItem(
            "access_token"
        );

    const response =
        await fetch(
            "/api/protected",
            {
                headers: {
                    "Authorization":
                        "Bearer " + token
                }
            }
        );

    const data =
        await response.json();

    console.log(data);
}
```

정상 결과:

```json
{
  "message": "Protected API access granted",
  "username": "jongkim"
}
```

---

# 11. Logout

```jsx
function logout() {

    localStorage.removeItem(
        "access_token"
    );

    localStorage.removeItem(
        "username"
    );
}
```

---

# 12. Backend 이미지 Build

## 실행 위치: worker-1

```bash
cd ~/webapp-crd/backend
```

```bash
docker build \
  -t 192.168.56.11:30002/webapp/backend:2.1 .
```

```bash
docker push \
  192.168.56.11:30002/webapp/backend:2.1
```

---

# 13. Frontend 이미지 Build

```bash
cd ~/webapp-crd/frontend
```

```bash
docker build \
  -t 192.168.56.11:30002/webapp/frontend:2.1 .
```

```bash
docker push \
  192.168.56.11:30002/webapp/frontend:2.1
```

---

# 14. WebApp CR 수정

## 실행 위치: control-plane

`my-webapp.yaml`:

```yaml
apiVersion: webapp.example.com/v1
kind: WebApp

metadata:
  name: my-webapp

spec:

  frontend:
    image: 192.168.56.11:30002/webapp/frontend:2.1
    replicas: 1

  backend:
    image: 192.168.56.11:30002/webapp/backend:2.1
    replicas: 1
```

적용:

```bash
kubectl apply -f my-webapp.yaml
```

---

# 15. Backend에 LDAP 환경변수 주입

현재 Controller가 아직 LDAP 설정을 자동 처리하지 않는다면 Backend Deployment에 임시로 직접 넣습니다.

```yaml
env:

  - name: LDAP_SERVER
    value: "ldap://openldap.ldap.svc.cluster.local:389"

  - name: LDAP_BASE_DN
    value: "ou=users,dc=hdaic,dc=com"

  - name: JWT_SECRET
    value: "change-this-secret"
```

확인:

```bash
kubectl exec \
  deployment/my-webapp-backend \
  -- env | grep LDAP
```

정상:

```
LDAP_SERVER=ldap://openldap.ldap.svc.cluster.local:389
LDAP_BASE_DN=ou=users,dc=hdaic,dc=com
```

---

# 16. Backend → LDAP 연결 테스트

먼저 Pod 확인:

```bash
kubectl get pods
```

Backend Pod에서 DNS 확인:

```bash
kubectl exec \
  deployment/my-webapp-backend \
  -- \
  getent hosts openldap.ldap.svc.cluster.local
```

TCP 연결 확인:

```bash
kubectl exec \
  deployment/my-webapp-backend \
  -- \
  python -c \
  "import socket; print(socket.create_connection(('openldap.ldap.svc.cluster.local',389),5))"
```

성공하면:

```
Backend
   │
   └── TCP 389
          │
          ▼
       OpenLDAP
```

네트워크 연결은 정상입니다.

---

# 17. Backend API 테스트

Backend의 기본 API:

```bash
kubectl port-forward \
  svc/my-webapp-backend \
  8000:8000
```

다른 터미널에서:

```bash
curl http://localhost:8000/api/hello
```

결과:

```json
{
  "message": "Hello from WebApp backend"
}
```

---

# 18. LDAP Login API 테스트

```bash
curl -X POST \
  http://localhost:8000/api/login \
  -H "Content-Type: application/json" \
  -d '{
    "username": "jongkim",
    "password": "실제비밀번호"
  }'
```

성공:

```json
{
  "access_token": "eyJ...",
  "token_type": "bearer",
  "username": "jongkim"
}
```

실패:

```json
{
  "detail": "Invalid username or password"
}
```

HTTP:

```
401 Unauthorized
```

---

# 19. JWT 인증 테스트

로그인 결과에서 Token을 복사합니다.

```bash
curl \
  http://localhost:8000/api/protected \
  -H "Authorization: Bearer <TOKEN>"
```

정상:

```json
{
  "message": "Protected API access granted",
  "username": "jongkim"
}
```

Token 없이 호출하면:

```bash
curl \
  http://localhost:8000/api/protected
```

결과:

```
401 Unauthorized
```

---

# 20. 최종 브라우저 테스트

Frontend 접속:

```
http://192.168.56.11:<Frontend-NodePort>
```

순서:

```
① Frontend 접속
       ↓
② Test Backend Connection
       ↓
③ Backend Connected
       ↓
④ LDAP Username 입력
       ↓
⑤ LDAP Password 입력
       ↓
⑥ Login
       ↓
⑦ FastAPI → LDAP Bind
       ↓
⑧ JWT 발급
       ↓
⑨ Browser localStorage 저장
       ↓
⑩ Protected API 호출
```

---

# 21. 문제가 발생할 경우 확인 순서

특히 이번에 발생했던 `401 Unauthorized`는 다음 순서로 확인하면 됩니다.

### ① LDAP Service

```bash
kubectl get svc -A | grep -i ldap
```

현재:

```
ldap    openldap    ClusterIP    10.104.20.241    389/TCP
```

### ② Backend LDAP 환경변수

```bash
kubectl exec \
  deployment/my-webapp-backend \
  -- env | grep LDAP
```

### ③ Backend 로그

```bash
kubectl logs \
  deployment/my-webapp-backend \
  --tail=50
```

특히:

```
LDAP authentication failed
```

뒤의 오류를 확인합니다.

### ④ LDAP 사용자

```bash
ldapsearch \
  -x \
  -H ldap://10.104.20.241:389 \
  -b "ou=users,dc=hdaic,dc=com" \
  "(uid=jongkim)"
```

### ⑤ 직접 LDAP Bind

```bash
ldapwhoami \
  -x \
  -H ldap://10.104.20.241:389 \
  -D "uid=jongkim,ou=users,dc=hdaic,dc=com" \
  -W
```

---

# 22. 최종 구조

현재 실습의 완성된 구조는 다음과 같습니다.

```
                       WebApp CR
                           │
                           ▼
                    Kopf Controller
                           │
             ┌─────────────┴─────────────┐
             │                           │
             ▼                           ▼
      Frontend Deployment         Backend Deployment
           Nginx                       FastAPI
             │                           │
             │ /api/*                    │
             ▼                           │
      Backend Service ◄──────────────────┘
             │
             │ LDAP
             ▼
       openldap Service
       namespace: ldap
             │
             ▼
         OpenLDAP
             │
             └── ou=users
                    │
                    └── uid=jongkim
```

인증 흐름은 한 줄로 정리하면:

```
Browser → Nginx → FastAPI → LDAP Bind → JWT → Browser → Protected API
```

입니다.

### 다음 CRD 발전 단계

현재는 LDAP 환경변수를 Backend Deployment에 직접 넣었지만, **CRD 실습의 최종 형태**에서는 이 부분도 Controller가 자동 처리하도록 만드는 것이 좋습니다.

즉 최종 `WebApp` CR을:

```yaml
spec:
  frontend:
    image: ...
  backend:
    image: ...

  auth:
    type: ldap

    ldap:
      server: openldap.ldap.svc.cluster.local
      port: 389
      baseDN: ou=users,dc=hdaic,dc=com
```

처럼 만들고,

```
WebApp CR
   ↓
Kopf Controller
   ↓
Backend Deployment
   ├── LDAP_SERVER 자동 주입
   ├── LDAP_BASE_DN 자동 주입
   └── JWT_SECRET → Kubernetes Secret
```

으로 자동 생성하게 만들면 **"WebApp CRD 하나로 LDAP 인증이 포함된 웹서비스를 배포"**하는 실습까지 완성됩니다.

# 

네. 이번에는 **현재 사용 중인 FastAPI 코드와 기존 WebApp CRD 구조를 그대로 유지**하는 것을 전제로 처음부터 다시 정리하겠습니다.

핵심은 이미 `prometheus-fastapi-instrumentator`가 들어가 있으므로 **FastAPI 코드를 수정하지 않고**, Kubernetes의 **Service → ServiceMonitor → Prometheus → Grafana**를 연결하는 것입니다.

---

# 1. 현재 실습 환경

```
control-plane
worker-1
```

- **코드 수정/이미지 빌드:** worker-1
- **kubectl 작업:** control-plane
- **WebApp:** nginx Frontend + FastAPI Backend
- **Grafana:** `http://192.168.56.11:30300/`

현재 FastAPI:

```python
from fastapi import FastAPI, HTTPException, Depends
from fastapi.middleware.cors import CORSMiddleware
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from prometheus_fastapi_instrumentator import Instrumentator

from pydantic import BaseModel
from ldap3 import Server, Connection, ALL
from jose import jwt, JWTError

import os

LDAP_SERVER = os.getenv(
    "LDAP_SERVER",
    "ldap://openldap.ldap.svc.cluster.local:389"
)

LDAP_BASE_DN = os.getenv(
    "LDAP_BASE_DN",
    "ou=users,dc=hdaic,dc=com"
)

JWT_SECRET = os.getenv(
    "JWT_SECRET",
    "change-this-secret"
)

JWT_ALGORITHM = "HS256"

app = FastAPI(
    title="WebApp Backend",
    version="2.0.0"
)

Instrumentator().instrument(app).expose(app)
```

여기서 이미 다음 코드가 있습니다.

```python
Instrumentator().instrument(app).expose(app)
```

따라서 **별도의 `/metrics` 코드를 작성할 필요가 없습니다.**

---

# 2. 전체 구조

이번 실습의 최종 구조는 다음입니다.

```
                 WebApp CRD
                     │
          ┌──────────┴──────────┐
          │                     │
          ▼                     ▼
     nginx Frontend       FastAPI Backend
                                │
                                │ /metrics
                                ▼
                         Backend Service
                                │
                                ▼
                         ServiceMonitor
                                │
                                ▼
                           Prometheus
                                │
                                │ PromQL
                                ▼
                            Grafana
                                │
                                ▼
                 192.168.56.11:30300
```

---

# 3. 1단계 — 현재 WebApp 상태 확인

## control-plane

```bash
kubectl get nodes
```

정상:

```
NAME           STATUS
control-plane  Ready
worker-1       Ready
```

WebApp 확인:

```bash
kubectl get webapp -A
```

Pod 확인:

```bash
kubectl get pods -A
```

Backend 확인:

```bash
kubectl get pods -A | grep backend
```

Service 확인:

```bash
kubectl get svc -A | grep backend
```

---

# 4. 2단계 — Prometheus 설치 상태 확인

## control-plane

```bash
kubectl get pods -A | grep -i prometheus
```

```bash
kubectl get svc -A | grep -i prometheus
```

Prometheus가 이미 실행 중이면 **재설치하지 않습니다.**

---

# 5. 3단계 — FastAPI `/metrics` 확인

현재 FastAPI에는 이미 Instrumentator가 적용되어 있습니다.

따라서 먼저 이것부터 확인합니다.

## control-plane

Backend Pod 확인:

```bash
kubectl get pods -A | grep my-webapp-backend
```

Pod 이름을 확인한 후:

```bash
kubectl exec -it <BACKEND-POD> -- curl localhost:8000/metrics
```

포트가 8000인지 모르겠다면:

```bash
kubectl get deployment my-webapp-backend -o yaml
```

에서 `containerPort`를 확인합니다.

정상이라면 다음과 비슷한 결과가 나옵니다.

```
# HELP http_requests_total ...
# TYPE http_requests_total counter
...
```

즉:

```
FastAPI
   ↓
Instrumentator
   ↓
/metrics
```

가 이미 완성되어 있습니다.

---

# 6. 4단계 — Backend Service 확인

## control-plane

```bash
kubectl get svc -A | grep my-webapp-backend
```

상세 확인:

```bash
kubectl get svc my-webapp-backend -o yaml
```

중요한 부분은 다음입니다.

```yaml
spec:
  ports:
    - port: 8000
      targetPort: 8000
```

ServiceMonitor에서 포트를 이름으로 참조하기 위해 다음처럼 만드는 것을 권장합니다.

```yaml
spec:
  ports:
    - name: http
      port: 8000
      targetPort: 8000
```

---

# 7. 5단계 — Backend Service Label 확인

```bash
kubectl get svc my-webapp-backend --show-labels
```

예를 들어:

```
app=my-webapp-backend
```

가 나온다면 ServiceMonitor에서 이 label을 사용합니다.

즉:

```yaml
selector:
  matchLabels:
    app: my-webapp-backend
```

으로 연결합니다.

---

# 8. 6단계 — ServiceMonitor CRD 확인

## control-plane

```bash
kubectl get crd | grep servicemonitor
```

다음이 나오면 됩니다.

```
servicemonitors.monitoring.coreos.com
```

없다면 현재 Prometheus가 ServiceMonitor를 지원하는 방식인지 먼저 확인해야 합니다.

---

# 9. 7단계 — ServiceMonitor 생성

현재 WebApp Backend Service가 `webapp` namespace라고 가정한 예입니다.

먼저 실제 namespace를 확인합니다.

```bash
kubectl get svc -A | grep my-webapp-backend
```

그 namespace에 맞춰 `servicemonitor.yaml`을 만듭니다.

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor

metadata:
  name: my-webapp-backend
  namespace: webapp

spec:
  selector:
    matchLabels:
      app: my-webapp-backend

  endpoints:
    - port: http
      path: /metrics
      interval: 15s
```

적용:

```bash
kubectl apply -f servicemonitor.yaml
```

확인:

```bash
kubectl get servicemonitor -A
```

---

# 10. 8단계 — Prometheus Target 확인

Prometheus UI에서:

```
Status
 → Targets
```

를 확인합니다.

WebApp Backend가 나타나야 합니다.

```
my-webapp-backend
        UP
```

**여기서 `UP`이 되어야 다음 단계로 넘어갑니다.**

전체 연결은:

```
FastAPI
   ↓
/metrics
   ↓
Service
   ↓
ServiceMonitor
   ↓
Prometheus Target
   ↓
UP
```

입니다.

---

# 11. 9단계 — 실제 API 호출

현재 사용 중인 API:

```
/api/hello
```

를 호출합니다.

기존 환경의 Backend NodePort를 알고 있다면:

```bash
curl http://192.168.56.11:<BACKEND-PORT>/api/hello
```

반복 호출:

```bash
for i in {1..100}; do
    curl -s http://192.168.56.11:<BACKEND-PORT>/api/hello > /dev/null
done
```

이 요청을 Instrumentator가 자동으로 기록합니다.

---

# 12. 10단계 — Prometheus에서 Metric 확인

Prometheus Query 화면에서 먼저:

```
http_requests_total
```

을 입력합니다.

만약 metric이 검색된다면 성공입니다.

중요한 것은 **현재 설치된 `prometheus-fastapi-instrumentator` 버전에 따라 실제 metric 이름/label이 조금 다를 수 있다는 점**입니다.

따라서 가장 정확한 방법은:

```bash
kubectl exec -it <BACKEND-POD> -- curl localhost:8000/metrics
```

결과를 기준으로 PromQL을 작성하는 것입니다.

---

# 13. 11단계 — Grafana 접속

브라우저에서:

```
http://192.168.56.11:30300/
```

접속합니다.

Grafana에서:

```
Connections
 → Data sources
```

로 이동합니다.

Prometheus Data Source가 연결되어 있는지 확인합니다.

---

# 14. 12단계 — Grafana Dashboard 생성

```
Dashboards
 → New
 → New Dashboard
 → Add visualization
```

Data source:

```
Prometheus
```

를 선택합니다.

---

## Panel 1 — 전체 HTTP 요청

우선:

```
sum(http_requests_total)
```

을 테스트합니다.

---

## Panel 2 — 초당 HTTP 요청

```
sum(
  rate(http_requests_total[1m])
)
```

그래프 형태로 표시합니다.

---

## Panel 3 — Endpoint별 요청

실제 `/metrics`에서 endpoint 관련 label 이름을 확인한 후 사용합니다.

예를 들어 label이 `handler`라면:

```
sum by (handler) (
  rate(http_requests_total[1m])
)
```

---

## Panel 4 — HTTP Status별 요청

status label이 있다면:

```
sum by (status) (
  rate(http_requests_total[1m])
)
```

그러면:

```
200
401
404
500
```

등을 구분해서 볼 수 있습니다.

---

# 15. 13단계 — 응답시간 Dashboard

Instrumentator가 제공하는 latency metric도 `/metrics`에서 확인합니다.

예를 들어 해당 metric이:

```
http_request_duration_seconds
```

계열이라면 이를 기반으로 평균/백분위 응답시간을 구성할 수 있습니다.

**여기 역시 metric 이름을 추측하기보다 실제 `/metrics` 출력값을 확인한 뒤 작성하는 것이 좋습니다.**

---

# 16. 14단계 — Grafana에서 실제 변화 확인

이제 API를 반복 호출합니다.

```bash
for i in {1..1000}; do
    curl -s http://192.168.56.11:<BACKEND-PORT>/api/hello > /dev/null
done
```

Grafana에서:

```
HTTP Request Rate
HTTP Request Count
Endpoint
HTTP Status
Response Time
```

그래프가 변화하는 것을 확인합니다.

---

# 17. 이번 실습에서 실제로 수정하는 것

현재 FastAPI 코드를 기준으로 하면 **FastAPI 소스 자체는 수정하지 않습니다.**

```
❌ Counter 직접 작성
❌ Histogram 직접 작성
❌ /metrics 직접 구현
❌ FastAPI middleware 추가
```

이미 이것이 있기 때문입니다.

```python
from prometheus_fastapi_instrumentator import Instrumentator

Instrumentator().instrument(app).expose(app)
```

이번 실습에서 집중할 부분은:

```
① Backend Service
       ↓
② ServiceMonitor
       ↓
③ Prometheus Target
       ↓
④ PromQL
       ↓
⑤ Grafana Dashboard
```

입니다.

---

# 18. 최종 실습 체크 순서

아래 순서대로 하나씩 확인하면 됩니다.

```
[1] WebApp Pod 정상
        ↓
[2] Backend Pod 정상
        ↓
[3] /metrics 정상
        ↓
[4] Backend Service 정상
        ↓
[5] ServiceMonitor 생성
        ↓
[6] Prometheus Target = UP
        ↓
[7] http_requests_total 확인
        ↓
[8] /api/hello 반복 호출
        ↓
[9] Prometheus 그래프 변화
        ↓
[10] Grafana Prometheus 연결
        ↓
[11] Grafana Dashboard 생성
        ↓
[12] Grafana에서 요청량/응답시간 확인
```

### 특히 지금 단계에서 가장 먼저 할 명령

**control-plane에서 아래 두 개를 실행해 보시는 것이 좋습니다.**

```bash
kubectl get pods -A | grep my-webapp-backend
```

그리고 Backend Pod 이름을 확인한 뒤:

```bash
kubectl exec -it <BACKEND-POD> -- curl localhost:8000/metrics
```

여기서 나오는 **실제 `/metrics` 출력**을 기준으로 다음 단계인 `ServiceMonitor + PromQL + Grafana Dashboard`를 정확하게 구성할 수 있습니다.
