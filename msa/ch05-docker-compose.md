# Chapter 5: Docker Compose & 배포

## 5.1 Docker Compose 구성

### 전체 구조
```yaml
version: '3.8'

services:
  nginx:
    image: nginx:latest
    ports: ["8080:80"]
    volumes:
      - ./frontend:/var/www/frontend:ro
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on: [gateway]
    networks: [app-network]

  gateway:
    build: ./gateway
    ports: ["5000:5000"]
    depends_on: [auth_server, employee_server, photo_service]
    networks: [app-network]

  auth_server:
    build: ./auth_server
    ports: ["5001:5001"]
    networks: [app-network]

  employee_server:
    build: ./employee_server
    ports: ["5002:5002"]
    depends_on: [photo_service]
    networks: [app-network]

  photo_service:
    build: ./photo_service
    ports: ["5003:5003"]
    volumes:
      - ./photo_service/data:/app/data
    networks: [app-network]

networks:
  app-network:
    driver: bridge
```

---

## 5.2 서비스별 Dockerfile

### Gateway Dockerfile
```dockerfile
FROM python:3.9-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 5000
CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "5000"]
```

### requirements.txt (Gateway)
```
fastapi
uvicorn
httpx
```

---

## 5.3 서비스 시작 순서

### depends_on 체크
```
nginx → gateway → auth_server, employee_server, photo_service
gateway → auth_server, employee_server, photo_service
employee_server → photo_service
```

### 시작 순서
```
1. photo_service (의존 없음)
2. auth_server (의존 없음)
3. employee_server → photo_service
4. gateway → auth_server, employee_server, photo_service
5. nginx → gateway
```

---

## 5.4 네트워크 구성

### Bridge Network
```yaml
networks:
  app-network:
    driver: bridge
```

### 서비스 간 통신
- Docker Compose 내에서 **서비스명**으로 접속
- 예: `http://gateway:5000`, `http://auth_server:5001`

---

## 5.5 실행 명령

### 서비스 시작
```bash
docker-compose up -d
```

### 서비스 중지
```bash
docker-compose down
```

### 로그 확인
```bash
docker-compose logs -f gateway
docker-compose logs -f employee_server
```

### 재빌드
```bash
docker-compose up -d --build
```

---

## 5.6 포트 매핑

| 서비스 | Container Port | Host Port |
|--------|----------------|-----------|
| **Nginx** | 80 | 8080 |
| **Gateway** | 5000 | 5000 |
| **Auth Server** | 5001 | 5001 |
| **Employee Server** | 5002 | 5002 |
| **Photo Service** | 5003 | 5003 |

---

## 5.7 외부 접속

### Browser 접속
```
http://localhost:8080              # Frontend
http://localhost:8080/api/auth/login  # Auth API
http://localhost:8080/api/employees   # Employee API
```

### Direct API 접속
```
http://localhost:5000/api/auth/login  # Gateway 직접
http://localhost:5001/login           # Auth Server 직접
http://localhost:5002/employees       # Employee Server 직접
```