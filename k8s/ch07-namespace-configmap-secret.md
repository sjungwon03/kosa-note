# Chapter 7: Namespace, ConfigMap, Secret

## 1. Namespace

### 1.1 정의
- 하나의 클러스터를 여러 논리적 가상 클러스터로 나누는 기능
- **폴더(디렉터리)와 비슷**: 같은 이름의 리소스가 ns별로 별개

### 1.2 사용 목적
| 목적 | 설명 |
|------|------|
| 리소스 격리 | 이름 충돌 방지 |
| 접근 제어 (RBAC) | 특정 사용자/팀에만 권한 부여 |
| 리소스 할당량 (Quota) | ns별 CPU/메모리 제한 |

### 1.3 기본 네임스페이스
| 네임스페이스 | 설명 |
|--------------|------|
| **default** | 사용자 리소스 기본 위치 |
| **kube-system** | 시스템 핵심 컴포넌트 (**수정 금지**) |
| **kube-public** | 모든 사용자가 읽을 수 있는 데이터 |
| **kube-node-lease** | 노드 상태 확인용 |

### 1.4 주요 명령어
```bash
kubectl get ns                          # 조회
kubectl create namespace dev            # 생성
kubectl get pods -n dev                 # 특정 ns 파드 조회
kubectl config set-context --current --namespace=dev  # 기본 ns 변경
kubectl delete namespace dev            # 삭제 (내부 리소스 모두 삭제)
```

### 1.5 네임스페이스 간 통신
- 기본적으로 다른 ns의 서비스에 접근 가능
- 주소 형식: `<서비스이름>.<네임스페이스이름>`
```bash
# frontend ns의 파드가 backend ns의 서비스 접근
http://api-server.backend
```

### 1.6 Best Practices
| 사례 | 설명 |
|------|------|
| 환경별 분리 | dev, staging, production |
| 팀/조직별 분리 | frontend-team, backend-team |
| 앱별 분리 | monitoring, logging, ingress-nginx |

---

## 2. ConfigMap

### 2.1 정의
- 설정(Configuration) 데이터를 키-값 쌍으로 저장
- 애플리케이션 코드와 설정을 **분리**

### 2.2 vs Secret
| ConfigMap | Secret |
|-----------|--------|
| 일반 설정 정보 | 민감 정보 (비밀번호, 토큰) |
| Base64 인코딩 없음 | Base64 인코딩 |
| 외부 노출 무관 | 외부 노출 금지 |

### 2.3 생성 방법

**방법 1: literal로 생성**
```bash
kubectl create configmap my-app-config \
  --from-literal=app.mode=production \
  --from-literal=app.version=1.5
```

**방법 2: 파일로 생성**
```bash
kubectl create configmap my-app-config-from-file \
  --from-file=app.mode \
  --from-file=app.properties
```

**방법 3: YAML로 생성 (권장)**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-nginx-config
data:
  app.name: "내가 만든 앱의 제목"
  nginx.conf: |
    server {
        listen 80;
        server_name example.com;
    }
```

### 2.4 파드에서 사용 방법

**방법 1: 환경 변수로 주입**
```yaml
env:
  - name: APP_NAME_IN_POD
    valueFrom:
      configMapKeyRef:
        name: my-nginx-config
        key: app.name
```

**방법 2: 모든 데이터를 환경 변수로 주입**
```yaml
envFrom:
  - configMapRef:
      name: my-nginx-config
```

**방법 3: 볼륨으로 마운트 (가장 많이 사용)**
```yaml
volumeMounts:
  - name: config-volume
    mountPath: /etc/config
volumes:
  - name: config-volume
    configMap:
      name: my-nginx-config
```

### 2.5 ConfigMap 업데이트
- ConfigMap 변경 시 **파드 자동 업데이트 안 됨**
- **Best Practice**: Deployment 재시작 (rollout restart)
```bash
kubectl rollout restart deployment my-app-deployment
```

### 2.6 체크섬 기법 (자동화)
```bash
checksum=$(kubectl get configmap my-app-config -o yaml | sha256sum | awk '{print $1}')
kubectl patch deployment my-app-deployment -p \
  '{"spec":{"template":{"metadata":{"annotations":{"checksum/config": "'$checksum'"}}}}}'
```

---

## 3. Secret

### 3.1 정의
- 비밀번호, OAuth 토큰, SSH 키, API 키 등 **민감 정보** 저장

### 3.2 ⚠️ 보안 경고
- **Base64는 암호화가 아님!**
- 누구나 `kubectl get secret -o yaml`로 디코딩 가능
- 진짜 보안은 **RBAC**로 접근 제어 + **etcd 암호화**

### 3.3 생성 방법

**방법 1: literal로 생성**
```bash
kubectl create secret generic my-db-credentials \
  --from-literal=DB_USER=kosa \
  --from-literal=DB_PASSWORD='kosa1004$'
```

**방법 2: 파일로 생성**
```bash
echo -n 'kosa' > ./username.txt
echo -n 'kosa1004$' > ./password.txt
kubectl create secret generic my-db-credentials-from-file \
  --from-file=DB_USER=./username.txt \
  --from-file=DB_PASSWORD=./password.txt
```

**방법 3: YAML로 생성 (권장)**
```bash
# Base64 인코딩
echo -n 'kosa' | base64        # a29zYQ==
echo -n 'kosa1004$' | base64   # a29zYTEwMDQk
```
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-db-credentials-yaml
type: Opaque
data:
  DB_USER: a29zYQ==
  DB_PASSWORD: a29zYTEwMDQk
```

### 3.4 파드에서 사용 방법

**방법 1: 환경 변수로 주입**
```yaml
env:
  - name: DB_USER
    valueFrom:
      secretKeyRef:
        name: my-db-credentials
        key: DB_USER
```

**방법 2: 볼륨으로 마운트**
```yaml
volumeMounts:
  - name: db-secret-volume
    mountPath: "/etc/db-credentials"
    readOnly: true
volumes:
  - name: db-secret-volume
    secret:
      secretName: my-db-credentials
```

### 3.5 보안 Best Practices
- 최소 권한 원칙: 필요한 Secret만 마운트
- RBAC 강화: 일반 사용자가 Secret 조회 못하게
- etcd 암호화: Encryption at Rest 설정