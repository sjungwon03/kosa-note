# Part 2: notify와 handlers

## 5.1 notify와 handlers란?

**handlers**: 특정 조건에서만 실행되는 특별한 태스크  
**notify**: 태스크가 "changed" 상태일 때 핸들러 호출

### 사용 이유 (멱등성)
- 설정 파일 변경 → 서비스 재시작
- 불필요한 재시작 방지 → 안정성 향상

### 동작 원리

| 특징 | 설명 |
|------|------|
| 실행 시점 | 모든 tasks 완료 후 실행 |
| 고유성 | 동일 핸들러는 플레이당 1회만 실행 |
| 호출 방식 | 핸들러 이름(`name`) 기반 |
| 실패 처리 | 알림 태스크 실패 → 핸들러 실행 안 함 |

---

## 5.2 기본 예제

```yaml
- name: nginx 설정 및 재시작
  hosts: webservers
  become: yes

  tasks:
    - name: nginx 설치
      apt:
        name: nginx
        state: present

    - name: nginx 시작
      service:
        name: nginx
        state: started

    - name: nginx.conf 복사
      copy:
        src: conf/nginx.conf
        dest: /etc/nginx/nginx.conf
        owner: root
        mode: '0644'
      notify: Restart nginx  # 변경 시 핸들러 호출

  handlers:
    - name: Restart nginx
      service:
        name: nginx
        state: restarted
```

### 실행 시나리오
| 상황 | 결과 |
|------|------|
| 최초 실행 | 파일 changed → 핸들러 실행 → 재시작 |
| 두 번째 실행 | 파일 ok → 핸들러 호출 안 함 → 재시작 안 함 |

---

## 5.3 여러 태스크 → 하나의 핸들러

```yaml
- name: 여러 설정 파일 변경
  hosts: webservers
  become: yes
  vars:
    http_port: 8080
    admin_email: admin@example.com

  tasks:
    - name: nginx 설치
      apt:
        name: nginx
        state: present

    - name: nginx.conf 복사
      copy:
        src: conf/nginx.conf
        dest: /etc/nginx/nginx.conf
      notify: Restart nginx

    - name: myapp.conf 템플릿
      template:
        src: templates/myapp.conf.j2
        dest: /etc/nginx/conf.d/myapp.conf
      notify: Restart nginx  # 동일 핸들러

  handlers:
    - name: Restart nginx
      service:
        name: nginx
        state: restarted
```

### 실행 결과
| 파일 변경 | 핸들러 실행 |
|----------|-------------|
| nginx.conf만 변경 | 1회 실행 |
| myapp.conf만 변경 | 1회 실행 |
| 두 파일 모두 변경 | 1회 실행 (알림 2회 → 실행 1회) |

---

## 5.4 핸들러 고급 기능

### 여러 핸들러 호출
```yaml
- name: 설정 파일 복사
  copy:
    src: app.conf
    dest: /etc/app/app.conf
  notify:
    - Restart nginx
    - Restart app
```

### 핸들러 체이닝
```yaml
handlers:
  - name: Restart nginx
    service:
      name: nginx
      state: restarted
    notify: Clear cache  # 다른 핸들러 호출

  - name: Clear cache
    file:
      path: /var/cache/app
      state: absent
```

### 핸들러 즉시 실행
```yaml
tasks:
  - name: 중요 설정 변경
    copy:
      src: critical.conf
      dest: /etc/critical.conf
    notify: Restart service
    # 다음 태스크 전에 핸들러 실행 필요
    # meta: flush_handlers 사용
```