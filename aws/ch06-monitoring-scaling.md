# Chapter 6: AWS 모니터링 & 확장

## 6.1 Amazon CloudWatch

### CloudWatch란?
- AWS 리소스 **실시간 모니터링** 서비스
- 지표 수집, 대시보드, 알람

### CloudWatch 대시보드
- EC2 CPU 사용률 등 시각화
- 위젯 생성 → 지표 선택 → 대시보드 구성

### CloudWatch 알람
| 설정 | 설명 |
|------|------|
| **지표** | CPUUtilization 등 |
| **기간** | 5분 |
| **임계값** | 70% 초과 |
| **알림** | SNS Topic → 이메일 발송 |

### 알람 상태
| 상태 | 설명 |
|------|------|
| **OK** | 정상 |
| **ALARM** | 임계값 초과 |
| **INSUFFICIENT_DATA** | 데이터 부족 |

### 알람 구성 순서
1. 지표 선택 (EC2 → CPUUtilization)
2. 조건 설정 (기간, 임계값)
3. 알림 구성 (SNS Topic, 이메일)
4. 이메일 구독 확인 (Confirm subscription)

---

## 6.2 Elastic Load Balancing (ELB)

### ELB란?
- **트래픽 자동 분산** 서비스
- 단일 접속 지점 (DNS) 제공
- 비정상 인스턴스 자동 제외

### ELB 유형

| 유형 | 설명 | 계층 |
|------|------|------|
| **ALB** | HTTP/HTTPS, 웹 앱용 | Layer 7 |
| **NLB** | TCP/UDP/TLS, 성능 중심 | Layer 4 |
| **GWLB** | 보안/서드파티 장비 라우팅 | - |

### ALB 구성 요소

| 요소 | 설명 |
|------|------|
| **리스너** | 요청 수신 (포트 + 프로토콜) |
| **대상 그룹** | 백엔드 그룹 (EC2, Lambda) |
| **규칙** | 트래픽 라우팅 방식 |
| **Health Check** | 정상 인스턴스만 트래픽 수신 |

### ALB 구성 순서
1. ALB 생성 (Internet-facing)
2. VPC & AZ 선택 (퍼블릭 서브넷 2개)
3. 보안 그룹 설정 (포트 80/443)
4. 대상 그룹 생성 (EC2 인스턴스 2개 등록)
5. DNS 주소로 접속 테스트

### ELB 특징
- **SPOF 아님**: AWS 리전 단위 고가용성
- 자동 확장: 트래픽 늘어나면 용량 자동 증가

---

## 6.3 Amazon EC2 Auto Scaling

### Auto Scaling이란?
- **EC2 인스턴스 자동 조정**
- 트래픽 ↑ → Scale Out (인스턴스 추가)
- 트래픽 ↓ → Scale In (인스턴스 제거)

### 구성 요소

| 요소 | 설명 |
|------|------|
| **Launch Template** | AMI, 인스턴스 타입, 보안 그룹, User Data |
| **Auto Scaling Group** | 최소/최대/원하는 인스턴스 수, AZ 배치 |
| **CloudWatch 연계** | CPU 등 지표 기반 확장/축소 |
| **ELB 연계** | 새 인스턴스 자동 등록 |

### 작동 방식
```
CPU 60% 초과 → CloudWatch 경보 → Auto Scaling 인스턴스 추가
부하 감소 → Auto Scaling 인스턴스 제거 → 최소 인스턴스 수 유지
```

### 장점
| 장점 | 설명 |
|------|------|
| **가용성** | 서버 죽어도 다른 서버 유지 |
| **확장성** | 트래픽 증가 시 즉시 확장 |
| **비용 효율성** | 트래픽 감소 시 자동 축소 |
| **자동화** | 관리자 직접 추가/삭제 불필요 |

---

## 6.4 아키텍처 다이어그램

### 고가용성 구조
```
          사용자 브라우저
                ↓
    [Application Load Balancer]
                ↓
        ┌──────────────┐
        │ Auto Scaling │
        │   Group      │
        └──────────────┘
         ↙      ↓      ↘
      EC2-A   EC2-B   EC2-C
      (AZ-1)  (AZ-2)  (AZ-1)
                ↓
          DynamoDB / S3
```

### 흐름
1. ALB가 트래픽 분산
2. Auto Scaling이 EC2 수 조정
3. CloudWatch가 지표 수집 → 알람 → Auto Scaling 트리거
4. EC2 장애 → Auto Scaling이 새 인스턴스 생성

---

## 6.5 모듈 지식

### Auto Scaling 구성 요소
> 시작 템플릿, 크기 조정 정책, Auto Scaling 그룹

### ELB 기능
> EC2 Auto Scaling 통합, 수신 트래픽 인스턴스 전달

### CloudWatch 경보 상태
> OK, ALARM, INSUFFICIENT_DATA