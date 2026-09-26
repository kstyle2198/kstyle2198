# [실습] Auto-Healing

# 기본 실습

좋습니다. 이번에는 **실제로 따라 할 수 있는 실습 과정**으로 진행하겠습니다.

현재 환경을 기준으로 다음을 전제로 하겠습니다.

- Kubernetes: `control-plane + worker-1`
- 코드 작성/이미지 빌드: **worker-1**
- Harbor: `192.168.56.11:30002`
- 별도 Namespace: `auto-heal`
- Auto-Healer: Python + Kubernetes Python Client
- Dashboard: FastAPI + HTML/JavaScript
- Dashboard 접근: NodePort
- 최종적으로 기존 WebApp CRD와 연결

---

# 0. 최종 실습 구조

먼저 전체 그림을 이해하고 시작하겠습니다.

```
                         Browser
                            │
                            │ :30888
                            ▼
              ┌─────────────────────────┐
              │ Auto-Healer Dashboard    │
              │ FastAPI + HTML/JS        │
              └────────────┬────────────┘
                           │
                           │ events
                           ▼
                    ┌──────────────┐
                    │ Shared PVC   │
                    │ events.json  │
                    └──────┬───────┘
                           │
                           │
              ┌────────────▼────────────┐
              │      Auto-Healer        │
              │       Python            │
              └────────────┬────────────┘
                           │
                           │ Kubernetes API
                           ▼
                ┌─────────────────────┐
                │ Kubernetes Cluster  │
                │                     │
                │  worker-1           │
                │                     │
                │  WebApp             │
                │   ├─ frontend       │
                │   └─ backend        │
                └─────────────────────┘
```

---

# STEP 1. 실습 디렉터리 생성

**worker-1에서 작업합니다.**

```bash
cd ~
mkdir -p auto-healer
cd auto-healer

mkdir -p healer
mkdir -p dashboard/static
mkdir -p k8s
```

확인:

```bash
find ~/auto-healer -maxdepth 2 -type d
```

예상:

```
/home/vboxuser/auto-healer
/home/vboxuser/auto-healer/healer
/home/vboxuser/auto-healer/dashboard
/home/vboxuser/auto-healer/dashboard/static
/home/vboxuser/auto-healer/k8s
```

---

# STEP 2. Namespace 생성

`k8s/namespace.yaml`을 만듭니다.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: auto-heal
```

적용:

```bash
kubectl apply -f ~/auto-healer/k8s/namespace.yaml
```

확인:

```bash
kubectl get namespace auto-heal
```

예:

```
NAME        STATUS   AGE
auto-heal   Active   5s
```

---

# STEP 3. Kubernetes 기본 Self-Healing 확인

우리가 만드는 Auto-Healer보다 먼저 **Kubernetes 자체의 자동복구 기능**을 확인합니다.

```bash
kubectl create deployment test-app \
  --image=nginx:latest \
  --replicas=2 \
  -n auto-heal
```

확인:

```bash
kubectl get pods -n auto-heal -o wide
```

예:

```
NAME                        READY   STATUS    NODE
test-app-xxxxxxxxxx         1/1     Running   worker-1
test-app-yyyyyyyyyy         1/1     Running   worker-1
```

Deployment도 확인합니다.

```bash
kubectl get deployment -n auto-heal
```

---

## 3-1. Pod 삭제

Pod 하나를 삭제합니다.

```bash
kubectl delete pod \
  $(kubectl get pod -n auto-heal -l app=test-app -o jsonpath='{.items[0].metadata.name}') \
  -n auto-heal
```

곧바로:

```bash
kubectl get pods -n auto-heal -w
```

확인합니다.

새 Pod가 자동으로 만들어집니다.

즉:

```
Pod 삭제
   ↓
ReplicaSet 감지
   ↓
replicas = 2 유지
   ↓
새 Pod 생성
```

**이것이 Kubernetes의 기본 Self-Healing입니다.**

---

# STEP 4. Auto-Healer 개발

이제 우리가 직접 만든 장애 감지 프로그램을 추가합니다.

디렉터리:

```bash
cd ~/auto-healer/healer
```

---

## 4-1. requirements.txt

```
kubernetes
```

---

## 4-2. healer.py

다음 코드로 시작합니다.

```python
import json
import os
import time
from datetime import datetime, timezone

from kubernetes import client, config

NAMESPACE = "auto-heal"
CHECK_INTERVAL = 5
EVENT_FILE = "/events/events.json"

def load_kubernetes():

    try:
        config.load_incluster_config()
        print("[INFO] Using in-cluster config")

    except Exception:
        config.load_kube_config()
        print("[INFO] Using local kube config")

def load_events():

    if not os.path.exists(EVENT_FILE):
        return []

    try:
        with open(EVENT_FILE, "r") as f:
            return json.load(f)

    except Exception:
        return []

def save_events(events):

    os.makedirs(os.path.dirname(EVENT_FILE), exist_ok=True)

    with open(EVENT_FILE, "w") as f:
        json.dump(
            events[-100:],
            f,
            indent=2
        )

def record_event(resource, event_type, message):

    event = {
        "time": datetime.now(
            timezone.utc
        ).isoformat(),

        "resource": resource,

        "type": event_type,

        "message": message
    }

    events = load_events()

    events.append(event)

    save_events(events)

    print(
        f"[EVENT] {event_type} "
        f"{resource}: {message}"
    )

def check_pods(core_api):

    pods = core_api.list_namespaced_pod(
        namespace=NAMESPACE
    ).items

    for pod in pods:

        name = pod.metadata.name
        phase = pod.status.phase

        print(
            f"[CHECK] Pod={name} "
            f"Phase={phase}"
        )

        if phase in ["Failed", "Unknown"]:

            record_event(
                name,
                "FAILURE",
                f"Pod state is {phase}"
            )

            try:

                core_api.delete_namespaced_pod(
                    name=name,
                    namespace=NAMESPACE
                )

                record_event(
                    name,
                    "RECOVERY",
                    "Failed Pod deleted"
                )

            except Exception as e:

                record_event(
                    name,
                    "ERROR",
                    str(e)
                )

def check_deployments(apps_api):

    deployments = apps_api.list_namespaced_deployment(
        namespace=NAMESPACE
    ).items

    for deployment in deployments:

        name = deployment.metadata.name

        desired = deployment.spec.replicas or 0

        available = (
            deployment.status.available_replicas or 0
        )

        print(
            f"[CHECK] Deployment={name} "
            f"desired={desired} "
            f"available={available}"
        )

        if desired > available:

            record_event(
                name,
                "WARNING",
                f"Replica shortage: "
                f"{available}/{desired}"
            )

def main():

    load_kubernetes()

    core_api = client.CoreV1Api()

    apps_api = client.AppsV1Api()

    print(
        "===================================="
    )

    print(
        " Kubernetes Auto-Healer Started"
    )

    print(
        "===================================="
    )

    while True:

        try:

            check_pods(core_api)

            check_deployments(apps_api)

        except Exception as e:

            print(
                f"[ERROR] {e}"
            )

        time.sleep(
            CHECK_INTERVAL
        )

if __name__ == "__main__":
    main()
```

---

# STEP 5. Auto-Healer Docker 이미지

`~/auto-healer/healer/Dockerfile`

```docker
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY healer.py .

RUN mkdir -p /events

CMD ["python", "healer.py"]
```

---

# STEP 6. RBAC 설정

Auto-Healer가 Kubernetes API를 호출하려면 권한이 필요합니다.

구조는:

```
ServiceAccount
      │
      ▼
     Role
      │
      ▼
 RoleBinding
      │
      ▼
Kubernetes API
```

---

## 6-1. ServiceAccount

`k8s/rbac.yaml`

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: auto-healer
  namespace: auto-heal
```

---

## 6-2. Role

```yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: auto-healer
  namespace: auto-heal

rules:

  - apiGroups: [""]
    resources:
      - pods
    verbs:
      - get
      - list
      - delete

  - apiGroups: ["apps"]
    resources:
      - deployments
    verbs:
      - get
      - list
      - patch
```

---

## 6-3. RoleBinding

```yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: auto-healer
  namespace: auto-heal

subjects:

  - kind: ServiceAccount
    name: auto-healer
    namespace: auto-heal

roleRef:

  kind: Role
  name: auto-healer
  apiGroup: rbac.authorization.k8s.io
```

적용:

```bash
kubectl apply -f ~/auto-healer/k8s/rbac.yaml
```

확인:

```bash
kubectl get sa,role,rolebinding -n auto-heal
```

---

# STEP 7. Auto-Healer 이미지 Build / Push

worker-1에서:

```bash
cd ~/auto-healer/healer
```

Build:

```bash
docker build \
  -t 192.168.56.11:30002/test-crd/auto-healer:1.0 .
```

확인:

```bash
docker images | grep auto-healer
```

Push:

```bash
docker push \
  192.168.56.11:30002/test-crd/auto-healer:1.0
```

---

# STEP 8. Auto-Healer Deployment

그런데 이벤트 파일을 Dashboard와 공유해야 합니다.

따라서 PVC를 먼저 만듭니다.

`k8s/storage.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: auto-healer-events
  namespace: auto-heal

spec:

  accessModes:
    - ReadWriteOnce

  resources:
    requests:
      storage: 1Gi
```

적용:

```bash
kubectl apply -f ~/auto-healer/k8s/storage.yaml
```

확인:

```bash
kubectl get pvc -n auto-heal
```

정상적으로:

```
NAME                 STATUS   VOLUME
auto-healer-events   Bound
```

이 되어야 합니다.

---

# STEP 9. Auto-Healer Deployment

`k8s/healer.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auto-healer
  namespace: auto-heal

spec:

  replicas: 1

  selector:
    matchLabels:
      app: auto-healer

  template:

    metadata:
      labels:
        app: auto-healer

    spec:

      serviceAccountName: auto-healer

      containers:

        - name: healer

          image: 192.168.56.11:30002/test-crd/auto-healer:1.0

          imagePullPolicy: Always

          volumeMounts:

            - name: events
              mountPath: /events

      volumes:

        - name: events

          persistentVolumeClaim:
            claimName: auto-healer-events
```

적용:

```bash
kubectl apply -f ~/auto-healer/k8s/healer.yaml
```

확인:

```bash
kubectl get pods -n auto-heal
```

---

# STEP 10. Auto-Healer 로그 확인

```bash
kubectl logs \
  -n auto-heal \
  deployment/auto-healer \
  -f
```

정상이라면:

```
====================================
 Kubernetes Auto-Healer Started
====================================

[CHECK] Pod=test-app-xxxxx Phase=Running
[CHECK] Pod=test-app-yyyyy Phase=Running

[CHECK] Deployment=test-app desired=2 available=2
[CHECK] Deployment=auto-healer desired=1 available=1
```

여기까지 성공하면 **Auto-Healer 자체가 정상 작동**하는 것입니다.

---

# STEP 11. Dashboard 만들기

이제 웹 UI를 만듭니다.

```bash
cd ~/auto-healer/dashboard
```

구조:

```
dashboard/
├── Dockerfile
├── requirements.txt
├── main.py
└── static/
    └── index.html
```

---

## 11-1. requirements.txt

```
fastapi
uvicorn
kubernetes
```

Dashboard에서도 Kubernetes 상태를 조회할 수 있도록 `kubernetes`를 추가합니다.

---

# STEP 12. Dashboard Backend

`dashboard/main.py`

```python
import json
import os

from fastapi import FastAPI
from fastapi.responses import FileResponse
from kubernetes import client, config

app = FastAPI(
    title="Kubernetes Auto-Healer Dashboard"
)

NAMESPACE = "auto-heal"
EVENT_FILE = "/events/events.json"

def load_kubernetes():

    try:
        config.load_incluster_config()

    except Exception:
        config.load_kube_config()

load_kubernetes()

core_api = client.CoreV1Api()
apps_api = client.AppsV1Api()

@app.get("/api/health")
def health():

    return {
        "status": "ok"
    }

@app.get("/api/events")
def events():

    if not os.path.exists(EVENT_FILE):
        return []

    try:

        with open(EVENT_FILE, "r") as f:
            return json.load(f)

    except Exception:

        return []

@app.get("/api/pods")
def pods():

    result = []

    items = core_api.list_namespaced_pod(
        namespace=NAMESPACE
    ).items

    for pod in items:

        result.append({
            "name": pod.metadata.name,
            "status": pod.status.phase,
            "node": pod.spec.node_name
        })

    return result

@app.get("/api/deployments")
def deployments():

    result = []

    items = apps_api.list_namespaced_deployment(
        namespace=NAMESPACE
    ).items

    for deployment in items:

        result.append({

            "name":
                deployment.metadata.name,

            "desired":
                deployment.spec.replicas or 0,

            "available":
                deployment.status.available_replicas or 0
        })

    return result

@app.get("/")
def index():

    return FileResponse(
        "/app/static/index.html"
    )
```

---

# STEP 13. Dashboard HTML

`dashboard/static/index.html`

```html
<!DOCTYPE html>

<html lang="ko">

<head>

<meta charset="UTF-8">

<title>
Kubernetes Auto-Healer
</title>

<style>

body {
    font-family: Arial, sans-serif;
    margin: 0;
    background: #f4f6f8;
}

header {
    background: #222;
    color: white;
    padding: 20px 30px;
}

.container {
    padding: 30px;
}

.cards {
    display: flex;
    gap: 20px;
    margin-bottom: 30px;
}

.card {
    background: white;
    padding: 20px;
    border-radius: 10px;
    min-width: 180px;
    box-shadow: 0 2px 8px #ddd;
}

.value {
    font-size: 30px;
    font-weight: bold;
    margin-top: 10px;
}

.running {
    color: green;
}

.failure {
    color: red;
}

.recovery {
    color: blue;
}

table {
    width: 100%;
    background: white;
    border-collapse: collapse;
    margin-bottom: 30px;
}

th, td {
    padding: 12px;
    border-bottom: 1px solid #ddd;
}

th {
    background: #eee;
}

</style>

</head>

<body>

<header>

<h1>
🚑 Kubernetes Auto-Healer
</h1>

</header>

<div class="container">

<div class="cards">

<div class="card">

<div>
Healer Status
</div>

<div
class="value running"
id="status">

RUNNING

</div>

</div>

<div class="card">

<div>
Total Events
</div>

<div
class="value"
id="total">

0

</div>

</div>

<div class="card">

<div>
Failures
</div>

<div
class="value failure"
id="failures">

0

</div>

</div>

<div class="card">

<div>
Recoveries
</div>

<div
class="value recovery"
id="recoveries">

0

</div>

</div>

</div>

<h2>
Pod Status
</h2>

<table>

<thead>

<tr>

<th>Pod</th>
<th>Status</th>
<th>Node</th>

</tr>

</thead>

<tbody id="pods">

</tbody>

</table>

<h2>
Recent Events
</h2>

<table>

<thead>

<tr>

<th>Time</th>
<th>Resource</th>
<th>Type</th>
<th>Message</th>

</tr>

</thead>

<tbody id="events">

</tbody>

</table>

</div>

<script>

async function loadEvents() {

    const response =
        await fetch("/api/events");

    const events =
        await response.json();

    document.getElementById("total")
        .innerText = events.length;

    document.getElementById("failures")
        .innerText =
        events.filter(
            e => e.type === "FAILURE"
        ).length;

    document.getElementById("recoveries")
        .innerText =
        events.filter(
            e => e.type === "RECOVERY"
        ).length;

    const table =
        document.getElementById("events");

    table.innerHTML = "";

    events
        .slice()
        .reverse()
        .slice(0, 30)
        .forEach(event => {

        const row =
            document.createElement("tr");

        row.innerHTML = `

            <td>${event.time}</td>

            <td>${event.resource}</td>

            <td>${event.type}</td>

            <td>${event.message}</td>

        `;

        table.appendChild(row);

    });

}

async function loadPods() {

    const response =
        await fetch("/api/pods");

    const pods =
        await response.json();

    const table =
        document.getElementById("pods");

    table.innerHTML = "";

    pods.forEach(pod => {

        const row =
            document.createElement("tr");

        row.innerHTML = `

            <td>${pod.name}</td>

            <td>${pod.status}</td>

            <td>${pod.node}</td>

        `;

        table.appendChild(row);

    });

}

async function refresh() {

    try {

        await loadEvents();

        await loadPods();

    }

    catch (error) {

        document.getElementById("status")
            .innerText = "ERROR";

    }

}

refresh();

setInterval(refresh, 3000);

</script>

</body>

</html>
```

---

# STEP 14. Dashboard Dockerfile

`dashboard/Dockerfile`

```docker
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY main.py .

COPY static ./static

RUN mkdir -p /events

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

# STEP 15. Dashboard 이미지 Build / Push

worker-1에서:

```bash
cd ~/auto-healer/dashboard
```

Build:

```bash
docker build \
  -t 192.168.56.11:30002/test-crd/auto-healer-dashboard:1.0 .
```

Push:

```bash
docker push \
  192.168.56.11:30002/test-crd/auto-healer-dashboard:1.0
```

---

# STEP 16. Dashboard RBAC

Dashboard도 Kubernetes API를 조회합니다.

따라서 별도의 ServiceAccount를 사용합니다.

`k8s/dashboard-rbac.yaml`

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: auto-healer-dashboard
  namespace: auto-heal

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: auto-healer-dashboard
  namespace: auto-heal

rules:

  - apiGroups: [""]
    resources:
      - pods
    verbs:
      - get
      - list

  - apiGroups: ["apps"]
    resources:
      - deployments
    verbs:
      - get
      - list

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: auto-healer-dashboard
  namespace: auto-heal

subjects:

  - kind: ServiceAccount
    name: auto-healer-dashboard
    namespace: auto-heal

roleRef:

  kind: Role
  name: auto-healer-dashboard
  apiGroup: rbac.authorization.k8s.io
```

적용:

```bash
kubectl apply \
  -f ~/auto-healer/k8s/dashboard-rbac.yaml
```

---

# STEP 17. Dashboard Deployment + Service

`k8s/dashboard.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: auto-healer-dashboard
  namespace: auto-heal

spec:

  replicas: 1

  selector:
    matchLabels:
      app: auto-healer-dashboard

  template:

    metadata:
      labels:
        app: auto-healer-dashboard

    spec:

      serviceAccountName: auto-healer-dashboard

      containers:

        - name: dashboard

          image: 192.168.56.11:30002/test-crd/auto-healer-dashboard:1.0

          imagePullPolicy: Always

          ports:

            - containerPort: 8000

          volumeMounts:

            - name: events

              mountPath: /events

      volumes:

        - name: events

          persistentVolumeClaim:
            claimName: auto-healer-events

---

apiVersion: v1
kind: Service

metadata:

  name: auto-healer-dashboard

  namespace: auto-heal

spec:

  type: NodePort

  selector:

    app: auto-healer-dashboard

  ports:

    - port: 8000

      targetPort: 8000

      nodePort: 30888
```

적용:

```bash
kubectl apply \
  -f ~/auto-healer/k8s/dashboard.yaml
```

---

# STEP 18. Dashboard 확인

```bash
kubectl get pods -n auto-heal
```

정상:

```
NAME                                      READY   STATUS
auto-healer-xxxxxxxx                      1/1     Running
auto-healer-dashboard-yyyyyyyy            1/1     Running
test-app-xxxxxxxx                         1/1     Running
test-app-yyyyyyyy                         1/1     Running
```

Service:

```bash
kubectl get svc -n auto-heal
```

예:

```
NAME                    TYPE       PORT(S)
auto-healer-dashboard   NodePort   8000:30888/TCP
```

이제 브라우저에서:

```
http://192.168.56.11:30888
```

접속합니다.

---

# STEP 19. ⭐ 실제 장애 발생

이제 드디어 핵심 실습입니다.

Dashboard를 브라우저에 띄워 놓습니다.

그리고 worker-1 터미널에서:

```bash
kubectl get pods -n auto-heal -o wide
```

`test-app` Pod 하나를 삭제합니다.

```bash
kubectl delete pod \
  $(kubectl get pod -n auto-heal \
  -l app=test-app \
  -o jsonpath='{.items[0].metadata.name}') \
  -n auto-heal
```

Dashboard를 관찰합니다.

---

# STEP 20. 첫 번째 장애 결과

Dashboard에서:

```
Total Events
1

Failures
1

Recoveries
1
```

와 비슷하게 나타나도록 만드는 것이 목표입니다.

이때 실제 Kubernetes에서는:

```
Pod 삭제
   ↓
ReplicaSet
   ↓
새 Pod 생성
```

이 이미 일어납니다.

Auto-Healer에서는:

```
장애 감지
   ↓
Event 기록
   ↓
복구 작업
   ↓
Recovery 기록
```

을 담당합니다.

---

# STEP 21. ⭐ Replica 장애 실습

이번에는:

```bash
kubectl scale deployment test-app \
  -n auto-heal \
  --replicas=0
```

확인:

```bash
kubectl get deployment -n auto-heal
```

현재 Auto-Healer는 replica 부족을 **감지만** 합니다.

Dashboard에는:

```
WARNING
Replica shortage: 0/0
```

같은 정보가 나올 수 있습니다.

여기서 중요한 문제가 발견됩니다.

```
사용자가 replicas 자체를 0으로 변경
```

한 상황과

```
원래 replicas=2였는데
실제 Pod가 0개가 된 상황
```

을 구별해야 합니다.

따라서 **다음 단계에서는 Desired State를 별도로 저장**해야 합니다.

이것이 실제 Controller 설계에서 중요한 개념입니다.

---

# STEP 22. ⭐⭐ Auto-Healer를 진짜 복구 시스템으로 개선

다음 버전에서는 다음 정보를 관리합니다.

```
Application
      │
      ├── desired replicas = 2
      ├── desired image = nginx:latest
      └── health endpoint
```

그리고:

```
현재 상태
      │
      ├── replicas = 0
      ├── image = broken
      └── health = failed
```

비교합니다.

```
Desired State
      VS
Actual State
      ↓
Difference
      ↓
Recovery
```

이것이 Kubernetes Controller의 핵심 철학입니다.

---

# STEP 23. CrashLoopBackOff 장애

테스트용 장애 Deployment를 하나 만듭니다.

```bash
kubectl create deployment broken-app \
  --image=busybox \
  -n auto-heal
```

그리고:

```bash
kubectl patch deployment broken-app \
  -n auto-heal \
  --type='strategic' \
  -p='{"spec":{"template":{"spec":{"containers":[{"name":"busybox","command":["sh","-c","exit 1"]}]}}}}'
```

확인:

```bash
kubectl get pods -n auto-heal
```

다음과 같은 상태가 됩니다.

```
broken-app-xxxxx
0/1
CrashLoopBackOff
```

Dashboard에도 이 상태가 보이게 발전시킬 수 있습니다.

---

# STEP 24. ⭐⭐ WebApp CRD와 연결

이제 지금까지 만든 WebApp CRD와 연결합니다.

기존 구조:

```
WebApp
  ↓
Kopf
  ↓
Frontend Deployment
Backend Deployment
Services
```

여기에:

```
             WebApp CRD
                  │
                  ▼
              Kopf
                  │
       ┌──────────┴──────────┐
       ▼                     ▼
   Frontend               Backend
       │                     │
       └──────────┬──────────┘
                  ▼
             Auto-Healer
                  │
          ┌───────┼────────┐
          ▼       ▼        ▼
         Pod    Replica    HTTP
        장애      장애      장애
          │       │        │
          └───────┼────────┘
                  ▼
              Recovery
                  │
                  ▼
              events.json
                  │
                  ▼
              Dashboard
```

로 연결합니다.

---

# STEP 25. 최종 실습 시나리오

완성 후에는 다음 시나리오를 직접 시연하면 좋습니다.

### 시나리오 A — Pod 장애

```bash
kubectl delete pod <backend-pod>
```

Dashboard:

```
FAILURE
    ↓
RECOVERY
```

---

### 시나리오 B — Backend 장애

```bash
kubectl exec <backend-pod> -- kill 1
```

Dashboard:

```
Backend
  ↓
Unhealthy
  ↓
Restart
  ↓
Healthy
```

---

### 시나리오 C — ImagePullBackOff

```
backend:1.0
    ↓
backend:999
    ↓
ImagePullBackOff
    ↓
Auto-Healer
    ↓
rollback
    ↓
backend:1.0
```

---

### 시나리오 D — 여러 장애 연속 발생

```
15:30:01 backend FAILURE
15:30:04 backend RECOVERY

15:31:12 frontend FAILURE
15:31:15 frontend RECOVERY

15:32:44 backend FAILURE
15:32:48 backend RECOVERY
```

Dashboard에서 장애 이력이 누적되는 모습을 확인합니다.

---

# STEP 26. 최종적으로 만들 시스템

이번 실습의 완성 형태는 다음입니다.

```
                       ┌──────────────┐
                       │    Browser   │
                       └──────┬───────┘
                              │
                         :30888
                              │
                              ▼
                  ┌────────────────────┐
                  │ Auto-Healer        │
                  │ Dashboard          │
                  │                    │
                  │ • Cluster Status   │
                  │ • Pod Status       │
                  │ • Failure Count    │
                  │ • Recovery Count   │
                  │ • Event History    │
                  └─────────┬──────────┘
                            │
                       events.json
                            │
                            ▼
                  ┌────────────────────┐
                  │    Auto-Healer     │
                  │                    │
                  │ Detect             │
                  │ Analyze            │
                  │ Recover            │
                  │ Record             │
                  └─────────┬──────────┘
                            │
                      Kubernetes API
                            │
          ┌─────────────────┼──────────────────┐
          ▼                 ▼                  ▼
      Frontend           Backend          Deployments
          │                 │                  │
          └─────────────────┼──────────────────┘
                            ▼
                       Kubernetes
```

## 지금은 여기까지 먼저 구축하면 됩니다

실습을 **한 단계씩 검증**하는 것이 중요합니다. 특히 현재 환경에서는 바로 전체를 적용하기보다 다음 순서로 진행하는 것을 권장합니다.

```
1. Namespace
   ↓
2. test-app
   ↓
3. Kubernetes 기본 Self-Healing 확인
   ↓
4. Auto-Healer 코드
   ↓
5. Harbor 이미지
   ↓
6. RBAC
   ↓
7. PVC
   ↓
8. Auto-Healer Pod
   ↓
9. Auto-Healer 로그 확인
   ↓
10. Dashboard 이미지
   ↓
11. Dashboard Pod
   ↓
12. NodePort :30888
   ↓
13. Browser 확인
   ↓
14. Pod 장애
   ↓
15. Dashboard에서 장애/복구 확인
```

**특히 1~9번을 먼저 완료한 뒤 10번부터 Dashboard를 붙이는 것을 추천합니다.** 그러면 문제가 발생했을 때 Kubernetes/RBAC 문제인지, Auto-Healer 문제인지, Dashboard 문제인지 쉽게 구분할 수 있습니다.

# Auto-healing 모니터링 범위 확장

네. **Dashboard의 적용 범위를 `auto-heal` namespace → 전체 namespace**로 확장할 때 핵심적으로 변경한 것은 **2가지**입니다.

## 1. `main.py` — 전체 Namespace 조회로 변경

### 기존

```python
core_api.list_namespaced_pod(
    namespace=NAMESPACE
)
```

```python
apps_api.list_namespaced_deployment(
    namespace=NAMESPACE
)
```

`NAMESPACE = "auto-heal"`에 한정되어 있었습니다.

### 변경

```python
core_api.list_pod_for_all_namespaces()
```

```python
apps_api.list_deployment_for_all_namespaces()
```

즉:

```
기존
Dashboard
  ↓
auto-heal namespace만 조회

변경
Dashboard
  ↓
모든 namespace 조회
```

그리고 Pod/Deployment 정보에 `namespace`를 추가했습니다.

```python
{
    "namespace": pod.metadata.namespace,
    "name": pod.metadata.name,
    "status": pod.status.phase,
    "node": pod.spec.node_name
}
```

---

## 2. Dashboard RBAC — `Role` → `ClusterRole`

이 부분이 **가장 중요**합니다.

### 기존

```
ServiceAccount
      ↓
Role
      ↓
RoleBinding
      ↓
auto-heal namespace
```

`Role`은 특정 namespace에 한정됩니다.

### 변경

```
ServiceAccount
      ↓
ClusterRole
      ↓
ClusterRoleBinding
      ↓
전체 namespace
```

핵심 YAML:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: auto-healer-dashboard

rules:
  - apiGroups: [""]
    resources:
      - pods
    verbs:
      - get
      - list
      - watch

  - apiGroups: ["apps"]
    resources:
      - deployments
    verbs:
      - get
      - list
      - watch
```

그리고:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: auto-healer-dashboard

subjects:
  - kind: ServiceAccount
    name: auto-healer-dashboard
    namespace: auto-heal

roleRef:
  kind: ClusterRole
  name: auto-healer-dashboard
  apiGroup: rbac.authorization.k8s.io
```

---

## 3. 반드시 권한 확인

```bash
kubectl auth can-i \
  list pods \
  --all-namespaces \
  --as=system:serviceaccount:auto-heal:auto-healer-dashboard
```

정상:

```
yes
```

Deployment도:

```bash
kubectl auth can-i \
  list deployments.apps \
  --all-namespaces \
  --as=system:serviceaccount:auto-heal:auto-healer-dashboard
```

정상:

```
yes
```

---

## 4. Dashboard HTML도 Namespace 표시

전체 namespace를 보여주므로 UI에:

```
Namespace | Pod | Status | Node
```

형태로 변경했습니다.

예:

```
kube-system | coredns-xxx       | Running | worker-1
auto-heal   | auto-healer-xxx   | Running | worker-1
default     | test-nginx-xxx    | Running | worker-1
webapp      | backend-xxx       | Running | worker-1
```

---

# 핵심만 한 줄로 정리

```
① main.py
   list_namespaced_*()
        ↓
   list_*_for_all_namespaces()

② RBAC
   Role + RoleBinding
        ↓
   ClusterRole + ClusterRoleBinding

③ UI
   Namespace 컬럼 추가
```

그리고 **Dashboard와 Auto-Healer의 RBAC은 별개**입니다.

```
Auto-Healer
  └─ auto-healer ServiceAccount
       └─ ClusterRole/ClusterRoleBinding

Dashboard
  └─ auto-healer-dashboard ServiceAccount
       └─ ClusterRole/ClusterRoleBinding
```

따라서 **둘 다 전체 namespace를 대상으로 하려면 각각 ClusterRole 기반으로 권한을 구성해야 합니다.**

# Test Case

좋습니다. 현재 구축한 **전체 Namespace 감시형 Auto-Healer + Dashboard**를 검증하기 위한 가장 간단한 테스트부터 진행하면 됩니다.

## 테스트 1 — 다른 Namespace의 Pod 조회 확인 ⭐

`auto-heal`이 아닌 `default`에 테스트 Pod를 하나 만듭니다.

```bash
kubectl create deployment test-nginx \
  --image=nginx:latest \
  --replicas=1 \
  -n default
```

확인:

```bash
kubectl get pods -n default
```

그리고 Dashboard:

```
http://192.168.56.11:30888
```

의 **Pod Status**에 다음처럼 나타나야 합니다.

```
Namespace   Pod                     Status    Node
---------------------------------------------------------
default     test-nginx-xxxxx        Running   worker-1
```

이것으로 **전체 namespace 조회가 정상**인지 확인할 수 있습니다.

---

# 테스트 2 — Pod 장애 감지 ⭐⭐⭐

가장 추천하는 테스트입니다.

먼저 Pod 이름을 확인합니다.

```bash
kubectl get pods -n default
```

예:

```
NAME                          READY   STATUS
test-nginx-7d8b7c9d8-x7abc    1/1     Running
```

Pod를 삭제합니다.

```bash
kubectl delete pod \
  test-nginx-7d8b7c9d8-x7abc \
  -n default
```

곧바로:

```bash
kubectl get pods -n default -w
```

하면 Kubernetes가 새로운 Pod를 생성합니다.

```
test-nginx-7d8b7c9d8-x7abc    Terminating
test-nginx-7d8b7c9d8-qwert    ContainerCreating
test-nginx-7d8b7c9d8-qwert    Running
```

### Dashboard에서는

Auto-Healer가 감지한 이벤트가 있다면:

```
Recent Events

Namespace   Resource       Type
---------------------------------------
default     test-nginx     ...
```

형태로 나타나야 합니다.

---

# 테스트 3 — 의도적으로 Failed Pod 만들기 ⭐⭐⭐

Auto-Healer의 **실제 복구 기능**을 테스트하려면 이것이 더 좋습니다.

```bash
kubectl run broken-test \
  --image=busybox \
  --restart=Never \
  -n default \
  -- sh -c "exit 1"
```

확인:

```bash
kubectl get pod broken-test -n default
```

예:

```
NAME          READY   STATUS
broken-test   0/1     Error
```

Auto-Healer 로그:

```bash
kubectl logs \
  -n auto-heal \
  deployment/auto-healer \
  -f
```

다음과 비슷한 로그가 나타나는지 확인합니다.

```
[CHECK] default/broken-test Phase=Failed

[EVENT] default/broken-test FAILURE:
Pod state is Failed

[EVENT] default/broken-test RECOVERY:
Failed Pod deleted
```

그리고:

```bash
kubectl get pod broken-test -n default
```

결과:

```
Error from server (NotFound):
pods "broken-test" not found
```

이면 **Auto-Healer가 실제로 장애 Pod를 발견하고 삭제한 것**입니다.

---

# 테스트 4 — Dashboard 실시간 확인 ⭐⭐⭐

테스트 3을 수행하면서 브라우저를 열어둡니다.

```
http://192.168.56.11:30888
```

Dashboard가 3초마다 갱신되도록 만들었기 때문에 이벤트가 표시되는지 확인합니다.

예:

```
Healer Status     RUNNING
Total Events      2
Failures          1
Recoveries        1
```

그리고:

```
Recent Events

Time                  Namespace   Resource       Type
---------------------------------------------------------
16:50:01              default     broken-test    FAILURE
16:50:06              default     broken-test    RECOVERY
```

가 보이면 아주 좋습니다.

---

# 테스트 5 — 다른 Namespace에서도 작동하는지 확인 ⭐⭐⭐⭐

현재 클러스터에 이미 `webapp`, `keycloak`, `crud-app` 등의 namespace가 있다면 **테스트용 Pod를 새로 만들 필요 없이** 해당 namespace의 Pod가 Dashboard에 표시되는지만 확인해도 됩니다.

```bash
kubectl get pods -A
```

예:

```
NAMESPACE     NAME                         STATUS
------------------------------------------------------
kube-system   coredns-xxxxx                Running
auto-heal     auto-healer-xxxxx            Running
default       test-nginx-xxxxx             Running
webapp        my-webapp-backend-xxxxx      Running
webapp        my-webapp-frontend-xxxxx     Running
```

Dashboard에도 여러 namespace가 나타나면:

> **Auto-Healer Dashboard의 전체 namespace 조회 기능 성공**
> 

입니다.

---

# 가장 간단한 테스트 순서

처음에는 **3개만** 해보세요.

### ① 정상 Pod 생성

```bash
kubectl create deployment test-nginx \
  --image=nginx:latest \
  --replicas=1 \
  -n default
```

Dashboard에서:

```
default / test-nginx-xxx / Running / worker-1
```

확인.

### ② 장애 Pod 생성

```bash
kubectl run broken-test \
  --image=busybox \
  --restart=Never \
  -n default \
  -- sh -c "exit 1"
```

### ③ Auto-Healer 로그 확인

```bash
kubectl logs \
  -n auto-heal \
  deployment/auto-healer \
  -f
```

다음 흐름이 나오면 성공입니다.

```
broken-test
    ↓
Failed
    ↓
FAILURE 이벤트
    ↓
Pod 삭제
    ↓
RECOVERY 이벤트
```
