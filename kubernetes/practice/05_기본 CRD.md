# [실습] 기본 CRD & Controller

# 기본 흐름(1단계)

좋습니다. 지금까지 진행한 실습은 단순히 CRD를 만드는 것에서 시작해서 **CRD → CR → Schema 검증 → Controller → Reconciliation → Controller Pod → Harbor 이미지 배포**까지 확장되었습니다.

아래와 같이 하나의 **Kubernetes CRD/Controller 실습 과정**으로 정리하면 좋습니다.

# Kubernetes CRD + Controller 실습 전체 과정

## 전체 실습 구조

최종적으로 만든 구조는 다음과 같습니다.

```
                    Kubernetes Cluster
              ┌─────────────────────────┐
              │                         │
              │  control-plane          │
              │                         │
              │  Kubernetes API Server  │
              │          │              │
              │          ▼              │
              │     AISTudio CR         │
              │          │              │
              │          │ watch         │
              │          ▼              │
              │                         │
              │  worker-1               │
              │          │              │
              │          ▼              │
              │  Controller Pod         │
              │          │              │
              │          ▼              │
              │  Deployment             │
              │          │              │
              │          ▼              │
              │      Application Pod     │
              │                         │
              └─────────────────────────┘
```

그리고 Controller 이미지는 Harbor에서 가져옵니다.

```
Harbor
192.168.56.11:30002
        │
        │ test-crd
        ▼
aistudio-controller:1.0
        │
        ▼
worker-1 containerd
        │
        ▼
Controller Pod
```

---

# 1단계. Kubernetes CRD 개념 이해

먼저 Kubernetes 기본 객체 외에 **사용자가 직접 새로운 Kubernetes 리소스 타입을 만들 수 있다는 것**을 확인했습니다.

기본 Kubernetes 객체:

```
Pod
Deployment
Service
ConfigMap
Secret
...
```

여기에 우리가 직접:

```
AISTudio
```

라는 새로운 리소스 타입을 추가합니다.

구조는:

```
Kubernetes
   │
   ├── Pod
   ├── Deployment
   ├── Service
   │
   └── AISTudio ← 우리가 추가
```

---

# 2단계. AISTudio CRD 작성

`AISTudio`라는 Custom Resource를 정의하는 CRD YAML을 작성했습니다.

개념적으로:

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition

metadata:
  name: aistudios.example.com

spec:
  group: example.com

  names:
    kind: AISTudio
    plural: aistudios
    singular: aistudio

  scope: Namespaced

  versions:
  - name: v1
    served: true
    storage: true
```

여기서 중요한 것은:

```
CRD
 ↓
AISTudio라는 새로운 Kubernetes 리소스 타입 정의
```

입니다.

---

# 3단계. CRD 생성

작성한 CRD를 Kubernetes에 등록했습니다.

```bash
kubectl apply -f aistudio-crd.yaml
```

확인:

```bash
kubectl get crd
```

예:

```
NAME
aistudios.example.com
```

---

# 4단계. `kubectl api-resources`로 확인

CRD가 Kubernetes API에 정상적으로 등록되었는지 확인했습니다.

```bash
kubectl api-resources
```

여기에서:

```
aistudios
```

가 나타납니다.

즉:

```
YAML 작성
   ↓
kubectl apply
   ↓
CRD 등록
   ↓
Kubernetes API Resource에 AISTudio 등장
```

이라는 흐름을 확인했습니다.

---

# 5단계. AISTudio CR 작성

CRD가 타입을 정의했다면 실제 객체를 생성할 수 있습니다.

예:

```yaml
apiVersion: example.com/v1
kind: AISTudio

metadata:
  name: my-aistudio

spec:
  replicas: 2
```

여기서 구분이 중요합니다.

### CRD

```
AISTudio라는 리소스 타입을 정의
```

### CR

```
my-aistudio라는 실제 AISTudio 객체 생성
```

즉:

```
CRD
 │
 │ defines
 ▼
AISTudio Type
 │
 │ creates
 ▼
AISTudio CR
my-aistudio
```

---

# 6단계. AISTudio CR 생성

```bash
kubectl apply -f aistudio.yaml
```

확인:

```bash
kubectl get aistudio
```

예:

```
NAME
my-aistudio
```

상세 확인:

```bash
kubectl describe aistudio my-aistudio
```

또는:

```bash
kubectl get aistudio my-aistudio -o yaml
```

---

# 7단계. CRD Schema 검증 실습

Controller를 만들기 전에 **CRD의 Schema Validation 기능**을 확인했습니다.

예를 들어 `replicas`가 정수여야 한다고 정의했다면:

```yaml
spec:
  replicas:
    type: integer
```

정상적인 CR:

```yaml
spec:
  replicas: 2
```

하지만 잘못된 값:

```yaml
spec:
  replicas: "two"
```

를 넣어:

```bash
kubectl apply -f aistudio-invalid.yaml
```

하면 Kubernetes API Server가 거부합니다.

핵심은:

```
사용자
 ↓
kubectl apply
 ↓
API Server
 ↓
CRD Schema Validation
 ↓
잘못된 데이터 → 거부
```

입니다.

### 중요한 점

이 단계에서는 **Controller가 없어도 검증이 됩니다.**

즉:

```
CRD Schema Validation
```

은 Kubernetes API Server의 기능입니다.

---

# 8단계. Controller의 필요성 이해

여기서 중요한 문제가 생깁니다.

우리가:

```yaml
kind: AISTudio

spec:
  replicas: 2
```

라고 작성했지만 Kubernetes는 `AISTudio`가 무엇을 의미하는지 모릅니다.

즉 CR만 생성하면:

```
AISTudio CR
```

은 생기지만 실제:

```
Deployment
Pod
Service
```

가 자동으로 생성되는 것은 아닙니다.

그래서 Controller를 추가했습니다.

---

# 9단계. AISTudio Controller 작성

Controller의 기본 역할:

```
AISTudio CR
     │
     ▼
Controller
     │
     ▼
Deployment 생성/수정
```

예를 들어:

```
AISTudio
name: my-aistudio
replicas: 2
```

가 있으면 Controller가:

```
Deployment
name: my-aistudio
replicas: 2
```

를 만들도록 구현했습니다.

---

# 10단계. Reconciliation 개념 실습

Controller의 핵심 개념은 **Reconciliation(일치, 조화)**입니다.

Controller는 단순히:

```
CR 생성 → Deployment 생성
```

만 하는 프로그램이 아닙니다.

계속해서:

```
Desired State
      VS
Actual State
```

를 비교합니다.

예:

```
AISTudio CR
replicas = 2
      │
      ▼
Desired State
Deployment replicas = 2
      │
      ▼
Actual State
```

둘이 다르면 Controller가 수정합니다.

---

# 11단계. Reconciliation 실습

예를 들어:

```
AISTudio
   ↓
Deployment
   ↓
Pod
```

가 정상적으로 존재하는 상태에서 Deployment를 삭제합니다.

```bash
kubectl delete deployment <deployment-name>
```

그러면:

```
Deployment 삭제
      ↓
Actual State 변경
      ↓
Controller 감지
      ↓
Reconcile
      ↓
Deployment 재생성
```

이라는 동작이 발생합니다.

이것이 Kubernetes Controller의 핵심 원리입니다.

---

# 12단계. Controller를 Docker 이미지로 빌드

Controller 프로그램을 컨테이너 이미지로 만들었습니다.

예:

```docker
FROM python:3.12-slim

WORKDIR /app

RUN pip install --no-cache-dir kubernetes

COPY controller.py /app/controller.py

CMD ["python", "/app/controller.py"]
```

```bash
docker build \
  -t 192.168.56.11:30002/test-crd/aistudio-controller:1.0 .
```

이미지 이름:

```
192.168.56.11:30002
        │
        └── Harbor Registry

test-crd
        │
        └── Harbor Project

aistudio-controller
        │
        └── Repository

1.0
        │
        └── Tag
```

---

# 13단계. Harbor에 Controller 이미지 Push

Harbor 프로젝트는 최종적으로:

```
test-crd
```

를 사용했습니다.

최종 이미지:

```
192.168.56.11:30002/test-crd/aistudio-controller:1.0
```

이 이미지를 Harbor에 Push했습니다.

---

# 14단계. Kubernetes에서 Controller Deployment 작성

Controller를 Kubernetes 안에서 실행하기 위해 Deployment를 만들었습니다.

구조:

```yaml
kind: Deployment

spec:
  replicas: 1

  template:
    spec:
      serviceAccountName: aistudio-controller

      containers:
      - name: controller
        image: 192.168.56.11:30002/test-crd/aistudio-controller:1.0
```

중요한 부분은:

```
Controller 프로그램
       ↓
Docker Image
       ↓
Harbor
       ↓
Controller Deployment
       ↓
Controller Pod
```

입니다.

---

# 15단계. HTTP Harbor Registry 문제 해결

처음 Controller Pod를 생성했을 때:

```
ImagePullBackOff
```

가 발생했습니다.

원인은:

```
worker-1
    ↓
HTTPS
    ↓
192.168.56.11:30002
    ↓
실제로는 HTTP
```

였습니다.

Pod 이벤트:

```
http: server gave HTTP response to HTTPS client
```

를 통해 확인했습니다.

---

# 16단계. Harbor Registry가 HTTP인지 확인

worker-1에서:

```bash
curl http://192.168.56.11:30002/v2/
```

결과:

```json
{
  "errors": [
    {
      "code": "UNAUTHORIZED",
      "message": "unauthorized: unauthorized"
    }
  ]
}
```

이 결과는 오히려 Registry가 정상적으로 응답하고 있다는 의미입니다.

즉:

```
HTTP 연결       정상
Registry 인증   필요
```

입니다.

HTTPS:

```bash
curl https://192.168.56.11:30002/v2/
```

에서는:

```
wrong version number
```

가 발생했습니다.

따라서 Registry가 HTTP임을 확인했습니다.

---

# 17단계. containerd 2.2.1 확인

worker-1에서:

```bash
containerd --version
```

결과:

```
containerd github.com/containerd/containerd/v2 2.2.1
```

이었습니다.

또한 CRI plugin도 정상임을 확인했습니다.

```bash
sudo ctr plugins ls | grep -E 'cri|images'
```

결과:

```
io.containerd.cri.v1   images    ok
io.containerd.cri.v1   runtime   linux/amd64   ok
io.containerd.grpc.v1  cri       ok
```

---

# 18단계. containerd HTTP Registry 설정

worker-1에:

```
/etc/containerd/certs.d/192.168.56.11:30002/
```

디렉터리를 만들고:

```
hosts.toml
```

을 구성했습니다.

내용:

```toml
server = "http://192.168.56.11:30002"

[host."http://192.168.56.11:30002"]
  capabilities = ["pull", "resolve", "push"]
```

그리고 containerd를 재시작했습니다.

```bash
sudo systemctl restart containerd
```

---

# 19단계. containerd Registry 설정 수정

containerd 2.2.1의 CRI 설정에서:

```toml
config_path = '/etc/containerd/certs.d:/etc/docker/certs.d'
```

를 사용하고 있었고 이를:

```toml
config_path = '/etc/containerd/certs.d'
```

로 변경했습니다.

이후:

```bash
sudo systemctl restart containerd
```

했습니다.

---

# 20단계. `crictl`을 이용한 실제 Image Pull 확인

이 단계가 매우 중요합니다.

`ctr`가 아니라 Kubernetes가 사용하는 **CRI 경로**를 직접 테스트했습니다.

```bash
sudo crictl pull \
  192.168.56.11:30002/test-crd/aistudio-controller:1.0
```

결과:

```
Image is up to date for sha256:e3cd4f16...
```

따라서 다음 경로가 정상임을 확인했습니다.

```
Kubernetes
    ↓
kubelet
    ↓
CRI
    ↓
containerd 2.2.1
    ↓
HTTP Registry
    ↓
Harbor
    ↓
test-crd/aistudio-controller:1.0
```

---

# 21단계. Controller Deployment 이미지 주소 수정

처음 Deployment에서는:  (이부분은 단순 오타 수정 부분임)

```
192.168.56.11:30002/test/aistudio-controller:1.0
```

를 사용했지만 실제 Harbor 프로젝트는:

```
test-crd
```

였습니다.

따라서 최종적으로:

```yaml
image: 192.168.56.11:30002/test-crd/aistudio-controller:1.0
```

로 변경했습니다.

---

# 22단계. Controller Pod 정상 실행

Deployment를 다시 적용한 후:

```bash
kubectl get pods -n aistudio-system -w
```

결과:

```
aistudio-controller-778df97cb7-26rnl
1/1 Running
```

이 확인되었습니다.

따라서 현재 상태는:

```
                    Kubernetes
                         │
                         ▼
              aistudio-system
                         │
                         ▼
               Controller Pod
                         │
                         │ Running
                         ▼
                AISTudio Controller
```

입니다.

---

# 23단계. 현재까지 완료된 실습

지금까지의 과정을 정리하면:

| 단계 | 실습 내용 | 상태 |
| --- | --- | --- |
| 1 | CRD 개념 이해 | ✅ |
| 2 | AISTudio CRD 작성 | ✅ |
| 3 | CRD 생성 | ✅ |
| 4 | `kubectl get crd` 확인 | ✅ |
| 5 | `kubectl api-resources` 확인 | ✅ |
| 6 | AISTudio CR 작성 | ✅ |
| 7 | AISTudio CR 생성 | ✅ |
| 8 | CRD Schema Validation | ✅ |
| 9 | 잘못된 `spec` 값 검증 | ✅ |
| 10 | Controller 작성 | ✅ |
| 11 | Deployment/Pod 자동 생성 로직 | ✅ |
| 12 | Reconciliation 개념 | ✅ |
| 13 | Controller Docker 이미지 생성 | ✅ |
| 14 | Harbor `test-crd` Push | ✅ |
| 15 | HTTP Registry 문제 해결 | ✅ |
| 16 | containerd 2.2.1 설정 | ✅ |
| 17 | `crictl pull` 확인 | ✅ |
| 18 | Controller Deployment 배포 | ✅ |
| 19 | Controller Pod Running | ✅ |

---

# 24단계. 앞으로 진행할 핵심 실습

이제부터가 **Kubernetes Controller 실습의 핵심**입니다.

현재:

```
Controller Pod
      │
      ▼
AISTudio CR
```

까지 준비되었습니다.

다음 단계에서는 다음 과정을 실제로 검증하면 됩니다.

### 실습 A — Controller가 CR을 감시하는지 확인

```
AISTudio CR 생성
       ↓
Controller Watch
       ↓
Reconcile 호출
```

### 실습 B — Controller가 Deployment 생성

```
AISTudio CR
       ↓
Controller
       ↓
Deployment
       ↓
Pod
```

### 실습 C — Deployment 삭제 실험

```
Deployment
     ↓
kubectl delete
     ↓
Deployment 없음
     ↓
Controller Reconcile
     ↓
Deployment 재생성
```

### 실습 D — CR의 `spec` 변경

예:

```yaml
# ~/crd-lab/aistudio.yaml
spec:
  replicas: 2
```

에서:

```yaml
# ~/crd-lab/aistudio.yaml
spec:
  replicas: 3
```

```bash
 # 수정 반영 
 k apply -f aistudio.yaml
```

으로 변경합니다. 

그러면:

```
k get deployment -w 

CR 변경
 ↓
Controller Watch
 ↓
Reconcile
 ↓
Deployment replicas 변경
 ↓
Pod 3개
```

가 되는 것을 확인합니다.

### 실습 E — Controller Pod 자체 삭제

```bash
kubectl delete pod \
  -n aistudio-system \
  -l app=aistudio-controller
```

그러면 Deployment가 Controller Pod를 다시 생성합니다.

그리고 새 Controller가 시작된 후 **현재 Kubernetes 상태를 다시 읽어 Desired State와 Actual State를 비교**하는 과정을 관찰합니다.

---

## 최종적으로 배우게 되는 Kubernetes Controller 구조

이번 실습을 하나의 그림으로 압축하면 다음과 같습니다.

```
┌──────────────────────────────────────────────┐
│              Kubernetes API Server           │
│                                              │
│   ┌──────────────┐                           │
│   │ AISTudio CRD │                           │
│   └──────┬───────┘                           │
│          │                                    │
│          ▼                                    │
│   ┌──────────────┐                            │
│   │ AISTudio CR  │                            │
│   │ my-aistudio  │                            │
│   └──────┬───────┘                            │
│          │ Watch                              │
│          ▼                                    │
│   ┌─────────────────────┐                    │
│   │ AISTudio Controller │                    │
│   │     Pod             │                    │
│   └──────────┬──────────┘                    │
│              │ Reconcile                     │
│              ▼                               │
│   ┌─────────────────────┐                    │
│   │ Deployment          │                    │
│   └──────────┬──────────┘                    │
│              │                               │
│              ▼                               │
│        ┌──────────┐                          │
│        │   Pod    │                          │
│        └──────────┘                          │
│                                              │
└──────────────────────────────────────────────┘
```

**핵심은 `CRD 자체가 무언가를 실행하는 것이 아니라, CR + Controller가 결합되어 Kubernetes에 새로운 동작을 추가한다는 점**입니다.

현재는 **Controller Pod가 클러스터 안에서 `Running`까지 성공한 상태**이므로, 다음 실습부터는 이미지/Registry 문제는 일단 내려놓고 **`AISTudio CR 생성 → Controller Reconcile → Deployment 자동 생성 → Pod 생성`**을 집중적으로 관찰하는 것이 가장 적합합니다.

# Python Controller

좋습니다. 지금부터는 **CRD/Harbor/containerd 배포 부분은 잠시 제외하고**, **Python으로 AISTudio Controller를 만드는 부분**만 집중해서 정리하겠습니다.

핵심 목표는 이것입니다.

```
AISTudio CR
     │
     │ watch
     ▼
Python Controller
     │
     │ reconcile
     ▼
Deployment
     │
     ▼
Pod
```

# 1. Python Controller가 하는 일

Python Controller는 쉽게 말하면:

> **AISTudio라는 리소스의 현재 상태를 계속 감시하고, 원하는 상태가 되도록 Kubernetes 객체를 생성/수정하는 프로그램**
> 

입니다.

예를 들어 AISTudio CR이:

```yaml
apiVersion: example.com/v1
kind: AISTudio
metadata:
  name: my-aistudio

spec:
  replicas: 2
  image: nginx:latest
```

라면 Controller가 이것을 보고:

```
AISTudio
replicas = 2
image = nginx:latest
```

를 읽습니다.

그리고 다음 Deployment를 만들도록 합니다.

```yaml
kind: Deployment

spec:
  replicas: 2

  template:
    spec:
      containers:
      - image: nginx:latest
```

---

# 2. Python Controller의 핵심 구조

초보자라면 Controller를 다음 **4부분**으로 나눠서 이해하면 쉽습니다.

```
controller.py
│
├── ① Kubernetes API 연결
│
├── ② AISTudio CR 감시
│
├── ③ Reconcile 함수
│
└── ④ Deployment 생성/수정
```

---

# 3. Python Kubernetes Client 설치

Controller는 Python Kubernetes Client를 사용합니다.

```bash
pip install kubernetes
```

주로 사용하는 라이브러리는:

```python
from kubernetes import client
from kubernetes import config
from kubernetes import watch
```

입니다.

---

# 4. Kubernetes API 연결

Controller가 Kubernetes API Server와 통신해야 합니다.

## 클러스터 안에서 실행되는 경우

현재처럼 Controller를 Pod로 실행한다면:

```python
from kubernetes import client, config

config.load_incluster_config()

apps_api = client.AppsV1Api()
custom_api = client.CustomObjectsApi()
```

핵심은:

```python
config.load_incluster_config()
```

입니다.

이 함수는 Controller Pod 내부에 Kubernetes가 자동으로 제공하는:

```
ServiceAccount
Token
CA 인증서
API Server 주소
```

를 이용해서 Kubernetes API Server에 연결합니다.

---

# 5. 왜 `CustomObjectsApi`가 필요한가?

AISTudio는 기본 Kubernetes 객체가 아닙니다.

따라서:

```python
client.CoreV1Api()
```

만으로는 AISTudio CR을 다루지 않습니다.

AISTudio 같은 Custom Resource는:

```python
custom_api = client.CustomObjectsApi()
```

를 사용합니다.

예:

```python
custom_api.get_namespaced_custom_object(
    group="example.com",
    version="v1",
    namespace="default",
    plural="aistudios",
    name="my-aistudio"
)
```

---

# 6. Controller가 AISTudio를 감시하는 방법

가장 이해하기 쉬운 방법은 Python Kubernetes Client의 `Watch`를 사용하는 것입니다.

```python
from kubernetes import client, config, watch

config.load_incluster_config()

custom_api = client.CustomObjectsApi()

w = watch.Watch()

for event in w.stream(
    custom_api.list_cluster_custom_object,
    group="example.com",
    version="v1",
    plural="aistudios"
):
    print(event)
```

여기서 Kubernetes가 이벤트를 보내줍니다.

예:

```
ADDED
MODIFIED
DELETED
```

---

# 7. 이벤트의 구조

예를 들어 사용자가:

```bash
kubectl apply -f aistudio.yaml
```

을 실행하면 Controller가:

```
ADDED
```

이벤트를 받을 수 있습니다.

Python에서는:

```python
for event in w.stream(...):

    event_type = event["type"]
    obj = event["object"]

    print(event_type)
```

`event["object"]`에는 AISTudio CR 전체가 들어옵니다.

---

# 8. CR의 `metadata`와 `spec` 읽기

예를 들어 CR:

```yaml
metadata:
  name: my-aistudio

spec:
  replicas: 2
  image: nginx:latest
```

가 있다면 Python에서는:

```python
name = obj["metadata"]["name"]

spec = obj.get("spec", {})

replicas = spec.get("replicas", 1)
image = spec.get("image", "nginx:latest")
```

결과:

```
name     = my-aistudio
replicas = 2
image    = nginx:latest
```

입니다.

---

# 9. 이것이 Reconcile 함수의 시작

이제 핵심 함수로 분리합니다.

```python
def reconcile(obj):

    name = obj["metadata"]["name"]

    spec = obj.get("spec", {})

    replicas = spec.get("replicas", 1)
    image = spec.get("image", "nginx:latest")

    print(
        f"Reconciling {name}: "
        f"replicas={replicas}, image={image}"
    )
```

그리고 이벤트가 발생하면:

```python
for event in w.stream(...):

    obj = event["object"]

    reconcile(obj)
```

구조가 됩니다.

```
Watch
 │
 ▼
Event
 │
 ▼
AISTudio CR
 │
 ▼
reconcile()
```

---

# 10. Deployment 생성

이제 실제 Kubernetes 객체를 생성합니다.

Python Kubernetes Client에서 Deployment는:

```python
apps_api = client.AppsV1Api()
```

를 사용합니다.

Deployment 객체를 구성합니다.

```python
deployment = client.V1Deployment(
    metadata=client.V1ObjectMeta(
        name=name
    ),

    spec=client.V1DeploymentSpec(
        replicas=replicas,

        selector=client.V1LabelSelector(
            match_labels={
                "app": name
            }
        ),

        template=client.V1PodTemplateSpec(
            metadata=client.V1ObjectMeta(
                labels={
                    "app": name
                }
            ),

            spec=client.V1PodSpec(
                containers=[
                    client.V1Container(
                        name="app",
                        image=image
                    )
                ]
            )
        )
    )
)
```

조금 복잡해 보이지만 YAML과 비교하면 쉽습니다.

---

# 11. Python 객체와 YAML 비교

YAML:

```yaml
spec:
  replicas: 2

  selector:
    matchLabels:
      app: my-aistudio

  template:
    metadata:
      labels:
        app: my-aistudio

    spec:
      containers:
      - name: app
        image: nginx:latest
```

Python:

```python
client.V1DeploymentSpec(
    replicas=2,

    selector=client.V1LabelSelector(
        match_labels={
            "app": "my-aistudio"
        }
    ),

    template=client.V1PodTemplateSpec(
        metadata=client.V1ObjectMeta(
            labels={
                "app": "my-aistudio"
            }
        ),

        spec=client.V1PodSpec(
            containers=[
                client.V1Container(
                    name="app",
                    image="nginx:latest"
                )
            ]
        )
    )
)
```

즉 **Python Kubernetes Client가 YAML을 Python 객체 형태로 표현하는 것**이라고 생각하면 됩니다.

---

# 12. Deployment 생성 API 호출

Deployment 객체를 만들었으면:

```python
apps_api.create_namespaced_deployment(
    namespace=namespace,
    body=deployment
)
```

를 호출합니다.

따라서:

```python
def reconcile(obj):

    name = obj["metadata"]["name"]

    spec = obj.get("spec", {})

    replicas = spec.get("replicas", 1)
    image = spec.get("image", "nginx:latest")

    deployment = create_deployment(
        name,
        replicas,
        image
    )

    apps_api.create_namespaced_deployment(
        namespace="default",
        body=deployment
    )
```

이런 구조가 됩니다.

---

# 13. 그런데 이것만으로는 진짜 Controller가 아닙니다

여기서 아주 중요한 문제가 있습니다.

현재 코드가:

```python
create_namespaced_deployment()
```

만 사용하면 이미 Deployment가 존재할 때:

```
409 Conflict
AlreadyExists
```

가 발생합니다.

더 중요한 문제는:

```
AISTudio
   ↓
Deployment 생성
```

까지만 있고,

```
Deployment가 삭제됨
```

을 처리하지 못합니다.

그래서 **Reconciliation**이 필요합니다.

---

# 14. 진짜 Reconciliation

Controller는 다음과 같이 동작해야 합니다.

```
          AISTudio CR
              │
              ▼
        Desired State
              │
              │ 비교
              ▼
        Actual State
              │
        ┌─────┴─────┐
        │           │
       같음        다름
        │           │
        ▼           ▼
     아무것도      수정
      안 함
```

예를 들어:

```
Desired:
replicas = 2

Actual:
replicas = 1
```

이면:

```
Controller
    ↓
Deployment replicas를 2로 수정
```

합니다.

---

# 15. Deployment 존재 여부 확인

```python
try:

    existing = apps_api.read_namespaced_deployment(
        name=name,
        namespace=namespace
    )

except client.exceptions.ApiException as e:

    if e.status == 404:
        # Deployment가 없음
        create_deployment(...)
```

즉:

```
Deployment 존재?
       │
   ┌───┴───┐
   │       │
  YES      NO
   │       │
   ▼       ▼
 UPDATE   CREATE
```

입니다.

---

# 16. 가장 단순한 Controller 예제

초보자 실습에서는 처음부터 복잡한 Controller Framework를 사용하지 않고 다음 정도로 시작하는 것이 좋습니다.

```python
from kubernetes import client, config, watch

GROUP = "example.com"
VERSION = "v1"
PLURAL = "aistudios"

config.load_incluster_config()

custom_api = client.CustomObjectsApi()
apps_api = client.AppsV1Api()

def create_deployment(name, namespace, replicas, image):

    return client.V1Deployment(

        metadata=client.V1ObjectMeta(
            name=name
        ),

        spec=client.V1DeploymentSpec(

            replicas=replicas,

            selector=client.V1LabelSelector(
                match_labels={
                    "app": name
                }
            ),

            template=client.V1PodTemplateSpec(

                metadata=client.V1ObjectMeta(
                    labels={
                        "app": name
                    }
                ),

                spec=client.V1PodSpec(

                    containers=[
                        client.V1Container(
                            name="app",
                            image=image
                        )
                    ]
                )
            )
        )
    )

def reconcile(obj):

    name = obj["metadata"]["name"]

    namespace = obj["metadata"].get(
        "namespace",
        "default"
    )

    spec = obj.get("spec", {})

    replicas = spec.get("replicas", 1)

    image = spec.get(
        "image",
        "nginx:latest"
    )

    print(
        f"[RECONCILE] "
        f"name={name}, "
        f"replicas={replicas}, "
        f"image={image}"
    )

    try:

        deployment = apps_api.read_namespaced_deployment(
            name=name,
            namespace=namespace
        )

        print(
            f"[EXISTS] Deployment {name}"
        )

    except client.exceptions.ApiException as e:

        if e.status == 404:

            deployment = create_deployment(
                name,
                namespace,
                replicas,
                image
            )

            apps_api.create_namespaced_deployment(
                namespace=namespace,
                body=deployment
            )

            print(
                f"[CREATE] Deployment {name}"
            )

        else:

            raise

def main():

    print(
        "AISTudio Controller started"
    )

    w = watch.Watch()

    for event in w.stream(

        custom_api.list_cluster_custom_object,

        group=GROUP,
        version=VERSION,
        plural=PLURAL

    ):

        event_type = event["type"]

        obj = event["object"]

        print(
            f"[EVENT] {event_type}"
        )

        if event_type in [
            "ADDED",
            "MODIFIED"
        ]:

            reconcile(obj)

if __name__ == "__main__":
    main()
```

---

# 17. 이 코드를 단계별로 읽는 방법

초보자라면 전체 코드를 한 번에 이해하려고 하지 않는 것이 좋습니다.

### 첫 번째

```python
config.load_incluster_config()
```

**Kubernetes API에 연결**

↓

### 두 번째

```python
custom_api = client.CustomObjectsApi()
```

**AISTudio CR을 다루기 위한 API**

↓

### 세 번째

```python
w = watch.Watch()
```

**Kubernetes 이벤트 감시**

↓

### 네 번째

```python
for event in w.stream(...):
```

**AISTudio 이벤트 수신**

↓

### 다섯 번째

```python
reconcile(obj)
```

**원하는 상태를 계산**

↓

### 여섯 번째

```python
apps_api.read_namespaced_deployment(...)
```

**현재 Deployment 상태 확인**

↓

### 일곱 번째

```python
apps_api.create_namespaced_deployment(...)
```

**필요하면 Deployment 생성**

이렇게 읽으면 됩니다.

---

# 18. 실제 동작 예

사용자가:

```bash
kubectl apply -f aistudio.yaml
```

실행:

```yaml
apiVersion: example.com/v1
kind: AISTudio

metadata:
  name: my-aistudio

spec:
  replicas: 2
  image: nginx:latest
```

그러면:

### ① API Server

```
AISTudio CR 저장
```

### ② Watch

```
ADDED event
```

### ③ Python

```python
reconcile(obj)
```

### ④ CR 읽기

```
name = my-aistudio
replicas = 2
image = nginx:latest
```

### ⑤ Deployment 확인

```
Deployment 없음
```

### ⑥ Deployment 생성

```
my-aistudio
replicas=2
image=nginx:latest
```

### ⑦ Kubernetes

```
Deployment
    ↓
ReplicaSet
    ↓
Pod
    ↓
Pod
```

---

# 19. Controller 실습에서 가장 중요한 부분

Python Controller를 공부할 때는 아래 5개만 먼저 확실히 이해하면 됩니다.

```
① Watch
   ↓
② Event
   ↓
③ CR 읽기
   ↓
④ Desired State 계산
   ↓
⑤ Actual State를 Desired State로 변경
```

즉:

```
             AISTudio CR
                  │
                  │ Watch
                  ▼
             Python Event
                  │
                  ▼
              reconcile()
                  │
                  ├── CR spec 읽기
                  │
                  ├── Desired State 계산
                  │
                  ├── Deployment 조회
                  │
                  └── CREATE / UPDATE
                         │
                         ▼
                    Kubernetes
```

이것이 **Python으로 구현하는 Kubernetes Controller의 본질**입니다.

---

## 다음 실습에서 추천하는 순서

현재 환경에서는 다음 순서로 Python 코드를 조금씩 발전시키는 것이 가장 좋습니다.

**실습 1**

```
AISTudio CR Watch
→ 이벤트 출력만
```

**실습 2**

```
AISTudio CR
→ spec 읽기
→ 로그 출력
```

**실습 3**

```
AISTudio CR
→ Deployment 생성
```

**실습 4**

```
Deployment 삭제
→ Controller가 다시 생성
```

**실습 5**

```
AISTudio replicas 변경
→ Deployment replicas 변경
```

**실습 6**

```
Controller Pod 삭제
→ 새 Controller 시작
→ 현재 상태 재확인
```

이 순서로 진행하면 **Python 코드 한 줄 한 줄이 Kubernetes Controller의 어떤 역할을 하는지** 직접 확인하면서 배울 수 있습니다.

# 

물론입니다. 지금까지의 **AISTudio Controller 실습에서 RBAC 부분만 분리**하면, 다음과 같은 독립적인 실습 과정으로 정리할 수 있습니다.

## 1. RBAC가 왜 필요한가?

현재 구조는 다음과 같습니다.

```
AISTudio CR
     │
     ▼
Controller Pod
     │
     ▼
Kubernetes API Server
     │
     ├── AISTudio CR 조회
     ├── Deployment 조회
     ├── Deployment 생성
     └── Deployment 수정
```

여기서 중요한 문제가 있습니다.

**Controller Pod가 Kubernetes API Server에 아무 권한이나 가지고 있으면 안 됩니다.**

따라서 Kubernetes에서는:

```
ServiceAccount
      ↓
Role / ClusterRole
      ↓
RoleBinding / ClusterRoleBinding
```

을 이용해서 Controller가 할 수 있는 작업을 제한합니다.

---

# 2. RBAC 전체 구조

이번 실습에서는 다음 구조를 사용합니다.

```
┌──────────────────────────────┐
│       Controller Pod         │
│                              │
│  Python Controller           │
│          │                   │
│          ▼                   │
│  ServiceAccount              │
│  aistudio-controller         │
└──────────────┬───────────────┘
               │
               │ RoleBinding
               ▼
┌──────────────────────────────┐
│            Role              │
│                              │
│ AISTudio CR   get/list/watch │
│ Deployment    get/list/watch │
│ Deployment    create/update  │
│              patch/delete?   │
└──────────────────────────────┘
```

핵심은:

> **Controller가 API Server에 접근할 때 어떤 Kubernetes 리소스에 어떤 동작을 할 수 있는지를 RBAC로 정의한다.**
> 

입니다.

---

# 3. Step 1 — Controller용 Namespace 생성

Controller를 별도의 Namespace에서 실행합니다.

```bash
kubectl create namespace aistudio-system
```

확인:

```bash
kubectl get namespace aistudio-system
```

---

# 4. Step 2 — ServiceAccount 생성

Controller 전용 ServiceAccount를 만듭니다.

`serviceaccount.yaml`

```yaml
apiVersion: v1
kind: ServiceAccount

metadata:
  name: aistudio-controller
  namespace: aistudio-system
```

적용:

```bash
kubectl apply -f serviceaccount.yaml
```

확인:

```bash
kubectl get serviceaccount -n aistudio-system
```

결과:

```
NAME
aistudio-controller
```

---

# 5. ServiceAccount의 역할

Controller Pod에서:

```yaml
spec:
  serviceAccountName: aistudio-controller
```

를 지정합니다.

즉:

```
Controller Pod
      │
      ▼
aistudio-controller
ServiceAccount
```

를 사용하게 됩니다.

Python Controller의:

```python
config.load_incluster_config()
```

는 Pod 내부의 ServiceAccount 인증 정보를 이용해서 Kubernetes API Server에 접근합니다.

---

# 6. Step 3 — Role 작성

이제 Controller에게 필요한 권한을 정의합니다.

`role.yaml`

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role

metadata:
  name: aistudio-controller
  namespace: aistudio-system

rules:

# AISTudio CR 접근
- apiGroups:
  - "example.com"

  resources:
  - aistudios

  verbs:
  - get
  - list
  - watch

# Deployment 접근
- apiGroups:
  - "apps"

  resources:
  - deployments

  verbs:
  - get
  - list
  - watch
  - create
  - update
  - patch
```

---

# 7. `apiGroups` 이해

RBAC에서 초보자가 가장 헷갈리는 부분입니다.

AISTudio CRD가:

```yaml
apiVersion: example.com/v1
```

이면:

```yaml
apiGroups:
- "example.com"
```

입니다.

반면 Deployment는:

```yaml
apiVersion: apps/v1
```

이므로:

```yaml
apiGroups:
- "apps"
```

입니다.

즉:

| Kubernetes 객체 | apiVersion | apiGroups |
| --- | --- | --- |
| AISTudio | `example.com/v1` | `example.com` |
| Deployment | `apps/v1` | `apps` |
| Pod | `v1` | `""` |
| Service | `v1` | `""` |

특히 Core API 리소스인 Pod, Service 등은:

```yaml
apiGroups:
- ""
```

입니다.

---

# 8. `resources` 이해

예를 들어:

```yaml
resources:
- aistudios
```

는 AISTudio CR을 의미합니다.

Deployment는:

```yaml
resources:
- deployments
```

입니다.

중요한 것은 **kind 이름이 아니라 plural resource 이름**을 사용한다는 점입니다.

```
AISTudio
   ↓
aistudios

Deployment
   ↓
deployments
```

확인은:

```bash
kubectl api-resources
```

로 할 수 있습니다.

---

# 9. `verbs` 이해

RBAC의 핵심입니다.

```yaml
verbs:
- get
- list
- watch
```

각각:

| verb | 의미 |
| --- | --- |
| get | 특정 객체 조회 |
| list | 여러 객체 조회 |
| watch | 변경 감시 |
| create | 생성 |
| update | 전체 수정 |
| patch | 부분 수정 |
| delete | 삭제 |

입니다.

---

# 10. Controller에 필요한 권한

AISTudio Controller가 하는 일을 생각해 보면:

```
AISTudio
   │
   ├── 조회
   ├── 목록 조회
   └── 변경 감시

Deployment
   │
   ├── 조회
   ├── 생성
   └── 수정
```

따라서 최소한:

### AISTudio

```yaml
verbs:
- get
- list
- watch
```

### Deployment

```yaml
verbs:
- get
- list
- watch
- create
- update
- patch
```

정도가 필요합니다.

---

# 11. Step 4 — RoleBinding

Role을 만들었다고 해서 ServiceAccount에 자동으로 권한이 부여되는 것은 아닙니다.

둘을 연결해야 합니다.

`rolebinding.yaml`

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding

metadata:
  name: aistudio-controller
  namespace: aistudio-system

subjects:
- kind: ServiceAccount
  name: aistudio-controller
  namespace: aistudio-system

roleRef:
  kind: Role
  name: aistudio-controller
  apiGroup: rbac.authorization.k8s.io
```

적용:

```bash
kubectl apply -f rolebinding.yaml
```

---

# 12. RBAC 전체 연결 확인

이제 구조는:

```
Controller Pod
      │
      ▼
ServiceAccount
aistudio-controller
      │
      │ RoleBinding
      ▼
Role
aistudio-controller
      │
      ├── AISTudio
      │    get/list/watch
      │
      └── Deployment
           get/list/watch/create/update/patch
```

입니다.

---

# 13. Step 5 — Controller Deployment에 ServiceAccount 지정

Controller Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: aistudio-controller
  namespace: aistudio-system

spec:
  replicas: 1

  selector:
    matchLabels:
      app: aistudio-controller

  template:
    metadata:
      labels:
        app: aistudio-controller

    spec:
      serviceAccountName: aistudio-controller

      containers:
      - name: controller
        image: 192.168.56.11:30002/test-crd/aistudio-controller:1.0
```

여기서 핵심:

```yaml
serviceAccountName: aistudio-controller
```

입니다.

---

# 14. Step 6 — Controller Pod가 어떤 ServiceAccount를 사용하는지 확인

```bash
kubectl get pod \
  -n aistudio-system \
  -o jsonpath='{.items[0].spec.serviceAccountName}{"\n"}'
```

결과:

```
aistudio-controller
```

이면 정상입니다.

또는:

```bash
kubectl describe pod \
  -n aistudio-system \
  <controller-pod-name>
```

에서:

```
Service Account:  aistudio-controller
```

를 확인할 수 있습니다.

---

# 15. Step 7 — `kubectl auth can-i`로 권한 테스트

이 부분은 **RBAC 실습에서 가장 중요한 테스트**입니다.

AISTudio 조회:

```bash
kubectl auth can-i get aistudios \
  --as=system:serviceaccount:aistudio-system:aistudio-controller
```

결과:

```
yes
```

AISTudio watch:

```bash
kubectl auth can-i watch aistudios \
  --as=system:serviceaccount:aistudio-system:aistudio-controller
```

결과:

```
yes
```

Deployment 생성:

```bash
kubectl auth can-i create deployments \
  --as=system:serviceaccount:aistudio-system:aistudio-controller
```

결과:

```
yes
```

Deployment 수정:

```bash
kubectl auth can-i update deployments \
  --as=system:serviceaccount:aistudio-system:aistudio-controller
```

결과:

```
yes
```

---

# 16. 권한이 없는 작업 테스트

RBAC를 공부할 때는 **허용되는 작업뿐만 아니라 금지되는 작업도 확인해야 합니다.**

예를 들어 Pod 삭제 권한을 Role에 주지 않았다면:

```bash
kubectl auth can-i delete pods \
  --as=system:serviceaccount:aistudio-system:aistudio-controller
```

결과:

```
no
```

가 나와야 합니다.

이것이 RBAC의 핵심입니다.

```
Controller
    │
    ├── AISTudio get       → YES
    ├── AISTudio watch     → YES
    ├── Deployment create  → YES
    ├── Deployment update  → YES
    │
    └── Pod delete         → NO
```

---

# 17. 아주 중요한 실습 — 권한을 일부러 제거

RBAC를 제대로 이해하려면 권한을 제거해서 Controller가 실패하는 것을 보는 것이 좋습니다.

예를 들어 Role에서:

```yaml
- create
```

를 제거합니다.

그러면 Controller가 Deployment를 생성하려고 할 때 API Server가 거부합니다.

Python Controller에서는 대략:

```
403 Forbidden
```

오류가 발생합니다.

Controller 로그:

```bash
kubectl logs \
  -n aistudio-system \
  deployment/aistudio-controller
```

에서 확인할 수 있습니다.

이것은 매우 좋은 실습입니다.

---

# 18. RBAC 권한을 다시 추가

다시:

```yaml
verbs:
- get
- list
- watch
- create
- update
- patch
```

로 복구합니다.

적용:

```bash
kubectl apply -f role.yaml
```

그리고:

```bash
kubectl auth can-i create deployments \
  --as=system:serviceaccount:aistudio-system:aistudio-controller
```

결과:

```
yes
```

가 되는지 확인합니다.

---

# 19. RBAC와 Python Controller의 관계

이 부분이 특히 중요합니다.

Python 코드에서는:

```python
config.load_incluster_config()

custom_api = client.CustomObjectsApi()
apps_api = client.AppsV1Api()
```

라고만 되어 있습니다.

Python 코드 안에는:

```
username
password
```

가 없습니다.

왜냐하면:

```
Controller Pod
      │
      ▼
ServiceAccount
      │
      ▼
Token
      │
      ▼
Kubernetes API Server
      │
      ▼
RBAC Authorization
```

구조로 인증/인가가 이루어지기 때문입니다.

---

# 20. Authentication과 Authorization 구분

초보자에게 매우 중요한 개념입니다.

### Authentication

> "너 누구야?"
> 

```
ServiceAccount
aistudio-controller
```

### Authorization

> "무엇을 할 수 있어?"
> 

```
AISTudio → get/list/watch
Deployment → get/list/watch/create/update/patch
```

즉:

```
Authentication
       ↓
ServiceAccount
       ↓
Authorization
       ↓
RBAC
       ↓
허용 / 거부
```

입니다.

---

# 21. 현재 실습에서 RBAC 파일 구성

현재 프로젝트를 깔끔하게 정리한다면:

```
aistudio-controller/
│
├── controller.py
│
├── Dockerfile
│
├── serviceaccount.yaml
├── role.yaml
├── rolebinding.yaml
│
├── controller-deployment.yaml
│
├── aistudio-crd.yaml
└── aistudio.yaml
```

역할을 분리하면:

```
aistudio-crd.yaml
    ↓
AISTudio 타입 정의

aistudio.yaml
    ↓
AISTudio 객체

serviceaccount.yaml
    ↓
Controller 신원

role.yaml
    ↓
Controller 권한

rolebinding.yaml
    ↓
신원 ↔ 권한 연결

controller-deployment.yaml
    ↓
Controller Pod 실행

controller.py
    ↓
Reconciliation 수행
```

---

# 22. 이번 RBAC 실습의 핵심 흐름

전체를 한 장으로 정리하면:

```
                 Controller Pod
                       │
                       │
                       ▼
              ServiceAccount
          aistudio-controller
                       │
                       │ RoleBinding
                       ▼
                    Role
                       │
          ┌────────────┴────────────┐
          │                         │
          ▼                         ▼
     AISTudio CR              Deployment
   get/list/watch        get/list/watch/create
                              update/patch
```

그리고 실제 요청은:

```
Python Controller
       │
       │ Kubernetes API 요청
       ▼
API Server
       │
       │ "이 ServiceAccount가
       │  이 작업을 할 수 있는가?"
       ▼
      RBAC
       │
    ┌──┴──┐
   YES    NO
    │      │
    ▼      ▼
실행    403 Forbidden
```

---

# 23. 초보자용 RBAC 실습 순서

현재 AISTudio Controller 실습에서는 RBAC를 다음 순서로 별도 진행하는 것을 추천합니다.

```
① ServiceAccount 생성
        ↓
② Role 생성
        ↓
③ RoleBinding 생성
        ↓
④ Controller Pod에 ServiceAccount 지정
        ↓
⑤ kubectl auth can-i 테스트
        ↓
⑥ 허용 권한 확인
        ↓
⑦ 금지 권한 확인
        ↓
⑧ 일부 권한 제거
        ↓
⑨ Controller 403 오류 확인
        ↓
⑩ 권한 복구
        ↓
⑪ Controller 정상 동작 확인
```

특히 **⑤ → ⑧ → ⑨ 과정**이 RBAC를 이해하는 데 가장 중요합니다.

그리고 현재 실습의 최종 구조에서는:

```
CRD
 +
CR
 +
Python Controller
 +
ServiceAccount
 +
RBAC
       ↓
완전한 Kubernetes Custom Controller
```

가 됩니다.

즉 **CRD는 "새로운 리소스 타입"을 만들고, Python Controller는 "그 리소스를 어떻게 동작시킬지"를 구현하며, RBAC는 "Controller가 Kubernetes에서 어디까지 조작할 수 있는지"를 제한하는 역할**을 합니다.
