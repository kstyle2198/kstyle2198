# Configmap

애플리케이션에 필요한 구성 정보(설정 값)을 클러스터 내엥서 쉽게 관리

애플리케이션 코드와 설정 정도를 분리하기 위해 사용 

즉, configmap은 설정 정보를 보관하는 상자!

# 샘플코드

네. `ConfigMap`은 Kubernetes에서 **애플리케이션의 설정값을 저장하는 객체**입니다.

쉽게 말하면:

> **코드/이미지와 설정을 분리하기 위한 저장소**
> 

예를 들어 애플리케이션에서:

```
DB_HOST=postgres
DB_PORT=5432
LOG_LEVEL=INFO
```

같은 설정을 Docker 이미지 안에 넣지 않고 ConfigMap으로 관리할 수 있습니다.

## 1. 가장 기본적인 ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap

metadata:
  name: app-config

data:
  DB_HOST: "postgres"
  DB_PORT: "5432"
  LOG_LEVEL: "INFO"
```

적용:

```bash
kubectl apply -f configmap.yaml
```

확인:

```bash
kubectl get configmap
```

상세 내용:

```bash
kubectl describe configmap app-config
```

---

# 2. ConfigMap을 Pod의 환경변수로 사용

ConfigMap을 만드는 것만으로는 Pod가 자동으로 사용하는 것은 아닙니다.

Pod에서 명시적으로 가져와야 합니다.

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: nginx-deployment

spec:
  replicas: 3

  selector:
    matchLabels:
      app: nginx

  template:
    metadata:
      labels:
        app: nginx

    spec:
      containers:
        - name: nginx
          image: nginx

          env:           # configmap에서 가져올  정보들 
            - name: DB_HOST
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: DB_HOST

            - name: DB_PORT
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: DB_PORT
```

구조는:

```
ConfigMap
app-config
   │
   ├── DB_HOST=postgres
   ├── DB_PORT=5432
   └── LOG_LEVEL=INFO
          │
          │
          ↓
       Deployment
          │
          ↓
         Pod
          │
          ↓
   Environment Variables
```

Pod 내부에서는:

```bash
echo $DB_HOST
```

결과:

```
postgres
```

---

# 3. ConfigMap 전체를 한꺼번에 환경변수로 가져오기

위처럼 하나씩 지정하지 않고 전체 ConfigMap을 가져올 수도 있습니다.

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: nginx-deployment

spec:
  replicas: 3

  selector:
    matchLabels:
      app: nginx

  template:
    metadata:
      labels:
        app: nginx

    spec:
      containers:
        - name: nginx
          image: nginx

          envFrom:
            - configMapRef:
                name: app-config
```

그러면:

```
ConfigMap
│
├── DB_HOST
├── DB_PORT
└── LOG_LEVEL
       │
       ↓
Pod Environment
│
├── DB_HOST
├── DB_PORT
└── LOG_LEVEL
```

처럼 모두 들어갑니다.

---

# 4. ConfigMap을 파일로 Mount

ConfigMap은 **파일 형태로 Pod에 전달**할 수도 있습니다.

ConfigMap:

```yaml
apiVersion: v1
kind: ConfigMap

metadata:
  name: app-config

data:
  application.yaml: |
    server:
      port: 8080

    logging:
      level: INFO

    database:
      host: postgres
      port: 5432
```

Deployment:

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
        - name: my-app
          image: my-app:1.0

          volumeMounts:
            - name: config-volume
              mountPath: /app/config

      volumes:
        - name: config-volume
          configMap:
            name: app-config
```

그러면 Pod 내부에:

```
/app/config/application.yaml
```

파일이 생성됩니다.

```
Pod
│
└── /app/config
      │
      └── application.yaml
```

---

# 5. ConfigMap과 Secret의 차이

여기서 **ConfigMap과 Secret을 구분하는 것이 중요합니다.**

### ConfigMap

일반적인 설정:

```
DB_HOST
DB_PORT
LOG_LEVEL
APP_MODE
API_URL
```

### Secret

민감한 정보:

```
DB_PASSWORD
API_KEY
TOKEN
PRIVATE_KEY
```

즉:

```
ConfigMap
    ↓
일반 설정

Secret
    ↓
민감한 설정
```

예를 들어 **DB 비밀번호를 ConfigMap에 넣는 것은 적절하지 않습니다.**

---

# 6. 지금까지 배운 Deployment와 연결

지금까지 공부한 내용을 합치면 이런 구조가 됩니다.

```
                    Deployment
                        │
                        ↓
                    ReplicaSet
                        │
             ┌──────────┼──────────┐
             ↓          ↓          ↓
            Pod        Pod        Pod
             │          │          │
             └──────────┼──────────┘
                        │
                  ConfigMap
                   app-config
                        │
              ┌─────────┼─────────┐
              ↓         ↓         ↓
           DB_HOST    DB_PORT   LOG_LEVEL
```

ConfigMap은 **Pod 자체를 관리하는 객체가 아니라 Pod에 설정값을 전달하는 객체**라고 생각하면 됩니다.

---

## 7. 실습용으로 가장 추천하는 예제

처음 실습한다면 아래 두 파일만 만들어보는 것이 좋습니다.

**configmap.yaml**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config

data:
  APP_NAME: "my-app"
  APP_ENV: "dev"
  LOG_LEVEL: "INFO"
```

**deployment.yaml**

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: my-app

spec:
  replicas: 2

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
            - configMapRef:
                name: app-config
```

적용:

```bash
kubectl apply -f configmap.yaml
kubectl apply -f deployment.yaml
```

Pod 확인:

```bash
kubectl get pods
```

그리고 Pod 안에서:

```bash
kubectl exec -it <pod-name> -- env | grep APP_
```

결과:

```
APP_NAME=my-app
APP_ENV=dev
```
