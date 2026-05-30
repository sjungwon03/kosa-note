# Chapter 1: Proxmox VE 기본

## 1. Proxmox 가상화 환경 시스템 구조

### 1.1 구조도
```
[ 사용자 (관리용 PC) ]          [ 인터넷 / 내부 네트워크 ]
         |                              |
         +------------------------------|
                                        |
  +-------------------------------------|------------------+
  |  +------------[ 물리 NIC ]----------|---------------+  |
  |  |                                  ^               |  |
  |  |  +-------------------------------|------------+  |  |
  |  |  |           가상 스위치 (vmbr0) |            |  |  |
  |  |  +---^-------^-------^-----------+------------+  |  |
  |  |      |       |       | (가상 랜선)               |  |
  |  |  +---|---+ +-|-----+ +-|----------------------+  |  |
  |  |  | VM1 | | VM2  | | VM3                    |  |  |
  |  |  +-----+ +------+ +------------------------+  |  |
  |  |                                             |  |
  |  +-------[ Proxmox VE (하이퍨바이저) ]----------+  |
  |                                                  |
  +----[ 물리 컴퓨터 (CPU, RAM, SSD) ]---------------+
```

### 1.2 구성 요소
| 구성 | 역할 |
|------|------|
| **물리 NIC** | 외부 요청이 도달하는 하드웨어 관문 |
| **가상 스위치 (vmbr0)** | 트래픽을 목적지 VM으로 전달하는 '교통 경찰' |
| **가상머신 (VM)** | 가상 스위치에 연결된 독립적인 컴퓨터 |
| **Proxmox VE** | 모든 VM과 네트워크를 관리하는 하이퍼바이저 OS |
| **물리 컴퓨터** | CPU, RAM, SSD 등 하드웨어 기반 |

### 1.3 하이브리드 설계 이유
- Proxmox는 서버용 OS → GUI 없음 → 웹 브라우저로 원격 관리
- 컴퓨터 1대 → Proxmox 설치 후 브라우저 실행할 OS 없음
- **해결책**: VM에 '메인 PC' 역할 부여 + Proxmox는 최소한의 GUI만

---

## 2. Proxmox VE 설치

### 2.1 설치 과정
```
1. Proxmox ISO 다운로드 → USB 부팅 디스크 제작
2. USB로 부팅 → Proxmox 설치
3. 네트워크 설정 (고정 IP, 게이트웨이, DNS)
4. 재부팅 → 웹 UI 접속 (https://<IP>:8006)
```

### 2.2 APT 리포지토리 수정 (필수)
- 기본: 유료 Enterprise 리포지토리 → 오류 발생
- 수정: 무료 No-Subscription 리포지토리로 변경

**Enterprise 비활성화:**
```bash
vi /etc/apt/sources.list.d/pve-enterprise.sources
# 모든 줄에 # 추가 (주석)
```

**No-Subscription 활성화:**
```bash
cat <<EOF >/etc/apt/sources.list.d/pve-no-subscription.sources
Types: deb
URIs: http://download.proxmox.com/debian/pve
Suites: trixie
Components: pve-no-subscription
Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
EOF
```

**Ceph 리포지토리도 동일하게 수정**

### 2.3 업데이트
```bash
apt update
apt upgrade -y
```

### 2.4 ISO 이미지 업로드
- 웹 UI → local (pve) → ISO 이미지 → 업로드

---

## 3. VM 생성

### 3.1 Xubuntu Desktop VM
- **역할**: 메인 PC (웹 브라우저, 터미널)
- **사양**: vCPU 2, RAM 3G, Disk 64G

### 3.2 Alpine Linux VM
- **역할**: 경량 서버
- **사양**: vCPU 2, RAM 2G, Disk 32G

### 3.3 Ubuntu Server VM
- **역할**: 경량 서버
- **사양**: vCPU 2, RAM 2G, Disk 32G

### 3.4 Qemu Guest Agent 설정
- VM과 호스트 간 통신
- VM 내부: `apt install qemu-guest-agent`
- Proxmox: VM Options → QEMU Guest Agent → Enabled

---

## 4. Proxmox 원격 접속 설정

### 4.1 FreeRDP 설치 (Proxmox 호스트)
```bash
apt install freerdp3-x11 xinit twm
```

### 4.2 .xinitrc 설정
```bash
vi ~/.xinitrc
# 내용:
xfreerdp /u:kosa /p:kosa1004 /v:192.168.x.x /f
```

### 4.3 접속
```bash
startx
# → Xubuntu VM 데스크톱 전체 화면 표시
```

---

## 5. Proxmox 웹 UI 접속

### 5.1 접속 주소
```
https://<Proxmox-IP>:8006
```

### 5.2 로그인
- 사용자: root
- 비밀번호: 설치 시 설정한 비밀번호

---

## 6. 요약

| 구성 | 설명 |
|------|------|
| Proxmox VE | 하이퍼바이저 (가상화 OS) |
| vmbr0 | 가상 스위치 (Linux Bridge) |
| VM | 가상머신 (Xubuntu, Ubuntu, Alpine) |
| FreeRDP | 원격 데스크톱 접속 |