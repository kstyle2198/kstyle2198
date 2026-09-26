# [실습] Grafana

# 기본 실습

좋습니다. 지금까지 구성한 내용을 기준으로 하면 **OpenLDAP → Grafana → LDAP 인증 → Kubernetes Secret으로 관리자 비밀번호 보호**까지 하나의 실습으로 정리하는 것이 가장 좋습니다.

아래 순서대로 진행하면 됩니다.

# Grafana + OpenLDAP LDAP 인증 실습

## 0. 최종 구성

현재 OpenLDAP이 이미 다음과 같이 구성되어 있다고 가정합니다.

```
Namespace: ldap

OpenLDAP Service
    이름: openldap
    Port: 389

LDAP Base DN
    dc=hdaic,dc=com

LDAP User Base
    ou=users,dc=hdaic,dc=com
```

전체 구조:

```
                         Kubernetes
┌──────────────────────────────────────────────────────────┐
│ namespace: ldap                                           │
│                                                          │
│   ┌─────────────────┐                                    │
│   │    OpenLDAP     │                                    │
│   │                 │                                    │
│   │ openldap:389    │                                    │
│   │                 │                                    │
│   │ ou=users        │                                    │
│   │ ├─ alice        │                                    │
│   │ ├─ bob          │                                    │
│   │ ├─ jongkim      │                                    │
│   │ └─ honglee      │                                    │
│   └────────▲────────┘                                    │
│            │ LDAP                                         │
│            │                                              │
│   ┌────────┴────────┐       ┌──────────────────────┐     │
│   │     Grafana     │       │ Kubernetes Secret    │     │
│   │                  │       │                      │     │
│   │ /etc/grafana/    │       │ LDAP_ADMIN_PASSWORD  │     │
│   │ ldap.toml        │       └──────────┬───────────┘     │
│   └────────┬─────────┘                  │                 │
│            │                            │ Secret mount    │
│            └──────────────┬─────────────┘                 │
│                           │                               │
│                           ▼                               │
│                    Grafana Pod                            │
│                    :3000                                  │
└──────────────────────────────────────────────────────────┘
```

---

# 1. 작업 디렉터리 생성

VM에서:

```bash
mkdir -p ~/grafana
cd ~/grafana
```

최종적으로 다음 파일을 만들겠습니다.

```
~/grafana/
│
├── ldap.toml
├── grafana-secret.yaml
├── grafana-ldap-config.yaml
├── grafana-deployment.yaml
└── grafana-service.yaml
```

각 파일의 역할은 다음과 같습니다.

| 파일 | 역할 |
| --- | --- |
| `ldap.toml` | Grafana LDAP 설정 원본 |
| `grafana-secret.yaml` | LDAP 관리자 비밀번호 Secret |
| `grafana-ldap-config.yaml` | `ldap.toml`을 Kubernetes ConfigMap으로 저장 |
| `grafana-deployment.yaml` | Grafana Pod 생성 및 ConfigMap/Secret 연결 |
| `grafana-service.yaml` | Grafana 외부 접속 |

---

# 2. OpenLDAP 상태 확인

Grafana를 만들기 전에 기존 LDAP가 정상인지 확인합니다.

```bash
kubectl get pod -n ldap
```

그리고 Service:

```bash
kubectl get svc -n ldap
```

다음과 같이 `openldap` Service가 있어야 합니다.

```
NAME       TYPE        CLUSTER-IP      PORT(S)
openldap   ClusterIP   10.xxx.xxx.xxx  389/TCP
```

즉 Grafana에서는:

```
ldap://openldap:389
```

으로 접근할 수 있습니다.

---

# 3. LDAP 사용자 확인

예를 들어 기존 `alice` 사용자를 확인합니다.

```bash
kubectl exec -it -n ldap deploy/openldap -- \
  ldapsearch -x \
  -H ldap://openldap:389 \
  -D "cn=admin,dc=hdaic,dc=com" \
  -W \
  -b "ou=users,dc=hdaic,dc=com" \
  "(uid=alice)"
```

정상적인 결과:

```
dn: uid=alice,ou=users,dc=hdaic,dc=com
uid: alice
...
```

이 구조를 Grafana가 사용합니다.

---

# 4. `ldap.toml` 생성

```bash
nano ~/grafana/ldap.toml
```

내용: (user grouping 하기 전..)

```toml
verbose_logging = true

[[servers]]
host = "openldap"
port = 389
use_ssl = false
start_tls = false
ssl_skip_verify = false

bind_dn = "cn=admin,dc=hdaic,dc=com"
bind_password = "$__file{/etc/grafana/secrets/LDAP_ADMIN_PASSWORD}"

search_filter = "(uid=%s)"
search_base_dns = ["ou=users,dc=hdaic,dc=com"]

[servers.attributes]
username = "uid"
name = "givenName"
surname = "sn"
email = "mail"
member_of = "memberOf"

[[servers.group_mappings]]
group_dn = "*"
org_role = "Viewer"
```

중요한 부분은:

```toml
bind_dn = "cn=admin,dc=hdaic,dc=com"
```

그리고:

```toml
bind_password = "$__file{/etc/grafana/secrets/LDAP_ADMIN_PASSWORD}"
```

입니다.

**실제 LDAP password는 `ldap.toml`에 넣지 않습니다.**

---

# 5. `ldap.toml` 확인

```bash
cat ~/grafana/ldap.toml
```

다음처럼 되어 있어야 합니다.

```
bind_dn = "cn=admin,dc=hdaic,dc=com"
bind_password = "$__file{/etc/grafana/secrets/LDAP_ADMIN_PASSWORD}"
```

---

# 6. Kubernetes Secret 생성

`grafana-secret.yaml` 생성:

```bash
nano ~/grafana/grafana-secret.yaml
```

내용:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: grafana-ldap-secret
  namespace: ldap
type: Opaque
stringData:
  LDAP_ADMIN_PASSWORD: "YOUR_LDAP_ADMIN_PASSWORD"
```

`YOUR_LDAP_ADMIN_PASSWORD`를 현재 OpenLDAP 관리자 password로 변경합니다.

예:

```yaml
stringData:
  LDAP_ADMIN_PASSWORD: "admin1234"
```

적용:

```bash
kubectl apply -f ~/grafana/grafana-secret.yaml
```

확인:

```bash
kubectl get secret -n ldap
```

예:

```
NAME                  TYPE     DATA   AGE
grafana-ldap-secret   Opaque   1      10s
```

**주의:** `grafana-secret.yaml`은 실제 password를 포함하므로 Git에 올리지 않는 것을 권장합니다.

---

# 7. ConfigMap 생성

`ldap.toml`을 ConfigMap으로 저장합니다.

방법 1 — 현재처럼 명령으로 생성:

```bash
kubectl create configmap grafana-ldap-config \
  -n ldap \
  --from-file=ldap.toml=~/grafana/ldap.toml \
  --dry-run=client \
  -o yaml | kubectl apply -f -
```

이 방식이 편리합니다.

결과:

```
ldap.toml
   ↓
ConfigMap
   ↓
grafana-ldap-config
```

확인:

```bash
kubectl get configmap grafana-ldap-config -n ldap
```

내용 확인:

```bash
kubectl get configmap grafana-ldap-config \
  -n ldap \
  -o yaml
```

여기에는 **실제 password가 없어야 합니다.**

---

# 8. ConfigMap YAML 파일로 저장

Git이나 실습 문서에서 YAML 파일로 관리하려면:

```bash
kubectl create configmap grafana-ldap-config \
  -n ldap \
  --from-file=ldap.toml=~/grafana/ldap.toml \
  --dry-run=client \
  -o yaml > ~/grafana/grafana-ldap-config.yaml
```

확인:

```bash
cat ~/grafana/grafana-ldap-config.yaml
```

이제 파일 구조:

```
~/grafana/
├── ldap.toml
├── grafana-secret.yaml
└── grafana-ldap-config.yaml
```

---

# 9. Grafana Deployment 생성

`grafana-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: ldap
spec:
  replicas: 1

  selector:
    matchLabels:
      app: grafana

  template:
    metadata:
      labels:
        app: grafana

    spec:
      containers:
        - name: grafana
          image: grafana/grafana:latest

          ports:
            - containerPort: 3000
              name: http

          env:
            - name: GF_AUTH_LDAP_ENABLED
              value: "true"

            - name: GF_AUTH_LDAP_CONFIG_FILE
              value: "/etc/grafana/ldap.toml"

            - name: GF_AUTH_LDAP_ALLOW_SIGN_UP
              value: "true"

          volumeMounts:
            # LDAP ConfigMap
            - name: ldap-config
              mountPath: /etc/grafana/ldap.toml
              subPath: ldap.toml
              readOnly: true

            # LDAP Secret
            - name: ldap-secret
              mountPath: /etc/grafana/secrets
              readOnly: true

      volumes:
        # ConfigMap
        - name: ldap-config
          configMap:
            name: grafana-ldap-config

        # Secret
        - name: ldap-secret
          secret:
            secretName: grafana-ldap-secret
            items:
              - key: LDAP_ADMIN_PASSWORD
                path: LDAP_ADMIN_PASSWORD
```

적용:

```bash
kubectl apply -f ~/grafana/grafana-deployment.yaml
```

---

# 10. Grafana Pod 확인

```bash
kubectl get pod -n ldap
```

예:

```
NAME                       READY   STATUS    RESTARTS
grafana-xxxxxxxxxx-xxxxx   1/1     Running   0
openldap-xxxxxxxxxx-xxxxx  1/1     Running   0
```

---

# 11. Secret이 Pod에 제대로 Mount됐는지 확인

실제 password를 출력하지 않고 파일 존재만 확인합니다.

```bash
kubectl exec -n ldap deploy/grafana -- \
  sh -c 'test -s /etc/grafana/secrets/LDAP_ADMIN_PASSWORD && echo "SECRET_FILE_OK" || echo "SECRET_FILE_ERROR"'
```

정상:

```
SECRET_FILE_OK
```

---

# 12. Grafana의 실제 `ldap.toml` 확인

```bash
kubectl exec -n ldap deploy/grafana -- \
  cat /etc/grafana/ldap.toml
```

핵심:

```toml
bind_dn = "cn=admin,dc=hdaic,dc=com"
bind_password = "$__file{/etc/grafana/secrets/LDAP_ADMIN_PASSWORD}"

search_filter = "(uid=%s)"
search_base_dns = ["ou=users,dc=hdaic,dc=com"]
```

실제 password가 출력되면 안 됩니다.

---

# 13. Grafana LDAP 설정 확인

로그:

```bash
kubectl logs -n ldap deploy/grafana | grep -i ldap
```

정상적으로:

```
LDAP enabled, reading config file
file=/etc/grafana/ldap.toml
```

가 나와야 합니다.

---

# 14. Grafana Service 생성

`grafana-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: ldap
spec:
  type: NodePort

  selector:
    app: grafana

  ports:
    - name: http
      port: 3000
      targetPort: 3000
      nodePort: 30300
```

적용:

```bash
kubectl apply -f ~/grafana/grafana-service.yaml
```

확인:

```bash
kubectl get svc -n ldap
```

예:

```
NAME       TYPE       CLUSTER-IP      PORT(S)
grafana    NodePort   10.xxx.xxx.xxx  3000:30300/TCP
openldap   ClusterIP  10.xxx.xxx.xxx  389/TCP
```

---

# 15. Grafana 접속

현재 VM 환경에서 Node IP가 `192.168.56.11`이라면:

```
http://192.168.56.11:30300
```

Grafana 로그인 화면이 나옵니다.

---

# 16. LDAP 사용자로 로그인

예를 들어:

```
Username: alice
Password: <alice LDAP password>
```

Grafana는 다음과 같은 방식으로 인증합니다.

```
alice
   │
   ▼
Grafana
   │
   │ LDAP search
   │ (uid=alice)
   ▼
ou=users,dc=hdaic,dc=com
   │
   ▼
uid=alice,ou=users,dc=hdaic,dc=com
   │
   │ password bind
   ▼
OpenLDAP
   │
   ▼
인증 성공
```

---

# 17. 인증 실패 시 로그 확인

```bash
kubectl logs -n ldap deploy/grafana | \
  grep -i -E 'ldap|error|bind'
```

예를 들어:

### 관리자 password가 전달되지 않는 경우

```
LDAP Result Code 53
"Unwilling To Perform"
unauthenticated bind
```

### LDAP Service 연결 문제

```
connection refused
```

### Service 이름 문제

```
no such host
```

### 사용자 검색 문제

```
user not found
```

이렇게 에러 종류를 기준으로 문제를 좁힐 수 있습니다.

---

# 18. ConfigMap 또는 Secret 변경 후

`ldap.toml`을 수정했다면:

```bash
kubectl create configmap grafana-ldap-config \
  -n ldap \
  --from-file=ldap.toml=~/grafana/ldap.toml \
  --dry-run=client \
  -o yaml | kubectl apply -f -
```

그 다음 Grafana 재시작:

```bash
kubectl rollout restart deployment grafana -n ldap
```

확인:

```bash
kubectl rollout status deployment grafana -n ldap
```

Secret을 변경한 경우에도:

```bash
kubectl apply -f ~/grafana/grafana-secret.yaml
kubectl rollout restart deployment grafana -n ldap
```

처럼 재시작하는 것이 실습 단계에서는 가장 확실합니다.

---

# 19. 최종 파일 구조

실습 완료 후에는 다음과 같습니다.

```
~/grafana/
│
├── ldap.toml
│
├── grafana-secret.yaml
│
├── grafana-ldap-config.yaml
│
├── grafana-deployment.yaml
│
└── grafana-service.yaml
```

각 파일의 관계를 정리하면:

```
                         ┌──────────────────────┐
                         │ ldap.toml            │
                         │                      │
                         │ host=openldap        │
                         │ bind_dn=cn=admin...  │
                         │ search_base=ou=users │
                         └──────────┬───────────┘
                                    │
                              ConfigMap 생성
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │ grafana-ldap-config  │
                         └──────────┬───────────┘
                                    │
                                    │ mount
                                    ▼
                         /etc/grafana/ldap.toml
                                    │
                                    │
                                    │ $__file{}
                                    │
┌──────────────────────┐            │
│ grafana-secret.yaml  │            │
│                      │            │
│ LDAP_ADMIN_PASSWORD  │            │
└──────────┬───────────┘            │
           │                        │
           │ Secret                 │
           ▼                        ▼
┌──────────────────────────────────────────┐
│              Grafana Pod                 │
│                                          │
│ /etc/grafana/secrets/                    │
│   └── LDAP_ADMIN_PASSWORD                │
│                                          │
│ /etc/grafana/ldap.toml                   │
│   └── bind_password=$__file{...}         │
└─────────────────────┬────────────────────┘
                      │
                      │ LDAP :389
                      ▼
             ┌─────────────────┐
             │    OpenLDAP     │
             │                 │
             │ dc=hdaic,dc=com │
             │                 │
             │ ou=users        │
             │ ├── alice       │
             │ ├── bob         │
             │ ├── jongkim     │
             │ └── honglee     │
             └─────────────────┘
```

## 20. 실습 전체 순서 요약

실제로 따라 할 때는 다음 순서만 기억하면 됩니다.

```
① OpenLDAP 확인
       ↓
② LDAP 사용자 확인
       ↓
③ ~/grafana 디렉터리 생성
       ↓
④ ldap.toml 작성
       ↓
⑤ grafana-secret.yaml 작성
       ↓
⑥ Secret 생성
       ↓
⑦ ldap.toml → ConfigMap 생성
       ↓
⑧ Grafana Deployment 생성
       ↓
⑨ Secret/ConfigMap Mount 확인
       ↓
⑩ Grafana Service 생성
       ↓
⑪ Grafana 접속
       ↓
⑫ LDAP 사용자(alice 등) 로그인
       ↓
⑬ Grafana LDAP 로그 확인
```

이번 실습의 핵심은 **"Grafana에 LDAP 인증을 붙이는 것"**이고, 그 과정에서 Kubernetes의 세 가지 핵심 기능도 함께 확인하는 것입니다.

```
ConfigMap → 설정 파일 관리
Secret    → 비밀번호/민감정보 관리
Volume    → ConfigMap/Secret을 Pod 내부 파일로 제공
```

따라서 이 실습은 이후 **JupyterHub, Gitea, Harbor, Keycloak 등의 LDAP 인증 연동**으로 확장하기에도 좋은 기본 실습이 됩니다.

# Ldap 사용자 그룹 설정

# Grafana + OpenLDAP 사용자 그룹 관리 실습

## 1. 최종 구성

현재 구성은 다음과 같습니다.

```
                    OpenLDAP
                       │
             dc=hdaic,dc=com
                       │
          ┌────────────┴────────────┐
          │                         │
     ou=users                   ou=groups
          │                         │
    ┌─────┼─────┐          ┌───────┼────────┐
    │     │     │          │       │        │
 jongkim alice  bob       admins  editors  viewers
    │     │     │          │       │        │
    └─────┴─────┴──────────┴───────┴────────┘
                       │
                       ▼
                    Grafana
                       │
             LDAP Group Mapping
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
      Admin          Editor         Viewer
```

사용자와 그룹은 다음과 같이 구성합니다.

| LDAP 사용자 | LDAP 그룹 | Grafana 권한 |
| --- | --- | --- |
| `jongkim` | `grafana-admins` | Admin + Server Admin |
| `honglee` | `grafana-editors` | Editor |
| `bob` | `grafana-viewers` | Viewer |

---

# 2. 1단계 — LDAP 사용자 확인

먼저 LDAP에 사용자가 정상적으로 존재하는지 확인합니다.

```bash
kubectl exec -it -n ldap deploy/openldap -- \
  ldapsearch -x \
  -H ldap://openldap:389 \
  -D "cn=admin,dc=hdaic,dc=com" \
  -W \
  -b "ou=users,dc=hdaic,dc=com"
```

예:

```
dn: uid=jongkim,ou=users,dc=hdaic,dc=com
uid: jongkim

dn: uid=alice,ou=users,dc=hdaic,dc=com
uid: honglee

dn: uid=bob,ou=users,dc=hdaic,dc=com
uid: bob
```

---

# 3. 2단계 — Grafana LDAP 인증만 먼저 확인

그룹 설정을 추가하기 전에 `jongkim`으로 Grafana 로그인을 테스트합니다.

현재 정상적으로 동작하는 핵심 설정은:

```toml
search_filter = "(uid=%s)"
search_base_dns = ["ou=users,dc=hdaic,dc=com"]
```

입니다.

이 단계에서는:

```
jongkim
   │
   ▼
LDAP 사용자 검색
   │
   ▼
uid=jongkim
   │
   ▼
Password 인증
   │
   ▼
Grafana 로그인 성공
```

을 확인합니다.

**이 단계가 성공한 상태에서만 다음 단계로 넘어가는 것이 중요합니다.**

---

# 4. 3단계 — LDAP 그룹 생성

이제 사용자 그룹을 만듭니다.

실습에서는 Grafana와 연동하기 편하도록 **`posixGroup` + `memberUid`** 구조를 사용합니다.

`groups.ldif`:

```
dn: cn=grafana-admins,ou=groups,dc=hdaic,dc=com
objectClass: posixGroup
cn: grafana-admins
gidNumber: 5000
memberUid: jongkim

dn: cn=grafana-editors,ou=groups,dc=hdaic,dc=com
objectClass: posixGroup
cn: grafana-editors
gidNumber: 5001
memberUid: honglee

dn: cn=grafana-viewers,ou=groups,dc=hdaic,dc=com
objectClass: posixGroup
cn: grafana-viewers
gidNumber: 5002
memberUid: bob
```

핵심은:

```
grafana-admins
    memberUid: jongkim

grafana-editors
    memberUid: alice

grafana-viewers
    memberUid: bob
```

입니다.

---

# 5. 4단계 — LDAP에 그룹 등록

예를 들어 `groups.ldif`를 OpenLDAP Pod에 복사합니다.

```bash
kubectl cp ~/ldap/groups.ldif \
  ldap/$(kubectl get pod -n ldap -l app=openldap \
  -o jsonpath='{.items[0].metadata.name}'):/tmp/groups.ldif
```

그리고 등록합니다.

```bash
kubectl exec -it -n ldap deploy/openldap -- \
  ldapadd -x \
  -H ldap://openldap:389 \
  -D "cn=admin,dc=hdaic,dc=com" \
  -W \
  -f /tmp/groups.ldif
```

성공하면:

```
adding new entry "cn=grafana-admins,ou=groups,dc=hdaic,dc=com"
adding new entry "cn=grafana-editors,ou=groups,dc=hdaic,dc=com"
adding new entry "cn=grafana-viewers,ou=groups,dc=hdaic,dc=com"
```

---

# 6. 5단계 — LDAP 그룹 확인

```bash
kubectl exec -it -n ldap deploy/openldap -- \
  ldapsearch -x \
  -H ldap://openldap:389 \
  -D "cn=admin,dc=hdaic,dc=com" \
  -W \
  -b "ou=groups,dc=hdaic,dc=com"
```

다음과 같이 확인합니다.

```
dn: cn=grafana-admins,ou=groups,dc=hdaic,dc=com
objectClass: posixGroup
cn: grafana-admins
gidNumber: 5000
memberUid: jongkim
```

---

# 7. 6단계 — LDAP에서 사용자 그룹 검색 테스트

**Grafana 설정을 변경하기 전에 LDAP 자체에서 검색이 되는지 확인합니다.**

`jongkim`:

```bash
kubectl exec -it -n ldap deploy/openldap -- \
  ldapsearch -x \
  -H ldap://openldap:389 \
  -D "cn=admin,dc=hdaic,dc=com" \
  -W \
  -b "ou=groups,dc=hdaic,dc=com" \
  "(&(objectClass=posixGroup)(memberUid=jongkim))"
```

정상:

```
dn: cn=grafana-admins,ou=groups,dc=hdaic,dc=com
memberUid: jongkim
```

이 테스트가 중요한 이유는:

```
LDAP Group 검색
       ↓
       성공
       ↓
Grafana Group Mapping
```

으로 단계별 문제를 분리할 수 있기 때문입니다.

---

# 8. 7단계 — Grafana LDAP Group 검색 설정

이제 `ldap.toml`에 그룹 검색 설정을 추가합니다.

기존 사용자 인증:

```toml
search_filter = "(uid=%s)"
search_base_dns = ["ou=users,dc=hdaic,dc=com"]
```

에 다음을 추가합니다.

```toml
group_search_filter = "(&(objectClass=posixGroup)(memberUid=%s))"
group_search_base_dns = ["ou=groups,dc=hdaic,dc=com"]
```

핵심은:

```toml
memberUid=%s
```

입니다.

Grafana가 `jongkim`으로 로그인하면:

```
%s
 ↓
jongkim

(&(objectClass=posixGroup)(memberUid=jongkim))
```

을 LDAP에 검색합니다.

---

# 9. 8단계 — Grafana Group Mapping 설정

이제 LDAP 그룹을 Grafana Role에 연결합니다.

```toml
[[servers.group_mappings]]
group_dn = "cn=grafana-admins,ou=groups,dc=hdaic,dc=com"
org_role = "Admin"
grafana_admin = true

[[servers.group_mappings]]
group_dn = "cn=grafana-editors,ou=groups,dc=hdaic,dc=com"
org_role = "Editor"

[[servers.group_mappings]]
group_dn = "cn=grafana-viewers,ou=groups,dc=hdaic,dc=com"
org_role = "Viewer"
```

결과:

```
LDAP Group                    Grafana
────────────────────────────────────────
grafana-admins       ──────► Admin
                              +
                              Server Admin

grafana-editors      ──────► Editor

grafana-viewers      ──────► Viewer
```

---

# 10. 9단계 — 최종 ldap.toml

현재 실습의 최종 형태는 다음과 같이 정리할 수 있습니다.

```toml
verbose_logging = true

[[servers]]
host = "openldap"
port = 389
use_ssl = false
start_tls = false
ssl_skip_verify = false

bind_dn = "cn=admin,dc=hdaic,dc=com"
bind_password = "$__file{/etc/grafana/secrets/LDAP_ADMIN_PASSWORD}"

search_filter = "(uid=%s)"
search_base_dns = ["ou=users,dc=hdaic,dc=com"]

group_search_filter = "(&(objectClass=posixGroup)(memberUid=%s))"
group_search_base_dns = ["ou=groups,dc=hdaic,dc=com"]

[servers.attributes]
username = "uid"
name = "givenName"
surname = "sn"
email = "mail"

[[servers.group_mappings]]
group_dn = "cn=grafana-admins,ou=groups,dc=hdaic,dc=com"
org_role = "Admin"
grafana_admin = true

[[servers.group_mappings]]
group_dn = "cn=grafana-editors,ou=groups,dc=hdaic,dc=com"
org_role = "Editor"

[[servers.group_mappings]]
group_dn = "cn=grafana-viewers,ou=groups,dc=hdaic,dc=com"
org_role = "Viewer"
```

---

# 11. 10단계 — ConfigMap 갱신

현재 `~/grafana`에서:

```bash
kubectl create configmap grafana-ldap-config \
  -n ldap \
  --from-file=ldap.toml=./ldap.toml \
  --dry-run=client \
  -o yaml | kubectl apply -f -
```

확인:

```bash
kubectl get configmap grafana-ldap-config -n ldap
```

---

# 12. 11단계 — Grafana 재시작

ConfigMap 변경 후 Grafana를 재시작합니다.

```bash
kubectl rollout restart deployment grafana -n ldap
```

상태 확인:

```bash
kubectl rollout status deployment grafana -n ldap
```

그리고 실제 Pod에 설정이 들어갔는지 확인합니다.

```bash
kubectl exec -n ldap deploy/grafana -- \
  cat /etc/grafana/ldap.toml
```

---

# 13. 12단계 — jongkim 로그인 테스트

`jongkim`으로 로그인합니다.

로그 확인:

```bash
kubectl logs -n ldap deploy/grafana --since=2m | \
grep -i -E 'ldap|group|authenticate|password-auth|identity'
```

정상이라면 다음과 비슷한 로그를 볼 수 있습니다.

```
Searching for user's groups
filter="(&(objectClass=posixGroup)(memberUid=jongkim))"
```

그리고 `groups=[]`가 아니라 `grafana-admins`가 검색되어야 합니다.

---

# 14. 13단계 — 사용자별 권한 테스트

### `jongkim`

```
LDAP User
   ↓
grafana-admins
   ↓
Grafana Admin
   +
Grafana Server Admin
```

관리 기능까지 확인합니다.

예:

- Data Sources
- Users
- Organizations
- Administration
- Dashboard 관리

---

### `alice`

```
LDAP User
   ↓
grafana-editors
   ↓
Grafana Editor
```

Dashboard 생성/수정은 가능하지만 서버 관리자 기능은 사용할 수 없어야 합니다.

---

### `bob`

```
LDAP User
   ↓
grafana-viewers
   ↓
Grafana Viewer
```

Dashboard 조회 중심의 권한을 갖습니다.

---

# 15. 전체 실습 흐름 요약

실습은 다음 순서로 진행하면 됩니다.

```
① OpenLDAP 설치
        ↓
② LDAP 사용자 생성
        ↓
③ Grafana 설치
        ↓
④ Grafana LDAP 인증 설정
        ↓
⑤ LDAP_ADMIN_PASSWORD → Kubernetes Secret
        ↓
⑥ ldap.toml → ConfigMap
        ↓
⑦ jongkim LDAP 로그인 테스트
        ↓
⑧ LDAP Group 생성
        ↓
⑨ memberUid 방식으로 사용자 연결
        ↓
⑩ LDAP에서 Group 검색 테스트
        ↓
⑪ Grafana group_search 설정
        ↓
⑫ Grafana group_mappings 설정
        ↓
⑬ ConfigMap 갱신
        ↓
⑭ Grafana 재시작
        ↓
⑮ jongkim / alice / bob 로그인
        ↓
⑯ Admin / Editor / Viewer 권한 확인
```

## 핵심적으로 기억할 3가지

이번 실습에서 가장 중요한 것은 다음 세 가지입니다.

**① 사용자 인증**

```toml
search_filter = "(uid=%s)"
search_base_dns = ["ou=users,dc=hdaic,dc=com"]
```

**② 사용자 그룹 검색**

```toml
group_search_filter = "(&(objectClass=posixGroup)(memberUid=%s))"
group_search_base_dns = ["ou=groups,dc=hdaic,dc=com"]
```

**③ Grafana 권한 매핑**

```toml
group_dn → org_role
```

즉,

```
LDAP User
    ↓
uid
    ↓
LDAP Group(memberUid)
    ↓
Grafana group_mappings
    ↓
Grafana Role
```

라는 구조입니다.
