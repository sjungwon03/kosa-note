# Chapter 1: ArgoCD 개요

## 1.1 ArgoCD 정의

**ArgoCD** = Kubernetes 환경에서 사용하는 **GitOps 기반 CD(Continuous Delivery) 도구**

> "Git에 있는 설정 그대로 Kubernetes를 자동으로 맞춰주는 도구"

---

## 1.2 GitOps 개념

### 정의
- Git 저장소를 **단일 진실 (Source of Truth)** 로 사용
- 클러스터 상태 = Git 상태와 항상 동일

### 핵심 원칙
```
Git에 YAML 수정 → 자동으로 Kubernetes 반영
```

---

## 1.3 ArgoCD 역할

| 역할 | 설명 |
|------|------|
| **배포 자동화** | Git YAML/Helm/Kustomize → Kubernetes 자동 배포 |
| **상태 동기화** | Git vs 클러스터 상태 비교 → 다르면 자동 수정 |
| **Drift 감지** | 원래 상태와 다른지 지속 감시 |
| **자동 복구** | kubectl로 몰래 수정 → ArgoCD가 원래대로 복구 |

---

## 1.4 ArgoCD 구성 요소

| 구성 요소 | 역할 |
|----------|------|
| **API Server** | UI / CLI / API 제공 |
| **Repository Server** | Git repo 가져오기 |
| **Application Controller** | 실제 상태 동기화 |
| **Redis** | 캐싱 |

---

## 1.5 동작 흐름

```
1. Git에 YAML push
2. ArgoCD가 변경 감지
3. Kubernetes 적용
4. 상태 계속 모니터링
```

---

## 1.6 전체 아키텍처

```
┌─────────────────────────────────────────────────────┐
│                  Git Repository                     │
│          (Kubernetes YAML / Helm / Kustomize)       │
└───────────────────────┬─────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│                    ArgoCD                           │
│   ┌─────────────────────────────────────────────┐   │
│   │ - Repository Server (Git 감시)              │   │
│   │ - Application Controller (Sync)             │   │
│   │ - API Server (UI/CLI)                       │   │
│   └─────────────────────────────────────────────┘   │
└───────────────────────┬─────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│              Kubernetes Cluster                     │
│   ┌─────────────────────────────────────────────┐   │
│   │ Deployment / Service / Pod                  │   │
│   └─────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

---

## 1.7 ArgoCD vs Traditional CD

| 특징 | Traditional CD | ArgoCD (GitOps) |
|------|---------------|-----------------|
| **배포 방식** | Push (CI에서 push) | Pull (ArgoCD가 감시) |
| **상태 저장** | CI 시스템 | Git |
| **롤백** | 복잡 | git revert로 간단 |
| **감시** | 없음 | 실시간 Drift 감지 |
| **보안** | CI에 kubectl 권한 필요 | ArgoCD만 권한 필요 |

---

## 1.8 프로젝트 구조

### argocd_exam 폴더 구조
```
argocd_exam/
├── echo-hostname/          # 애플리케션 소스
│   ├── Dockerfile
│   ├── main.py
│   ├── requirements.txt
│   └── build.sh
├── app_exam-k8s/           # ArgoCD가 보는 폴더
│   ├── k8s/
│   │   ├── deployment.yaml
│   │   └── svc.yaml
│   └── argocd/
│       └── echo-hostname.yaml
├── .github/
│   └── workflows/
│       └── deploy.yml
└── readme.md
```

### 2 Repo 구조 (실무 표준)
| Repo | 역할 |
|------|------|
| **Source Repo** | 애플리케션 코드 (echo-hostname) |
| **Manifest Repo** | Kubernetes YAML (app_exam-k8s) |