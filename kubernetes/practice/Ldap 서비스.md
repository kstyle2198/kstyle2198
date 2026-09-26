# [실습] Ldap 서비스

# 기본 흐름

지금까지 진행한 **VirtualBox + kubeadm Kubernetes 환경에서 OpenLDAP을 설치하고, PV/PVC를 이용해 사용자 정보를 영속화한 과정**을 순서대로 정리하면 다음과 같습니다.

# 1. 전체 구성

현재 구성은 다음과 같습니다.

```
                         Kubernetes Cluster
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  control-plane                                               │
│  192.168.56.10                                               │
│                                                              │
│  worker-1                                                    │
│  192.168.56.11                                               │
│      │                                                       │
│      │                                                        │
│      ├── OpenLDAP Pod                                        │
│      │      │                                                 │
│      │      ├── 389 LDAP                                     │
│      │      └── 636 LDAPS                                    │
│      │                                                        │
│      ├── Service: openldap                                   │
│      │      └── ClusterIP: 10.104.20.241                     │
│      │                                                        │
│      ├── PVC: openldap-data                                  │
│      │      └── PV: openldap-data-pv                         │
│      │             └── /data/openldap/ldap                   │
│      │                                                        │
│      └── PVC: openldap-config                                │
│             └── PV: openldap-config-pv                       │
│                    └── /data/openldap/config                  │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

# 2. LDAP Namespace 생성

먼저 OpenLDAP을 별도의 namespace에서 관리했습니다.

```bash
kubectl create namespace ldap
```

확인:

```bash
kubectl get namespace ldap
```

구성:

```
ldap
└── OpenLDAP 관련 리소스
```

---

# 3. 처음에는 StorageClass가 없는 상태 확인

현재 kubeadm + VirtualBox 클러스터에는 기본 StorageClass가 없었습니다.

```bash
kubectl get storageclass
```

결과:

```
No resources found
```

따라서 일반적인 PVC만 생성하면 `Pending` 상태가 될 수 있었습니다.

그래서 **Local PersistentVolume + PVC 방식**으로 OpenLDAP 저장소를 구성했습니다.

---

# 4. worker-1에 OpenLDAP 데이터 디렉터리 생성

OpenLDAP Pod가 `worker-1`에서 실행되도록 구성할 예정이므로 worker-1에 디렉터리를 생성했습니다.

```bash
sudo mkdir -p /data/openldap/ldap
sudo mkdir -p /data/openldap/config
```

그리고 실습 환경에서 권한 문제를 피하기 위해:

```bash
sudo chmod 777 /data/openldap/ldap
sudo chmod 777 /data/openldap/config
```

구조:

```
worker-1
└── /data/openldap
    ├── ldap
    └── config
```

---

# 5. Local PersistentVolume 생성

두 개의 PV를 생성했습니다.

### LDAP 데이터

```
openldap-data-pv
5Gi
```

실제 저장 위치:

```
worker-1:/data/openldap/ldap
```

### LDAP 설정

```
openldap-config-pv
1Gi
```

실제 저장 위치:

```
worker-1:/data/openldap/config
```

두 PV 모두:

```yaml
accessModes:
  - ReadWriteOnce

persistentVolumeReclaimPolicy: Retain
```

으로 구성했습니다.

그리고 중요한 부분으로 Local PV가 `worker-1`을 사용하도록 `nodeAffinity`를 설정했습니다.

```yaml
nodeAffinity:
  required:
    nodeSelectorTerms:
      - matchExpressions:
          - key: kubernetes.io/hostname
            operator: In
            values:
              - worker-1
```

---

# 6. PVC 생성

PV에 연결하기 위해 PVC를 생성했습니다.

### LDAP 데이터 PVC

```
openldap-data
```

요청 용량:

```
5Gi
```

### LDAP 설정 PVC

```
openldap-config
```

요청 용량:

```
1Gi
```

두 PVC의 StorageClass:

```
openldap-local
```

---

# 7. PV/PVC 연결 확인

다음 명령으로 확인했습니다.

```bash
kubectl get pv
kubectl get pvc -n ldap
```

최종 상태:

```
NAME                 CAPACITY   ACCESS MODES   STATUS   CLAIM
openldap-config-pv   1Gi        RWO            Bound    ldap/openldap-config
openldap-data-pv     5Gi        RWO            Bound    ldap/openldap-data
```

PVC도:

```
NAME              STATUS   VOLUME
openldap-config   Bound    openldap-config-pv
openldap-data     Bound    openldap-data-pv
```

따라서:

```
PV ←→ PVC
```

연결이 정상적으로 완료되었습니다.

---

# 8. OpenLDAP Deployment 생성

OpenLDAP 이미지로 다음 이미지를 사용했습니다.

```yaml
image: osixia/openldap:1.5.0
```

환경 변수:

```yaml
LDAP_ORGANISATION: HDAIC
LDAP_DOMAIN: hdaic.com
LDAP_ADMIN_PASSWORD: admin123
```

따라서 기본 LDAP 구조의 Base DN은:

```
dc=hdaic,dc=com
```

관리자 DN은:

```
cn=admin,dc=hdaic,dc=com
```

입니다.

---

# 9. OpenLDAP에 Volume 연결

Deployment에 다음 Volume을 연결했습니다.

```yaml
volumeMounts:
  - name: ldap-data
    mountPath: /var/lib/ldap

  - name: ldap-config
    mountPath: /etc/ldap/slapd.d
```

그리고:

```yaml
volumes:
  - name: ldap-data
    persistentVolumeClaim:
      claimName: openldap-data

  - name: ldap-config
    persistentVolumeClaim:
      claimName: openldap-config
```

결과적으로:

```
/var/lib/ldap
      ↓
openldap-data PVC
      ↓
openldap-data-pv
      ↓
/data/openldap/ldap
```

그리고:

```
/etc/ldap/slapd.d
      ↓
openldap-config PVC
      ↓
openldap-config-pv
      ↓
/data/openldap/config
```

구조가 만들어졌습니다.

---

# 10. Pod를 worker-1에 고정

Local PV가 worker-1에 있기 때문에 Deployment에 다음을 추가했습니다.

```yaml
nodeSelector:
  kubernetes.io/hostname: worker-1
```

따라서 OpenLDAP Pod는 worker-1에서 실행됩니다.

실제 확인 결과:

```
Node: worker-1/192.168.56.11
```

---

# 11. OpenLDAP Pod 정상 실행 확인

확인:

```bash
kubectl get pod -n ldap -o wide
```

결과:

```
NAME                       READY   STATUS    RESTARTS   NODE
openldap-7949d8bb8-rwnpv   1/1     Running   0          worker-1
```

정상 상태였습니다.

또한 `describe` 결과에서:

```
/var/lib/ldap from ldap-data
/etc/ldap/slapd.d from ldap-config
```

이 확인되었습니다.

즉 Volume Mount도 정상입니다.

---

# 12. OpenLDAP 포트 확인

컨테이너 내부에서:

```bash
kubectl exec -it -n ldap deploy/openldap -- ss -lntp
```

결과:

```
0.0.0.0:389   LISTEN
0.0.0.0:636   LISTEN
```

따라서 OpenLDAP 서비스 자체도 정상입니다.

```
389 → LDAP
636 → LDAPS
```

---

# 13. Kubernetes Service 생성

OpenLDAP Pod에 접근하기 위해 `openldap` Service를 생성했습니다.

```
Service Name: openldap
Type: ClusterIP
ClusterIP: 10.104.20.241
```

포트:

```
389/TCP
636/TCP
```

확인:

```bash
kubectl get svc -n ldap
```

결과:

```
NAME       TYPE        CLUSTER-IP      PORT(S)
openldap   ClusterIP   10.104.20.241   389/TCP,636/TCP
```

---

# 14. Service → Pod 연결 확인

```bash
kubectl get endpoints -n ldap
```

결과:

```
openldap
10.0.1.94:389
10.0.1.94:636
```

즉:

```
Service
10.104.20.241
      │
      ▼
Pod
10.0.1.94
      │
      ├── 389
      └── 636
```

연결이 정상입니다.

---

# 15. LDAP 관리자 Bind 테스트

다음과 같이 LDAP 서버에 접속했습니다.

```bash
kubectl exec -it -n ldap deploy/openldap -- \
  ldapsearch -x \
  -H ldap://openldap:389 \
  -D "cn=admin,dc=hdaic,dc=com" \
  -w admin123 \
  -b "ou=users,dc=hdaic,dc=com"
```

여기서:

```
result: 32 No such object
matchedDN: dc=hdaic,dc=com
```

이 나왔습니다.

이것은 **LDAP 서버 오류가 아니었습니다.**

의미는:

```
dc=hdaic,dc=com       ← 존재
└── ou=users          ← 아직 없음
```

이었습니다.

---

# 16. `ou=users` 생성

LDAP의 사용자 저장 위치를 다음과 같이 구성하기로 했습니다.

```
dc=hdaic,dc=com
└── ou=users
    ├── uid=alice
    ├── uid=bob
    └── uid=jongkim
```

`ou=users` 생성용 LDIF:

```
dn: ou=users,dc=hdaic,dc=com
objectClass: top
objectClass: organizationalUnit
ou: users
```

이를 `ldapadd`로 등록합니다.

```bash
ldapadd -x \
  -H ldap://openldap:389 \
  -D "cn=admin,dc=hdaic,dc=com" \
  -w admin123 \
  -f /tmp/users-ou.ldif
```

---

# 17. LDAP 사용자 생성

예를 들어 `jongkim` 사용자를 다음 구조로 생성할 수 있습니다.

```
uid=jongkim
ou=users
dc=hdaic
dc=com
```

즉 전체 DN:

```
uid=jongkim,ou=users,dc=hdaic,dc=com
```

LDIF 예:

```
dn: uid=jongkim,ou=users,dc=hdaic,dc=com
objectClass: inetOrgPerson
objectClass: organizationalPerson
objectClass: person
objectClass: top
uid: jongkim
cn: Jong Kim
sn: Kim
givenName: Jong
mail: jongkim@hdaic.com
userPassword: password123
```

---

# 18. LDAP 사용자 인증 테스트

사용자가 생성된 후 다음과 같이 인증 테스트를 할 수 있습니다.

```bash
kubectl exec -it -n ldap deploy/openldap -- \
  ldapwhoami -x \
  -H ldap://openldap:389 \
  -D "uid=jongkim,ou=users,dc=hdaic,dc=com" \
  -w password123
```

정상이라면:

```
dn:uid=jongkim,ou=users,dc=hdaic,dc=com
```

가 반환됩니다.

---

# 19. Pod 삭제 후 데이터 유지 테스트

현재 구성에서 가장 중요한 테스트입니다.

```bash
kubectl delete pod -n ldap -l app=openldap
```

Deployment가 자동으로 새로운 Pod를 생성합니다.

```bash
kubectl get pod -n ldap -o wide
```

새 Pod가 다시:

```
Running
worker-1
```

상태가 됩니다.

그 후 LDAP 사용자 검색:

```bash
kubectl exec -it -n ldap deploy/openldap -- \
  ldapsearch -x \
  -H ldap://openldap:389 \
  -D "cn=admin,dc=hdaic,dc=com" \
  -w admin123 \
  -b "ou=users,dc=hdaic,dc=com"
```

기존 사용자가 그대로 존재하면 **LDAP 데이터 영속화가 성공한 것**입니다.

---

# 20. 현재까지 완성된 구조

최종적으로 현재 LDAP 시스템은 다음과 같습니다.

```
                    Kubernetes Cluster
                           │
                           │
                    ┌──────▼──────┐
                    │   Service   │
                    │   openldap  │
                    │  ClusterIP  │
                    │ 10.104.20.241
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │ OpenLDAP Pod │
                    │  worker-1    │
                    │ 10.0.1.94   │
                    └──────┬──────┘
                           │
              ┌────────────┴────────────┐
              │                         │
       /var/lib/ldap             /etc/ldap/slapd.d
              │                         │
              ▼                         ▼
     openldap-data PVC        openldap-config PVC
              │                         │
              ▼                         ▼
     openldap-data-pv        openldap-config-pv
              │                         │
              ▼                         ▼
 /data/openldap/ldap      /data/openldap/config
       worker-1                  worker-1
```

LDAP 데이터 구조는:

```
dc=hdaic,dc=com
│
├── cn=admin
│
└── ou=users
    │
    ├── uid=alice
    ├── uid=bob
    └── uid=jongkim
```

---

## 21. 현재 상태에서 다음 단계

현재 LDAP 자체는 거의 완성된 상태입니다.

다음 순서로 진행하면 좋습니다.

```
① ou=users 생성
       ↓
② LDAP 사용자 생성
       ↓
③ ldapwhoami로 사용자 인증 테스트
       ↓
④ Pod 삭제
       ↓
⑤ 사용자 정보 유지 확인
       ↓
⑥ JupyterHub / 애플리케이션에서 LDAP 인증
       ↓
⑦ 필요하면 LDAP 외부 접근 설정
```

특히 **JupyterHub와 연결할 목적이라면 다음 단계는 `LDAPAuthenticator` 설정**입니다.

JupyterHub에서는 보통 다음 정보를 사용하게 됩니다.

```
LDAP Server:
openldap.ldap.svc.cluster.local

Port:
389

Base DN:
dc=hdaic,dc=com

User Search Base:
ou=users,dc=hdaic,dc=com

User Attribute:
uid
```

즉 지금까지 만든 OpenLDAP 구조가 이후 **JupyterHub LDAP 로그인 인증의 기반**이 됩니다.

# 사용자 추가 방법

네. 현재 구성에서는 **LDIF 파일을 작성 → Pod에 복사 → `ldapadd`로 등록**하는 방식이 가장 명확합니다.

현재 LDAP 구조는 다음과 같습니다.

```
dc=hdaic,dc=com
└── ou=users
    ├── uid=alice
    ├── uid=bob
    └── uid=jongkim
```

## 1. 사용자 LDIF 파일 작성

예를 들어 `honglee` 사용자를 추가하겠습니다.

`honglee.ldif`:

```
dn: uid=honglee,ou=users,dc=hdaic,dc=com
objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: inetOrgPerson
uid: honglee
cn: Hong Lee
sn: Lee
givenName: Hong
mail: honglee@hdaic.com
userPassword: password123
```

파일 생성:

```bash
cat > honglee.ldif <<'EOF'
dn: uid=honglee,ou=users,dc=hdaic,dc=com
objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: inetOrgPerson
uid: honglee
cn: Hong Lee
sn: Lee
givenName: Hong
mail: honglee@hdaic.com
userPassword: password123
EOF
```

---

## 2. LDIF 파일을 OpenLDAP Pod로 복사

먼저 Pod 이름을 확인합니다.

```bash
kubectl get pod -n ldap
```

예:

```
openldap-7949d8bb8-rwnpv
```

그 다음:

```bash
kubectl cp honglee.ldif \
  ldap/openldap-7949d8bb8-rwnpv:/tmp/honglee.ldif
```

Pod 이름을 매번 입력하기 번거롭다면:

```bash
POD=$(kubectl get pod -n ldap -l app=openldap -o jsonpath='{.items[0].metadata.name}')

kubectl cp honglee.ldif \
  ldap/$POD:/tmp/honglee.ldif
```

---

## 3. LDAP 사용자 추가

관리자 계정으로 `ldapadd`를 실행합니다.

```bash
kubectl exec -it -n ldap deploy/openldap -- \
  ldapadd -x \
  -H ldap://openldap:389 \
  -D "cn=admin,dc=hdaic,dc=com" \
  -w admin123 \
  -f /tmp/honglee.ldif
```

정상적으로 추가되면:

```
adding new entry "uid=honglee,ou=users,dc=hdaic,dc=com"
```

가 출력됩니다.

---

## 4. 사용자 조회

등록되었는지 확인합니다.

```bash
kubectl exec -it -n ldap deploy/openldap -- \
  ldapsearch -x \
  -H ldap://openldap:389 \
  -D "cn=admin,dc=hdaic,dc=com" \
  -w admin123 \
  -b "ou=users,dc=hdaic,dc=com" \
  "(uid=honglee)"
```

정상이라면:

```
dn: uid=honglee,ou=users,dc=hdaic,dc=com
uid: honglee
cn: Hong Lee
sn: Lee
givenName: Hong
mail: honglee@hdaic.com
```

---

## 5. 사용자 비밀번호로 로그인 테스트

관리자 계정이 아니라 방금 만든 `honglee` 계정으로 LDAP Bind가 되는지 확인합니다.

```bash
kubectl exec -it -n ldap deploy/openldap -- \
  ldapwhoami -x \
  -H ldap://openldap:389 \
  -D "uid=honglee,ou=users,dc=hdaic,dc=com" \
  -w password123
```

정상이면:

```
dn:uid=honglee,ou=users,dc=hdaic,dc=com
```

가 출력됩니다.

이 테스트가 성공하면 **사용자 생성 + 비밀번호 인증까지 정상**입니다.

---

## 6. 여러 사용자를 한 번에 추가

여러 명을 등록할 경우 하나의 LDIF 파일에 여러 사용자를 넣을 수도 있습니다.

```
dn: uid=alice,ou=users,dc=hdaic,dc=com
objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: inetOrgPerson
uid: alice
cn: Alice Kim
sn: Kim
givenName: Alice
mail: alice@hdaic.com
userPassword: alice123

dn: uid=bob,ou=users,dc=hdaic,dc=com
objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: inetOrgPerson
uid: bob
cn: Bob Kim
sn: Kim
givenName: Bob
mail: bob@hdaic.com
userPassword: bob123

dn: uid=honglee,ou=users,dc=hdaic,dc=com
objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: inetOrgPerson
uid: honglee
cn: Hong Lee
sn: Lee
givenName: Hong
mail: honglee@hdaic.com
userPassword: password123
```

그리고 한 번에:

```bash
ldapadd -x \
  -H ldap://openldap:389 \
  -D "cn=admin,dc=hdaic,dc=com" \
  -w admin123 \
  -f users.ldif
```

### 현재 실습에서 권장하는 테스트 순서

```
ou=users 생성
    ↓
alice 생성
    ↓
bob 생성
    ↓
honglee 생성
    ↓
ldapsearch로 전체 사용자 조회
    ↓
ldapwhoami로 각 사용자 인증
    ↓
OpenLDAP Pod 삭제
    ↓
Pod 재생성
    ↓
사용자 다시 조회
    ↓
사용자 정보가 유지되는지 확인
```

특히 **Pod 삭제 후 `alice`, `bob`, `honglee`가 그대로 조회되는 것까지 확인하면 현재 구성한 PV/PVC 영속화가 실제로 제대로 동작하는지 검증할 수 있습니다.**

# Sample codes

```python
apiVersion: apps/v1
kind: Deployment
metadata:
  name: openldap
  namespace: ldap

spec:
  replicas: 1

  selector:
    matchLabels:
      app: openldap

  template:
    metadata:
      labels:
        app: openldap

    spec:
      nodeSelector:
        kubernetes.io/hostname: worker-1

      containers:
        - name: openldap
          image: osixia/openldap:1.5.0

          ports:
            - containerPort: 389
            - containerPort: 636

          env:
            - name: LDAP_ORGANISATION
              value: "HDAIC"

            - name: LDAP_DOMAIN
              value: "hdaic.com"

            - name: LDAP_ADMIN_PASSWORD
              value: "admin123"

          volumeMounts:
            - name: ldap-data
              mountPath: /var/lib/ldap

            - name: ldap-config
              mountPath: /etc/ldap/slapd.d

      volumes:
        - name: ldap-data
          persistentVolumeClaim:
            claimName: openldap-data

        - name: ldap-config
          persistentVolumeClaim:
            claimName: openldap-config
```

```python
apiVersion: v1
kind: PersistentVolume
metadata:
  name: openldap-data-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: openldap-local

  local:
    path: /data/openldap/ldap

  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - worker-1

---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: openldap-config-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: openldap-local

  local:
    path: /data/openldap/config

  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - worker-1

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: openldap-data
  namespace: ldap
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: openldap-local
  resources:
    requests:
      storage: 5Gi

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: openldap-config
  namespace: ldap
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: openldap-local
  resources:
    requests:
      storage: 1Gi
```

```python
apiVersion: v1
kind: Service
metadata:
  name: openldap
  namespace: ldap
spec:
  selector:
    app: openldap
  ports:
    - name: ldap
      port: 389
      targetPort: 389
    - name: ldaps
      port: 636
      targetPort: 636
```
