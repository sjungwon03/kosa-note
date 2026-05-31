# Chapter 3: ArgoCD Application 생성

## 3.1 Application 정의

### Application YAML (echo-hostname.yaml)
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: echo-hostname
  namespace: argocd

spec:
  project: default

  source:
    repoURL: https://github.com/masungil70/argocd_exam.git
    targetRevision: main
    path: app_exam-k8s/k8s   # Kubernetes YAML 위치

  destination:
    server: https://kubernetes.default.svc
    namespace: default

  syncPolicy:
    automated:
      prune: true      # 삭제된 리소스 자동 제거
      selfHeal: true    # Drift 자동 복구
```

---

## 3.2 Application 구성 요소

### Source 설정
| 필드 | 설명 | 예시 |
|------|------|------|
| `repoURL` | Git 저장소 URL | https://github.com/... |
| `targetRevision` | Branch/Tag | main, v1.0 |
| `path` | YAML 위치 | app_exam-k8s/k8s |

### Destination 설정
| 필드 | 설명 | 예시 |
|------|------|------|
| `server` | K8s API Server | https://kubernetes.default.svc |
| `namespace` | 배포 Namespace | default |

### SyncPolicy 설정
| 필드 | 설명 |
|------|------|
| `automated.prune` | Git에서 삭제된 리소스 자동 제거 |
| `automated.selfHeal` | 클러스터 Drift 자동 복구 |

---

## 3.3 Application 생성 방법

### 방법 1: YAML 적용
```bash
kubectl apply -f app_exam-k8s/argocd/echo-hostname.yaml
```

### 방법 2: UI 생성
```
ArgoCD UI → NEW APP

Application Name: echo-hostname
Project: default
Sync Policy: Automatic

Source:
  Repo URL: https://github.com/...
  Path: app_exam-k8s/k8s
  Branch: main

Destination:
  Cluster: https://kubernetes.default.svc
  Namespace: default
```

### 방법 3: CLI 생성
```bash
argocd app create echo-hostname \
  --repo https://github.com/masungil70/argocd_exam.git \
  --path app_exam-k8s/k8s \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default \
  --sync-policy automated
```

---

## 3.4 Application 상태 확인

### CLI 확인
```bash
argocd app get echo-hostname
```

### 결과 예시
```
Name:               argocd/echo-hostname
Project:            default
Server:             https://kubernetes.default.svc
Namespace:          default
URL:                https://192.168.80.165:30118/applications/echo-hostname
Source:
- Repo:             https://github.com/masungil70/argocd_exam
  Target:           main
  Path:             app_exam-k8s/k8s
Sync Policy:        Automated
Sync Status:        Synced to main
Health Status:      Healthy
```

---

## 3.5 Sync 실행

### 자동 Sync
- `syncPolicy.automated` 설정 시 자동 동기화

### 수동 Sync
```bash
argocd app sync echo-hostname
```

### Sync 결과
```
GROUP  KIND        NAMESPACE  NAME                 STATUS  HEALTH
       Service     default    echo-hostname-svc    Synced  Healthy
apps   Deployment  default    hostname-deployment  Synced  Healthy
```

---

## 3.6 Health Status

| 상태 | 설명 |
|------|------|
| **Healthy** | 모든 리소스 정상 |
| **Progressing** | 배포 진행 중 |
| **Degraded** | 장애 발생 |
| **Suspended** | 중지 상태 |

---

## 3.7 Sync Status

| 상태 | 설명 |
|------|------|
| **Synced** | Git = 클러스터 상태 일치 |
| **OutOfSync** | Git ≠ 클러스터 상태 |
| **Unknown** | 상태 확인 불가 |