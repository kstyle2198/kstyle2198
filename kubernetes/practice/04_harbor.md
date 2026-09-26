# [실습] Harbor 설치/실행

# 기본 설치 실행

네. 지금까지 진행한 내용을 기준으로 **`control-plane + worker-1` Kubernetes 클러스터에서 Harbor 설치 → 실행 → 이미지 Push 테스트**까지의 전체 과정을 순서대로 재정리하면 다음과 같습니다.

# Harbor 설치 및 이미지 Push 실습 전체 순서

## 1. Kubernetes 클러스터 확인

**control-plane에서**

```bash
kubectl get nodes -o wide
```

정상적인 상태:

```
NAME            STATUS   ROLES           AGE   VERSION
control-plane   Ready    control-plane   ...
worker-1        Ready    <none>          ...
```

두 노드 모두 `Ready`인지 확인합니다.

---

## 2. worker-1에 Harbor 전용 Label 지정

Harbor의 모든 Pod가 `worker-1`에서 실행되도록 Label을 추가합니다.

```bash
kubectl label node worker-1 harbor=true
```

확인:

```bash
kubectl get node worker-1 --show-labels
```

다음 Label이 있어야 합니다.

```
harbor=true
```

---

## 3. Harbor Namespace 생성

```bash
kubectl create namespace harbor
```

확인:

```bash
kubectl get ns harbor
```

결과:

```
NAME     STATUS
harbor   Active
```

---

## 4. Helm 설치 확인

```bash
helm version
```

Helm이 정상적으로 설치되어 있어야 합니다.

---

## 5. Harbor Helm Repository 등록

```bash
helm repo add harbor https://helm.goharbor.io
```

Repository 확인:

```bash
helm repo list
```

예:

```
NAME    URL
harbor  https://helm.goharbor.io
```

Repository 업데이트:

```bash
helm repo update
```

---

## 6. Harbor Chart 확인

```bash
helm search repo harbor/harbor
```

기본 설정을 확인할 수도 있습니다.

```bash
helm show values harbor/harbor > harbor-values-default.yaml
```

---

# 7. Harbor 설치 디렉터리 생성

```bash
mkdir -p ~/harbor
cd ~/harbor
```

이 디렉터리에서 Harbor 설정을 관리합니다.

구조:

```
~/harbor/
├── values.yaml
└── harbor-values-default.yaml
```

---

# 8. StorageClass 확인

Harbor는 Registry 이미지와 DB 등의 데이터를 저장해야 하므로 Storage가 필요합니다.

```bash
kubectl get storageclass
```

예:

```
NAME                   PROVISIONER
local-path             rancher.io/local-path
```

여기서 사용하는 StorageClass가 정상적으로 동작해야 합니다.

PVC 상태도 나중에 반드시 확인해야 합니다.

---

# 9. Harbor `values.yaml` 작성

실습 초기에는 **HTTP + NodePort** 방식으로 구성했습니다.

```yaml
expose:
  type: nodePort

  tls:
    enabled: false

  nodePort:
    ports:
      http:
        nodePort: 30002
      https:
        nodePort: 30003

externalURL: http://192.168.56.11:30002

harborAdminPassword: "Harbor12345"

persistence:
  enabled: true

  persistentVolumeClaim:
    registry:
      storageClass: ""

    jobservice:
      jobLog:
        storageClass: ""

    database:
      storageClass: ""

    redis:
      storageClass: ""

    trivy:
      storageClass: ""

nodeSelector:
  harbor: "true"
```

여기서 중요한 부분은:

```yaml
nodeSelector:
  harbor: "true"
```

입니다.

앞에서 worker-1에:

```bash
kubectl label node worker-1 harbor=true
```

를 지정했기 때문에 Harbor Pod가 worker-1에 배치됩니다.

---

# 10. Harbor 설치

```bash
helm install harbor harbor/harbor \
  -n harbor \
  -f values.yaml
```

설치 결과 확인:

```bash
helm list -n harbor
```

정상:

```
NAME    NAMESPACE   STATUS
harbor  harbor      deployed
```

---

# 11. Harbor Pod 실행 확인

```bash
kubectl get pods -n harbor -o wide
```

중요한 것은 `STATUS`와 `NODE`입니다.

예:

```
NAME                       READY   STATUS    NODE
harbor-core-xxxx           1/1     Running   worker-1
harbor-portal-xxxx         1/1     Running   worker-1
harbor-registry-xxxx       2/2     Running   worker-1
harbor-database-0          1/1     Running   worker-1
harbor-redis-0             1/1     Running   worker-1
harbor-jobservice-xxxx     1/1     Running   worker-1
harbor-trivy-0             1/1     Running   worker-1
```

즉 Harbor 관련 Pod가 모두 `worker-1`에 배치되었는지 확인합니다.

---

# 12. Harbor PVC 확인

```bash
kubectl get pvc -n harbor
```

정상적으로 Storage가 연결되었다면:

```
NAME                         STATUS
harbor-registry              Bound
harbor-database              Bound
harbor-redis                 Bound
harbor-jobservice            Bound
harbor-trivy                 Bound
```

핵심은:

```
STATUS = Bound
```

입니다.

`Pending`이면 StorageClass/PV 문제를 먼저 해결해야 합니다.

---

# 13. Harbor Service 확인

```bash
kubectl get svc -n harbor
```

또는:

```bash
kubectl get svc -n harbor -o wide
```

Harbor Service가 정상적으로 생성되었는지 확인합니다.

NodePort도 확인합니다.

```bash
kubectl get svc -n harbor harbor
```

---

# 14. Harbor 웹 UI 접속

worker-1의 IP가 예를 들어:

```
192.168.56.11
```

이고 NodePort가:

```
30002
```

이면 브라우저에서:

```
http://192.168.56.11:30002
```

로 접속합니다.

Harbor 로그인 계정:

```
Username: admin
Password: Harbor12345
```

---

# 15. Harbor 프로젝트 생성

Harbor Web UI에서 프로젝트를 하나 생성합니다.

예:

```
Project Name: test
```

그러면 Docker 이미지 경로는 다음 형태가 됩니다.

```
192.168.56.11:30002/test/nginx:latest
```

구조를 보면:

```
Harbor
└── test
    └── nginx
        └── latest
```

입니다.

---

# 16. 테스트 Docker 이미지 다운로드

Docker 클라이언트에서:

```bash
docker pull nginx:latest
```

확인:

```bash
docker images
```

예:

```
REPOSITORY   TAG       IMAGE ID
nginx        latest    xxxxx
```

---

# 17. Harbor Registry 주소로 이미지 Tag 변경

기존:

```
nginx:latest
```

를 Harbor Registry 주소를 포함하도록 변경합니다.

```bash
docker tag nginx:latest \
  192.168.56.11:30002/test/nginx:latest
```

확인:

```bash
docker images
```

예:

```
REPOSITORY                           TAG
nginx                                latest
192.168.56.11:30002/test/nginx      latest
```

---

# 18. HTTP Registry를 위한 Docker 설정

현재 Harbor는 TLS를 사용하지 않는 HTTP 방식이므로 Docker에서 insecure registry로 허용해야 합니다.

Docker 클라이언트에서:

```bash
sudo nano /etc/docker/daemon.json
```

다음 내용을 설정합니다.

```json
{
  "insecure-registries": [
    "192.168.56.11:30002"
  ]
}
```

Docker 재시작:

```bash
sudo systemctl restart docker
```

확인:

```bash
docker info | grep -A5 "Insecure Registries"
```

---

# 19. Harbor 로그인

```bash
docker login 192.168.56.11:30002
```

입력:

```
Username: admin
Password: Harbor12345
```

성공하면:

```
Login Succeeded
```

가 출력됩니다.

---

# 20. Harbor에 이미지 Push

이제 실제 Push 테스트입니다.

```bash
docker push \
  192.168.56.11:30002/test/nginx:latest
```

정상적으로 진행되면:

```
latest: digest: sha256:...
```

등의 메시지가 출력됩니다.

이 단계가 성공하면 **Harbor Registry가 실제 Docker 이미지 저장소로 정상 동작**하는 것입니다.

---

# 21. Harbor Web UI에서 이미지 확인

Harbor Web UI:

```
Projects
  ↓
test
  ↓
Repositories
  ↓
nginx
  ↓
latest
```

에서 Push한 이미지를 확인합니다.

즉,

```
Docker Client
     │
     │ docker push
     ▼
192.168.56.11:30002
     │
     ▼
Harbor
     │
     ▼
test/nginx:latest
```

구조입니다.

---

# 22. 최종적으로 확인해야 할 상태

지금까지 실습이 정상적으로 완료되었다면 다음 명령들이 모두 정상이어야 합니다.

### Kubernetes

```bash
kubectl get nodes
```

```
control-plane   Ready
worker-1        Ready
```

### Harbor Pod

```bash
kubectl get pods -n harbor -o wide
```

```
STATUS = Running
NODE   = worker-1
```

### Harbor PVC

```bash
kubectl get pvc -n harbor
```

```
STATUS = Bound
```

### Harbor Helm

```bash
helm list -n harbor
```

```
STATUS = deployed
```

### Docker Login

```bash
docker login 192.168.56.11:30002
```

```
Login Succeeded
```

### Docker Push

```bash
docker push 192.168.56.11:30002/test/nginx:latest
```

```
Successfully pushed
```

---

# 전체 실습 흐름 요약

```
┌──────────────────────────────┐
│ 1. Kubernetes Cluster 확인   │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 2. worker-1 Label            │
│    harbor=true                │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 3. harbor Namespace 생성     │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 4. Helm Repository 등록      │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 5. StorageClass 확인         │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 6. values.yaml 작성          │
│    NodeSelector=worker-1     │
│    NodePort=30002            │
│    TLS=false                 │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 7. helm install              │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 8. Harbor Pod 확인           │
│    worker-1 / Running        │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 9. PVC / Service 확인        │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 10. Harbor Web UI 접속       │
│     :30002                   │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 11. test 프로젝트 생성       │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 12. nginx 이미지 Pull        │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 13. docker tag               │
│     → Harbor 주소            │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 14. docker login             │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 15. docker push              │
│     nginx:latest             │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 16. Harbor UI에서 이미지 확인│
└──────────────────────────────┘
```

## 다음 단계

현재까지의 **설치 + 실행 + Push 테스트**가 완료된 상태라면 다음 단계는 단순히 Harbor를 더 설치하는 것이 아니라, 지금 구성한 Harbor를 실제 운영에 가까운 형태로 발전시키는 것입니다.

권장 순서는:

**① HTTP → HTTPS 전환 → ② Ingress 연결 → ③ cert-manager 인증서 적용 → ④ `harbor.example.com` 형태의 도메인 구성 → ⑤ OpenLDAP 연동 → ⑥ LDAP 사용자 로그인 테스트 → ⑦ LDAP 그룹/프로젝트 권한 설정 → ⑧ GitLab CI/CD에서 Harbor로 이미지 Push**

입니다.

특히 앞서 구성하셨던 **OpenLDAP + Harbor 연동**까지 이어가려면, 다음에는 **현재 Harbor 구성은 유지하면서 HTTPS/Ingress를 먼저 구성한 뒤 LDAP를 연결하는 순서**가 가장 깔끔합니다.

# 

## 1. 실습 목표

Kubernetes 환경에서 실행 중인 **Harbor와 OpenLDAP을 연동**하여 LDAP 계정으로 Harbor에 로그인하고, 이후 LDAP 그룹을 이용해 Harbor 권한을 관리한다.

```
사용자
  │
  │ ID / Password
  ▼
Harbor
  │
  │ LDAP 인증
  ▼
OpenLDAP
  │
  ├── ou=users
  │     ├── jongkim
  │     └── honglee
  │
  └── ou=groups
        └── harbor-admins
```

---

# 2. 현재 환경 확인

## 2.1 LDAP Pod 확인

```bash
kubectl get pods -n ldap
```

정상 예:

```
NAME                         READY   STATUS
openldap-xxxxxxxxxx          1/1     Running
```

## 2.2 LDAP Service 확인

```bash
kubectl get svc -n ldap
```

예:

```
NAME       TYPE        CLUSTER-IP      PORT
openldap   ClusterIP   10.xxx.xxx.xxx  389
```

---

# 3. LDAP 사용자 확인

LDAP 사용자 검색:

```bash
kubectl exec -it -n ldap deploy/openldap -- \
  ldapsearch -x \
  -H ldap://openldap:389 \
  -D "cn=admin,dc=hdaic,dc=com" \
  -W \
  -b "ou=users,dc=hdaic,dc=com"
```

다음과 같이 사용자가 존재하는지 확인한다.

```
dn: uid=jongkim,ou=users,dc=hdaic,dc=com
uid: jongkim
```

---

# 4. LDAP 사용자 인증 테스트

`jongkim` 계정으로 직접 LDAP 인증을 테스트한다.

```bash
kubectl exec -it -n ldap deploy/openldap -- \
  ldapwhoami -x \
  -H ldap://openldap:389 \
  -D "uid=jongkim,ou=users,dc=hdaic,dc=com" \
  -W
```

성공하면:

```
dn:uid=jongkim,ou=users,dc=hdaic,dc=com
```

> 이 단계가 실패하면 Harbor 설정 전에 LDAP 계정 또는 비밀번호 문제를 먼저 해결한다.
> 

---

# 5. Harbor 상태 확인

Harbor Namespace 확인:

```bash
kubectl get pods -n harbor-jongkim
```

모든 주요 Pod가 `Running` 또는 정상 상태인지 확인한다.

Helm Release 확인:

```bash
helm list -n harbor-jongkim
```

예:

```
NAME      NAMESPACE        STATUS
harbor    harbor-jongkim   deployed
```

---

# 6. Harbor에서 LDAP 서버 주소 확인

Harbor와 LDAP가 서로 다른 Namespace에 있으므로 Kubernetes Service DNS를 사용한다.

LDAP Service:

```
openldap
```

Namespace:

```
ldap
```

따라서 Harbor에서 LDAP 주소는:

```
openldap.ldap.svc.cluster.local
```

LDAP 포트:

```
389
```

최종 LDAP URL:

```
ldap://openldap.ldap.svc.cluster.local:389
```

---

# 7. Harbor LDAP 설정

Harbor 웹 UI에 관리자 계정으로 로그인한다.

메뉴:

```
Administration
    ↓
Configuration
    ↓
Authentication
```

Authentication Type을:

```
Database
```

에서:

```
LDAP
```

로 변경한다.

---

# 8. LDAP 설정값 입력

현재 LDAP 구조를 기준으로 다음과 같이 입력한다.

| Harbor 설정 | 값 |
| --- | --- |
| Authentication Type | LDAP |
| LDAP URL | `ldap://openldap.ldap.svc.cluster.local:389` |
| LDAP Search DN | `cn=admin,dc=hdaic,dc=com` |
| LDAP Search Password | LDAP admin 비밀번호 |
| LDAP Base DN | `dc=hdaic,dc=com` |
| LDAP User Search Base | `ou=users,dc=hdaic,dc=com` |
| LDAP UID | `uid` |

핵심 관계:

```
LDAP Base DN
    dc=hdaic,dc=com

       ↓

User Search Base
    ou=users,dc=hdaic,dc=com

       ↓

UID
    uid

       ↓

사용자
    uid=jongkim
```

---

# 9. LDAP 연결 테스트

Harbor 설정 화면에서:

```
Verify LDAP Connection
```

을 실행한다.

### 성공

Harbor가 LDAP 서버에 정상적으로 연결된 것이다.

### 실패

다음 항목을 우선 확인한다.

```
① LDAP URL
② LDAP Port
③ Search DN
④ Search Password
⑤ Base DN
⑥ User Search Base
⑦ UID
```

---

# 10. LDAP 계정으로 Harbor 로그인

Harbor 로그인 화면에서 LDAP 계정을 입력한다.

```
Username: jongkim
Password: <LDAP jongkim 비밀번호>
```

정상적으로 로그인되면:

```
LDAP
 ↓
jongkim 인증
 ↓
Harbor 로그인
```

이 과정이 성공한 것이다.

---

# 11. Harbor에서 사용자 확인

Harbor 관리자 화면:

```
Administration
    ↓
Users
```

LDAP 사용자 `jongkim`이 정상적으로 확인되는지 확인한다.

---

# 12. LDAP 그룹 구성

LDAP에 그룹을 추가한다.

예:

```
ou=groups,dc=hdaic,dc=com
```

그룹:

```
cn=harbor-admins
```

구성 예:

```
ou=groups
    │
    ├── harbor-admins
    │       └── jongkim
    │
    ├── harbor-developers
    │       └── honglee
    │
    └── harbor-readonly
            └── ...
```

---

# 13. Harbor 프로젝트 권한 테스트

예를 들어 Harbor Project를 생성한다.

```
test
```

LDAP 사용자/그룹에 역할을 부여한다.

| LDAP 그룹 | Harbor 역할 | 권한 |
| --- | --- | --- |
| `harbor-admins` | Admin | 전체 관리 |
| `harbor-developers` | Developer | 이미지 Push/Pull |
| `harbor-readonly` | Guest | 이미지 Pull 중심 |

---

# 14. Docker Push 테스트

LDAP 인증이 성공한 후 실제 이미지 Push까지 테스트한다.

로그인:

```bash
docker login <Harbor주소>
```

예:

```bash
docker login 192.168.56.11:30002
```

이미지 생성:

```bash
docker pull nginx:latest
```

Tag:

```bash
docker tag nginx:latest \
  192.168.56.11:30002/test/nginx:latest
```

Push:

```bash
docker push \
  192.168.56.11:30002/test/nginx:latest
```

정상적으로 Push되면:

```
LDAP 인증
   ↓
Harbor 로그인
   ↓
Project 권한 확인
   ↓
Docker Registry 인증
   ↓
Image Push
```

전체 과정이 정상적으로 동작한 것이다.

---

# 15. 전체 실습 순서 요약

```
[1] LDAP 실행 확인
       ↓
[2] LDAP 사용자 확인
       ↓
[3] LDAP 사용자 Bind 테스트
       ↓
[4] Harbor 실행 확인
       ↓
[5] Harbor → LDAP Service 주소 확인
       ↓
[6] Harbor Authentication = LDAP
       ↓
[7] LDAP 설정값 입력
       ↓
[8] Verify LDAP Connection
       ↓
[9] jongkim LDAP 로그인
       ↓
[10] Harbor 사용자 확인
       ↓
[11] LDAP Group 구성
       ↓
[12] Harbor Project 권한 설정
       ↓
[13] docker login
       ↓
[14] docker push 테스트
```

## 핵심 설정값

```
LDAP Server
    openldap.ldap.svc.cluster.local:389

Base DN
    dc=hdaic,dc=com

User Base
    ou=users,dc=hdaic,dc=com

User Attribute
    uid

Admin Bind DN
    cn=admin,dc=hdaic,dc=com

Test User
    jongkim
```

**실습에서는 ① LDAP 사용자 인증 → ② Harbor LDAP 로그인 → ③ LDAP 그룹 → ④ Harbor 프로젝트 권한 → ⑤ Docker Push** 순서로 진행하면 문제 발생 지점을 쉽게 구분할 수 있습니다.
