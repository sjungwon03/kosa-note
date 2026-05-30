# 2. 기본 환경 구축

## 2.1 환경 구축 개요

| 노드 | 설명 |
|------|------|
| **컨트롤 노드** | Ansible 명령을 내리는 중앙 서버 |
| **매니지드 노드** | 컨트롤 노드에 의해 관리되는 대상 서버 |

---

## 2.2 전체 절차

1. 컨트롤 노드: Ansible 설치
2. 매니지드 노드: Python 설치, SSH 접속 계정 확인
3. SSH 키 기반 인증 설정
4. 인벤토리 파일 작성
5. 연결 테스트 (Ad-hoc 명령)
6. 플레이북 작성 및 실행

---

## 2.3 컨트롤 노드 Ansible 설치

### Ubuntu/Debian
```bash
sudo apt update
sudo apt install software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install ansible -y

ansible --version
```

---

## 2.4 매니지드 노드 준비

- **Python 설치 필수** (Ansible 모듈이 Python으로 작성)
- **SSH 접속 계정** 필요

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install python3 -y
```

---

## 2.5 SSH 키 기반 인증 설정

### 1단계: SSH 키 생성 (컨트롤 노드)
```bash
ssh-keygen
# Enter만 눌러서 기본값으로 생성 (~/.ssh/id_rsa, id_rsa.pub)
```

### 2단계: 공개 키 복사
```bash
ssh-copy-id user@192.168.0.10
ssh-copy-id user@192.168.0.11
```

### 3단계: 접속 확인
```bash
ssh user@192.168.0.10
# 비밀번호 없이 접속되면 성공
```

---

## 2.6 인벤토리 파일 작성

### 프로젝트 폴더 생성
```bash
mkdir ex02 && cd ex02
touch inventory.ini
```

### inventory.ini (INI 형식)
```ini
[webservers]
web1 ansible_host=192.168.0.10
web2 ansible_host=192.168.0.11

[dbservers]
db1 ansible_host=192.168.0.20

[all:vars]
ansible_user=user
```

**구성 요소:**
- `[그룹이름]`: 서버 그룹 정의
- `ansible_host`: 실제 접속 IP
- `[all:vars]`: 모든 서버 공통 변수

---

## 2.7 연결 테스트 (Ad-hoc 명령)

```bash
ansible all -i inventory.ini -m ping
```

### 성공 결과
```
web1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
web2 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

---

## 2.8 플레이북 작성 및 실행

### install_nginx.yml
```yaml
---
- name: Install and start Nginx
  hosts: webservers
  become: yes
  tasks:
    - name: Install nginx package
      apt:
        name: nginx
        state: latest
        update_cache: yes

    - name: Start nginx service
      service:
        name: nginx
        state: started
        enabled: yes
```

### 실행
```bash
ansible-playbook -i inventory.ini install_nginx.yml --ask-become-pass
```

---

## 2.9 최종 디렉토리 구조

```
ex02/
├── inventory.ini         # 관리 서버 목록
├── install_nginx.yml     # Nginx 설치 플레이북
└── uninstall_nginx.yml   # Nginx 제거 플레이북
```

---

## 2.10 주요 옵션

| 옵션 | 설명 |
|------|------|
| `hosts: webservers` | inventory.ini의 webservers 그룹 대상 |
| `become: yes` | sudo (root 권한) |
| `tasks` | 실행할 작업 목록 |