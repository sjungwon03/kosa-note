# Chapter 4: DNS 서버 (BIND9)

## 1. DNS 서버 구성

### 1.1 아키텍처
```
[ Client PCs ]
192.168.80.100 ~ 192.168.80.199
        |
        | DNS Query
        v
[ DNS Server ]
Ubuntu Server + BIND9
IP: 192.168.80.5
Zone: kosa.co.kr
        |
        +--> team1.kosa.co.kr → 192.168.80.6 (Web)
        +--> team2.kosa.co.kr → 192.168.80.7 (Web)
```

---

## 2. BIND9 설치

### 2.1 설치
```bash
sudo apt update
sudo apt install -y bind9 bind9utils dnsutils
```

### 2.2 서비스 확인
```bash
systemctl status bind9
```

---

## 3. BIND9 기본 설정

### 3.1 named.conf.options
```bash
sudo vi /etc/bind/named.conf.options
```

```conf
options {
    directory "/var/cache/bind";

    recursion yes;  # 재귀 쿼리 허용

    allow-recursion {
        localhost;
        192.168.80.0/24;  # 내부망만 허용 (보안)
    };

    listen-on {
        127.0.0.1;
        192.168.80.5;  # DNS 서버 IP
    };

    allow-query {
        localhost;
        192.168.80.0/24;  # 내부망만 질의 허용
    };

    forwarders {
        8.8.8.8;   # Google DNS
        8.8.4.4;
    };

    dnssec-validation auto;
    listen-on-v6 { none; };
};
```

### 3.2 설정 의미
| 설정 | 의미 |
|------|------|
| recursion yes | 클라이언트 대신 외부 DNS 조회 |
| allow-recursion | 재귀 요청 허용 범위 (내부망만) |
| allow-query | DNS 질의 허용 범위 |
| forwarders | 상위 DNS (Google) |

---

## 4. Zone 파일 설정

### 4.1 Zone 선언
```bash
sudo vi /etc/bind/named.conf.local
```

```conf
zone "kosa.co.kr" {
    type master;
    file "/etc/bind/db.kosa.co.kr";
};
```

### 4.2 정방향 Zone 파일
```bash
sudo vi /etc/bind/db.kosa.co.kr
```

```dns
$TTL 3600

@   IN  SOA ns.kosa.co.kr. admin.kosa.co.kr. (
        2025121901  ; Serial (YYYYMMDDnn)
        3600        ; Refresh (1시간)
        1800        ; Retry (30분)
        604800      ; Expire (7일)
        86400       ; Minimum (1일)
)

@       IN  NS  ns.kosa.co.kr.
ns      IN  A   192.168.80.5
@       IN  A   192.168.80.5

team1           IN  A   192.168.80.6
www.team1       IN  A   192.168.80.6

team2           IN  A   192.168.80.7
www.team2       IN  A   192.168.80.7
```

### 4.3 Zone 파일 설명
| 레코드 | 설명 |
|--------|------|
| **SOA** | Zone 권한 시작 정보 |
| **NS** | 네임서버 지정 |
| **A** | 도메인 → IP 매핑 |

---

## 5. 설정 검사

### 5.1 문법 검사
```bash
sudo named-checkconf
sudo named-checkzone kosa.co.kr /etc/bind/db.kosa.co.kr
```

### 5.2 BIND 재시작
```bash
sudo systemctl restart bind9
sudo systemctl enable bind9
```

---

## 6. 방화벽 설정

```bash
sudo ufw allow 53
sudo ufw reload
```

---

## 7. 클라이언트 DNS 설정

### 7.1 Ubuntu
```bash
sudo vi /etc/netplan/00-installer-config.yaml
```

```yaml
nameservers:
  addresses:
    - 192.168.80.5
```

```bash
sudo netplan apply
```

### 7.2 Windows
- 네트워크 설정 → IPv4 → DNS 서버 → 192.168.80.5

---

## 8. DNS 테스트

```bash
nslookup team1.kosa.co.kr
# 결과: 192.168.80.6

nslookup www.team1.kosa.co.kr
# 결과: 192.168.80.6
```

---

## 9. Reverse DNS (역방향 DNS)

### 9.1 정의
- IP → 도메인 매핑
- IP 주소로 도메인 이름 찾기

### 9.2 Zone 선언
```conf
zone "80.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.192.168.80";
};
```

### 9.3 역방향 Zone 파일
```dns
$TTL 3600

@   IN  SOA ns.kosa.co.kr. admin.kosa.co.kr. (
        2025121901
        3600
        1800
        604800
        86400
)

@       IN  NS  ns.kosa.co.kr.

5       IN  PTR ns.kosa.co.kr.
6       IN  PTR team1.kosa.co.kr.
7       IN  PTR team2.kosa.co.kr.
```

### 9.4 테스트
```bash
nslookup 192.168.80.6
# 결과: team1.kosa.co.kr
```

---

## 10. 요약

| 구성 | 설명 |
|------|------|
| **BIND9** | DNS 서버 소프트웨어 |
| **Zone 파일** | 도메인 → IP 매핑 |
| **Forward DNS** | 도메인 → IP |
| **Reverse DNS** | IP → 도메인 |
| **forwarders** | 상위 DNS (Google) |