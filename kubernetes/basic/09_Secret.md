# Secret

비밀번호나 키와 같은 민감한 데이터를 안전하게 저장하고 관리하는데 사용 

configmap과 유사하나 더 민감한 데이터로 간주되어 RBAC(Roll based Access Control) 등으로 엄격하게 접근 제어

# 샘플 코드

Kubernetes **Secret**은 비밀번호, API Key, Token 같은 **민감한 설정값**을 저장할 때 사용합니다.

### 1. Secret YAML

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:
  DB_USER: admin
  DB_PASSWORD: mypassword
```

적용:

```bash
kubectl apply -f secret.yaml
```

확인:

```bash
kubectl get secret db-secret
```

---

### 2. Pod에서 환경변수로 사용

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app

spec:
  replicas: 1

  selector:
    matchLabels:
      app: my-app

  template:
    metadata:
      labels:
        app: my-app

    spec:
      containers:
        - name: nginx
          image: nginx

          envFrom:
            - secretRef:
                name: db-secret
```

그러면 Pod 내부에서:

```bash
echo $DB_USER
echo $DB_PASSWORD
```

처럼 사용할 수 있습니다.

```python
import os

db_user = os.getenv("DB_USER")
db_password = os.getenv("DB_PASSWORD")

print(f"DB User: {db_user }")
```

### ConfigMap과 비교

```
ConfigMap
  → 일반적인 설정
  → DB_HOST, LOG_LEVEL 등

Secret
  → 민감한 설정
  → PASSWORD, API_KEY, TOKEN 등
```

**주의:** Secret의 값은 기본적으로 단순 Base64 인코딩 형태로 저장되므로, Secret 자체가 암호화된 금고를 의미하는 것은 아닙니다. 운영 환경에서는 Kubernetes의 암호화-at-rest, RBAC 등의 보안 설정도 함께 고려해야 합니다.
