# Chapter 10: 보안 (RBAC)

## 1. 인증 vs 인가

### 1.1 개념
| 인증 (Authentication) | 인가 (Authorization) |
|----------------------|---------------------|
| "당신은 누구입니까?" | "무엇을 할 수 있습니까?" |
| 신원 확인 | 권한 확인 |
| 로그인 | 권한 검사 |

### 1.2 API 요청 처리 순서
```
API 요청 → 인증 → 인가 → 실행
```

---

## 2. 인증 (Authentication)

### 2.1 주요 방식
| 방식 | 설명 |
|------|------|
| **X.509 클라이언트 인증서** | kubectl 기본 방식, CN=사용자명, O=그룹 |
| **Service Account** | 파드 내 프로세스용 ID |
| **OIDC** | 외부 ID 공급자 연동 (Google, Okta) |
| **정적 토큰/비밀번호 파일** | 거의 사용 안 함 (보안 취약) |

### 2.2 X.509 인증서 생성 과정
```bash
# 1. 개인 키 생성
openssl genrsa -out dev-user.key 2048

# 2. CSR 생성
openssl req -new -key dev-user.key -out dev-user.csr -subj "/CN=dev-user"

# 3. CSR base64 인코딩
export CSR_CONTENT=$(cat dev-user.csr | base64 | tr -d '\n')

# 4. CertificateSigningRequest 생성
cat <<EOF > csr.yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: dev-user-csr
spec:
  request: ${CSR_CONTENT}
  signerName: kubernetes.io/kube-apiserver-client
  usages:
  - client auth
EOF

# 5. CSR 생성 및 승인
kubectl apply -f csr.yaml
kubectl certificate approve dev-user-csr

# 6. 인증서 추출
kubectl get csr dev-user-csr -o jsonpath='{.status.certificate}' | base64 --decode > dev-user.crt

# 7. kubeconfig 설정
kubectl config set-credentials dev-user --client-key=dev-user.key --client-certificate=dev-user.crt --embed-certs=true
kubectl config set-context dev-user-context --cluster=kubernetes --user=dev-user --namespace=development
```

---

## 3. 인가 (Authorization) - RBAC

### 3.1 RBAC 구성 요소
| 리소스 | 범위 | 설명 |
|--------|------|------|
| **Role** | 네임스페이스 | 특정 ns 내 권한 정의 |
| **ClusterRole** | 클러스터 전체 | 클러스터 범위 권한 정의 |
| **RoleBinding** | 네임스페이스 | Role을 사용자/SA에 연결 |
| **ClusterRoleBinding** | 클러스터 전체 | ClusterRole을 주체에 연결 |

### 3.2 관계
```
Who? → User, Group, ServiceAccount
What? → Role, ClusterRole
Connection → RoleBinding, ClusterRoleBinding
```

---

## 4. Role 예제

### 4.1 Role 생성
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: development
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
```

### 4.2 RoleBinding 생성
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-in-development
  namespace: development
subjects:
- kind: User
  name: dev-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### 4.3 권한 확인
```bash
# 허용된 작업
kubectl auth can-i list pods --namespace development --as dev-user
# 결과: yes

# 거부된 작업
kubectl auth can-i delete pods --namespace development --as dev-user
# 결과: no

# 다른 네임스페이스
kubectl auth can-i list pods --namespace default --as dev-user
# 결과: no
```

---

## 5. ClusterRole

### 5.1 용도
| 용도 | 설명 |
|------|------|
| 클러스터 자원 | nodes, persistentvolumes |
| 모든 네임스페이스 | 모든 ns의 pods 조회 |

### 5.2 예제
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: nodes-reader
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]
```

---

## 6. ServiceAccount

### 6.1 정의
- 파드 내 프로세스가 API 서버와 통신할 때 사용하는 **"기계용" ID**
- User: 사람용, ServiceAccount: 로봇용

### 6.2 작동 원리
```
1. 파드 생성 → default SA 자동 할당
2. 토큰 마운트 → /var/run/secrets/kubernetes.io/serviceaccount/token
3. API 요청 → Authorization: Bearer <토큰>
4. API 서버 → 인증 + 인가
```

### 6.3 예제: 파드 목록 조회 파드
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pod-lister-sa
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-lister-role
  namespace: default
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-lister-binding
  namespace: default
subjects:
- kind: ServiceAccount
  name: pod-lister-sa
roleRef:
  kind: Role
  name: pod-lister-role
---
apiVersion: v1
kind: Pod
metadata:
  name: pod-lister-test-pod
spec:
  serviceAccountName: pod-lister-sa
  containers:
  - name: kubectl-container
    image: bitnami/kubectl:latest
    command:
      - "/bin/sh"
      - "-c"
      - |
        kubectl get pods
        sleep 3600
```

---

## 7. verbs (동사)

| verb | 설명 |
|------|------|
| get | 단일 리소스 조회 |
| list | 리소스 목록 조회 |
| watch | 리소스 변경 감시 |
| create | 리소스 생성 |
| update | 리소스 수정 |
| delete | 리소스 삭제 |

---

## 8. Best Practices

| Practice | 설명 |
|----------|------|
| 최소 권한 원칙 | 필요한 최소한의 권한만 부여 |
| 네임스페이스 분리 | 팀/환경별 ns로 격리 |
| ServiceAccount 제한 | 파드에 필요한 SA만 할당 |
| 정기 권한 검토 | 불필요한 권한 정기 삭제 |