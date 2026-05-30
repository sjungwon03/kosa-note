# 1. Ansible 소개

## 1.1 Ansible이란?

**Ansible은 "IT 작업을 아주 쉽게 자동화해주는 오픈소스 도구"입니다.**  
컴퓨터(서버)를 관리할 때 발생하는 반복적인 작업을 미리 작성된 코드(플레이북)를 통해 자동으로 처리합니다.

### 주요 역할 (3가지)

| 역할 | 설명 |
|------|------|
| **구성 관리** | 여러 서버의 상태를 원하는 '특정 상태'로 정의하고 일관되게 유지 |
| **애플리케이션 배포** | 소프트웨어를 실제 서비스 서버로 옮기고 실행시키는 과정 자동화 |
| **오케스트레이션** | 여러 서버에 걸친 복잡한 작업을 정해진 순서에 맞게 조율하고 실행 |

---

## 1.2 구성 관리 예제

### 시나리오
신규 입사한 개발자 `new_dev`를 위해 개발팀 서버에 표준 환경 구성

### inventory.ini
```ini
[dev_servers]
192.168.40.171
192.168.40.172
```

### configure_servers.yml
```yaml
---
- name: 개발 서버 표준 환경 구성
  hosts: dev_servers
  remote_user: kosa
  become: yes
  tasks:
    - name: 신규 개발자 'new_dev' 계정 생성
      ansible.builtin.user:
        name: new_dev
        state: present
        comment: "New Developer"
        shell: /bin/bash

    - name: ufw 패키지 설치
      ansible.builtin.apt:
        name: ufw
        state: present
        update_cache: yes

    - name: ufw 서비스 활성화 및 기본 정책 설정
      community.general.ufw:
        state: enabled
        policy: deny

    - name: 방화벽 허용 규칙 설정
      community.general.ufw:
        rule: allow
        port: "{{ item }}"
        proto: tcp
      loop:
        - "22"
        - "80"

    - name: MOTD 파일 설정
      ansible.builtin.copy:
        content: "이 서버는 Ansible에 의해 관리됩니다.\n"
        dest: /etc/motd
        owner: root
        group: root
        mode: '0644'

    - name: nginx 웹 서버 설치
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: yes

    - name: nginx 서비스 활성화 및 실행
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: yes
```

---

## 1.3 애플리케이션 배포 예제

### 시나리오
v1.2 버전의 웹 애플리케이션을 webservers 그룹에 배포

### deploy_webapp.yml
```yaml
---
- name: 웹 애플리케션 v1.2 배포
  hosts: webservers
  remote_user: kosa
  become: yes
  vars:
    app_source_repo: https://github.com/masungil70/my-webapp.git
    app_deploy_path: /var/www/html
    app_version: v1.2

  tasks:
    - name: Git 설치
      ansible.builtin.apt:
        name: git
        state: present

    - name: /var/www/html 삭제
      ansible.builtin.file:
        path: "{{ app_deploy_path }}"
        state: absent

    - name: Git 저장소에서 최신 소스코드 가져오기
      ansible.builtin.git:
        repo: "{{ app_source_repo }}"
        dest: "{{ app_deploy_path }}"
        version: "{{ app_version }}"
        update: yes
        force: yes

    - name: Nginx 서비스 재시작
      ansible.builtin.service:
        name: nginx
        state: restarted
```

---

## 1.4 오케스트레이션 예제

### 시나리오
DB 스키마 변경 후 웹 애플리케이션 업데이트 (순서 중요)

### orchestrate_update.yml
```yaml
---
# 플레이 1: 데이터베이스 서버 작업
- name: 데이터베이스 스키마 업데이트
  hosts: db_servers
  become: yes
  tasks:
    - name: 데이터베이스 백업
      community.mysql.mysql_db:
        name: all
        state: dump
        target: "/opt/backup/db_backup_{{ ansible_date_time.iso8601 }}.sql"
      run_once: true

    - name: 새로운 DB 스키마 적용
      ansible.builtin.shell: mysql -u root < /opt/new_schema.sql

# 플레이 2: 웹 서버 작업
- name: 웹 애플리케션 업데이트
  hosts: webservers
  become: yes
  tasks:
    - name: 새로운 버전 배포
      ansible.builtin.git:
        repo: https://github.com/mycompany/my-webapp.git
        dest: /var/www/html
        version: main

# 플레이 3: 알림 작업
- name: 작업 완료 알림
  hosts: localhost
  become: no
  tasks:
    - name: 슬랙으로 알림 보내기
      community.general.slack:
        token: "xoxb-your-slack-token-here"
        msg: "DB 스키마 및 웹 애플리케션 업데이트 완료"
        channel: "#devops-alerts"
```

---

## 1.5 Ansible의 특징 및 철학

### 1. 에이전트리스 (Agentless)
- 관리 서버에 **별도 프로그램(Agent) 설치 불필요**
- Linux: SSH, Windows: WinRM으로 통신

**장점:**
- 초기 구성 간편
- 관리 서버 자원 소모 없음
- 에이전트 버전 관리/보안 문제 없음

### 2. 선언적 (Declarative)
- '어떤 상태여야 하는지'를 정의 (명령형과 차이)

| 방식 | 예시 |
|------|------|
| 명령형 (Shell Script) | "직진 500m, 좌회전 후 200m..." |
| 선언형 (Ansible) | "목적지는 서울시청" |

### 3. 멱등성 (Idempotency)
- 작업을 여러 번 실행해도 **결과 항상 동일**
- 이미 설치된 패키지 → 재설치 안 함
- 플레이북 100번 실행 → 100번 바뀌지 않음, 최종 상태로 수렴

### 4. 단순성과 가독성
- YAML 형식 사용 (key: value)
- 프로그래밍 지식 없어도 작성 가능

---

## 1.6 다른 도구와의 차이점

| 구분 | Ansible | Shell Script | Puppet/Chef |
|------|---------|--------------|-------------|
| 관리 구조 | Agentless | Agentless | Agent-based |
| 방식 | 선언적 | 명령형 | 선언적 |
| 언어 | YAML | Bash | DSL/Ruby |
| 멱등성 | 기본 보장 | 직접 구현 | 기본 보장 |
| 학습 곡선 | 낮음 | 낮음 | 높음 |

### vs Shell Script
- 복잡한 로직, 오류 처리, 멱등성 보장 어려움 → Ansible이 해결

### vs Puppet/Chef
- Master-Agent 구조 → 초기 설정 복잡
- Ansible은 Agentless → 가볍고 유연