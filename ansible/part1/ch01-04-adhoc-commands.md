# 4. Ad-Hoc 명령어 (Ad-Hoc Commands)

## 4.1 Ad-Hoc 명령어란?

**Ad-Hoc = '임시의', '특별한 목적을 위한'**

- 플레이북 없이 **일회성**으로 실행하는 단일 태스크
- 신속, 간단한 작업에 적합

### 주요 특징
| 특징 | 설명 |
|------|------|
| 일회성 | 반복 사용 전제하지 않음 |
| 신속성 | 전체 플레이북 작성 필요 없음 |
| 단순 작업 | 서버 재부팅, 상태 확인, 파일 전송 등 |

---

## 4.2 주요 모듈

| 모듈 | 설명 |
|------|------|
| ping | 연결 테스트 |
| command | 기본 명령어 실행 (파이프/리다이렉션 불가) |
| shell | 쉘 명령어 실행 (파이프/리다이렉션 가능) |
| apt/yum/dnf | 패키지 관리 |
| debug | 변수/메시지 출력 |

---

## 4.3 ping: 연결 테스트

```bash
ansible all -m ping -u kosa
```

### 성공 결과
```
192.168.40.176 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3.12"
    },
    "changed": false,
    "ping": "pong"
}
```

---

## 4.4 command vs shell

### command 모듈
- 기본 명령어 실행
- **파이프(|), 리다이렉션(>), 변수($HOME) 불가**
- 더 안전하고 예측 가능

```bash
ansible all -m command -a "df -h" -u kosa
```

### shell 모듈
- `/bin/sh` 쉘 통해 실행
- **파이프, 리다이렉션 등 모든 쉘 기능 가능**

```bash
ansible webservers -m shell -a "ps -ef | grep nginx" -u kosa
```

---

## 4.5 apt/yum: 패키지 관리

### apt (Debian/Ubuntu)
```bash
ansible webservers -m apt -a "name=nginx state=latest" \
  -u kosa --become --ask-become-pass
```

### yum (RedHat/CentOS)
```bash
ansible webservers -m yum -a "name=httpd state=present" \
  -u kosa --become --ask-become-pass
```

### state 옵션
| state | 설명 |
|-------|------|
| present | 설치된 상태 보장 |
| latest | 최신 버전 설치/업데이트 |
| absent | 패키지 삭제 |

---

## 4.6 debug: 메시지 출력

```bash
ansible localhost -m debug -a 'msg="Hello Ansible!"'
```

### 결과
```
localhost | SUCCESS => {
    "msg": "Hello Ansible!"
}
```

---

## 4.7 주요 옵션

| 옵션 | 설명 |
|------|------|
| `-m` | 모듈 지정 |
| `-a` | 모듈 인자 |
| `-u` | 접속 사용자 |
| `-i` | 인벤토리 파일 |
| `--become` | sudo 권한 상승 |
| `--ask-become-pass` | sudo 비밀번호 입력 |
| `-k` | SSH 비밀번호 입력 |

---

## 4.8 사용 예시

```bash
# 모든 서버 연결 테스트
ansible all -m ping

# 웹 서버 재부팅
ansible webservers -b -a "/sbin/reboot" --ask-become-pass

# 디스크 사용량 확인
ansible all -b -a "df -h" --ask-become-pass
```