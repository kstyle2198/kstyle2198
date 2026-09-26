# Volume

쿠버네티스에서 볼륨은 컨테이너가 데이터를 저장하거나 공유할 수 있는 파일 시스템 

컨테이너 내부는 휘발성이기 때문에 이를 해결하기 위해 볼륨 사용 

# StorageClass, PVC, PV, Pod 관계

Kubernetes에서 **StorageClass → PVC → PV** 관계는 처음 보면 헷갈리지만, **"저장공간을 주문하고(PVC), 실제 저장공간을 제공받는(PV)"** 구조로 이해하면 쉽습니다.

## 1. 먼저 비유로 이해하기

아파트 주차장으로 비유하면:

| Kubernetes | 비유 | 역할 |
| --- | --- | --- |
| **StorageClass** | 주차장 상품/등급 | 어떤 방식의 저장공간을 사용할지 정의 |
| **PVC** | 주차 공간 신청서 | "100GB짜리 공간이 필요합니다"라고 요청 |
| **PV** | 실제 주차 공간 | 실제로 할당된 저장공간 |
| **Pod** | 자동차 | 실제 저장공간을 사용하는 주체 |

관계는 대략 이렇게 됩니다.

```
              StorageClass
             "어떤 저장소를?"
                   │
                   ▼
              PVC
        "10Gi 필요합니다"
                   │
                   ▼
                PV
          "10Gi 할당했습니다"
                   │
                   ▼
                 Pod
           "이 공간을 사용"
```

---

# 2. PV란?

**PV(PersistentVolume)**는 Kubernetes 클러스터에서 사용할 수 있는 **실제 저장공간**입니다.

예를 들어:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 10Gi

  accessModes:
    - ReadWriteOnce

  hostPath:
    path: /data/myapp
```

이렇게 하면 Kubernetes 입장에서는:

> "10Gi 크기의 저장공간이 하나 존재한다."
> 

라고 인식합니다.

```
PV
┌─────────────────────┐
│ 이름: my-pv         │
│ 크기: 10Gi          │
│ Access: RWO         │
│ 실제 위치: /data/...│
└─────────────────────┘
```

다만 `hostPath`는 테스트/실습에서는 편하지만, 실제 운영 환경에서는 보통 NFS, Ceph, 클라우드 디스크, SAN 등의 스토리지를 사용합니다.

---

# 3. PVC란?

**PVC(PersistentVolumeClaim)**는 Pod가 직접 저장공간을 지정하는 것이 아니라,

> "나에게 이 정도 저장공간을 주세요."
> 

라고 **저장공간을 요청하는 객체**입니다.

예:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce

  resources:
    requests:
      storage: 10Gi
```

여기서 중요한 것은:

```yaml
resources:
  requests:
    storage: 10Gi
```

입니다.

즉,

> **"10Gi짜리 저장공간이 필요합니다."**
> 

라는 의미입니다.

---

# 4. Pod는 PVC를 사용한다

Pod에서는 PV를 직접 지정하지 않습니다.

**PVC를 지정합니다.**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
    - name: app
      image: nginx

      volumeMounts:
        - mountPath: /data
          name: storage

  volumes:
    - name: storage
      persistentVolumeClaim:
        claimName: my-pvc
```

그러면 관계는:

```
Pod
 │
 │ 사용
 ▼
PVC
 │
 │ 연결
 ▼
PV
 │
 │ 실제 저장공간
 ▼
Storage
```

입니다.

---

# 5. 그런데 StorageClass는 왜 필요한가?

여기서 StorageClass가 등장합니다.

PV를 직접 만드는 방식은 다음과 같습니다.

```
관리자
  │
  │ PV 10Gi 생성
  ▼
PV
  │
  ▼
PVC
  │
  ▼
Pod
```

그런데 실제 운영환경에서는 사용자가 PVC를 만들 때마다 관리자가 PV를 직접 만들어주는 것은 번거롭습니다.

그래서 **StorageClass + Dynamic Provisioning**을 사용합니다.

---

# 6. StorageClass의 핵심

StorageClass는 쉽게 말하면:

> **"이 종류의 저장공간이 필요하면 이렇게 만들어라."**
> 

라는 **저장소 생성 정책**입니다.

예:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: my-storage
provisioner: ...
```

여기서 중요한 것이 `provisioner`입니다.

예를 들어 환경에 따라:

```
StorageClass
      │
      ├── AWS EBS
      ├── GCP Persistent Disk
      ├── Azure Disk
      ├── NFS
      ├── Ceph
      └── Local Storage
```

등과 연결될 수 있습니다.

---

# 7. StorageClass를 사용한 PVC

예를 들어 다음 PVC를 만든다고 해보겠습니다.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  storageClassName: my-storage

  accessModes:
    - ReadWriteOnce

  resources:
    requests:
      storage: 10Gi
```

여기서:

```yaml
storageClassName: my-storage
```

가 핵심입니다.

Kubernetes에게:

> "my-storage라는 StorageClass를 이용해서 10Gi 저장공간을 만들어주세요."
> 

라고 요청하는 것입니다.

---

# 8. Dynamic Provisioning

이제 Kubernetes가 자동으로 PV를 만들어 줍니다.

전체 과정은:

```
① StorageClass 생성
          │
          ▼
② 사용자가 PVC 생성
   "10Gi 필요합니다"
          │
          ▼
③ StorageClass 확인
   "my-storage를 사용해야 하는구나"
          │
          ▼
④ Provisioner가 실제 저장소 생성
          │
          ▼
⑤ PV 자동 생성
          │
          ▼
⑥ PVC ↔ PV 연결
          │
          ▼
⑦ Pod가 PVC 사용
```

즉:

```
                    StorageClass
                         │
                    생성 정책
                         │
                         ▼
PVC ────────────────► PV
 │                    │
 │                    │
 └──────── Pod ◄──────┘
```

---

# 9. 가장 중요한 관계

세 가지를 한 문장씩 기억하면 됩니다.

### StorageClass

> **"어떤 방식으로 저장공간을 만들 것인가?"**
> 

### PVC

> **"저장공간이 이만큼 필요하다."**
> 

### PV

> **"실제로 이 저장공간을 할당했다."**
> 

따라서:

```
StorageClass
   ↓
저장공간 생성 방법

PVC
   ↓
저장공간 요청

PV
   ↓
실제 저장공간
```

입니다.

---

# 10. 실제 Kubernetes에서는 이렇게 사용

예를 들어 사용자가 다음 PVC를 만들었다고 해봅시다.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: database-pvc
spec:
  storageClassName: standard
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
```

그러면:

```
database-pvc
     │
     │ storageClassName: standard
     ▼
 StorageClass
   "standard"
     │
     │ Dynamic Provisioning
     ▼
    PV
   20Gi
     │
     ▼
 database Pod
```

가 됩니다.

---

# 11. Kubernetes에서 직접 확인해 보면

현재 클러스터에서 다음 명령을 실행하면 됩니다.

### StorageClass 확인

```bash
kubectl get storageclass
```

또는:

```bash
kubectl get sc
```

예:

```
NAME                 PROVISIONER
standard (default)   ...
```

---

### PV 확인

```bash
kubectl get pv
```

예:

```
NAME       CAPACITY   ACCESS MODES   STATUS   CLAIM
pvc-xxx    20Gi       RWO            Bound    default/database-pvc
```

여기서:

```
STATUS = Bound
```

이면 PVC와 PV가 연결된 상태입니다.

---

### PVC 확인

```bash
kubectl get pvc
```

예:

```
NAME           STATUS   VOLUME    CAPACITY
database-pvc   Bound    pvc-xxx   20Gi
```

즉:

```
PVC database-pvc
       │
       │ Bound
       ▼
PV pvc-xxx
       │
       ▼
20Gi 저장공간
```

입니다.

---

# 12. 특히 헷갈리는 부분: PVC와 PV는 1:1 관계

일반적으로 하나의 PVC는 하나의 PV에 연결됩니다.

```
PVC-1 ───── PV-1
PVC-2 ───── PV-2
PVC-3 ───── PV-3
```

그리고 여러 Pod가 **하나의 PVC를 공유할 수 있는지**는 `accessModes`와 스토리지 구현에 따라 결정됩니다.

대표적으로:

| Access Mode | 의미 |
| --- | --- |
| RWO | ReadWriteOnce |
| ROX | ReadOnlyMany |
| RWX | ReadWriteMany |

예를 들어:

```yaml
accessModes:
  - ReadWriteOnce
```

이면 일반적으로 하나의 노드에서 읽기/쓰기가 가능한 형태입니다.

---

# 13. 지금 사용하시는 Auto-Healer 환경으로 생각하면

예전에 구성하신 Auto-Healer에서 이벤트를 저장한다고 하면:

```
                StorageClass
                     │
                     │
                     ▼
                auto-heal-pvc
                     │
                     ▼
              Persistent Volume
                     │
                     ▼
             /events/events.json
                     ▲
                     │
                healer Pod
```

예를 들어:

```yaml
volumes:
  - name: events
    persistentVolumeClaim:
      claimName: events-pvc
```

그리고:

```yaml
volumeMounts:
  - name: events
    mountPath: /events
```

라고 하면 애플리케이션에서는 그냥:

```
/events/events.json
```

에 파일을 저장합니다.

**실제 디스크가 어디 있는지는 Pod 입장에서는 신경 쓸 필요가 없습니다.**

이게 Kubernetes Storage의 중요한 장점입니다.

---

## 14. 최종적으로 이렇게 기억하세요

가장 쉽게 그림 하나로 정리하면:

```
┌─────────────────────────┐
│      StorageClass       │
│                         │
│ "어떤 저장소를 사용할까?" │
└────────────┬────────────┘
             │
             │ Dynamic Provisioning
             ▼
┌─────────────────────────┐
│          PV             │
│                         │
│  실제 저장공간 100Gi     │
└────────────┬────────────┘
             │
             │ Binding
             ▼
┌─────────────────────────┐
│          PVC            │
│                         │
│  "100Gi 필요합니다"      │
└────────────┬────────────┘
             │
             │ Mount
             ▼
┌─────────────────────────┐
│          Pod            │
│                         │
│   /data                  │
└─────────────────────────┘
```

다만 **실제 생성 순서와 논리적 관계는 조금 다르게 이해하는 것이 좋습니다.** 사용자가 먼저 **PVC를 생성**하면 PVC가 StorageClass를 지정하고, StorageClass의 provisioner가 **PV를 동적으로 생성**하여 PVC와 Binding하는 것이 일반적인 흐름입니다.

**핵심 한 줄:**

> **StorageClass = 저장소 만드는 방법, PVC = 저장소 신청서, PV = 실제 저장공간, Pod = 그 공간을 사용하는 애플리케이션**입니다.
> 

# 샘플 코드

Kubernetes의 **Volume**은 Pod의 컨테이너가 데이터를 저장하거나 공유하기 위한 공간입니다.

가장 기본적인 `emptyDir` 예제로 보면 쉽습니다.

### 1. `emptyDir` Volume

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-pod

spec:
  containers:
    - name: nginx
      image: nginx

      volumeMounts:
        - name: data-volume
          mountPath: /data

  volumes:
    - name: data-volume
      emptyDir: {}
```

구조는:

```
Pod
│
├── Container
│     └── /data
│
└── Volume
      └── data-volume
```

컨테이너에서:

```bash
kubectl exec -it volume-pod -- bash
```

그리고:

```bash
echo "hello" > /data/test.txt
cat /data/test.txt
```

하면 `/data/test.txt`가 Volume에 저장됩니다.

### 2. `volumeMounts`와 `volumes` 관계

가장 중요한 부분입니다.

```yaml
volumeMounts:
  - name: data-volume
    mountPath: /data
```

→ **컨테이너의 어느 경로에 Volume을 연결할지**

```yaml
volumes:
  - name: data-volume
    emptyDir: {}
```

→ **어떤 Volume을 사용할지**

즉:

```
volumes
   ↓
data-volume
   ↓
volumeMounts
   ↓
/data
   ↓
Container
```

### 3. `emptyDir`의 특징

`emptyDir`은 **Pod가 생성될 때 빈 공간이 만들어지고, Pod가 삭제되면 데이터도 사라집니다.**

```
Pod 생성
   ↓
emptyDir 생성
   ↓
데이터 저장
   ↓
Pod 삭제
   ↓
데이터 삭제
```

따라서 **임시 데이터, 캐시, 컨테이너 간 파일 공유** 등에 적합합니다.

반대로 DB 데이터처럼 **Pod가 삭제되어도 데이터를 유지해야 한다면 `PersistentVolume(PV) + PersistentVolumeClaim(PVC)`를 사용**합니다.

```
emptyDir
→ 임시 저장

PV + PVC
→ 영구 저장
```

지금 Kubernetes 학습 순서에서는 **`emptyDir → hostPath → PV/PVC`** 순서로 이해하면 좋습니다.

# StorageClass 샘플 코드

네. `persistentVolumeReclaimPolicy`까지 포함해서 **StorageClass → PVC → PV → Pod** 관계를 한 번에 볼 수 있는 예제로 구성하면 다음과 같습니다.

### 1. StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass

metadata:
  name: my-storage

provisioner: rancher.io/local-path

reclaimPolicy: Retain
# Retain: 데이터를 보존 (관리자가 수동으로 정리)
# Recycle: 데이터를 삭제하고 PV를 다시 사용 가능 
# Delete: PVC 삭제시 PV와 데이터를 모두 삭제 
```

여기서:

```yaml
reclaimPolicy: Retain
```

은 **PVC가 삭제되어도 PV와 실제 데이터는 보존**하겠다는 의미입니다.

---

### 2. PVC

```yaml
apiVersion: v1
kind: PersistentVolumeClaim

metadata:
  name: my-pvc

spec:
  storageClassName: my-storage

  accessModes:
    - ReadWriteOnce

  resources:
    requests:
      storage: 1Gi
```

---

### 3. Pod

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: my-pod

spec:
  containers:
    - name: nginx
      image: nginx

      volumeMounts:
        - name: data
          mountPath: /data

  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: my-pvc
```

### 전체 관계

```
StorageClass
my-storage
    │
    │ Dynamic Provisioning
    ↓
   PV
    │
    │ Bound
    ↓
   PVC
my-pvc
    │
    │ mount
    ↓
   Pod
    │
    ↓
 /data
```

`Retain`까지 포함하면:

```
Pod 삭제
   ↓
PVC 삭제
   ↓
PV 유지
   ↓
데이터 유지
```

### ReclaimPolicy 주요 3가지

| 정책 | PVC 삭제 후 PV | 데이터 |
| --- | --- | --- |
| `Delete` | 삭제 | 삭제될 수 있음 |
| `Retain` | 유지 | 유지 |
| `Recycle` | 현재 일반적으로 사용하지 않음 | 삭제 후 재사용 |

실습에서는 **`Retain`을 사용하면 데이터 보존 동작을 확인하기 좋습니다.**

> 참고로 동적 프로비저닝에서는 `StorageClass.reclaimPolicy`가 생성된 PV의 기본 reclaim policy가 됩니다. 즉, 위 예제에서는 PVC 요청으로 만들어진 PV가 `Retain` 정책을 갖게 됩니다.
> 

# ConfigMap과 Secret를 Volume으로 마운트

네. **`configmap.yaml`과 `secret.yaml`을 별도 파일로 만들고, Pod YAML에서 이 둘을 Volume으로 마운트**하는 형태로 작성하면 다음과 같습니다.

### 1. `configmap.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  app.conf: 
    PORT=8080
    LOG_LEVEL=INFO
```

---

### 2. `secret.yaml`

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
stringData:
  username: admin
  password: mypassword
```

---

### 3. `pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-secret-pod

spec:
  containers:
    - name: nginx
      image: nginx

      volumeMounts:
        - name: config-volume
          mountPath: /etc/app/config

        - name: secret-volume
          mountPath: /etc/app/secret

  volumes:
    - name: config-volume
      configMap:
        name: app-config

    - name: secret-volume
      secret:
        secretName: app-secret
```

### 4. 한 번에 적용

```bash
kubectl apply -f configmap.yaml
kubectl apply -f secret.yaml
kubectl apply -f pod.yaml
```

또는 파일을 한 디렉터리에 넣었다면:

```bash
kubectl apply -f .
```

### 5. 확인

```bash
kubectl get configmap
kubectl get secret
kubectl get pod
```

Pod 내부 확인:

```bash
kubectl exec -it config-secret-pod -- bash
```

ConfigMap:

```bash
cat /etc/app/config/app.conf
```

결과:

```
PORT=8080
LOG_LEVEL=INFO
```

Secret:

```bash
cat /etc/app/secret/username
cat /etc/app/secret/password
```

결과:

```
admin
mypassword
```

전체 구조는 간단히:

```
configmap.yaml
      │
      ↓
 ConfigMap
      │
      ↓
   Volume
      │
      ↓
/etc/app/config/app.conf

secret.yaml
      │
      ↓
   Secret
      │
      ↓
   Volume
      │
      ↓
/etc/app/secret/
 ├── username
 └── password
```
