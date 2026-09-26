# K8S yaml 파일

쿠버네티스에서 yaml 파일은 리소스를 정의하는 기본 방법으로 클러스터에 배포할 오브젝트 상태를 선언적으로 기술합니다. 

```yaml
# 아래는 nginx 이미지 배포 샘플 
apiVersion: v1  # 리소스의 API 버전 
kind: Pod       # 리소스의 종류
matadata:       # 리소스의 메타데이터 (주로 name, labes)
	name: nginx-pod
	labels:
		app: nginx 
spec:           # 리소스의 사양 (동작, 속성, 구성요소 등)
	containers:   # pod 안에서 실행될 컨테이너 정의 
		-name: nginx-container   # - 는 yaml 문법에서 리스트(배열)을 나타냄 (이하 값들의 리스트)
		 image: nginx:1.21
		 ports:
		  - containerPort: 80
```

```yaml
# Two container + volume 사례

apiVersion: v1
kind: Pod
metadata: 
	name: shared-pod
	labels:
		app: shared-app
		
spec:
	containers:
		- name: writer-container
			image: busybox
			command: ["sh", "-c", "echo 'Hello' > /shared-data/message.txt; sleep 3600"]
			volumeMounts:
				- name: shared-volume
					mountPath: /shared-data
					
		- name: reader-container
			image: busybox
			command: ["sh", "-c", "cat /shared-data/message.txt; sleep 3600"]
			volumeMounts:
				- name: shared-volume
					mountPath: /shared-data
	volumes:
		- name: shared-volume
			emtpyDir: {} # Pod가 실행되는 동안 임시로 데이터를 저장하는 공유 디스크 생성 
```

### metadata name과 labeds 차이

쿠버네티스 YAML의 `metadata`에서 **`name`과 `labels`는 역할이 완전히 다릅니다.**

가장 간단하게 말하면:

> **`name` = 이 리소스 자체의 이름**
> 
> 
> **`labels` = 이 리소스를 분류·그룹화하기 위한 태그**
> 

예를 들어:

```
apiVersion: apps/v1
kind: Deployment

metadata:
  name: my-app
  labels:
    app: my-app
    environment: dev
```

여기서:

```
name
└── Deployment 자체의 이름
    → my-app

labels
├── app= my-app
└── environment= dev
```

## 1. `metadata.name`

`name`은 **Kubernetes 리소스를 식별하는 고유한 이름**입니다.

```
metadata:
  name: my-app
```

그러면:

```
kubectl get deployment
```

결과에서:

```
NAME
my-app
```

으로 나타납니다.

그리고 특정 Deployment를 조회할 때:

```
kubectl get deployment my-app
```

처럼 사용합니다.

같은 namespace에서는 동일한 `kind`의 리소스에 같은 이름을 사용할 수 없습니다.

예:

```
kind: Deployment
metadata:
  name: my-app
```

이미 존재한다면 같은 namespace에 또 다른 `my-app` Deployment를 만들 수 없습니다.

---

# 2. `metadata.labels`

`labels`는 **리소스를 분류하거나 다른 리소스가 특정 리소스를 선택하기 위해 사용하는 key-value 정보**입니다.

```
metadata:
  name: my-app
  labels:
    app: my-app
    environment: dev
```

여기서:

```
name = my-app

labels:
  app         = my-app
  environment = dev
```

입니다.

`labels`는 여러 개 붙일 수 있습니다.

```
labels:
  app: my-app
  environment: dev
  version: v1
  team: backend
```

이렇게 하면 리소스를 여러 관점으로 분류할 수 있습니다.

---

# 3. 가장 중요한 차이: Selector

Kubernetes에서 `label`이 특히 중요한 이유는 **Service, Deployment 등이 label을 이용해서 Pod를 찾기 때문**입니다.

예를 들어 Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: my-app

spec:
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
```

여기서 중요한 관계가 있습니다.

```
Deployment
   │
   │ selector
   ↓
app=my-app
   │
   │
   ↓
Pod
metadata:
  labels:
    app=my-app
```

즉 Deployment가:

```
selector:
  matchLabels:
    app: my-app
```

라고 하면

```
app=my-app
```

이라는 label을 가진 Pod를 자신의 관리 대상으로 선택합니다.

---

# 4. Service에서도 똑같이 사용

예를 들어:

```
apiVersion: v1
kind: Service

metadata:
  name: my-app-service

spec:
  selector:
    app: my-app

  ports:
    - port: 80
```

이 Service는:

```
label:
  app=my-app
```

을 가지고 있는 Pod로 트래픽을 전달합니다.

따라서 다음 Pod가 있다면:

```
metadata:
  name: my-app-pod-1
  labels:
    app: my-app
```

```
metadata:
  name: my-app-pod-2
  labels:
    app: my-app
```

Service가 둘을 모두 찾습니다.

```
                  Service
               my-app-service
                     │
             selector: app=my-app
                     │
          ┌──────────┴──────────┐
          ↓                     ↓
    my-app-pod-1          my-app-pod-2
    app=my-app             app=my-app
```

여기서 중요한 것은 **Pod의 `name`이 아니라 `label`을 보고 선택한다는 것**입니다.

---

# 5. `name`은 고유 식별자, `label`은 그룹

예를 들어 Pod가 3개 있다고 해보겠습니다.

```
metadata:
  name: backend-1
  labels:
    app: backend
```

```
metadata:
  name: backend-2
  labels:
    app: backend
```

```
metadata:
  name: backend-3
  labels:
    app: backend
```

그러면:

```
name
────────────────
backend-1
backend-2
backend-3
```

각각 **서로 다른 리소스**입니다.

반면:

```
label
────────────────
app=backend
app=backend
app=backend
```

이므로 **하나의 그룹**으로 볼 수 있습니다.

그래서:

```
kubectl get pods -l app=backend
```

를 실행하면 세 Pod가 모두 조회됩니다.

---

# 6. `name`과 `label`은 같을 필요가 없습니다

다음처럼 해도 됩니다.

```
metadata:
  name: payment-api-7d8f9c
  labels:
    app: payment
    tier: backend
```

여기서:

```
name = payment-api-7d8f9c

labels:
  app  = payment
  tier = backend
```

전혀 다른 값입니다.

특히 실제 Kubernetes 환경에서는 Pod 이름이 자동으로 생성되기 때문에 더 명확합니다.

Deployment:

```
metadata:
  name: payment-api
```

Deployment가 생성한 Pod:

```
payment-api-7f8d6c9b7-xk92p
payment-api-7f8d6c9b7-m4t7q
payment-api-7f8d6c9b7-p2z8k
```

하지만 세 Pod 모두:

```
labels:
  app: payment
```

같은 label을 가질 수 있습니다.

그래서 Service는 Pod 이름을 알 필요 없이:

```
selector:
  app: payment
```

으로 세 Pod를 찾아갑니다.

---

## 7. 한 가지 더 중요한 점

`metadata.labels`와 `spec.selector`는 **서로 다른 필드**입니다.

예:

```
metadata:
  name: my-app
  labels:
    app: my-app

spec:
  selector:
    matchLabels:
      app: my-app
```

관계를 그림으로 보면:

```
Deployment
│
├── metadata.name
│       └── my-app
│
├── metadata.labels
│       └── app=my-app
│
└── spec.selector
        └── app=my-app
                 │
                 ↓
              Pod
              │
              └── metadata.labels
                      └── app=my-app
```
