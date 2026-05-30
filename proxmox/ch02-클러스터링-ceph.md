# Chapter 2: Proxmox 클러스터링 & Ceph

## 1. Proxmox VE 클러스터링

### 1.1 정의
- 여러 Proxmox 서버를 **하나의 그룹으로 묶어** 중앙 관리
- 고가용성(HA), 실시간 마이그레이션(Live Migration) 가능

### 1.2 사전 준비 (매우 중요)
| 준비 사항 | 설명 |
|-----------|------|
| **버전 통일** | 모든 노드의 Proxmox 버전 동일 |
| **전용 NIC** | 클러스터 통신용 별도 네트워크 카드 (1Gbps+) |
| **NTP 동기화** | 모든 노드 시간 동기화 필수 |
| **/etc/hosts** | 각 노드의 호스트 이름과 IP 등록 |
| **SSH 접속** | 노드 간 SSH 접속 가능 |
| **VM 없음** | 추가될 노드는 VM/CT 없는 초기 상태 |

### 1.3 /etc/hosts 설정
```bash
192.168.0.3 masungil1.co.kr masungil1
192.168.0.4 masungil2.co.kr masungil2
192.168.0.5 masungil3.co.kr masungil3
```

### 1.4 클러스터 생성
**첫 번째 노드:**
```bash
pvecm create kosa-cluster
```

**두 번째 노드 추가:**
```bash
pvecm add 192.168.0.3
# → 첫 번째 노드 IP 입력
# → yes 입력하여 인증
```

### 1.5 클러스터 상태 확인
```bash
pvecm status    # 전반적인 상태
pvecm nodes     # 노드 목록
```

**정상 상태:**
- Quorate: Yes → 쿼럼 형성됨
- Nodes: 2 → 2개 노드 참여

---

## 2. 클러스터링 주요 기능

| 기능 | 설명 |
|------|------|
| **중앙 관리** | 한 노드의 GUI에서 모든 노드 관리 |
| **Live Migration** | VM 실시간 이동 (무중단) |
| **HA (High Availability)** | 노드 장애 시 VM 자동 이동 |
| **공유 스토리지** | NFS, Ceph 등 연동 |

---

## 3. Ceph (분산 스토리지)

### 3.1 정의
- 여러 서버를 묶어 **하나의 거대한 스토리지**처럼 만드는 오픈소스
- 소프트웨어 정의 스토리지 (SDS)

### 3.2 비유
```
일반 스토리지: 각자 배낭 분리 → 하나 찢어지면 데이터 손실
Ceph 스토리지: 배낭 마법 연결 → 데이터 복제 (3개) → 하나 사라져도 안전
```

### 3.3 주요 특징
| 특징 | 설명 |
|------|------|
| **통합 스토리지** | 블록(RBD), 파일(CephFS), 오브젝트(S3) |
| **자가 복구** | 장애 시 자동 복제본 생성 |
| **확장성** | 노드 추가 → 자동 분산 |
| **비종속성** | 일반 x86 서버 사용 (비용 효율) |

### 3.4 핵심 구성 요소
| 구성 | 역할 |
|------|------|
| **OSD** | 실제 데이터 저장 '일꾼' (디스크 수 = OSD 수) |
| **MON** | 클러스터 상태 관리 '감독관' (최소 3개, 홀수) |
| **MGR** | 모니터 부담 덜어주는 '관리자' |

### 3.5 HCI (하이퍼컨버지드 인프라)
- Proxmox + Ceph = **HCI**
- 한 서버가 컴퓨팅 + 스토리지 동시 수행
- 외부 스토리지 없이 고가용성, 고성능 구축

---

## 4. Ceph 설정

### 4.1 Ceph 설치 (Proxmox GUI)
- 데이터센터 → Ceph → Install

### 4.2 OSD 생성
- 각 노드의 디스크를 OSD로 등록

### 4.3 Ceph Pool 생성
- 저장소 풀 생성 (replicated: 3)

### 4.4 Ceph RBD 사용
- VM 디스크 저장소로 Ceph 선택

---

## 5. NFS 연동

### 5.1 NFS 서버 설정
```bash
apt install nfs-kernel-server
mkdir /export/vms
chmod 755 /export/vms
echo "/export/vms *(rw,sync,no_subtree_check)" >> /etc/exports
exportfs -a
systemctl restart nfs-kernel-server
```

### 5.2 Proxmox NFS 추가
- 데이터센터 → Storage → Add → NFS
- Server: NFS 서버 IP
- Export: /export/vms
- Content: Images, Containers

---

## 6. Wireshark

### 6.1 정의
- 네트워크 패킷 캡처 및 분석 도구
- 네트워크 "현미경"

### 6.2 주요 사용 목적
| 목적 | 설명 |
|------|------|
| 문제 해결 | 네트워크 느림, 접속 불가 원인 파악 |
| 보안 분석 | 의심스러운 활동, 악성코드 탐지 |
| 프로토콜 학습 | TCP/IP, HTTP, DNS 동작 확인 |
| 성능 분석 | 트래픽 패턴, 병목 현상 찾기 |

### 6.3 설치
```bash
apt install wireshark
```

---

## 7. DuckDNS (무료 도메인)

### 7.1 정의
- 무료 DNS 서비스 (동적 IP용)

### 7.2 설정
1. DuckDNS 사이트 접속 → 계정 생성
2. 도메인 생성 (예: myserver.duckdns.org)
3. IP 주소 업데이트 스크립트 작성

```bash
curl -k "https://www.duckdns.org/update?domains=myserver&token=xxx&ip="
```

---

## 8. 요약

| 구성 | 설명 |
|------|------|
| **클러스터링** | 여러 노드 묶어 중앙 관리 |
| **Ceph** | 분산 스토리지 (자가 복구, 확장성) |
| **NFS** | 외부 스토리지 연동 |
| **Wireshark** | 패킷 분석 도구 |
| **DuckDNS** | 무료 동적 DNS |