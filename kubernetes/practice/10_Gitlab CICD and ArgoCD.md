# [실습] Gitlab CI/CD & Argo CD

# 기본 실습

# GitLab CI/CD + Kubernetes Auto-Healer 실습 정리

## 1. 실습 목표

기존에 Kubernetes에서 동작하던 **Auto-Healer 서비스**를 GitLab과 연결하여 다음 자동화 구조를 만드는 것이 목표였습니다.

```
개발자가 코드 수정
      ↓
git push
      ↓
GitLab
      ↓
GitLab Runner
      ↓
Python 코드 테스트
      ↓
Docker 이미지 Build
      ↓
Harbor Registry Push
      ↓
Kubernetes Deployment 업데이트
      ↓
새로운 Pod 생성
      ↓
수정된 Auto-Healer 실행
```

---

# 2. GitLab Repository 구성

GitLab 프로젝트:

```
kstyle1/k8s-auto-healer
```

프로젝트 구조는 대략 다음과 같습니다.

```
auto-healer/
├── healer/
│   ├── healer.py
│   ├── requirements.txt
│   └── Dockerfile
│
├── dashboard/
│   └── ...
│
├── k8s/
│   ├── namespace.yaml
│   ├── storage.yaml
│   ├── healer.yaml
│   └── dashboard.yaml
│
└── .gitlab-ci.yml
```

---

# 3. GitLab Runner 설치 및 등록

worker-1에 GitLab Runner를 설치하고:

```
GitLab
   ↓
auto-healer-runner
   ↓
worker-1
```

구조를 만들었습니다.

Runner는 Shell executor를 사용했습니다.

실제 Pipeline 로그:

```
Running with gitlab-runner 19.3.1
Using Shell (bash) executor...
Running on worker-1...
```

즉 **GitLab이 worker-1의 GitLab Runner에서 명령을 실행**합니다.

---

# 4. CI 단계 구성

`.gitlab-ci.yml`에서 Pipeline을 세 단계로 구성했습니다.

```yaml
stages:
  - test
  - build
  - deploy
```

전체 흐름:

```
test
 ↓
build
 ↓
deploy
```

---

# 5. Test 단계

먼저 Python 코드가 정상인지 검사합니다.

```bash
python3 --version
python3 -m py_compile healer/healer.py
```

즉 코드에 문법 오류가 있으면 **Docker Build나 Kubernetes Deploy까지 진행하지 않습니다.**

```
코드 수정
   ↓
Python syntax test
   ↓
실패 → Pipeline 중단
성공 → build
```

---

# 6. Build 단계

Docker 이미지를 Git commit 기준으로 생성하도록 개선했습니다.

기존에는:

```
auto-healer:1.3
```

처럼 고정된 태그를 사용했습니다.

이를:

```
auto-healer:<commit-sha>
```

형태로 변경했습니다.

예를 들어 commit이:

```
b3b1c0eb
```

이면:

```
192.168.56.11:30002/auto-healer/auto-healer:b3b1c0eb
```

이미지를 생성합니다.

Dockerfile:

```docker
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY healer.py .

RUN mkdir -p /events

ENV PYTHONUNBUFFERED=1

CMD ["python", "-u", "healer.py"]
```

---

# 7. Harbor에 이미지 Push

worker-1의 Docker를 사용하여:

```
Docker Build
    ↓
192.168.56.11:30002
    ↓
Harbor
    ↓
auto-healer/auto-healer:<commit>
```

형태로 이미지를 저장합니다.

이 과정에서 GitLab Runner 사용자에게 Docker 권한이 없었던 문제도 확인했습니다.

처음:

```
permission denied while trying to connect to the Docker API
```

였지만 `gitlab-runner`를 Docker 사용 가능하도록 구성한 후:

```bash
sudo -u gitlab-runner docker ps
```

가 정상 동작했습니다.

---

# 8. Kubernetes 배포

기존 Kubernetes Deployment는:

```yaml
image: 192.168.56.11:30002/auto-healer/auto-healer:1.3
```

를 사용했습니다.

최종적으로 CI/CD에서는 고정 `1.3` 대신 commit SHA 이미지를 사용합니다.

```
192.168.56.11:30002/auto-healer/auto-healer:<commit-sha>
```

그리고:

```bash
kubectl set image deployment/auto-healer ...
```

명령으로 Deployment의 이미지를 변경합니다.

---

# 9. 중요한 문제: RBAC

처음에는 GitLab Runner가 Kubernetes에 접근할 때:

```
Forbidden
```

오류가 발생했습니다.

예:

```
User "system:serviceaccount:auto-heal:gitlab-runner"
cannot list resource "nodes"
```

이것은:

```
인증 성공
+
권한 부족
```

이라는 의미였습니다.

따라서 `ServiceAccount`, `Role/ClusterRole`, `RoleBinding/ClusterRoleBinding`을 구성했습니다.

---

# 10. ClusterRole을 CI에서 생성하면 안 된다는 점 확인

처음에는 GitLab Pipeline에서:

```bash
kubectl apply -f k8s/rbac.yaml
```

를 수행하려 했습니다.

하지만 여기에는:

```
ClusterRole
ClusterRoleBinding
```

이 포함되어 있었습니다.

GitLab Runner에게 클러스터 전체 RBAC 권한을 주지 않았기 때문에:

```
clusterroles ... is forbidden
clusterrolebindings ... is forbidden
```

이 발생했습니다.

따라서 **RBAC의 관리자 영역과 애플리케이션 배포 영역을 분리**했습니다.

```
관리자(control-plane)
        ↓
ServiceAccount
Role/ClusterRole
RoleBinding/ClusterRoleBinding
        ↓
GitLab Runner

----------------------------

GitLab Runner
        ↓
Deployment
Service
PVC
Pod
```

---

# 11. Namespace 문제도 해결

Pipeline에서:

```bash
kubectl apply -f k8s/namespace.yaml
```

를 실행했을 때도:

```
namespaces "auto-heal" is forbidden
```

이 발생했습니다.

Namespace 역시 클러스터 수준 리소스이므로 **GitLab Runner가 생성하도록 하지 않고 관리자가 미리 생성**하는 구조로 정리했습니다.

즉:

```bash
kubectl create namespace auto-heal
```

또는 관리자 권한으로 namespace manifest를 적용합니다.

CI/CD는 이미 존재하는:

```
auto-heal
```

namespace 안의 애플리케이션만 관리합니다.

---

# 12. Kubernetes 인증 문제

이후에는 RBAC 문제가 아니라:

```
Unauthorized
```

가 발생했습니다.

```
You must be logged in to the server
```

이것은:

```
Forbidden
```

과 다릅니다.

```
Unauthorized
    ↓
인증정보/토큰 문제

Forbidden
    ↓
인증은 성공했지만 권한 부족
```

GitLab Runner가 사용하는 kubeconfig:

```
/home/gitlab-runner/.kube/config
```

를 확인했습니다.

그리고 실제 Runner 사용자로:

```bash
sudo -u gitlab-runner \
KUBECONFIG=/home/gitlab-runner/.kube/config \
kubectl auth whoami
```

를 테스트했습니다.

---

# 13. 현재 마지막으로 확인된 문제

토큰을 갱신한 후에는 오류가:

```
Unauthorized
```

에서:

```
TLS handshake timeout
```

으로 바뀌었습니다.

즉 현재는 Kubernetes 인증보다 먼저:

```
worker-1
    ↓
192.168.56.10:6443
    ↓
control-plane API Server
```

네트워크/API Server 연결 상태를 확인해야 하는 단계입니다.

확인 명령:

```bash
ping -c 3 192.168.56.10
```

```bash
nc -vz 192.168.56.10 6443
```

```bash
curl -k --connect-timeout 5 \
https://192.168.56.10:6443/version
```

---

# 14. 최종적으로 만든 CI/CD 구조

오늘 실습의 핵심 결과는 다음 구조입니다.

```
                 ┌──────────────┐
                 │    GitLab    │
                 └──────┬───────┘
                        │
                     git push
                        │
                        ▼
              ┌──────────────────┐
              │  GitLab Runner   │
              │    worker-1      │
              └────────┬─────────┘
                       │
              ┌────────▼────────┐
              │      TEST       │
              │ Python syntax   │
              └────────┬────────┘
                       │
              ┌────────▼────────┐
              │      BUILD      │
              │ Docker image    │
              └────────┬────────┘
                       │
                       ▼
              ┌─────────────────┐
              │     Harbor      │
              │ image:<commit>  │
              └────────┬────────┘
                       │
                       ▼
              ┌─────────────────┐
              │   Kubernetes    │
              │     deploy      │
              └────────┬────────┘
                       │
                       ▼
              ┌─────────────────┐
              │  Auto-Healer    │
              │      Pod        │
              └─────────────────┘
```

## 핵심적으로 기억할 5가지

1. **GitLab** → 소스코드 관리
2. **GitLab Runner** → CI/CD 명령 실행
3. **Docker/Harbor** → 이미지 Build 및 저장
4. **Kubernetes** → 실제 서비스 실행
5. **commit SHA 이미지 태그** → 코드 수정 내용이 실제 Pod에 반영되도록 보장

특히 오늘 실습에서 가장 중요한 개선은 **`1.3` 같은 고정 이미지 태그를 계속 덮어쓰는 방식에서 `CI_COMMIT_SHORT_SHA`를 이용한 버전별 이미지 방식으로 변경한 것**입니다. 이를 통해 `git push → 새 이미지 → 새 Pod`라는 실제 CI/CD 흐름을 명확하게 만들었습니다.

# RBAC 권한 오류 해결

좋습니다. 오늘 발생했던 문제는 크게 **① RBAC 권한 문제**와 **② GitLab Runner 인증(kubeconfig/token) 문제**가 섞여서 발생했습니다. 최종적으로 CI/CD가 정상 작동한 상태를 기준으로 정리하면 다음과 같습니다.

# GitLab CI/CD RBAC 권한 오류 정리

## 1. 전체 구조

현재 Auto-Healer CI/CD 구조는 다음과 같습니다.

```
GitLab
   │
   │ push
   ▼
GitLab Pipeline
   │
   ▼
GitLab Runner
(worker-1)
   │
   │ kubectl
   ▼
Kubernetes API Server
(control-plane)
   │
   ▼
ServiceAccount
gitlab-runner
   │
   ▼
ClusterRoleBinding
gitlab-runner-auto-healer
   │
   ▼
ClusterRole
auto-healer
```

여기서 중요한 것은 **GitLab Runner가 worker-1에서 실행되더라도 Kubernetes 권한은 control-plane의 Kubernetes API Server에서 검사한다는 점**입니다.

---

# 2. 첫 번째 문제: `Forbidden`

초기에 다음 오류가 발생했습니다.

```
User "system:serviceaccount:auto-heal:gitlab-runner"
cannot list resource "nodes"
at the cluster scope
```

또는:

```
cannot get resource "clusterroles"
at the cluster scope
```

이것은:

```
인증(Authentication)     ✅
        ↓
RBAC 권한(Authorization) ❌
```

상태입니다.

즉 Kubernetes가:

> "너는 누구인지 알고 있다. 하지만 이 작업을 할 권한은 없다."
> 

라고 판단한 것입니다.

---

# 3. 원인은 ServiceAccount가 달랐던 것

당시 ClusterRoleBinding은:

```yaml
subjects:
- kind: ServiceAccount
  name: auto-healer
  namespace: auto-heal
```

였습니다.

즉:

```
ClusterRole auto-healer
        ↓
ServiceAccount auto-healer
```

로 연결되어 있었습니다.

그런데 GitLab Runner는:

```
system:serviceaccount:auto-heal:gitlab-runner
```

로 Kubernetes에 접근하고 있었습니다.

따라서:

```
GitLab Runner
     ↓
gitlab-runner ServiceAccount
     ↓
❌ ClusterRole과 연결되지 않음
```

상태였습니다.

---

# 4. 해결: GitLab Runner용 Binding 추가

기존 Auto-Healer의 권한은 유지해야 했습니다.

왜냐하면 `healer.yaml`에서:

```yaml
serviceAccountName: auto-healer
```

를 사용하고 있기 때문입니다.

따라서 기존 Binding을 변경하지 않고 **GitLab Runner용 Binding을 추가**했습니다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: gitlab-runner-auto-healer

subjects:
- kind: ServiceAccount
  name: gitlab-runner
  namespace: auto-heal

roleRef:
  kind: ClusterRole
  name: auto-healer
  apiGroup: rbac.authorization.k8s.io
```

최종 구조:

```
ClusterRole: auto-healer
        │
        ├── ClusterRoleBinding: auto-healer
        │       ↓
        │   ServiceAccount: auto-healer
        │       ↓
        │   Auto-Healer Pod
        │
        └── ClusterRoleBinding: gitlab-runner-auto-healer
                ↓
            ServiceAccount: gitlab-runner
                ↓
            GitLab Runner
```

이렇게 하면 **Auto-Healer와 GitLab Runner가 각각 필요한 권한을 사용할 수 있습니다.**

---

# 5. 두 번째 문제: `Unauthorized`

RBAC을 수정한 이후에도 다음 오류가 발생했습니다.

```
error: You must be logged in to the server (Unauthorized)
```

이것은 `Forbidden`과 완전히 다른 문제입니다.

```
Unauthorized
    ↓
인증(Authentication) 실패

Forbidden
    ↓
인증 성공
    ↓
권한(Authorization) 부족
```

즉 RBAC을 아무리 수정해도 kubeconfig 인증 자체가 실패하면:

```bash
kubectl get pods
```

를 사용할 수 없습니다.

---

# 6. kubeconfig는 존재했지만 인증정보가 문제

worker-1에서:

```bash
sudo -u gitlab-runner \
env KUBECONFIG=/home/gitlab-runner/.kube/config \
kubectl auth whoami
```

를 실행했을 때:

```
Unauthorized
```

가 발생했습니다.

하지만:

```bash
kubectl config current-context
```

는 정상적으로:

```
auto-heal-context
```

를 반환했습니다.

또한 API Server 주소도:

```
https://192.168.56.10:6443
```

로 정상적으로 설정되어 있었습니다.

따라서:

```
kubeconfig 파일 존재       ✅
context 설정               ✅
API Server 주소            ✅
인증 정보(token)            ❌
```

상태로 판단했습니다.

---

# 7. 해결: GitLab Runner ServiceAccount Token 재설정

worker-1에서 `gitlab-runner` ServiceAccount의 token을 생성했습니다.

```bash
kubectl -n auto-heal create token gitlab-runner
```

그리고 GitLab Runner의 kubeconfig에 해당 token을 설정했습니다.

```bash
sudo -u gitlab-runner \
kubectl --kubeconfig=/home/gitlab-runner/.kube/config \
config set-credentials gitlab-runner \
--token="$TOKEN"
```

그 결과:

```bash
sudo -u gitlab-runner \
env KUBECONFIG=/home/gitlab-runner/.kube/config \
kubectl auth whoami
```

에서 정상적으로:

```
system:serviceaccount:auto-heal:gitlab-runner
```

를 확인할 수 있게 되었습니다.

---

# 8. 최종적으로 두 가지를 모두 확인

## 인증 확인

```bash
sudo -u gitlab-runner \
env KUBECONFIG=/home/gitlab-runner/.kube/config \
kubectl auth whoami
```

정상:

```
system:serviceaccount:auto-heal:gitlab-runner
```

## 권한 확인

```bash
kubectl auth can-i get pods \
  --as=system:serviceaccount:auto-heal:gitlab-runner \
  -n auto-heal
```

```
yes
```

그리고:

```bash
kubectl auth can-i patch deployments \
  --as=system:serviceaccount:auto-heal:gitlab-runner \
  -n auto-heal
```

```
yes
```

가 되면 GitLab Runner가 실제 배포 작업을 수행할 수 있습니다.

---

# 9. `rbac.yaml`은 어디에 있어야 하는가?

오늘 확인했던 것처럼:

```
control-plane
└── ~/auto-healer/rbac.yaml

worker-1
└── ~/auto-healer/k8s/rbac.yaml
```

두 곳에 파일이 있어도 **문제가 없습니다.**

중요한 것은 파일의 위치가 아니라 **Kubernetes API에 적용되는 내용**입니다.

### control-plane

```bash
kubectl apply -f rbac.yaml
```

→ 실제 Kubernetes RBAC 리소스 생성/수정

### worker-1

```
~/auto-healer/k8s/rbac.yaml
```

→ GitLab Repository에서 관리하는 소스 파일

따라서 일반적으로:

```
worker-1
   ↓
git push
   ↓
GitLab
   ↓
Pipeline
```

으로 관리하고,

**ClusterRole/ClusterRoleBinding 같은 클러스터 관리 리소스는 control-plane에서 관리자 권한으로 적용**하는 것이 좋습니다.

---

# 10. 왜 Pipeline을 반복 실행하면 성공/실패가 반복되었나?

오늘 가장 혼란스러웠던 부분입니다.

원인은 크게 두 문제가 섞여 있었기 때문입니다.

### 문제 A — RBAC

```
gitlab-runner
      ↓
ClusterRoleBinding 없음
      ↓
Forbidden
```

### 문제 B — 인증 Token

```
gitlab-runner
      ↓
kubeconfig
      ↓
잘못되었거나 사용할 수 없는 인증정보
      ↓
Unauthorized
```

그래서 어떤 시점에는:

```
Pipeline
   ↓
kubectl
   ↓
성공
```

했다가 이후에는:

```
Pipeline
   ↓
kubectl
   ↓
Unauthorized
   ↓
실패
```

하는 현상이 발생했습니다.

최종적으로는 **RBAC 연결과 ServiceAccount 인증정보를 모두 정상화**하면서 문제가 해결되었습니다.

---

# 11. 이번 CI/CD의 최종 권한 구조

현재 구조를 한 장으로 정리하면:

```
                    Kubernetes
                  Control Plane
                       │
                 Kubernetes API
                       │
          ┌────────────┴────────────┐
          │                         │
   ServiceAccount              ServiceAccount
   auto-healer                 gitlab-runner
          │                         │
          │                         │
   ClusterRoleBinding        ClusterRoleBinding
   auto-healer               gitlab-runner-auto-healer
          │                         │
          └──────────┬──────────────┘
                     │
             ClusterRole
             auto-healer
                     │
          ┌──────────┴──────────┐
          │                     │
     Auto-Healer Pod       GitLab Runner
                           (worker-1)
                                │
                                │ kubectl
                                ▼
                         Kubernetes Deploy
```

---

# 12. 앞으로 문제 발생 시 판단 기준

앞으로 GitLab CI/CD에서 Kubernetes 오류가 발생하면 **먼저 오류 종류부터 구분**하면 됩니다.

| 오류 | 의미 | 확인할 것 |
| --- | --- | --- |
| `Unauthorized` | 인증 실패 | kubeconfig / token |
| `Forbidden` | RBAC 권한 부족 | ServiceAccount / ClusterRole / Binding |
| `no route to host` | 네트워크 문제 | worker → control-plane |
| `TLS handshake timeout` | API Server 통신 문제 | 네트워크/API Server |
| `connection refused` | API Server 접근 실패 | kube-apiserver |
| `You must be logged in` | 인증 실패 | kubeconfig/token |

가장 중요한 두 가지는:

```bash
kubectl auth whoami
```

→ **나는 누구인가?**

```bash
kubectl auth can-i <verb> <resource>
```

→ **내가 무엇을 할 수 있는가?**

입니다.

---

## 최종 결론

오늘 문제의 핵심은 **GitLab Runner의 Kubernetes 접근에서 인증과 권한을 별도로 이해하는 것**입니다.

```
① Authentication
   "너 누구야?"
        ↓
   ServiceAccount + Token + kubeconfig

② Authorization
   "그 작업 할 수 있어?"
        ↓
   ClusterRole + ClusterRoleBinding
```

이번에는 최종적으로:

```
gitlab-runner
    ↓
ServiceAccount Token 정상
    ↓
kubeconfig 정상
    ↓
ClusterRoleBinding 정상
    ↓
ClusterRole auto-healer
    ↓
kubectl 정상
    ↓
GitLab CI/CD deploy 정상
```

상태가 된 것입니다.

그리고 **현재처럼 `deploy` 단계에서 `kubectl set image`로 commit별 Docker 이미지를 배포하는 구조라면, 앞으로는 일반적인 코드 수정 → Git push → test → build → Harbor push → Kubernetes rollout까지 자동화된 CI/CD 실습을 안정적으로 진행할 수 있습니다.**

# Gitlab과 Argo CD 연계 의미

네. **현재까지 만든 GitLab CI/CD 실습에 Argo CD를 추가하는 것은 매우 의미가 있습니다.**

오히려 지금 단계가 **GitLab CI/CD → GitLab CI + Argo CD**로 발전시키기에 딱 좋은 시점입니다.

핵심은 **GitLab CI/CD와 Argo CD가 경쟁 관계가 아니라 역할을 나누는 관계**라는 것입니다.

## 1. 현재 구조

지금은 대략 이렇게 되어 있습니다.

```
개발자
  │
  │ git push
  ▼
GitLab Repository
  │
  ▼
GitLab CI
  │
  ├── test
  ├── docker build
  ├── Harbor push
  │
  └── kubectl set image
          │
          ▼
     Kubernetes
          │
          ├── Auto-Healer
          └── Dashboard
```

즉 현재 GitLab CI가 **빌드뿐 아니라 Kubernetes 배포까지 직접 담당**하고 있습니다.

특히 현재 `.gitlab-ci.yml`의:

```bash
kubectl set image deployment/auto-healer ...
```

같은 명령이 핵심입니다.

---

# 2. Argo CD를 추가하면

Argo CD를 추가하면 구조가 이렇게 바뀝니다.

```
                 GitLab
                   │
             git push source
                   │
                   ▼
             GitLab CI
             ┌─────┴─────┐
             │           │
            test       build
                         │
                         ▼
                       Harbor
                         │
                         │ image
                         ▼

GitLab Repository
        │
        │ Kubernetes manifest
        ▼
      Argo CD
        │
        │ GitOps
        ▼
   Kubernetes
        │
        ├── Auto-Healer
        └── Dashboard
```

여기서 중요한 변화는:

> **GitLab CI가 Kubernetes에 직접 `kubectl`을 실행하지 않는 것**
> 

입니다.

---

# 3. 역할이 명확하게 나뉩니다

### GitLab CI

**"소프트웨어를 만들어라"**

담당:

```
코드 테스트
   ↓
Docker build
   ↓
Docker image
   ↓
Harbor push
```

### Argo CD

**"Kubernetes를 Git에 정의된 상태로 만들어라"**

담당:

```
Git Repository
     ↓
Kubernetes YAML/Helm
     ↓
Argo CD
     ↓
Kubernetes
```

즉:

| 역할 | GitLab CI | Argo CD |
| --- | --- | --- |
| Git push 감지 | ✅ | ✅ |
| Python 테스트 | ✅ | ❌ |
| Docker build | ✅ | ❌ |
| Harbor push | ✅ | ❌ |
| Kubernetes 배포 | 기존에는 ✅ | ✅ |
| GitOps | ❌ | ✅ |
| Desired/Actual 상태 비교 | ❌ | ✅ |
| Drift 감지 | ❌ | ✅ |
| 자동 Sync | ❌ | ✅ |
| Rollback | 제한적 | ✅ |

---

# 4. 현재 Auto-Healer 프로젝트와 특히 잘 맞습니다

지금 프로젝트에는 이미:

```
GitLab
Harbor
Kubernetes
Auto-Healer
Dashboard
```

가 구축되어 있습니다.

따라서 Argo CD를 추가하면 상당히 좋은 **실전형 DevOps 실습 환경**이 됩니다.

최종적으로:

```
Developer
   │
   │ git push
   ▼
GitLab
   │
   ├───────────────┐
   │               │
   ▼               ▼
GitLab CI       Kubernetes manifests
   │               │
   ▼               ▼
Harbor          Argo CD
   │               │
   │ image         │ sync
   └───────┐       │
           ▼       ▼
          Kubernetes
              │
       ┌──────┴──────┐
       │             │
   Auto-Healer    Dashboard
```

라는 구조를 만들 수 있습니다.

---

# 5. 특히 지금의 CI/CD 문제를 생각하면 의미가 큽니다

지금까지 GitLab Runner에서 상당히 많은 시간을:

```
kubeconfig
ServiceAccount
Token
RBAC
ClusterRole
ClusterRoleBinding
Unauthorized
Forbidden
```

문제를 해결하는 데 사용했습니다.

Argo CD를 도입하면 **GitLab Runner가 Kubernetes API에 직접 접근할 필요가 없어집니다.**

즉:

```
현재

GitLab Runner
     │
     │ kubectl
     ▼
Kubernetes API
```

에서:

```
개선

GitLab Runner
     │
     ▼
Harbor

Argo CD
     │
     ▼
Kubernetes API
```

가 됩니다.

따라서 GitLab Runner의 Kubernetes 권한을 최소화할 수 있습니다.

이것이 **GitOps의 중요한 장점 중 하나**입니다.

---

# 6. 다만 처음부터 구조를 완전히 바꾸지는 않는 것을 추천합니다

현재 실습이 정상적으로 작동하므로 바로 기존 deploy를 제거하기보다는 **2단계로 실습하는 것이 좋습니다.**

### Phase 1 — 현재 구조 유지

```
GitLab
 ↓
CI
 ↓
Docker
 ↓
Harbor
 ↓
kubectl
 ↓
Kubernetes
```

현재 구조를 기준선으로 유지합니다.

### Phase 2 — Argo CD 추가

먼저 별도의 manifest repository 또는 디렉터리를 만듭니다.

예:

```
k8s-auto-healer
├── app
│   ├── deployment.yaml
│   ├── service.yaml
│   └── pvc.yaml
└── argocd
    └── application.yaml
```

그리고:

```
GitLab CI
   ↓
Docker image build
   ↓
Harbor
   ↓
Manifest의 image tag 변경
   ↓
Git commit
   ↓
Argo CD 감지
   ↓
Kubernetes Sync
```

를 구현합니다.

---

# 7. 현재 프로젝트에서는 이것이 가장 좋은 실습

현재 이미지가:

```
192.168.56.11:30002/auto-healer/auto-healer:<commit>
```

형태로 만들어지고 있으므로 아주 좋습니다.

예를 들어 코드 수정:

```
git push
```

하면:

```
1. GitLab CI test
        ↓
2. Docker build
        ↓
3. Harbor push
   :cf9f3a6a
        ↓
4. Kubernetes manifest
   image: ...:cf9f3a6a
        ↓
5. Git commit
        ↓
6. Argo CD detects change
        ↓
7. Kubernetes rollout
        ↓
8. 새로운 Auto-Healer Pod
```

가 됩니다.

이것이 바로 **CI와 CD를 분리한 GitOps 구조**입니다.

---

# 8. 재미있는 실습도 가능합니다

Argo CD를 추가하면 단순 배포보다 훨씬 재미있는 실습을 할 수 있습니다.

### 실습 ① Git 변경 → 자동 배포

```
image tag 변경
   ↓
Git push
   ↓
Argo CD Sync
   ↓
Pod 교체
```

### 실습 ② Kubernetes에서 수동 변경

예를 들어:

```bash
kubectl scale deployment auto-healer --replicas=3
```

그런데 Git에는:

```yaml
replicas: 1
```

이라면 Argo CD가 **Drift**를 감지합니다.

```
Desired State = 1
Actual State  = 3
       ↓
    Out of Sync
```

자동 Sync를 켜면 다시:

```
3 → 1
```

로 돌아옵니다.

이 실습은 **GitOps 개념을 이해하는 데 매우 좋습니다.**

---

# 9. Auto-Healer와 Argo CD의 조합도 재미있습니다

현재 만든 Auto-Healer가 Kubernetes 장애를 감지해서:

```
Pod 장애
 ↓
Auto-Healer
 ↓
Pod 삭제/복구
```

한다면 Argo CD는:

```
Git Desired State
 ↓
Argo CD
 ↓
Kubernetes Desired State 유지
```

를 담당합니다.

즉 두 시스템의 역할이 다릅니다.

```
             Git
              │
              ▼
           Argo CD
              │
        Desired State
              │
              ▼
        Kubernetes
              │
          Pod 장애
              │
              ▼
        Auto-Healer
              │
              ▼
          Pod 복구
```

상당히 좋은 **Kubernetes 운영 자동화 실습**이 됩니다.

---

# 10. 제가 추천하는 다음 실습 순서

현재 상태라면 다음 순서가 가장 좋습니다.

```
현재 완료
────────────────────────
① GitLab Repository
② GitLab Runner
③ GitLab CI
④ Docker Build
⑤ Harbor
⑥ Kubernetes Deploy
⑦ Auto-Healer
⑧ Dashboard
```

↓

```
다음 단계
────────────────────────
⑨ Argo CD 설치
        ↓
⑩ Argo CD ↔ GitLab Repository 연결
        ↓
⑪ Auto-Healer Application 생성
        ↓
⑫ Git 변경 → Argo CD 자동 Sync
        ↓
⑬ GitLab CI에서 kubectl 제거
        ↓
⑭ GitLab CI → Harbor
        ↓
⑮ Argo CD → Kubernetes
```

그리고 마지막으로:

```
⑯ GitOps Drift 실습
⑰ Rollback 실습
⑱ Auto-Healer + Argo CD 연동
⑲ Dashboard에서 배포 상태 확인
```

까지 하면 상당히 완성도 높은 실습이 됩니다.

**결론적으로, 지금 Argo CD를 추가하는 것은 매우 의미가 있습니다.**

특히 지금까지 만든 환경을 버리는 것이 아니라 **현재의 GitLab CI/CD를 CI 영역으로 유지하고, Argo CD를 CD/GitOps 영역으로 추가**하는 방식이 가장 교육적입니다.

그리고 지금까지 겪었던 `gitlab-runner → Kubernetes API → kubeconfig → RBAC → Unauthorized` 문제도 **왜 GitOps에서 CD 도구를 별도로 사용하는지 이해하는 좋은 배경**이 됩니다.

# Agro CD 연계 실습

네. 지금까지 진행한 내용 중 **GitLab CI/CD 일반 내용은 제외하고, Argo CD와 직접 관련된 실습만** 순서대로 정리하면 아래와 같습니다.

# Argo CD 실습 전체 순서

## 1단계. Argo CD의 역할 이해

기존 구조:

```
GitLab
  ↓
GitLab CI
  ↓
Docker Image Build
  ↓
Harbor
  ↓
kubectl
  ↓
Kubernetes
```

Argo CD를 추가하면:

```
GitLab
  ↓
GitLab CI
  ↓
Docker Image Build
  ↓
Harbor
  ↓
GitOps Repository
  ↓
Argo CD
  ↓
Kubernetes
```

핵심은 **Argo CD가 Kubernetes에 직접 배포를 담당**한다는 것입니다.

---

# 2단계. GitOps Repository 준비

Argo CD가 바라볼 Git Repository가 필요합니다.

예:

```
k8s-auto-healer-gitops/
├── healer.yaml
└── dashboard.yaml
```

각 파일에는 Kubernetes Deployment가 정의됩니다.

예:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auto-healer
  namespace: auto-heal

spec:
  replicas: 1

  template:
    spec:
      containers:
        - name: healer
          image: 192.168.56.11:30002/auto-healer/auto-healer:1.3
```

Dashboard도 동일하게 관리합니다.

---

# 3단계. GitOps Repository에 Kubernetes Manifest 저장

Argo CD는 Kubernetes YAML을 직접 읽어 Kubernetes에 적용합니다.

따라서 Git Repository의 YAML이 **실제 원하는 Kubernetes 상태(desired state)**가 됩니다.

```
GitOps Repository
       │
       ├── healer.yaml
       │
       └── dashboard.yaml
```

---

# 4단계. Kubernetes에 Argo CD 설치

Argo CD namespace 생성:

```bash
kubectl create namespace argocd
```

Argo CD 설치:

```bash
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

설치 확인:

```bash
kubectl get pods -n argocd
```

정상이라면 여러 Argo CD Pod가 `Running` 상태가 됩니다.

---

# 5단계. Argo CD Server 접속

Argo CD Server 확인:

```bash
kubectl get svc -n argocd
```

실습 환경에서는 NodePort 등을 이용해서 외부에서 접속할 수 있습니다.

예:

```
https://<control-plane-ip>:<nodeport>
```

CLI를 사용하는 경우:

```bash
argocd login <ARGOCD_SERVER> --insecure
```

---

# 6단계. Argo CD 관리자 비밀번호 확인

초기 관리자 비밀번호 확인:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
echo
```

사용자:

```
admin
```

비밀번호는 위 명령으로 확인한 값을 사용합니다.

---

# 7단계. Argo CD에서 Git Repository 등록

Argo CD가 GitOps Repository를 읽을 수 있도록 Repository를 등록합니다.

개념적으로:

```
Argo CD
   │
   │ Git Repository 연결
   ▼
GitLab
   │
   └── k8s-auto-healer-gitops
```

CLI 예:

```bash
argocd repo add https://gitlab.com/<사용자>/<gitops-repository>.git
```

Private Repository라면 GitLab 인증정보가 필요합니다.

확인:

```bash
argocd repo list
```

---

# 8단계. Argo CD Application 생성

이 단계가 **Argo CD 실습의 핵심**입니다.

Application은 다음 정보를 가지고 있습니다.

```
Git Repository
      ↓
어느 디렉터리를 사용할 것인가?
      ↓
어느 Kubernetes Cluster에 배포할 것인가?
      ↓
어느 Namespace에 배포할 것인가?
```

예를 들어:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: auto-healer
  namespace: argocd

spec:
  project: default

  source:
    repoURL: https://gitlab.com/<사용자>/<gitops-repository>.git
    targetRevision: main
    path: .

  destination:
    server: https://kubernetes.default.svc
    namespace: auto-heal

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

적용:

```bash
kubectl apply -f argocd/application.yaml
```

---

# 9단계. Application 상태 확인

```bash
kubectl get applications -n argocd
```

또는:

```bash
argocd app list
```

정상적인 상태는 대략:

```
NAME          SYNC STATUS   HEALTH STATUS
auto-healer   Synced        Healthy
```

입니다.

### 주요 상태

**Synced**

```
Git 상태 = Kubernetes 상태
```

**OutOfSync**

```
Git 상태 ≠ Kubernetes 상태
```

**Healthy**

```
Kubernetes 리소스가 정상
```

---

# 10단계. 최초 Sync 테스트

자동 Sync를 사용하지 않는 경우:

```bash
argocd app sync auto-healer
```

확인:

```bash
argocd app get auto-healer
```

또는:

```bash
kubectl get pods -n auto-heal
```

---

# 11단계. Git 변경 → Argo CD 자동 배포 테스트

이제 실제 GitOps 실습입니다.

GitOps Repository의:

```
healer.yaml
```

에서 image를 변경합니다.

예:

```yaml
image: 192.168.56.11:30002/auto-healer/auto-healer:1.3
```

↓

```yaml
image: 192.168.56.11:30002/auto-healer/auto-healer:2.0
```

Git push:

```bash
git add .
git commit -m "update auto-healer image"
git push origin main
```

그러면:

```
GitLab
  ↓
GitOps 변경
  ↓
Argo CD 감지
  ↓
OutOfSync
  ↓
자동 Sync
  ↓
Kubernetes Deployment 변경
  ↓
새 Pod
```

가 됩니다.

---

# 12단계. Self-Heal 테스트

Argo CD의 중요한 기능입니다.

현재:

```yaml
syncPolicy:
  automated:
    selfHeal: true
```

라면 Kubernetes에서 직접 Deployment를 변경해도 Argo CD가 원래 Git 상태로 되돌립니다.

예:

```bash
kubectl scale deployment auto-healer \
  -n auto-heal \
  --replicas=3
```

확인:

```bash
kubectl get deployment auto-healer -n auto-heal
```

잠시 후 Argo CD가 Git의 desired state를 기준으로 복구합니다.

---

# 13단계. Prune 테스트

GitOps Repository에서 Kubernetes 리소스를 삭제합니다.

예를 들어 Git에:

```
dashboard.yaml
```

이 존재하다가 삭제:

```bash
git rm dashboard.yaml
git commit -m "remove dashboard"
git push origin main
```

Argo CD에서:

```yaml
prune: true
```

가 활성화되어 있다면 Git에서 제거된 리소스도 Kubernetes에서 제거합니다.

즉:

```
Git에서 삭제
      ↓
Argo CD 감지
      ↓
Kubernetes 리소스 삭제
```

---

# 14단계. Auto-Healer와 Argo CD의 역할 비교

이 부분이 이번 실습에서 특히 중요합니다.

### Argo CD

```
Git
 ↓
Desired State
 ↓
Kubernetes
```

즉 **배포 상태를 관리**합니다.

### Auto-Healer

```
Kubernetes
 ↓
장애 감지
 ↓
Pod 삭제/복구
```

즉 **런타임 장애를 복구**합니다.

둘은 경쟁 관계가 아니라 서로 다른 역할입니다.

```
             GitLab
                │
                ▼
           GitOps Repo
                │
                ▼
             Argo CD
                │
                ▼
          Kubernetes
                │
        ┌───────┴────────┐
        ▼                ▼
   Auto-Healer       Dashboard
        │
        ▼
    장애 감지
        │
        ▼
      복구
```

---

# 15단계. 최종 통합 테스트

최종적으로 다음 시나리오를 수행하면 됩니다.

### 테스트 A — Git 변경 배포

```
코드 수정
 ↓
GitLab Push
 ↓
CI
 ↓
Harbor Image
 ↓
GitOps Manifest 변경
 ↓
Argo CD
 ↓
Kubernetes
```

### 테스트 B — Kubernetes 장애

```
Pod 삭제
 ↓
Auto-Healer 감지
 ↓
복구
```

### 테스트 C — Kubernetes 상태 변경

```
kubectl로 replicas 변경
 ↓
Argo CD Self-Heal
 ↓
Git desired state로 복구
```

### 테스트 D — Git에서 리소스 삭제

```
GitOps YAML 삭제
 ↓
Argo CD
 ↓
Prune
 ↓
Kubernetes 리소스 삭제
```

---

## 최종적으로 구축하려는 구조

```
                    GitLab
                       │
                 Application Code
                       │
                       ▼
                 GitLab CI/CD
                  │          │
                  │          ▼
                  │        Harbor
                  │
                  ▼
              GitOps Repo
                  │
                  ▼
                Argo CD
                  │
                  ▼
             Kubernetes
                  │
        ┌─────────┴─────────┐
        ▼                   ▼
   Auto-Healer          Dashboard
        │
        ▼
   장애 자동복구
```

**이번 Argo CD 실습에서 핵심적으로 기억할 것은 딱 3가지입니다.**

1. **Git = Desired State**
2. **Argo CD = Git과 Kubernetes를 동기화**
3. **Auto-Healer = Kubernetes 런타임 장애를 복구**

따라서 지금까지 만든 **GitLab CI/CD + Harbor + Auto-Healer**에 Argo CD를 추가하면, 단순 CI/CD 실습에서 **GitOps 기반 자동 배포 + 런타임 자동복구 구조**로 한 단계 발전시키는 의미가 있습니다.

# .gitlab-ci.yml syntax 오류 관련

좋습니다. 현재 정상 작동하는 상태를 기준으로 **`.gitlab-ci.yml`에서 YAML indent/문법 오류가 발생했던 원인과 해결 방법**을 정리하면 다음과 같습니다.

## 1. 가장 큰 원인: YAML과 Bash 문법이 충돌

GitLab CI의:

```yaml
script:
  - 명령어
```

부분은 YAML 문법이면서 동시에 Bash 명령을 작성하는 영역입니다.

예를 들어 다음과 같은 복잡한 명령은:

```yaml
- 'git commit -m "chore: update images to $CI_COMMIT_SHORT_SHA [skip ci]" || echo "No manifest changes"'
```

따옴표가 여러 겹으로 사용되면서 YAML parser가 문자열의 끝을 잘못 판단할 수 있습니다.

그 결과:

```
mapping values are not allowed in this context
did not find expected key
block sequence entries are not allowed in this context
script config should be a string
```

등의 오류가 발생했습니다.

---

# 2. 가장 안전한 해결 방법: `|-` 사용

기존:

```yaml
deploy:
  script:
    - echo "Start"
    - if [ -z "$TOKEN" ]; then echo "error"; exit 1; fi
    - git commit -m "update: image"
```

보다:

```yaml
deploy:
  script:
    - |
      echo "Start"

      if [ -z "$TOKEN" ]; then
        echo "error"
        exit 1
      fi

      git commit -m "update: image"
```

형태가 훨씬 안전합니다.

`|`를 사용하면 **여러 줄의 Bash 명령 전체가 하나의 YAML 문자열**로 처리됩니다.

즉:

```yaml
script:
  - |
      Bash 명령 1
      Bash 명령 2
      Bash 명령 3
```

구조가 됩니다.

---

# 3. `script`의 indent를 정확하게 유지

정상:

```yaml
deploy:
  stage: deploy
  script:
    - |
      echo "hello"
      echo "world"
```

계층을 보면:

```
deploy:
  stage:
  script:
    -
      명령
      명령
```

입니다.

잘못된 예:

```yaml
deploy:
  stage: deploy
  script:
  - |
    echo "hello"
```

또는:

```yaml
deploy:
  stage: deploy
    script:
      - echo "hello"
```

처럼 계층이 어긋나면 YAML parser 오류가 발생합니다.

---

# 4. 복잡한 Bash는 한 개의 block으로 묶기

이번 실습에서 가장 유용했던 방식입니다.

### 권장

```yaml
deploy:
  script:
    - |
      set -e

      echo "Starting deployment"

      if [ -z "${GITOPS_TOKEN}" ]; then
        echo "ERROR: GITOPS_TOKEN is empty"
        exit 1
      fi

      git clone \
        "https://oauth2:${GITOPS_TOKEN}@gitlab.com/..."

      cd k8s-auto-healer-gitops

      git add manifests/
      git commit -m "update images"
      git push origin main
```

### 피하는 방식

```yaml
deploy:
  script:
    - 'if [ -z "$TOKEN" ]; then echo "ERROR"; exit 1; fi'
    - 'git clone "https://oauth2:${TOKEN}@gitlab.com/..."'
    - 'git commit -m "update: image" || echo "No changes"'
```

후자의 경우 YAML 따옴표와 Bash 따옴표가 충돌하기 쉽습니다.

---

# 5. 특히 `:`가 들어가는 문자열 주의

YAML에서는 `:`가 특별한 의미를 갖습니다.

예를 들어:

```yaml
- echo "image: ${IMAGE}"
```

같은 문자열이 복잡한 상황에서 YAML parser를 혼란시킬 수 있습니다.

특히 다음처럼 여러 문법이 섞이면 위험합니다.

```yaml
- 'sed -i "s|image: .*|image: $IMAGE|" file.yaml'
```

따라서 이런 명령은:

```yaml
- |
    sed -i \
      "s|image: .*|image: ${IMAGE}|" \
      file.yaml
```

처럼 block scalar 안에 넣는 것이 안전합니다.

---

# 6. `sed`의 YAML indentation 문제도 주의

이번 실습에서 실제로 중요한 부분입니다.

잘못된 방식:

```bash
sed -i "s|image: .*|image: ${IMAGE}|" manifest.yaml
```

이 방식은 경우에 따라 기존 indentation을 잃어버릴 수 있습니다.

예:

```yaml
        image: old-image
```

가:

```yaml
image: new-image
```

처럼 변경되면 Kubernetes YAML 구조가 깨질 수 있습니다.

현재 사용한 방식처럼:

```bash
sed -i \
  "s|^\([[:space:]]*\)image: .*|\1image: ${IMAGE}|" \
  manifest.yaml
```

하면 기존 공백을 캡처해서 다시 사용합니다.

```
기존 공백
   ↓
\([[:space:]]*\)

다시 사용
   ↓
\1
```

따라서:

```yaml
        image: old
```

→

```yaml
        image: new
```

로 유지됩니다.

---

# 7. GitLab CI에서 `script`는 반드시 문자열이어야 함

이번에 발생했던:

```
jobs:build:script config should be a string or a nested array of strings
```

오류도 같은 맥락입니다.

정상:

```yaml
script:
  - echo "hello"
  - python3 main.py
```

또는:

```yaml
script:
  - |
      echo "hello"
      python3 main.py
```

둘 다 `script`의 각 항목이 **문자열**입니다.

반면 YAML indentation이 깨져서:

```yaml
script:
  - echo "hello"
    if [ ... ]
```

처럼 해석되면 GitLab은 `script` 항목을 문자열로 인식하지 못합니다.

---

# 8. CI 파일 수정 후 바로 Push하지 않기

앞으로는 다음 순서를 추천합니다.

### ① 수정

```bash
vi .gitlab-ci.yml
```

### ② 문법 확인

가능하면 GitLab의 **CI/CD → Editor → Validate**에서 먼저 검증합니다.

또는 로컬에서 YAML parser를 이용해 확인합니다.

### ③ Git diff 확인

```bash
git diff -- .gitlab-ci.yml
```

### ④ Commit

```bash
git add .gitlab-ci.yml
git commit -m "fix: update gitlab ci"
```

### ⑤ Push

```bash
git push origin main
```

---

# 9. 현재 프로젝트에서 권장하는 `.gitlab-ci.yml` 구조

현재 구축한 Auto-Healer 구조에서는 다음 패턴을 유지하면 좋습니다.

```yaml
stages:
  - test
  - build
  - deploy

variables:
  ...

test:
  stage: test
  tags:
    - auto-healer
  script:
    - |
      # 테스트 명령

build:
  stage: build
  tags:
    - auto-healer
  script:
    - |
      # Docker build/push 명령

deploy:
  stage: deploy
  tags:
    - auto-healer
  script:
    - |
      # GitOps clone
      # manifest image 변경
      # git commit
      # git push
```

즉 **각 job의 `script`를 하나의 `|-` block으로 관리**하는 방식입니다.

---

# 10. 이번 실습에서 얻은 핵심 원칙

앞으로 `.gitlab-ci.yml`을 수정할 때 아래 6가지만 기억하시면 됩니다.

| 원칙 | 권장 |
| --- | --- |
| 복잡한 Bash | `- |
| 따옴표 중첩 | 최대한 피하기 |
| `script` indentation | `script → - → 명령` 유지 |
| `sed`로 YAML 수정 | 기존 indentation 보존 |
| CI 수정 직후 | GitLab CI Lint 먼저 |
| Kubernetes 배포 | 현재 구조에서는 Argo CD 담당 |

특히 현재 프로젝트에서는:

```
GitLab CI
   ↓
Docker Image Build
   ↓
Harbor
   ↓
GitOps Repository
   ↓
Argo CD
   ↓
Kubernetes
```

구조이므로 **CI에서 Kubernetes에 직접 `kubectl apply`하지 않는 것**도 중요한 원칙입니다.

결론적으로 앞으로 `.gitlab-ci.yml`은 **"YAML 안에 Bash를 한 줄씩 억지로 넣기"보다 `script: - |` 아래에 Bash 스크립트를 작성하는 방식**을 사용하면 이번에 겪었던 indent/parser 오류 대부분을 피할 수 있습니다.
