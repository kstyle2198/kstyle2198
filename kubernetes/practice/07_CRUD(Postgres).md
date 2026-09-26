# [실습] CRUD (Postgres & Redis)

# 기본 실습

좋습니다. **기존 WebApp CRD 실습과 완전히 분리해서**, 이번에는 다음과 같은 구조로 새 클러스터 실습을 구성하는 것이 좋습니다.

## 1. 이번 실습의 최종 목표

```
                 Client
                   │
                   ▼
          ┌─────────────────┐
          │ FastAPI Backend  │
          │   worker-1       │
          └────────┬────────┘
                   │
          ┌────────┴────────┐
          ▼                 ▼
   ┌─────────────┐   ┌─────────────┐
   │ PostgreSQL  │   │    Redis    │
   │   DB        │   │ Cache       │
   └─────────────┘   └─────────────┘
```

Kubernetes 클러스터는:

```
control-plane
     │
     └── worker-1
           ├── FastAPI
           ├── PostgreSQL
           └── Redis
```

그리고 **코드 작성은 가능한 한 worker-1에서 수행**합니다.

이번 실습에서는 단순히 Redis/PostgreSQL을 설치하는 데서 끝내지 않고,

- PostgreSQL에 데이터 저장
- Redis에 캐시 저장
- FastAPI API 개발
- 데이터 입력
- 데이터 조회
- 데이터 수정
- 데이터 삭제
- Redis 캐시 확인
- Kubernetes Service를 통한 연결

까지 하나의 작은 프로젝트로 묶겠습니다.

---

# 2. 실습 단계 전체

전체 과정을 다음 순서로 진행하는 것을 권장합니다.

| 단계 | 내용 | 작업 위치 |
| --- | --- | --- |
| 1 | Kubernetes 클러스터 확인 | control-plane |
| 2 | 작업 디렉터리 생성 | worker-1 |
| 3 | PostgreSQL 설치 | control-plane |
| 4 | Redis 설치 | control-plane |
| 5 | PostgreSQL/Redis 상태 확인 | control-plane |
| 6 | FastAPI 프로젝트 생성 | worker-1 |
| 7 | PostgreSQL 연결 코드 작성 | worker-1 |
| 8 | CRUD API 작성 | worker-1 |
| 9 | Redis 캐시 추가 | worker-1 |
| 10 | Docker 이미지 생성 | worker-1 |
| 11 | Kubernetes Deployment/Service 작성 | worker-1 |
| 12 | FastAPI 배포 | control-plane |
| 13 | API 테스트 | worker-1 또는 PC |
| 14 | PostgreSQL 데이터 확인 | control-plane |
| 15 | Redis 캐시 확인 | control-plane |
| 16 | 장애/Pod 재시작 실습 | control-plane |

---

# 3. 1단계 — 클러스터 확인

**control-plane에서 실행**

```bash
kubectl get nodes -o wide
```

정상적으로 다음과 비슷해야 합니다.

```
NAME           STATUS   ROLES           AGE
control-plane  Ready    control-plane   ...
worker-1       Ready    <none>          ...
```

그리고 기본 Namespace를 확인합니다.

```bash
kubectl get ns
```

---

# 4. 2단계 — 이번 프로젝트 디렉터리 생성

이번에는 **worker-1에서 코드 작업**을 진행합니다.

worker-1:

```bash
mkdir -p ~/k8s-crud-app
cd ~/k8s-crud-app
```

앞으로 대략 다음과 같은 구조가 됩니다.

```
k8s-crud-app/
├── app/
│   ├── main.py
│   ├── database.py
│   ├── models.py
│   └── schemas.py
├── requirements.txt
├── Dockerfile
└── k8s/
    ├── namespace.yaml
    ├── postgres.yaml
    ├── redis.yaml
    └── backend.yaml
```

---

# 5. 3단계 — PostgreSQL 설치

이번 실습에서는 우선 **Kubernetes YAML을 이용해서 PostgreSQL을 직접 배포**하겠습니다.

다만 실습 목적상 다음 단계로 발전시킬 수 있습니다.

```
1단계
PostgreSQL Deployment
        ↓
2단계
PersistentVolume
        ↓
3단계
StatefulSet
        ↓
4단계
백업/복구
```

처음부터 StatefulSet과 PVC까지 복잡하게 들어가기보다는 **CRUD가 먼저 정상 동작하도록 구성한 뒤 저장소를 개선**하는 것이 좋습니다.

### Namespace 생성

control-plane:

```bash
kubectl create namespace crud-app
```

확인:

```bash
kubectl get ns
```

---

# 6. PostgreSQL Deployment

control-plane에서 파일을 하나 만듭니다.

```bash
vi postgres.yaml
```

내용:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: crud-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16
          ports:
            - containerPort: 5432

          env:
            - name: POSTGRES_DB
              value: cruddb

            - name: POSTGRES_USER
              value: appuser

            - name: POSTGRES_PASSWORD
              value: apppassword

---
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: crud-app
spec:
  selector:
    app: postgres

  ports:
    - port: 5432
      targetPort: 5432

  type: ClusterIP
```

적용:

```bash
kubectl apply -f postgres.yaml
```

확인:

```bash
kubectl get pods -n crud-app
```

예:

```
NAME                        READY   STATUS    RESTARTS   AGE
postgres-xxxxxxxxxx-xxxxx   1/1     Running   0          20s
```

Service 확인:

```bash
kubectl get svc -n crud-app
```

```
NAME       TYPE        CLUSTER-IP      PORT(S)
postgres   ClusterIP   10.xxx.xxx.xxx  5432/TCP
```

---

# 7. PostgreSQL 동작 테스트

control-plane:

```bash
kubectl exec -it \
  -n crud-app \
  deployment/postgres \
  -- psql -U appuser -d cruddb
```

PostgreSQL에 들어가면:

```sql
\l
```

데이터베이스 목록을 볼 수 있습니다.

종료:

```sql
\q
```

---

# 8. 4단계 — Redis 설치

이번에는 Redis입니다.

control-plane:

```bash
vi redis.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: crud-app
spec:
  replicas: 1

  selector:
    matchLabels:
      app: redis

  template:
    metadata:
      labels:
        app: redis

    spec:
      containers:
        - name: redis
          image: redis:7
          ports:
            - containerPort: 6379

---
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: crud-app
spec:
  selector:
    app: redis

  ports:
    - port: 6379
      targetPort: 6379

  type: ClusterIP
```

적용:

```bash
kubectl apply -f redis.yaml
```

확인:

```bash
kubectl get pods -n crud-app
```

정상이라면:

```
postgres-xxxxx   1/1   Running
redis-xxxxx      1/1   Running
```

Service:

```bash
kubectl get svc -n crud-app
```

---

# 9. Redis 테스트

Redis Pod에 접속합니다.

```bash
kubectl exec -it \
  -n crud-app \
  deployment/redis \
  -- redis-cli
```

Redis에서:

```
SET test "hello"
```

결과:

```
OK
```

조회:

```
GET test
```

결과:

```
"hello"
```

삭제:

```
DEL test
```

종료:

```
exit
```

이 단계까지 완료하면:

```
Kubernetes
│
├── PostgreSQL
│     └── cruddb
│
└── Redis
      └── cache
```

가 정상적으로 동작하는 것입니다.

---

# 10. 5단계 — FastAPI 프로젝트

이제부터는 **worker-1에서 코드 작성**을 합니다.

```bash
cd ~/k8s-crud-app
mkdir -p app
cd app
```

Python 패키지:

```bash
cd ~/k8s-crud-app
vi requirements.txt
```

내용:

```
fastapi
uvicorn[standard]
psycopg2-binary
redis
sqlalchemy
```

---

# 11. PostgreSQL 연결 코드

worker-1:

```bash
vi app/database.py
```

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, declarative_base

DATABASE_URL = (
    "postgresql://appuser:apppassword"
    "@postgres.crud-app.svc.cluster.local:5432/cruddb"
)

engine = create_engine(DATABASE_URL)

SessionLocal = sessionmaker(
    autocommit=False,
    autoflush=False,
    bind=engine
)

Base = declarative_base()
```

여기서 중요한 부분은:

```
postgres.crud-app.svc.cluster.local
```

입니다.

Kubernetes 내부 DNS를 이용하여 PostgreSQL Service에 접근합니다.

즉 FastAPI에서:

```
postgres
```

라는 Kubernetes Service를 통해 PostgreSQL에 접근하게 됩니다.

---

# 12. 데이터 모델

이번 실습에서는 아주 간단하게 `Item`을 사용하겠습니다.

```
id
name
description
price
```

worker-1:

```bash
vi app/models.py
```

```python
from sqlalchemy import Column, Integer, String, Float
from .database import Base

class Item(Base):
    __tablename__ = "items"

    id = Column(Integer, primary_key=True, index=True)
    name = Column(String(100), nullable=False)
    description = Column(String(255))
    price = Column(Float, nullable=False)
```

---

# 13. Pydantic Schema

```bash
vi app/schemas.py
```

```python
from pydantic import BaseModel

class ItemCreate(BaseModel):
    name: str
    description: str | None = None
    price: float

class ItemResponse(ItemCreate):
    id: int

    class Config:
        from_attributes = True
```

---

# 14. CRUD API

이제 핵심인 FastAPI입니다.

```bash
vi app/main.py
```

```python
from fastapi import FastAPI, Depends, HTTPException
from sqlalchemy.orm import Session

from .database import Base, engine, SessionLocal
from .models import Item
from .schemas import ItemCreate, ItemResponse

import redis

app = FastAPI(
    title="Kubernetes CRUD API",
    version="1.0"
)

# DB 테이블 생성
Base.metadata.create_all(bind=engine)

# Redis
redis_client = redis.Redis(
    host="redis.crud-app.svc.cluster.local",
    port=6379,
    decode_responses=True
)

def get_db():
    db = SessionLocal()

    try:
        yield db
    finally:
        db.close()

@app.get("/")
def root():
    return {
        "message": "Kubernetes CRUD API"
    }

@app.get("/health")
def health():
    return {
        "status": "ok"
    }

# CREATE
@app.post("/items", response_model=ItemResponse)
def create_item(
    item: ItemCreate,
    db: Session = Depends(get_db)
):
    db_item = Item(
        name=item.name,
        description=item.description,
        price=item.price
    )

    db.add(db_item)
    db.commit()
    db.refresh(db_item)

    # Redis cache
    redis_client.set(
        f"item:{db_item.id}",
        db_item.name
    )

    return db_item

# READ
@app.get("/items/{item_id}", response_model=ItemResponse)
def get_item(
    item_id: int,
    db: Session = Depends(get_db)
):
    db_item = db.query(Item).filter(
        Item.id == item_id
    ).first()

    if not db_item:
        raise HTTPException(
            status_code=404,
            detail="Item not found"
        )

    return db_item

# UPDATE
@app.put("/items/{item_id}", response_model=ItemResponse)
def update_item(
    item_id: int,
    item: ItemCreate,
    db: Session = Depends(get_db)
):
    db_item = db.query(Item).filter(
        Item.id == item_id
    ).first()

    if not db_item:
        raise HTTPException(
            status_code=404,
            detail="Item not found"
        )

    db_item.name = item.name
    db_item.description = item.description
    db_item.price = item.price

    db.commit()
    db.refresh(db_item)

    redis_client.set(
        f"item:{db_item.id}",
        db_item.name
    )

    return db_item

# DELETE
@app.delete("/items/{item_id}")
def delete_item(
    item_id: int,
    db: Session = Depends(get_db)
):
    db_item = db.query(Item).filter(
        Item.id == item_id
    ).first()

    if not db_item:
        raise HTTPException(
            status_code=404,
            detail="Item not found"
        )

    db.delete(db_item)
    db.commit()

    redis_client.delete(
        f"item:{item_id}"
    )

    return {
        "message": "Item deleted",
        "id": item_id
    }
```

---

# 15. 이번 API에서 배우게 되는 것

API 구조는 다음과 같습니다.

| HTTP | API | 기능 |
| --- | --- | --- |
| GET | `/` | API 확인 |
| GET | `/health` | 상태 확인 |
| POST | `/items` | 데이터 입력 |
| GET | `/items/{id}` | 데이터 조회 |
| PUT | `/items/{id}` | 데이터 수정 |
| DELETE | `/items/{id}` | 데이터 삭제 |

즉 전형적인 **REST CRUD API** 실습입니다.

---

# 16. Docker 이미지 생성

worker-1에서:

```bash
cd ~/k8s-crud-app
```

Dockerfile:

```bash
vi Dockerfile
```

```docker
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY app ./app

EXPOSE 8000

CMD [
    "uvicorn",
    "app.main:app",
    "--host",
    "0.0.0.0",
    "--port",
    "8000"
]
```

이미지 생성:

```bash
docker build -t k8s-crud-api:1.0 .
```

---

# 17. Kubernetes에서 이미지 사용

여기서 **중요한 부분**이 하나 있습니다.

worker-1에서 만든 Docker 이미지를 Kubernetes가 실행하려면 이미지가 worker-1의 container runtime에서 사용할 수 있어야 합니다.

처음 실습에서는 가장 간단하게:

```
worker-1
   │
   └── Docker build
          │
          ▼
     k8s-crud-api:1.0
```

그리고 Kubernetes Pod가 worker-1에서 실행되도록 구성할 수 있습니다.

나중에는 여기서:

```
worker-1
   │
   │ docker build
   ▼
Private Registry
   │
   ├── worker-1 pull
   └── worker-2 pull
```

형태로 발전시키면 **Harbor/Private Registry + Kubernetes 배포 실습**으로 자연스럽게 확장됩니다.

---

# 18. FastAPI Deployment

worker-1에서 YAML을 작성합니다.

```bash
mkdir -p ~/k8s-crud-app/k8s
cd ~/k8s-crud-app/k8s
```

```bash
vi backend.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: crud-api
  namespace: crud-app

spec:
  replicas: 1

  selector:
    matchLabels:
      app: crud-api

  template:
    metadata:
      labels:
        app: crud-api

    spec:
      nodeSelector:
        kubernetes.io/hostname: worker-1

      containers:
        - name: api
          image: k8s-crud-api:1.0

          imagePullPolicy: IfNotPresent

          ports:
            - containerPort: 8000

---
apiVersion: v1
kind: Service
metadata:
  name: crud-api
  namespace: crud-app

spec:
  selector:
    app: crud-api

  ports:
    - port: 8000
      targetPort: 8000

  type: NodePort
```

---

# 19. 배포

control-plane에서:

```bash
kubectl apply -f backend.yaml
```

확인:

```bash
kubectl get pods -n crud-app -o wide
```

예:

```
NAME                        READY   STATUS    NODE
crud-api-xxxxxxxxxx-xxxxx   1/1     Running   worker-1
postgres-xxxxxxxxxx-xxxxx   1/1     Running   worker-1
redis-xxxxxxxxxx-xxxxx      1/1     Running   worker-1
```

---

# 20. Service 확인

```bash
kubectl get svc -n crud-app
```

예:

```
NAME        TYPE        CLUSTER-IP      PORT(S)
postgres    ClusterIP   10.100.x.x      5432/TCP
redis       ClusterIP   10.100.x.x      6379/TCP
crud-api    NodePort    10.100.x.x      8000:30080/TCP
```

그러면 외부에서:

```
http://worker-1-IP:30080
```

형태로 접근할 수 있습니다.

---

# 21. Swagger UI

FastAPI의 장점 중 하나가 자동 API 문서입니다.

브라우저에서:

```
http://worker-1-IP:30080/docs
```

접속합니다.

그러면:

```
Kubernetes CRUD API

GET     /
GET     /health
POST    /items
GET     /items/{item_id}
PUT     /items/{item_id}
DELETE  /items/{item_id}
```

를 브라우저에서 직접 테스트할 수 있습니다.

---

# 22. CRUD 실습

## CREATE

Swagger의 `POST /items`를 실행합니다.

입력:

```json
{
  "name": "keyboard",
  "description": "Mechanical keyboard",
  "price": 89000
}
```

결과:

```json
{
  "id": 1,
  "name": "keyboard",
  "description": "Mechanical keyboard",
  "price": 89000
}
```

---

## READ

```
GET /items/1
```

결과:

```json
{
  "id": 1,
  "name": "keyboard",
  "description": "Mechanical keyboard",
  "price": 89000
}
```

---

## UPDATE

```
PUT /items/1
```

입력:

```json
{
  "name": "gaming keyboard",
  "description": "RGB mechanical keyboard",
  "price": 120000
}
```

결과:

```json
{
  "id": 1,
  "name": "gaming keyboard",
  "description": "RGB mechanical keyboard",
  "price": 120000
}
```

---

## DELETE

```
DELETE /items/1
```

결과:

```json
{
  "message": "Item deleted",
  "id": 1
}
```

---

# 23. PostgreSQL에서 실제 데이터 확인

control-plane:

```bash
kubectl exec -it \
  -n crud-app \
  deployment/postgres \
  -- psql -U appuser -d cruddb
```

그리고:

```sql
SELECT * FROM items;
```

이를 통해 중요한 것을 확인할 수 있습니다.

```
FastAPI
   │
   │ SQL
   ▼
PostgreSQL
   │
   ▼
items table
```

즉 Swagger에서 입력한 데이터가 실제 PostgreSQL에 저장됩니다.

---

# 24. Redis 확인

Redis:

```bash
kubectl exec -it \
  -n crud-app \
  deployment/redis \
  -- redis-cli
```

확인:

```
KEYS *
```

예:

```
1) "item:1"
```

조회:

```
GET item:1
```

결과:

```
"gaming keyboard"
```

이렇게 하면 Redis가 단순히 설치된 것이 아니라 **FastAPI와 실제 연결되어 사용되는 것**까지 확인할 수 있습니다.

---

# 25. 이 실습에서 중요한 Kubernetes 구조

최종적으로 다음 구조가 됩니다.

```
                    Browser
                       │
                       │ :30080
                       ▼
              ┌─────────────────┐
              │    crud-api      │
              │    FastAPI       │
              │    worker-1      │
              └────────┬────────┘
                       │
             ┌─────────┴──────────┐
             │                    │
             ▼                    ▼
      ┌─────────────┐      ┌─────────────┐
      │ PostgreSQL  │      │    Redis    │
      │   :5432     │      │   :6379     │
      └─────────────┘      └─────────────┘
```

그리고 Kubernetes 관점에서는:

```
Namespace: crud-app

Deployment
├── crud-api
├── postgres
└── redis

Service
├── crud-api       NodePort
├── postgres       ClusterIP
└── redis          ClusterIP
```

---

# 26. 이후 실습을 확장하면 매우 재미있습니다

이번 실습은 여기서 끝내기보다 다음 순서로 확장하는 것을 강력하게 추천합니다.

### Phase 1 — 기본 CRUD

```
FastAPI
   ↓
PostgreSQL
```

### Phase 2 — Redis Cache

```
FastAPI
 ├── PostgreSQL
 └── Redis
```

### Phase 3 — Redis 캐시 효과 확인

```
GET /items/1

첫 요청
→ PostgreSQL

두 번째 요청
→ Redis
```

실제로 응답 시간을 측정합니다.

### Phase 4 — PostgreSQL StatefulSet + PVC

```
PostgreSQL
    │
    ▼
PersistentVolume
```

Pod를 삭제해도 데이터가 살아있는지 테스트합니다.

### Phase 5 — FastAPI Scale-out

```
crud-api
replicas: 3
```

그리고:

```bash
kubectl get pods -n crud-app -o wide
```

로 여러 worker에 Pod가 배치되는 것을 확인합니다.

### Phase 6 — HPA

CPU 사용량에 따라:

```
1 Pod
 ↓
2 Pod
 ↓
3 Pod
 ↓
...
```

자동 확장합니다.

### Phase 7 — Redis 장애 실습

Redis Pod를 삭제합니다.

```bash
kubectl delete pod -n crud-app -l app=redis
```

FastAPI가 어떻게 반응하는지 확인합니다.

### Phase 8 — PostgreSQL 장애/복구

PostgreSQL Pod를 삭제하고 데이터 보존 여부를 테스트합니다.

### Phase 9 — Prometheus + Grafana

이미 실습하셨던 모니터링 개념을 이번 프로젝트에 붙이면 좋습니다.

```
FastAPI
  │
  └── /metrics
          │
          ▼
      Prometheus
          │
          ▼
       Grafana
```

### Phase 10 — 최종 프로젝트

최종적으로는:

```
                    Browser
                       │
                       ▼
                  FastAPI API
                       │
             ┌─────────┴─────────┐
             ▼                   ▼
        PostgreSQL              Redis
             │                   │
             └─────────┬─────────┘
                       ▼
                  Prometheus
                       │
                       ▼
                    Grafana
```

까지 만들어 보면 **Kubernetes + Backend + Database + Cache + Monitoring**을 한 번에 경험할 수 있습니다.

**우선 다음 단계에서는 위 전체를 한꺼번에 진행하지 말고, `1단계 클러스터 확인 → 2단계 PostgreSQL → 3단계 Redis`부터 실제 명령어를 하나씩 실행하면서 진행하는 방식**이 가장 좋습니다. 이후 제가 그 결과를 보고 다음 단계로 넘어가면 됩니다.

# Redis Cache 조회 실습

# Redis Cache 실습 — 간명한 전체 절차

현재 환경:

```
Kubernetes
├── control-plane
└── worker-1
      └── FastAPI 코드 작업

crud-app Namespace
├── crud-api     FastAPI
├── postgres     PostgreSQL
└── redis        Redis
```

목표:

```
GET /items/1
      ↓
Redis 확인
 ┌────┴────┐
HIT       MISS
 ↓          ↓
Redis    PostgreSQL
 ↓          ↓
응답     Redis 저장
            ↓
           응답
```

---

## 1. Redis 상태 확인

**control-plane**

```bash
kubectl get pods -n crud-app
kubectl get svc -n crud-app
```

Redis가 `Running`인지 확인합니다.

```bash
kubectl exec -it \
  -n crud-app \
  deployment/redis \
  -- redis-cli
```

```
KEYS *
```

종료:

```
exit
```

---

# 2. FastAPI Redis 연결 확인

**worker-1**

```bash
cd ~/k8s-crud-app
```

`main.py`에 Redis 연결이 있어야 합니다.

```python
import json
import redis

redis_client = redis.Redis(
    host="redis.crud-app.svc.cluster.local",
    port=6379,
    decode_responses=True
)
```

---

# 3. CREATE API의 Redis 저장 형식 수정

`POST /items`에서 Redis에 **JSON 전체 객체**를 저장합니다.

```python
result = {
    "id": db_item.id,
    "name": db_item.name,
    "description": db_item.description,
    "price": db_item.price
}

redis_client.set(
    f"item:{db_item.id}",
    json.dumps(result),
    ex=60
)

print(
    f"Redis CACHE SET: item:{db_item.id}",
    flush=True
)
```

`ex=60` → 캐시가 60초 후 자동 삭제됩니다.

---

# 4. GET API에 Cache 적용

`GET /items/{item_id}`:

```python
@app.get("/items/{item_id}", response_model=ItemResponse)
def get_item(
    item_id: int,
    db: Session = Depends(get_db)
):
    cache_key = f"item:{item_id}"

    # 1. Redis 확인
    cached_item = redis_client.get(cache_key)

    if cached_item:
        print(
            f"Redis CACHE HIT: {cache_key}",
            flush=True
        )

        return json.loads(cached_item)

    # 2. Cache MISS → PostgreSQL 조회
    print(
        f"Redis CACHE MISS: {cache_key}",
        flush=True
    )

    db_item = db.query(Item).filter(
        Item.id == item_id
    ).first()

    if not db_item:
        raise HTTPException(
            status_code=404,
            detail="Item not found"
        )

    result = {
        "id": db_item.id,
        "name": db_item.name,
        "description": db_item.description,
        "price": db_item.price
    }

    # 3. PostgreSQL 결과를 Redis에 저장
    redis_client.set(
        cache_key,
        json.dumps(result),
        ex=60
    )

    print(
        f"Redis CACHE SET: {cache_key}",
        flush=True
    )

    return result
```

---

# 5. UPDATE / DELETE도 Redis와 동기화

### UPDATE

```python
redis_client.set(
    f"item:{db_item.id}",
    json.dumps({
        "id": db_item.id,
        "name": db_item.name,
        "description": db_item.description,
        "price": db_item.price
    }),
    ex=60
)
```

### DELETE

```python
redis_client.delete(
    f"item:{item_id}"
)
```

즉:

```
POST → PostgreSQL INSERT + Redis SET
PUT  → PostgreSQL UPDATE + Redis SET
DELETE → PostgreSQL DELETE + Redis DELETE
```

---

# 6. Redis 초기화

테스트 전에 캐시를 비웁니다.

**control-plane**

```bash
kubectl exec \
  -n crud-app \
  deployment/redis \
  -- redis-cli FLUSHALL
```

확인:

```bash
kubectl exec \
  -n crud-app \
  deployment/redis \
  -- redis-cli KEYS '*'
```

결과:

```
(empty array)
```

---

# 7. Docker 이미지 재빌드

코드를 수정했으므로 **worker-1에서 반드시 이미지를 다시 빌드**합니다.

```bash
cd ~/k8s-crud-app

docker build -t k8s-crud-api:2.1 .
```

---

# 8. Kubernetes Deployment 이미지 변경

`k8s/backend.yaml`:

```yaml
image: k8s-crud-api:2.1
```

기존:

```yaml
image: k8s-crud-api:2.0
```

에서 변경합니다.

**control-plane**

```bash
kubectl apply -f ~/k8s-crud-app/k8s/backend.yaml
```

확인:

```bash
kubectl rollout status \
  deployment/crud-api \
  -n crud-app
```

---

# 9. FastAPI 로그 확인

**control-plane**

```bash
kubectl logs -f \
  -n crud-app \
  deployment/crud-api
```

이 상태에서 Swagger에서 API를 실행합니다.

---

# 10. 데이터 생성

Swagger:

```
POST /items
```

```json
{
  "name": "keyboard",
  "description": "Mechanical keyboard",
  "price": 89000
}
```

예를 들어 `id=1` 생성.

Redis 확인:

```bash
kubectl exec \
  -n crud-app \
  deployment/redis \
  -- redis-cli GET item:1
```

JSON 데이터가 나오면 정상입니다.

---

# 11. 첫 번째 GET — Cache MISS

Swagger:

```
GET /items/1
```

처음에는 Redis에 없으므로:

```
Redis CACHE MISS: item:1
Redis CACHE SET: item:1
```

흐름:

```
GET
 ↓
Redis MISS
 ↓
PostgreSQL
 ↓
Redis 저장
 ↓
응답
```

---

# 12. 두 번째 GET — Cache HIT

다시:

```
GET /items/1
```

이번에는:

```
Redis CACHE HIT: item:1
```

흐름:

```
GET
 ↓
Redis HIT
 ↓
응답
```

**PostgreSQL에는 조회하지 않습니다.**

---

# 13. 세 번째 GET

다시 실행:

```
GET /items/1
```

예상 로그:

```
Redis CACHE HIT: item:1
```

즉:

```
1회차 → MISS → PostgreSQL → Redis SET
2회차 → HIT  → Redis
3회차 → HIT  → Redis
```

가 핵심 실습 결과입니다.

---

# 14. TTL 확인

Redis:

```bash
kubectl exec -it \
  -n crud-app \
  deployment/redis \
  -- redis-cli
```

```
TTL item:1
```

예:

```
(integer) 48
```

60초가 지나면:

```
(integer) -2
```

캐시가 삭제됩니다.

다시:

```
GET /items/1
```

하면:

```
MISS
 ↓
PostgreSQL
 ↓
Redis SET
```

이 됩니다.

---

# 15. Redis 장애 테스트

Redis Pod 삭제:

```bash
kubectl delete pod \
  -n crud-app \
  -l app=redis
```

자동으로 새 Pod가 생성됩니다.

```bash
kubectl get pods -n crud-app -w
```

새 Redis는 캐시가 비어 있습니다.

따라서:

```
GET /items/1
```

→ `CACHE MISS`

→ PostgreSQL 조회

→ Redis 재저장

되는 것을 확인합니다.

---

# 최종 실습 결과

```
             GET /items/1
                    │
                    ▼
                  Redis
                 /     \
              HIT       MISS
               │          │
               │          ▼
               │      PostgreSQL
               │          │
               │          ▼
               │        Redis SET
               │          │
               └────┬─────┘
                    ▼
                  응답
```

### 핵심 확인 항목

| 실습 | 확인 내용 |
| --- | --- |
| Redis 설치 | Redis Pod/Service |
| Cache SET | PostgreSQL 결과를 Redis에 저장 |
| Cache MISS | Redis → PostgreSQL |
| Cache HIT | Redis에서 바로 응답 |
| TTL | 60초 후 자동 삭제 |
| UPDATE | PostgreSQL + Redis 동기화 |
| DELETE | PostgreSQL + Redis 삭제 |
| Redis Pod 삭제 | 캐시는 사라지지만 Redis Pod 자동 복구 |
| Cache 재생성 | MISS → PostgreSQL → Redis SET |

**이 Phase의 핵심은 `1회차 GET = MISS`, `2·3회차 GET = HIT`를 로그로 직접 확인하는 것입니다.**

# StatefulSet 배포 실습

좋습니다. **Phase 4의 목적은 "PostgreSQL Pod를 삭제해도 데이터가 유지되는 것"을 직접 확인하는 것**입니다.

현재 Phase 1~3의 PostgreSQL은 `Deployment` + Pod 내부 저장공간이므로 Pod 삭제 시 데이터가 사라질 수 있습니다. 이를 `StatefulSet + PVC` 구조로 변경합니다.

```
기존
PostgreSQL Deployment
      ↓
   PostgreSQL Pod
      ↓
  Pod 내부 저장공간
      ↓
Pod 삭제 → 데이터 소실 ❌

Phase 4
PostgreSQL StatefulSet
      ↓
PostgreSQL Pod
      ↓
    PVC
      ↓
    PV
      ↓
Pod 삭제 → 데이터 유지 ✅
```

---

# 1. 실습 최종 구조

이번 단계에서는 다음과 같이 구성합니다.

```
crud-app
│
├── crud-api
│
├── redis
│
└── postgres
      │
      └── StatefulSet
             │
             └── postgres-0
                    │
                    ▼
                   PVC
                    │
                    ▼
                   PV
```

그리고 중요한 관계는:

```
postgres-0
    │
    └── postgres-data-postgres-0
                  │
                  ▼
                 PV
```

입니다.

---

# 2. 현재 PostgreSQL 상태 확인

먼저 **control-plane**에서 현재 구성을 확인합니다.

```bash
kubectl get deployment -n crud-app
```

아마:

```
NAME         READY
crud-api     1/1
postgres     1/1
redis        1/1
```

처럼 나올 것입니다.

PostgreSQL Deployment:

```bash
kubectl get deployment postgres -n crud-app -o yaml
```

현재는:

```
Deployment
   ↓
postgres Pod
```

구조입니다.

---

# 3. 현재 데이터 확인

StatefulSet으로 변경하기 전에 현재 DB 데이터를 확인합니다.

```bash
kubectl exec -it \
  -n crud-app \
  deployment/postgres \
  -- psql -U appuser -d cruddb
```

```sql
SELECT * FROM items;
```

예:

```
 id |   name   |     description      | price
----+----------+----------------------+-------
  1 | keyboard | Mechanical keyboard | 89000
  2 | mouse    | Wireless mouse      | 35000
```

**이 데이터를 StatefulSet 전환 후에도 유지시키는 것이 이번 실습의 목표입니다.**

---

# 4. PVC 개념 이해

먼저 Kubernetes 저장 구조를 간단히 이해하면 됩니다.

```
Pod
 │
 │ mount
 ▼
PVC
 │
 │ request
 ▼
PV
 │
 ▼
실제 저장공간
```

### PV

PersistentVolume입니다.

실제 저장 공간을 나타냅니다.

### PVC

PersistentVolumeClaim입니다.

Pod가:

> "나 PostgreSQL 데이터를 저장할 공간이 필요합니다."
> 

라고 요청하는 객체입니다.

---

# 5. 현재 클러스터의 StorageClass 확인

**control-plane**

```bash
kubectl get storageclass
```

예를 들어:

```
NAME                   PROVISIONER
local-path             rancher.io/local-path
```

또는 다른 StorageClass가 나올 수 있습니다.

기본 StorageClass 확인:

```bash
kubectl get storageclass
```

`(default)`가 붙은 StorageClass가 있다면 그것을 사용하면 됩니다.

---

# 6. PostgreSQL StatefulSet YAML 작성

이번에는 기존 `postgres.yaml`을 그대로 수정하기보다 새로운 파일을 만드는 것을 권장합니다.

**worker-1**

```bash
cd ~/k8s-crud-app
mkdir -p k8s
vi k8s/postgres-statefulset.yaml
```

다음 내용을 사용합니다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: crud-app

spec:
  clusterIP: None

  selector:
    app: postgres

  ports:
    - port: 5432
      targetPort: 5432

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: crud-app

spec:
  serviceName: postgres
  replicas: 1

  selector:
    matchLabels:
      app: postgres

  template:
    metadata:
      labels:
        app: postgres

    spec:
      containers:
        - name: postgres
          image: postgres:16

          ports:
            - containerPort: 5432

          env:
            - name: POSTGRES_DB
              value: cruddb

            - name: POSTGRES_USER
              value: appuser

            - name: POSTGRES_PASSWORD
              value: apppassword

          volumeMounts:
            - name: postgres-data
              mountPath: /var/lib/postgresql/data

  volumeClaimTemplates:

    - metadata:
        name: postgres-data

      spec:
        accessModes:
          - ReadWriteOnce

        resources:
          requests:
            storage: 5Gi
```

---

# 7. 중요한 부분 설명

가장 중요한 부분은 이것입니다.

```yaml
volumeMounts:
  - name: postgres-data
    mountPath: /var/lib/postgresql/data
```

PostgreSQL의 실제 데이터 디렉터리를 PVC와 연결합니다.

그리고:

```yaml
volumeClaimTemplates:
```

를 사용하면 StatefulSet이 자동으로 PVC를 생성합니다.

예를 들어:

```
StatefulSet
   │
   ▼
postgres-0
   │
   ▼
postgres-data-postgres-0
   │
   ▼
PV
```

가 만들어집니다.

---

# 8. 기존 PostgreSQL Deployment 중지

**중요합니다.**

기존 PostgreSQL Deployment와 새로운 StatefulSet이 동시에 PostgreSQL을 실행하면 안 됩니다.

현재 Deployment를 확인합니다.

```bash
kubectl get deployment postgres -n crud-app
```

기존 Deployment를 삭제합니다.

```bash
kubectl delete deployment postgres -n crud-app
```

그리고 확인:

```bash
kubectl get pods -n crud-app
```

PostgreSQL Pod가 없어져야 합니다.

---

# 9. 기존 Service 처리

기존 PostgreSQL Service는 그대로 사용할 수도 있지만, StatefulSet에서는 Headless Service를 사용하는 것이 좋습니다.

기존 Service를 삭제합니다.

```bash
kubectl delete svc postgres -n crud-app
```

**주의:** Service 삭제는 DB 데이터 삭제와 관계없습니다.

---

# 10. StatefulSet 배포

**control-plane**

worker-1에서 작성한 YAML을 control-plane에서 사용할 수 있도록 복사했다면:

```bash
kubectl apply -f postgres-statefulset.yaml
```

또는 파일을 worker-1에서 control-plane으로 복사한 경우:

```bash
kubectl apply -f ~/k8s-crud-app/k8s/postgres-statefulset.yaml
```

---

# 11. Pod 확인

```bash
kubectl get pods -n crud-app
```

이번에는 이름이 Deployment와 다릅니다.

기존:

```
postgres-65854df76d-xxxxx
```

StatefulSet:

```
postgres-0
```

입니다.

정상:

```
postgres-0    1/1    Running
```

---

# 12. StatefulSet 확인

```bash
kubectl get statefulset -n crud-app
```

결과:

```
NAME       READY
postgres   1/1
```

---

# 13. PVC 확인

가장 중요한 확인입니다.

```bash
kubectl get pvc -n crud-app
```

예:

```
NAME                     STATUS   VOLUME   CAPACITY
postgres-data-postgres-0 Bound    pvc-xxx  5Gi
```

`STATUS`가:

```
Bound
```

여야 합니다.

---

# 14. PV 확인

```bash
kubectl get pv
```

예:

```
NAME                                       CAPACITY   STATUS
pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx   5Gi        Bound
```

현재 구조:

```
postgres-0
    │
    ▼
postgres-data-postgres-0
    │
    ▼
PVC
    │
    ▼
PV
```

가 완성된 것입니다.

---

# 15. PostgreSQL 기동 확인

```bash
kubectl logs -n crud-app postgres-0
```

정상적으로 다음과 비슷한 메시지를 확인합니다.

```
database system is ready to accept connections
```

---

# 16. 새로운 PostgreSQL이 정상인지 확인

```bash
kubectl exec -it \
  -n crud-app \
  postgres-0 \
  -- psql -U appuser -d cruddb
```

테이블:

```sql
\dt
```

여기서 중요한 상황이 발생합니다.

**기존 Deployment의 DB를 삭제하고 새로운 PVC를 처음 만든 경우 기존 데이터는 자동으로 PVC에 들어가지 않습니다.**

즉:

```
기존 Deployment
   ↓
기존 Pod 내부 데이터
   ↓
Deployment 삭제
   ↓
데이터 삭제

새 StatefulSet
   ↓
새 PVC
   ↓
새로운 PostgreSQL
```

입니다.

따라서 이번 실습에서는 **먼저 새로운 StatefulSet + PVC가 정상적으로 동작하는지 확인한 후 데이터를 새로 넣는 방식**이 가장 안전합니다.

---

# 17. 테스트 데이터 생성

Swagger에서:

```
POST /items
```

```json
{
  "name": "persistent-keyboard",
  "description": "PVC test",
  "price": 100000
}
```

확인:

```bash
kubectl exec -it \
  -n crud-app \
  postgres-0 \
  -- psql -U appuser -d cruddb \
  -c "SELECT * FROM items;"
```

예:

```
 id |        name         | description | price
----+---------------------+-------------+--------
  1 | persistent-keyboard | PVC test    | 100000
```

---

# 18. 핵심 실습 — PostgreSQL Pod 삭제

이제 이번 Phase의 핵심입니다.

**control-plane**

```bash
kubectl delete pod postgres-0 -n crud-app
```

결과:

```
pod "postgres-0" deleted
```

---

# 19. Pod 자동 복구 확인

```bash
kubectl get pods -n crud-app -w
```

잠시 후:

```
postgres-0    0/1   ContainerCreating
postgres-0    1/1   Running
```

다시 **동일한 이름**인:

```
postgres-0
```

이 생성됩니다.

Deployment와 중요한 차이입니다.

Deployment:

```
postgres-65854df76d-xxxxx
```

StatefulSet:

```
postgres-0
```

---

# 20. PVC 확인

Pod가 복구된 후:

```bash
kubectl get pvc -n crud-app
```

여전히:

```
postgres-data-postgres-0   Bound
```

입니다.

즉:

```
Pod 삭제
   ↓
PVC 유지
   ↓
새 postgres-0
   ↓
기존 PVC 연결
```

입니다.

---

# 21. 데이터 유지 확인

다시 PostgreSQL에 접속합니다.

```bash
kubectl exec -it \
  -n crud-app \
  postgres-0 \
  -- psql -U appuser -d cruddb
```

```sql
SELECT * FROM items;
```

아까 넣었던:

```
persistent-keyboard
```

가 **그대로 존재해야 합니다.**

이것이 이번 Phase의 핵심 성공 조건입니다.

---

# 22. Redis와 비교

이번 실습에서는 Redis와 PostgreSQL의 차이도 명확하게 볼 수 있습니다.

Redis는 현재:

```
Redis Pod
   ↓
컨테이너 저장공간
```

구조라면 Pod 삭제 후:

```
Redis Pod 삭제
   ↓
새 Redis Pod
   ↓
Cache 초기화
```

됩니다.

반면 PostgreSQL은:

```
PostgreSQL Pod
      ↓
PVC
      ↓
PV
```

이므로:

```
PostgreSQL Pod 삭제
      ↓
새 PostgreSQL Pod
      ↓
기존 PVC 연결
      ↓
데이터 유지
```

됩니다.

---

# 23. PostgreSQL Service 확인

현재 Service는 Headless Service이므로:

```bash
kubectl get svc -n crud-app postgres
```

예:

```
NAME       TYPE        CLUSTER-IP   PORT(S)
postgres   ClusterIP   None         5432/TCP
```

`CLUSTER-IP`가:

```
None
```

인 것이 정상입니다.

FastAPI에서는 기존처럼:

```python
postgres.crud-app.svc.cluster.local
```

을 사용할 수 있습니다.

따라서 **FastAPI 코드는 변경하지 않아도 됩니다.**

---

# 24. 전체 테스트

이번 Phase에서는 다음 순서로 테스트하면 됩니다.

```
① PostgreSQL StatefulSet 생성
        ↓
② PVC 생성 확인
        ↓
③ postgres-0 Running
        ↓
④ 데이터 INSERT
        ↓
⑤ SELECT로 데이터 확인
        ↓
⑥ postgres-0 삭제
        ↓
⑦ postgres-0 자동 생성
        ↓
⑧ PVC 확인
        ↓
⑨ SELECT 다시 실행
        ↓
⑩ 데이터 유지 확인
```

핵심 명령어만 정리하면:

```bash
# StatefulSet
kubectl get statefulset -n crud-app

# Pod
kubectl get pods -n crud-app

# PVC
kubectl get pvc -n crud-app

# PV
kubectl get pv

# Pod 삭제
kubectl delete pod postgres-0 -n crud-app

# 데이터 확인
kubectl exec -it \
  -n crud-app \
  postgres-0 \
  -- psql -U appuser -d cruddb \
  -c "SELECT * FROM items;"
```

---

# 25. Phase 4에서 꼭 이해해야 할 것

이번 실습에서 가장 중요한 것은 YAML 문법이 아니라 다음 관계입니다.

```
Deployment
   ↓
Pod
   ↓
Pod가 삭제되면
   ↓
컨테이너 저장공간도 사라질 수 있음

StatefulSet
   ↓
Pod
   ↓
PVC
   ↓
PV
   ↓
Pod가 삭제되어도
   ↓
PVC/PV 유지
   ↓
새 Pod가 기존 데이터 사용
```

그리고 한 가지 더 중요한 점은 **StatefulSet이 데이터를 보존해 주는 것이 아니라 PVC/PV가 데이터를 보존한다는 것**입니다.

StatefulSet은 `postgres-0` 같은 안정적인 Pod identity와 `volumeClaimTemplates`를 통해 **각 Pod에 영구 저장공간을 연결하고 관리하기 편하게 해주는 역할**을 합니다.

---

## 다음 단계 추천

Phase 4를 완료하면 다음은 **Phase 5 — PostgreSQL StatefulSet + PVC 장애/복구 실습**으로 이어가는 것이 좋습니다.

예를 들어:

```
postgres-0 삭제
      ↓
자동 복구
      ↓
데이터 유지

        +

worker-1 장애
      ↓
Pod 재배치 가능 여부
      ↓
PVC가 어느 노드에 존재하는가?
```
