# Chapter 5: VPN (IPsec Site-to-Site)

## 1. VPN (Virtual Private Network)

### 1.1 정의
- 공용 인터넷망 위에 **암호화된 터널** 생성
- 사설 네트워크처럼 안전하게 통신

### 1.2 주요 목적
| 목적 | 설명 |
|------|------|
| 🔒 보안 | 데이터 암호화 |
| 🌐 사설 IP 통신 | 사설 IP 기반 통신 |
| 🏢 거리 극복 | 물리적 거리 문제 해결 |

---

## 2. IPsec (Internet Protocol Security)

### 2.1 정의
- VPN을 구현하기 위한 **표준 보안 프로토콜**
- VPN은 개념, IPsec은 기술

### 2.2 보안 기능
| 기능 | 설명 |
|------|------|
| 🔐 암호화 | 데이터 내용 보호 (AES) |
| 🧾 인증 | 상대방 확인 (PSK, 인증서) |
| 🔁 무결성 | 중간 변조 방지 |
| 🚫 재전송 방지 | Replay Attack 방어 |

---

## 3. Site-to-Site VPN

### 3.1 정의
- 두 네트워크(사이트)를 **IPsec 터널로 직접 연결**
- PC ↔ PC가 아님 → **네트워크 ↔ 네트워크**

### 3.2 구성 예시
```
[온프레미스 사설망]              [AWS VPC]
192.168.0.0/24  ── IPsec 터널 ── 10.1.0.0/16
        ER605                StrongSwan / VGW
```

### 3.3 특징
- 한 번 연결 → 내부 장비 모두 통신 가능
- PC마다 VPN 설치 불필요
- 라우터/게이트웨이 1번 설정

---

## 4. 개념 관계
```
VPN (개념)
 └─ IPsec (기술)
     └─ Site-to-Site VPN (구현 방식)
```

---

## 5. 동작 구조

### 5.1 Phase 1 (IKE)
- 상대 인증
- 암호화 알고리즘 협상
- IKE SA 생성

### 5.2 Phase 2 (ESP)
- 실제 데이터 암호화
- IPsec SA 생성

---

## 6. Site-to-Site VPN 장점

| 장점 | 설명 |
|------|------|
| 전용선 없이 연결 | 비용 절감 |
| 내부 IP 그대로 사용 | NAT 불필요 |
| 높은 보안성 | AES-256, SHA-2 |
| 서버 설정 최소화 | 라우터만 설정 |
| 중앙 집중 관리 | 하이브리드 클라우드 필수 |

---

## 7. ER605 ↔ AWS VPN (IPsec)

### 7.1 구성
```
[ ER605 (온프레미스) ]          [ AWS VGW ]
192.168.0.0/24                  10.1.0.0/16
        |                              |
        +---- IPsec Tunnel -----------+
```

### 7.2 ER605 설정
1. VPN → IPsec → Add
2. Remote Gateway: AWS VGW IP
3. Local Network: 192.168.0.0/24
4. Remote Network: 10.1.0.0/16
5. Pre-Shared Key: 설정

### 7.3 AWS VGW 설정
1. VPC → VPN Connections → Create
2. Customer Gateway: ER605 Public IP
3. Virtual Private Gateway: VPC 연결
4. Route Tables: VPN 경로 추가

---

## 8. 보안성

| 항목 | 설명 |
|------|------|
| 암호화 | AES-256 |
| 인증 | PSK / 인증서 |
| 무결성 | SHA-2 |
| 키 교환 | Diffie-Hellman |

---

## 9. 단점 및 고려사항

| 단점 | 설명 |
|------|------|
| 인터넷 품질 의존 | 대역폭, 지연 문제 |
| 설정 복잡도 | 전문 지식 필요 |
| 이중화 필요 | HA 구성 |

---

## 10. AWS 활용 사례

| 사례 | 설명 |
|------|------|
| 온프레미스 ↔ AWS | EC2/RDS/EKS 연동 |
| 사내 AD 통합 | DNS 통합 |
| 환경 분리 | 개발/운영 분리 |
| DMZ 구성 | 보안 아키텍처 |

---

## 11. 요약

| 구성 | 설명 |
|------|------|
| **VPN** | 보안 터널 개념 |
| **IPsec** | VPN 구현 표준 기술 |
| **Site-to-Site** | 네트워크 간 연결 |
| **ER605** | TP-Link VPN 라우터 |
| **AWS VGW** | AWS Virtual Private Gateway |