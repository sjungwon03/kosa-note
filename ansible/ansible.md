# Ansible 학습 노트

## 1. Ansible이란?

### 1.1 정의
- **IT 자동화 도구** (Configuration Management, Deployment, Orchestration)
- 복잡한 IT 작업을 간단하고 사람이 읽기 쉬운 YAML로 자동화

### 1.2 주요 사용 목적
| 목적 | 설명 |
|------|------|
| **구성 관리** | 여러 서버 설정 일관성 유지 |
| **앱 배포** | 애플리케이션 자동 배포 |
| **오케스트레이션** | 여러 서버 순차 작업 실행 |
| **프로비저닝** | 클라우드 인프라 생성 |

---

## 2. Ansible 핵심 특징

### 2.1 Agentless (에이전트리스)
- 관리 서버에 **별도 프로그램 설치 불필요**
- SSH (Linux) / WinRM (Windows)로 통신

**장점:**
- 설치 간편
- 리소스 차지 없음
- 보안 업데이트 부담 적음

### 2.2 제어 노드 vs 관리 노드
| 노드 | 설명 |
|------|------|
| **제어 노드** | Ansible 설치된 명령 내리는 컴퓨터 |
| **관리 노드** | 명령 받아 작업 수행하는 서버 |

### 2.3 멱등성 (Idempotency)
- 작업을 여러 번 실행해도 **결과 항상 동일**
- 이미 설치된 패키지 → 재설치 안 함 (OK 상태)
- 시스템 항상 원하는 상태 유지 보장

---

## 3. Ansible 구성 요소

### 3.1 Playbook (플레이북)
- 자동화 작업의 **'대본'** 또는 '레시피'
- YAML 형식 (사람이 읽기 쉬움)
- Task (작업)의 연속

### 3.2 Inventory (인벤토리)
- 관리할 서버 목록
- IP/도메인 + 그룹 분류 가능

### 3.3 Module (모듈)
- 실제 작업 수행하는 **부품**
- 수천 개 기본 제공

**주요 모듈:**
| 모듈 | 설명 |
|------|------|
| apt | Debian/Ubuntu 패키지 관리 |
| yum | CentOS/RHEL 패키지 관리 |
| service | 서비스 제어 |
| copy | 파일 복사 |
| template | Jinja2 템플릿 |
| file | 파일/디렉터리 관리 |
| user | 사용자 관리 |
| shell | 쉘 명령 실행 |
| command | 명령 실행 |

---

## 4. Ansible 비유

```
지휘자 → 제어 노드 (Control Node)
악보   → 플레이북 (Playbook)
연주자 → 관리 노드 (Managed Nodes)
지시   → 모듈 (Module)
```

---

## 5. Ansible 설치

### 5.1 Ubuntu/Debian
```bash
sudo apt update
sudo apt install ansible -y
```

### 5.2 설치 확인
```bash
ansible --version
```

---

## 6. SSH 키 설정 (필수)

### 6.1 SSH 키 생성
```bash
ssh-keygen
# Enter 몇 번 → 기본값으로 생성
```

### 6.2 공개 키 복사
```bash
ssh-copy-id kosa@managed_node_ip
```

### 6.3 접속 확인
```bash
ssh kosa@managed_node_ip
# 비밀번호 없이 접속
```

---

## 7. Inventory 파일

### 7.1 파일 생성
```bash
mkdir ansible-test
cd ansible-test
touch inventory.ini
```

### 7.2 inventory.ini 예시
```ini
# 웹서버 그룹
[webservers]
192.168.1.10
192.168.1.11

# 데이터베이스 서버 그룹
[dbservers]
db01.example.com

# 그룹 없는 서버
192.168.1.100
```

---

## 8. Ad-Hoc 명령어

### 8.1 정의
- 플레이북 없이 **간단 작업 1회 실행**

### 8.2 연결 테스트 (ping)
```bash
ansible all -i inventory.ini -m ping
```

**결과:**
```
192.168.1.10 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

### 8.3 명령어 실행
```bash
ansible webservers -i inventory.ini -m command -a "uptime"
```

### 8.4 옵션
| 옵션 | 설명 |
|------|------|
| `-i` | 인벤토리 파일 지정 |
| `-m` | 모듈 지정 |
| `-a` | 모듈 인자 |
| `all` | 모든 서버 |
| `webservers` | 특정 그룹 |

---

## 9. Playbook 작성

### 9.1 파일 생성
```bash
touch install_nginx.yml
```

### 9.2 install_nginx.yml
```yaml
---
- name: Nginx 설치 및 실행
  hosts: webservers
  become: yes  # sudo 권한

  tasks:
    - name: Nginx 패키지 설치
      ansible.builtin.apt:
        name: nginx
        state: present    # 설치 상태 보장
        update_cache: yes # 패키지 목록 업데이트

    - name: Nginx 서비스 시작
      ansible.builtin.service:
        name: nginx
        state: started    # 실행 상태 보장
        enabled: yes      # 부팅 시 자동 실행
```

### 9.3 Playbook 구조
```yaml
---
- name: 플레이 이름
  hosts: 대상 그룹
  become: yes/no

  tasks:
    - name: 작업 이름
      모듈:
        파라미터: 값
```

---

## 10. Playbook 실행

### 10.1 실행 명령
```bash
ansible-playbook -i inventory.ini install_nginx.yml
```

### 10.2 실행 결과 (PLAY RECAP)
```
PLAY RECAP *********************************************************************
192.168.1.10               : ok=3    changed=2    unreachable=0    failed=0
192.168.1.11               : ok=3    changed=2    unreachable=0    failed=0
```

| 항목 | 설명 |
|------|------|
| **ok** | 실행한 작업 수 |
| **changed** | 시스템 변경 발생 수 |
| **unreachable** | 접속 불가 수 |
| **failed** | 실패 수 |

---

## 11. 주요 모듈 예시

### 11.1 apt (패키지 관리)
```yaml
- name: Nginx 설치
  ansible.builtin.apt:
    name: nginx
    state: present
    update_cache: yes
```

### 11.2 service (서비스 제어)
```yaml
- name: Nginx 시작
  ansible.builtin.service:
    name: nginx
    state: started
    enabled: yes
```

### 11.3 copy (파일 복사)
```yaml
- name: 설정 파일 복사
  ansible.builtin.copy:
    src: ./nginx.conf
    dest: /etc/nginx/nginx.conf
    owner: root
    group: root
    mode: '0644'
```

### 11.4 template (Jinja2 템플릿)
```yaml
- name: 템플릿 배포
  ansible.builtin.template:
    src: ./nginx.conf.j2
    dest: /etc/nginx/nginx.conf
```

### 11.5 file (파일/디렉터리)
```yaml
- name: 디렉터리 생성
  ansible.builtin.file:
    path: /var/www/html
    state: directory
    mode: '0755'
```

### 11.6 user (사용자 관리)
```yaml
- name: 사용자 생성
  ansible.builtin.user:
    name: deploy
    shell: /bin/bash
    groups: sudo
```

---

## 12. Playbook 실전 예제

### 12.1 전체 예시
```yaml
---
- name: Web Server 구성
  hosts: webservers
  become: yes

  vars:
    nginx_port: 80
    doc_root: /var/www/html

  tasks:
    - name: 패키지 업데이트
      ansible.builtin.apt:
        update_cache: yes

    - name: Nginx 설치
      ansible.builtin.apt:
        name: nginx
        state: present

    - name: 웹 루트 디렉터리 생성
      ansible.builtin.file:
        path: "{{ doc_root }}"
        state: directory
        mode: '0755'

    - name: index.html 생성
      ansible.builtin.copy:
        content: "<h1>Hello Ansible!</h1>"
        dest: "{{ doc_root }}/index.html"

    - name: Nginx 시작
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: yes

  handlers:
    - name: Nginx 재시작
      ansible.builtin.service:
        name: nginx
        state: restarted
```

---

## 13. Variables (변수)

### 13.1 Playbook 내 변수
```yaml
vars:
  nginx_port: 80
  doc_root: /var/www/html
```

### 13.2 변수 사용
```yaml
path: "{{ doc_root }}"
```

### 13.3 Inventory 변수
```ini
[webservers]
192.168.1.10 nginx_port=80
192.168.1.11 nginx_port=8080

[webservers:vars]
doc_root=/var/www/html
```

---

## 14. Handlers

### 14.1 정의
- Task 완료 후 **조건부 실행**
- 설정 파일 변경 → 서비스 재시작

### 14.2 예시
```yaml
tasks:
  - name: nginx.conf 복사
    ansible.builtin.copy:
      src: nginx.conf
      dest: /etc/nginx/nginx.conf
    notify: Nginx 재시작

handlers:
  - name: Nginx 재시작
    ansible.builtin.service:
      name: nginx
      state: restarted
```

---

## 15. Roles

### 15.1 정의
- Playbook을 **재사용 가능한 단위**로 구성
- 변수, Task, Handler, 파일 등 모듈화

### 15.2 Role 구조
```
roles/
  nginx/
    tasks/
      main.yml
    handlers/
      main.yml
    templates/
      nginx.conf.j2
    files/
      index.html
    vars/
      main.yml
    defaults/
      main.yml
```

### 15.3 Role 사용
```yaml
- name: Web Server 구성
  hosts: webservers
  roles:
    - nginx
```

---

## 16. Ansible Galaxy

### 16.1 정의
- Ansible Role **공유 커뮤니티**
- https://galaxy.ansible.com

### 16.2 Role 설치
```bash
ansible-galaxy install geerlingguy.nginx
```

---

## 17. Ansible vs Terraform

| Ansible | Terraform |
|---------|-----------|
| **구성 관리** | **인프라 프로비저닝** |
| 서버 설정 | 클라우드 리소스 생성 |
| Agentless | Provider 기반 |
| YAML | HCL |
| 멱등성 | 멱등성 |
| 반복 실행 가능 | State 파일 기반 |

---

## 18. Best Practices

| Practice | 설명 |
|----------|------|
| SSH 키 사용 | 비밀번호 없이 자동화 |
| Inventory 그룹화 | 서버 분류 관리 |
| Variables 사용 | 재사용성 향상 |
| Handlers 활용 | 변경 시 서비스 재시작 |
| Roles 사용 | 모듈화, 재사용 |
| ansible.cfg | 기본 설정 파일 |

---

## 19. 요약

| 구성 | 설명 |
|------|------|
| **Ansible** | IT 자동화 도구 |
| **Agentless** | SSH 기반, 에이전트 없음 |
| **Playbook** | YAML 작업 대본 |
| **Inventory** | 관리 서버 목록 |
| **Module** | 작업 수행 부품 |
| **Idempotency** | 멱등성 (동일 결과 보장) |