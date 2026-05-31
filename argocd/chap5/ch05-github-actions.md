# Chapter 5: GitHub Actions CI/CD

## 5.1 CI/CD Pipeline 구조

### 전체 흐름
```
GitHub Push
    ↓
GitHub Actions (CI)
    ↓
Docker Build
    ↓
Docker Hub Push
    ↓
Kubernetes Manifest Update (image tag)
    ↓
Git Push (manifest repo)
    ↓
ArgoCD (자동 감지)
    ↓
Kubernetes 자동 배포
```

---

## 5.2 GitHub Actions Workflow

### deploy.yml
```yaml
name: CI-CD Pipeline

permissions:
  contents: write

on:
  push:
    branches:
      - main

env:
  IMAGE_NAME: masungil/echo-hostname

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Login to DockerHub
      uses: docker/login-action@v3
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}

    - name: Build Docker image
      run: |
        docker build -t $IMAGE_NAME:${{ github.sha }} echo-hostname/.

    - name: Push Docker image
      run: |
        docker push $IMAGE_NAME:${{ github.sha }}

    - name: Update Kubernetes manifest
      run: |
        cd app_exam-k8s/k8s/

        sed -i "s|image:.*|image: $IMAGE_NAME:${{ github.sha }}|g" deployment.yaml

        git config user.name "github-actions"
        git config user.email "github-actions@github.com"

        git add .
        git commit -m "update image tag ${{ github.sha }}"
        git push https://x-access-token:${{ secrets.GH_TOKEN }}@github.com/masungil70/argocd_exam.git
```

---

## 5.3 Workflow 구성 요소

### Trigger
```yaml
on:
  push:
    branches:
      - main
```

### Environment Variables
```yaml
env:
  IMAGE_NAME: masungil/echo-hostname
```

### Steps
| Step | 설명 |
|------|------|
| Checkout | Git clone |
| Login DockerHub | Docker 인증 |
| Build | 이미지 생성 |
| Push | Docker Hub 업로드 |
| Update manifest | deployment.yaml 이미지 태그 변경 |
| Git push | manifest repo 업데이트 |

---

## 5.4 GitHub Secrets 설정

### Secrets 목록
| Secret | 설명 |
|--------|------|
| `DOCKER_USERNAME` | Docker Hub ID |
| `DOCKER_PASSWORD` | Docker Hub Password |
| `GH_TOKEN` | GitHub Personal Access Token |

### 설정 위치
```
GitHub → Settings → Secrets and variables → Actions → New repository secret
```

---

## 5.5 이미지 태그 업데이트

### sed 명령
```bash
sed -i "s|image:.*|image: $IMAGE_NAME:${{ github.sha }}|g" deployment.yaml
```

### 결과
```yaml
# 변경 전
image: masungil/echo-hostname:1.0

# 변경 후
image: masungil/echo-hostname:457a5ca0d489404f5884019f4c2244db3bf58613
```

---

## 5.6 자동 배포 테스트

### 코드 수정
```python
# main.py
return {
    "message": f"Hello from version: v2.0",
    ...
}
```

### Git push
```bash
git add .
git commit -m "update to v2.0"
git push origin main
```

### 자동 실행
```
GitHub Actions → 실행
ArgoCD → sync
Pod → 자동 재배포
```

---

## 5.7 ArgoCD Application 확인

### 상태 확인
```bash
argocd app get echo-hostname
```

### 결과
```
Sync Status:        Synced to main (c796417)
Health Status:      Healthy

GROUP  KIND        NAMESPACE  NAME                 STATUS  HEALTH
       Service     default    echo-hostname-svc    Synced  Healthy
apps   Deployment  default    hostname-deployment  Synced  Healthy
```

---

## 5.8 수동 Sync

```bash
argocd app sync echo-hostname --prune
```

### --prune 옵션
- Git에서 삭제된 리소스 → 클러스터에서도 삭제