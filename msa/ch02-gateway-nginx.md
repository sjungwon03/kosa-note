# Chapter 2: Gateway & Nginx

## 2.1 Gateway (API Gateway)

### Gateway 역할
- **단일 진입점**: 모든 API 요청의 중심
- **Routing**: 요청 경로에 따라 적절한 서비스로 전달
- **CORS 처리**: Cross-Origin Resource Sharing 허용
- **비동기 처리**: httpx AsyncClient 사용

---

## 2.2 Gateway 구현 (FastAPI)

### 기본 설정
```python
from fastapi import FastAPI, Request, Response, HTTPException
from fastapi.middleware.cors import CORSMiddleware
import httpx

app = FastAPI()

# CORS 미들웨어
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# 서비스 URL
AUTH_SERVER_URL = "http://localhost:5001"
EMPLOYEE_SERVER_URL = "http://localhost:5002"
PHOTO_SERVICE_URL = "http://localhost:5003"

client = httpx.AsyncClient()
```

---

## 2.3 Routing 구조

### 라우트 매핑

| Gateway Route | Target Service | Target Path |
|---------------|----------------|-------------|
| `/api/auth/{path}` | Auth Server | `/{path}` |
| `/api/employee/{path}` | Employee Server | `/{path}` |
| `/static/uploads/{filename}` | Photo Service | `/photos/{filename}` |

### Auth Server 프록시
```python
@app.api_route("/api/auth/{path:path}", methods=["GET", "POST", ...])
async def proxy_auth_requests(path: str, request: Request):
    url = f"{AUTH_SERVER_URL}/{path}"
    headers = {k: v for k, v in request.headers.items() 
               if k.lower() not in ["host", "content-length"]}
    body = await request.body()
    
    resp = await client.request(
        method=request.method,
        url=url,
        headers=headers,
        content=body,
        params=request.query_params
    )
    
    excluded_headers = ["content-encoding", "content-length", "transfer-encoding"]
    response_headers = {k: v for k, v in resp.headers.items() 
                        if k.lower() not in excluded_headers}
    
    return Response(content=resp.content, status_code=resp.status_code, 
                    headers=response_headers)
```

### Employee Server 프록시
```python
@app.api_route("/api/employee/{path:path}", methods=["GET", "POST", ...])
async def proxy_employee_requests(path: str, request: Request):
    url = f"{EMPLOYEE_SERVER_URL}/{path}"
    # ... (프록시 로직)
```

### Photo Service 프록시
```python
@app.api_route("/static/uploads/{filename:path}", methods=["GET"])
async def proxy_employee_photo_requests(filename: str, request: Request):
    url = f"{PHOTO_SERVICE_URL}/photos/{filename}"
    # ... (프록시 로직)
```

---

## 2.4 실행 시간 측정

```python
start_time = time.time()

resp = await client.request(...)

end_time = time.time()
execution_time = (end_time - start_time) * 1000
print(f"Gateway: Proxy executed in {execution_time:.2f} ms")
```

---

## 2.5 Nginx (Reverse Proxy)

### nginx.conf
```nginx
events {}

http {
    include /etc/nginx/mime.types;

    server {
        listen 80;
        server_name localhost;

        # Frontend 정적 파일
        location / {
            root /var/www/frontend;
            try_files $uri $uri/ /index.html;
        }

        # API Gateway 프록시
        location ~ ^/(api|static/uploads)/ {
            proxy_pass http://gateway:5000;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}
```

### 역할
| 역할 | 설명 |
|------|------|
| **Static Files** | Frontend (Vue.js) 파일 서빅 |
| **Reverse Proxy** | `/api/*`, `/static/uploads/*` → Gateway |
| **Load Balancing** | 여러 Gateway 인스턴스 분산 가능 |

---

## 2.6 요청 흐름

```
Browser Request
    ↓
http://localhost:8080/api/auth/login
    ↓
Nginx (port 80) → proxy_pass to gateway:5000
    ↓
Gateway (port 5000) → httpx.AsyncClient
    ↓
Auth Server (port 5001) → JWT Token
    ↓
Response → Gateway → Nginx → Browser
```