# Chapter 4: Kubernetes Manifest

## 4.1 Deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hostname-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webserver
  template:
    metadata:
      name: my-webserver
      labels:
        app: webserver
    spec:
      containers:
      - name: my-webserver
        image: masungil/echo-hostname:1.0
        imagePullPolicy: Always
        ports:
        - containerPort: 8000
```

### Deployment 구성

| 필드 | 설명 |
|------|------|
| `replicas` | Pod 복제본 수 |
| `selector.matchLabels` | Pod 선택 레이블 |
| `template.labels` | Pod 레이블 |
| `containers.image` | Docker 이미지 |
| `containerPort` | 컨테이너 포트 |

---

## 4.2 Service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: echo-hostname-svc
spec:
  ports:
    - name: web-port
      port: 8080
      targetPort: 8000
  selector:
    app: webserver
  type: ClusterIP
```

### Service 구성

| 필드 | 설명 |
|------|------|
| `port` | Service 포트 |
| `targetPort` | 컨테이너 포트 |
| `selector` | 연결할 Pod 레이블 |
| `type` | ClusterIP / NodePort / LoadBalancer |

---

## 4.3 Dockerfile

### Multi-stage Build
```dockerfile
# Stage 1: 빌더 스테이지
FROM python:3.9-slim AS builder

RUN apt-get update && apt-get install -y --no-install-recommends \
    curl build-essential libc6-dev && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Stage 2: 실행 스테이지
FROM python:3.9-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
    curl libgcc1 libstdc++6 && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

WORKDIR /app

COPY --from=builder /usr/local/lib/python3.9/site-packages \
     /usr/local/lib/python3.9/site-packages
COPY --from=builder /usr/local/bin/uvicorn /usr/local/bin/uvicorn

COPY . .

EXPOSE 8000

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Multi-stage Build 장점
- 이미지 크기 감소
- 빌드 캐시 활용
- 최종 이미지에 불필요한 파일 제외

---

## 4.4 main.py (FastAPI)

```python
from fastapi import FastAPI
import os
import socket

app = FastAPI()

@app.get("/")
def read_root():
    hostname = socket.gethostname()
    container_id = os.getenv("HOSTNAME", hostname)
    return {
        "message": f"Hello from version: v1.0",
        "container_id": container_id,
        "hostname": hostname
    }

@app.get("/healthz")
def healthz():
    return {"status": "OK"}
```

### 엔드포인트
| 경로 | 설명 |
|------|------|
| `/` | 호스트명/컨테이너 ID 반환 |
| `/healthz` | Health Check용 |

---

## 4.5 Docker Hub Push

### 이미지 빌드
```bash
cd echo-hostname
docker build -t masungil/echo-hostname:1.0 .
```

### Docker Hub 로그인
```bash
docker login
```

### Push
```bash
docker push masungil/echo-hostname:1.0
```

---

## 4.6 이미지 태그 업데이트

### Deployment 이미지 변경
```yaml
image: masungil/echo-hostname:457a5ca0d489...
```

### ArgoCD 자동 감지
- Git에 push → ArgoCD가 감지 → 자동 Sync