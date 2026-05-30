# Chapter 3: pfSense & 네트워크 구성

## 1. 네트워크 시스템 구조

### 1.1 전체 구조
```
ER605 (단순 인터넷) (port1 WAN 입력)
  |     |
WAN ---+     |
     (port1)  | (port2)
              |
  +-----------+
  |     
  |    관리형 스위치 (TRUNK)
  | (port1)|     | (port2)   | (port3)
  +--------+     |           |
                 |           +---> 관리형 노트북 연결  
                 |
             Proxmox #1 
                 |
           pfSense (VLAN 라우팅)

  +---------+---------+---------+
  |         |         |         |
vlan10   vlan20    vlan30    vlan40
공개망    DMZ       내부망     관리망

Proxmox #2  Proxmox #3 Proxmox #4
```

### 1.2 VLAN 구성
| VLAN | 이름 | 용도 |
|------|------|------|
| VLAN 10 | 공개망 | 외부 공개 서비스 |
| VLAN 20 | DMZ | 외부 접근 가능 서비스 |
| VLAN 30 | 내부망 | 내부 서비스 |
| VLAN 40 | 관리망 | Proxmox 관리 |

### 1.3 Proxmox NIC TRUNK 설정
```bash
# /etc/network/interfaces
auto vmbr0
iface vmbr0 inet static
        address 192.168.3.2/24
        gateway 192.168.3.1
        bridge-ports nic0
        bridge-stp off
        bridge-fd 0
        bridge-vlan-aware yes  # TRUNK 가능 상태
```

---

## 2. pfSense 설치

### 2.1 VMware 설치
1. pfSense ISO 다운로드
2. VMware에 VM 생성
3. pfSense 설치 → WAN/LAN 인터페이스 설정

### 2.2 Proxmox 설치
1. pfSense ISO 업로드
2. VM 생성 (CPU 2, RAM 2G)
3. 2개 NIC 설정 (WAN, LAN)
4. pfSense 설치

### 2.3 초기 설정
- WAN: 외부 네트워크 (공유기 연결)
- LAN: 내부 네트워크 (192.168.x.x)

---

## 3. VLAN 생성

### 3.1 pfSense VLAN 설정
1. Interfaces → Assignments → VLANs
2. Add 버튼 → VLAN 설정
   - Parent Interface: LAN 인터페이스
   - VLAN Tag: 10, 20, 30, 40
   - Description: 공개망, DMZ, 내부망, 관리망

### 3.2 VLAN 인터페이스 등록
1. Interfaces → Assignments
2. VLAN 인터페이스를 각각 추가
3. 각 인터페이스 Enable

---

## 4. DHCP 서버 설정

### 4.1 각 VLAN DHCP 설정
1. Services → DHCP Server
2. 각 VLAN별 DHCP 설정
   - Range: IP 할당 범위
   - Gateway: VLAN 게이트웨이

**예시:**
| VLAN | IP Range | Gateway |
|------|----------|---------|
| VLAN 10 | 192.168.10.100-200 | 192.168.10.1 |
| VLAN 20 | 192.168.20.100-200 | 192.168.20.1 |
| VLAN 30 | 192.168.30.100-200 | 192.168.30.1 |
| VLAN 40 | 192.168.40.100-200 | 192.168.40.1 |

---

## 5. OPT NIC 설정

### 5.1 추가 인터페이스 설정
1. Interfaces → Assignments
2. 새 인터페이스 추가
3. OPT1, OPT2 등으로 이름 변경

### 5.2 인터페이스 활성화
- Enable: 체크
- IPv4 Configuration Type: Static IPv4
- IP Address: 해당 VLAN IP

---

## 6. 방화벽 설정

### 6.1 방화벽 규칙 추가
1. Firewall → Rules
2. 각 인터페이스별 규칙 추가

### 6.2 규칙 예시
| 인터페이스 | Action | Source | Destination | Port |
|------------|--------|--------|-------------|------|
| WAN | Pass | Any | LAN | HTTP/HTTPS |
| LAN | Pass | LAN Net | Any | Any |
| VLAN10 | Pass | VLAN10 Net | Any | HTTP |

### 6.3 시스템 로그
1. Status → System Logs
2. Firewall 로그 확인
3. 차단/허용 패킷 확인

---

## 7. 관리형 스위치 설정

### 7.1 TRUNK 포트 설정
- pfSense 연결 포트: VLAN 10, 20, 30, 40 Tagged
- Proxmox 연결 포트: VLAN 10, 20, 30, 40 Tagged

### 7.2 ACCESS 포트 설정
- VLAN 10 포트: VLAN 10 Untagged
- VLAN 20 포트: VLAN 20 Untagged
- VLAN 30 포트: VLAN 30 Untagged
- VLAN 40 포트: VLAN 40 Untagged

---

## 8. Proxmox 서버 배치

### 8.1 물리적 배치
- 모든 Proxmox 서버 → 관리형 스위치 TRUNK 포트 연결
- 스위치 포트: VLAN 10, 20, 30, 40 모두 통과

### 8.2 논리적 배치
- pfSense가 모든 호스트의 게이트웨이 역할
- Traffic Flow: VM → 스위치 → pfSense → 인터넷

### 8.3 VLAN Aware Bridge
- 모든 Proxmox 노드에 VLAN Aware 설정
- vmbr0에 `bridge-vlan-aware yes`

### 8.4 관리 IP
- 모든 노드의 관리 IP: VLAN 40 대역
- 예: PVE #1(192.168.3.2), PVE #2(192.168.3.3)

---

## 9. 고가용성 (HA) 고려

### 9.1 pfSense HA
- Proxmox #2에도 pfSense 설치
- pfSense HA 구성 → #1 장애 시 #2 작동

### 9.2 공유 스토리지
- NAS (NFS/iSCSI) 또는 Ceph 연동
- VM Live Migration 가능

---

## 10. 요약

| 구성 | 설명 |
|------|------|
| **pfSense** | 방화벽/라우터 VM |
| **VLAN** | 네트워크 분리 (공개, DMZ, 내부, 관리) |
| **관리형 스위치** | TRUNK/ACCESS 포트 설정 |
| **DHCP** | 각 VLAN별 IP 자동 할당 |
| **방화벽 규칙** | 인터페이스별 트래픽 제어 |