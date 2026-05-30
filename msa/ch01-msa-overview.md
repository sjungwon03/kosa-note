# Chapter 1: MSA 프로젝트 개요

## 1.1 MSA (Microservices Architecture) 개요

### 정의
- **MSA**: 애플리케션을 여러 개의 작은 서비스로 분리하는 아키텍처
- 각 서비스는 **독립적으로 배포, 확장, 관리** 가능

### 모놀리식 vs MSA

| 특징 | Monolith | MSA |
|------|----------|-----|
| **구조** | 하나의 큰 앱 | 여러 작은 서비스 |
| **배포** | 전체 재배포 | 서비스별 배포 |
| **확장** | 전체 확장 | 서비스별 확장 |
| **장애** | 전체 장애 | 서비스별 장애 격리 |
| **기술** | 단일 스택 | 서비스별 선택 |

---

## 1.2 MSA_Project 구조

### 서비스 구성
```
MSA_Project/
├── docker-compose.yml      # Docker 컨테이너 orchestration
├── nginx/
│   └── nginx.conf          # Reverse Proxy 설정
├── gateway/
│   └── app.py              # API Gateway (FastAPI, port 5000)
├── auth_server/
│   └── app.py              # 인증 서버 (FastAPI, port 5001)
├── employee_server/
│   ├── application.py      # 직원 서버 (FastAPI, port 5002)
│   ├── database.py         # RDS MySQL 연동
│   ├── database_dynamo.py  # DynamoDB 연동
│   ├── models.py           # Pydantic 모델
│   └── util.py             # 이미지 처리
├── photo_service/
│   └── app.py              # 사진 서비스 (FastAPI, port 5003)
└── frontend/
    └── index.html          # Vue.js SPA
```

---

## 1.3 아키텍처 구조

### 전체 구조
```
┌─────────────────────────────────────────────────────┐
│                   사용자 (Browser)                   │
│                  http://localhost:8080               │
└───────────────────────┬─────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│               Nginx (Reverse Proxy)                  │
│                     port 80                          │
│   / → frontend (static files)                       │
│   /api/* → gateway:5000                             │
│   /static/uploads/* → gateway:5000                  │
└───────────────────────┬─────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│             Gateway (API Gateway)                    │
│                   port 5000                          │
│   FastAPI + CORS + httpx async client               │
└───────────────────────┬─────────────────────────────┘
                        ↓
        ┌───────────────┴───────────────┐
        ↓               ↓               ↓
┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│ Auth Server │ │Employee Srv │ │Photo Service│
│  port 5001  │ │  port 5002  │ │  port 5003  │
│ JWT 인증    │ │ CRUD + DB   │ │ 사진 저장   │
└─────────────┘ └─────────────┘ └─────────────┘
```

---

## 1.4 서비스 역할

| 서비스 | 포트 | 역할 | 프레임워크 |
|--------|------|------|-----------|
| **Nginx** | 80 | Reverse Proxy, Frontend Serving | Nginx |
| **Gateway** | 5000 | API Routing, CORS | FastAPI |
| **Auth Server** | 5001 | JWT 인증 | FastAPI |
| **Employee Server** | 5002 | 직원 CRUD | FastAPI |
| **Photo Service** | 5003 | 사진 저장/조회 | FastAPI |

---

## 1.5 통신 패턴

### 동기 통신 (REST API)
- Gateway → Auth Server: `/api/auth/*`
- Gateway → Employee Server: `/api/employee/*`
- Gateway → Photo Service: `/static/uploads/*`

### 인증 흐름
```
1. Client → Gateway → Auth Server (login)
2. Auth Server → JWT Token 반환
3. Client → Gateway → Employee Server (with JWT)
4. Employee Server → Token 검증 → 요청 처리
```

---

## 1.6 Docker Compose 구성

### 서비스 정의
```yaml
services:
  nginx:
    image: nginx:latest
    ports: ["8080:80"]
    volumes:
      - ./frontend:/var/www/frontend:ro
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on: [gateway]
    
  gateway:
    build: ./gateway
    ports: ["5000:5000"]
    depends_on: [auth_server, employee_server, photo_service]
    
  auth_server:
    build: ./auth_server
    ports: ["5001:5001"]
    
  employee_server:
    build: ./employee_server
    ports: ["5002:5002"]
    depends_on: [photo_service]
    
  photo_service:
    build: ./photo_service
    ports: ["5003:5003"]
    volumes:
      - ./photo_service/data:/app/data
```

### 네트워크
```yaml
networks:
  app-network:
    driver: bridge
```