# Chapter 2: ArgoCD 설치 및 설정

## 2.1 ArgoCD 설치

### Namespace 생성
```bash
kubectl create namespace argocd
```

### ArgoCD 설치
```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 설치 확인
```bash
kubectl get pods -n argocd
kubectl get svc -n argocd
```

---

## 2.2 UI 접속 설정

### NodePort로 변경
```bash
kubectl patch svc argocd-server -n argocd \
  -p '{"spec": {"type": "NodePort"}}'
```

### 포트 확인
```bash
kubectl get svc -n argocd
# argocd-server NodePort  <NODE_IP>:<NODE_PORT>
```

### 접속
```
https://<NODE_IP>:<NODE_PORT>
```

---

## 2.3 초기 비밀번호

### 비밀번호 확인
```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

### 로그인 정보
```
ID: admin
PW: <위에서 나온 값>
```

---

## 2.4 ArgoCD CLI 설치

### Linux 설치
```bash
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd
sudo mv argocd /usr/local/bin/
```

### 버전 확인
```bash
argocd version
```

---

## 2.5 CLI 로그인

### 서버 IP 확인
```bash
kubectl get svc -n argocd
# argocd-server NodePort  192.168.80.165:30118
```

### 로그인
```bash
argocd login 192.168.80.165:30118 --insecure
# Username: admin
# Password: <initial password>
```

---

## 2.6 ArgoCD 주요 리소스

| 리소스 | Kind | 설명 |
|--------|------|------|
| **Application** | argoproj.io/v1alpha1 | 배포할 앱 정의 |
| **AppProject** | argoproj.io/v1alpha1 | 앱 그룹/권한 관리 |
| **Repository** | v1 | Git 저장소 연결 |

---

## 2.7 설치 후 상태

### 확인 명령
```bash
kubectl get all -n argocd
```

### Pod 목록
```
argocd-application-controller
argocd-api-server
argocd-repo-server
argocd-redis
argocd-dex-server (SSO용)
```