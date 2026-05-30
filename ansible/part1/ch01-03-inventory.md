# 3. 인벤토리 (Inventory)

## 3.1 인벤토리란?

**인벤토리 = "Ansible이 관리할 대상 서버들의 목록과 정보가 담긴 파일"**  
→ 서버 주소록 역할

### 핵심 기능
- **관리 대상 정의**: 어떤 서버를 자동화할지 명시
- **그룹화**: 역할/환경에 따라 서버 묶기
- **변수 저장**: 각 서버/그룹에 특화된 변수 저장

### 종류
| 종류 | 설명 |
|------|------|
| **정적 인벤토리** | 관리자가 직접 파일에 작성 |
| **동적 인벤토리** | 클라우드/CMDB에서 실시간 가져오기 |

---

## 3.2 INI 형식 (.ini)

### 기본 구조
```ini
# 가장 기본적인 형태
192.168.1.10
server.example.com

# 호스트 별칭과 연결 변수
web1 ansible_host=192.168.1.11 ansible_user=ubuntu
db1 ansible_host=192.168.1.21 ansible_port=2222
```

### 파일 예시
```ini
[webservers]
web1.example.com
web2.example.com ansible_host=192.168.10.20

[dbservers]
db1 ansible_host=192.168.40.175
db2 ansible_host=192.168.40.176

[backup]
backup1 ansible_host=192.168.40.175
```

---

## 3.3 YAML 형식 (.yml)

### 기본 구조
```yaml
all:
  hosts:
    ungrouped_host:
      ansible_host: 192.168.40.175
      ansible_user: kosa
  children:
    webservers:
      hosts:
        web1.example.com:
          ansible_host: 192.168.40.175
          ansible_user: kosa
        web2.example.com:
          ansible_host: 192.168.40.176
          ansible_user: kosa
    dbservers:
      hosts:
        db1:
          ansible_host: 192.168.40.177
          ansible_user: kosa
          ansible_port: 2222
```

---

## 3.4 호스트 그룹핑

### 목적
- **역할 기반**: webservers, dbservers, api_servers
- **환경 기반**: production, staging, development
- **지역 기반**: datacenter_seoul, datacenter_tokyo

### 중첩 그룹 (INI)
```ini
[webservers]
web1 ansible_host=...
web2 ansible_host=...

[dbservers]
db1 ansible_host=...

[datacenter_a:children]
webservers
dbservers
```

### 중첩 그룹 (YAML)
```yaml
all:
  children:
    webservers:
      hosts:
        web1: ...
        web2: ...
    dbservers:
      hosts:
        db1: ...
    datacenter_a:
      children:
        webservers:
        dbservers:
```

---

## 3.5 그룹 변수 (Group Variables)

### INI 형식
```ini
[webservers]
web1 ansible_host=192.168.10.10
web2 ansible_host=192.168.10.20

[webservers:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=~/.ssh/web_key.pem
http_port=80
ntp_server=time.google.com
```

### YAML 형식
```yaml
all:
  children:
    webservers:
      hosts:
        web1:
          ansible_host: 192.168.10.10
        web2:
          ansible_host: 192.168.10.20
      vars:
        ansible_user: ubuntu
        ansible_ssh_private_key_file: ~/.ssh/web_key.pem
        http_port: 80
        ntp_server: time.google.com
```

---

## 3.6 변수 우선순위

> **더 구체적인 변수가 덜 구체적인 변수를 덮어쓴다**

| 순위 | 변수 종류 |
|------|----------|
| 1 (最高) | 호스트 변수 |
| 2 | 그룹 변수 |
| 3 (最低) | `all` 그룹 변수 |

### 예시
- `group_vars/webservers.yml`: `http_port: 80`
- `host_vars/web01.yml`: `http_port: 8080`
  
→ web01 실행 시: **8080** (호스트 변수 우선)