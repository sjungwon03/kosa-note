# KOSA 클라우드 인프라 엔지니어링 학습 노트

## 📁 디렉토리 구조

```
kosa-note/
├── ansible/          # Ansible 학습 노트
├── aws/              # AWS 클라우드 학습 노트
├── argocd/           # ArgoCD GitOps 학습 노트
├── k8s/              # Kubernetes 학습 노트
├── msa/              # MSA & FlaskApp 학습 노트
├── proxmox/          # Proxmox & Ceph 학습 노트
└── README.md
```

---

## 📚 학습 노트 목록

### Ansible
IT 자동화 및 구성 관리 학습 노트

| Chapter | 내용 |
|---------|------|
| **Part 1-1** | [Ansible 소개](ansible/part1/ch01-01-ansible-intro.md) |
| **Part 1-2** | [환경 구축](ansible/part1/ch01-02-environment-setup.md) |
| **Part 1-3** | [Inventory](ansible/part1/ch01-03-inventory.md) |
| **Part 1-4** | [Ad-Hoc 명령어](ansible/part1/ch01-04-adhoc-commands.md) |
| **Part 2-1** | [플레이북 기초](ansible/part2/ch02-01-플레이북-기초.md) |
| **Part 2-2** | [핵심 모듈](ansible/part2/ch02-02-핵심-모듈.md) |
| **Part 2-3** | [변수](ansible/part2/ch02-03-변수.md) |
| **Part 2-4** | [조건문과 루프](ansible/part2/ch02-04-조건문과-루프.md) |
| **Part 2-5** | [Handlers](ansible/part2/ch02-05-handlers.md) |

**주요 학습 내용:**

- Agentless 구조, YAML 기반 선언적 구성
- Playbook, Inventory, Module 활용
- 멱등성(Idempotency) 보장

---

### AWS (Amazon Web Services)

AWS 클라우드 서비스 학습 노트

| Chapter                | 내용                                                              |
| ---------------------- | ----------------------------------------------------------------- |
| **1. AWS 입문**        | [글로벌 인프라, 공동책임모델, IAM](aws/ch01-aws-intro.md)         |
| **2. 컴퓨팅**          | [EC2, AMI, 인스턴스 유형, ECS/EKS, Lambda](aws/ch02-computing.md) |
| **3. 네트워킹**        | [VPC, 서브넷, IGW, NACL, 보안그룹](aws/ch03-networking.md)        |
| **4. 스토리지**        | [EBS, S3, EFS, 인스턴스 스토어](aws/ch04-storage.md)              |
| **5. 데이터베이스**    | [RDS MySQL, DynamoDB](aws/ch05-database.md)                       |
| **6. 모니터링 & 확장** | [CloudWatch, ELB, Auto Scaling](aws/ch06-monitoring-scaling.md)   |

**주요 학습 내용:**

- EC2 인스턴스 생성 및 관리
- VPC 네트워크 구성
- S3/RDS 데이터 저장
- 고가용성 아키텍처 구축

---

### ArgoCD

GitOps 기반 Continuous Delivery 학습 노트

| Chapter               | 내용                                                                        |
| --------------------- | --------------------------------------------------------------------------- |
| **1. ArgoCD 개요**    | [GitOps 개념, 구성 요소, 동작 흐름](argocd/chap1/ch01-argocd-overview.md)   |
| **2. ArgoCD 설치**    | [kubectl 설치, UI/CLI 설정](argocd/chap2/ch02-argocd-install.md)            |
| **3. Application**    | [Application YAML, SyncPolicy](argocd/chap3/ch03-argocd-application.md)     |
| **4. K8s Manifest**   | [Deployment, Service, Dockerfile](argocd/chap4/ch04-kubernetes-manifest.md) |
| **5. GitHub Actions** | [CI/CD Pipeline, 자동 배포](argocd/chap5/ch05-github-actions.md)            |

**주요 학습 내용:**

- Git → Kubernetes 자동 배포
- Drift 감지 및 자동 복구
- GitHub Actions 연동 CI/CD

---

### Kubernetes (k8s)

컨테이너 오케스트레이션 학습 노트

| Chapter                           | 내용                                                              |
| --------------------------------- | ----------------------------------------------------------------- |
| **5. 기본**                       | [쿠버네티스 기본 개념](k8s/ch05-쿠버네티스-기본.md)               |
| **6. 핵심 오브젝트**              | [Pod, ReplicaSet, Deployment, Service](k8s/ch06-핵심-오브젝트.md) |
| **7. Namespace/ConfigMap/Secret** | [네임스페이스, 설정관리](k8s/ch07-namespace-configmap-secret.md)  |
| **8. Ingress**                    | [인그레스 네트워크 설정](k8s/ch08-ingress.md)                     |
| **9. Volume/PV/PVC**              | [스토리지 관리](k8s/ch09-volume-pv-pvc.md)                        |
| **10. 보안/RBAC**                 | [인증/인가, RBAC](k8s/ch10-보안-rbac.md)                          |
| **11-1. 리소스 관리**             | [Request/Limit, QoS](k8s/ch11-1-리소스-관리.md)                   |
| **11-2. 스케줄링**                | [스케줄러, Node Selector](k8s/ch11-2-스케줄링.md)                 |
| **11-3. Pod 생명주기/HPA**        | [HPA, 생명주기](k8s/ch11-3-pod-생명주기-hpa.md)                   |
| **12. CRD/Operator**              | [Custom Resource, Operator](k8s/ch12-crd-operator.md)             |
| **13. Job/CronJob/DaemonSet**     | [배치 워크로드](k8s/ch13-job-cronjob-daemonset-statefulset.md)    |
| **14. 모니터링**                  | [프로메테우스, 그라파나](k8s/ch14-모니터링.md)                    |

**주요 학습 내용:**

- Pod, Deployment, Service 구성
- PV/PVC 스토리지 관리
- Ingress 네트워크 설정
- RBAC 보안 설정

---

### MSA (Microservices Architecture)

MSA 및 FlaskApp 프로젝트 학습 노트

| Chapter                       | 내용                                                       |
| ----------------------------- | ---------------------------------------------------------- |
| **1. MSA 개요**               | [MSA vs Monolith, 아키텍처 구조](msa/ch01-msa-overview.md) |
| **2. Gateway & Nginx**        | [API Gateway, Reverse Proxy](msa/ch02-gateway-nginx.md)    |
| **3. Auth & Employee Server** | [JWT 인증, CRUD API](msa/ch03-auth-employee-server.md)     |
| **4. Photo Service**          | [사진 저장, 이미지 처리](msa/ch04-photo-service.md)        |
| **5. Docker Compose**         | [서비스 orchestration](msa/ch05-docker-compose.md)         |
| **6. Galera Cluster**         | [MariaDB Galera, Ceph RBD](msa/ch06-galera-cluster.md)     |

**주요 학습 내용:**

- FastAPI 기반 마이크로서비스
- Gateway/Nginx 라우팅
- JWT 토큰 인증
- Galera Cluster DB 구성

---

### Proxmox & Ceph

가상화 및 분산 스토리지 학습 노트

| Chapter                  | 내용                                                      |
| ------------------------ | --------------------------------------------------------- |
| **1. Proxmox 기본**      | [설치, VM/CT 생성](proxmox/ch01-proxmox-기본.md)          |
| **2. 클러스터링 & Ceph** | [HA 구성, Ceph 스토리지](proxmox/ch02-클러스터링-ceph.md) |
| **3. pfSense 네트워크**  | [방화벽, VLAN 구성](proxmox/ch03-pfsense-네트워크.md)     |
| **4. DNS 서버**          | [BIND9, DNS 설정](proxmox/ch04-dns-서버.md)               |
| **5. VPN/IPsec**         | [VPN 연결, IPsec 구성](proxmox/ch05-vpn-ipsec.md)         |
| **6. Terraform Proxmox** | [IaC 자동화](proxmox/ch06-terraform-proxmox.md)           |

**주요 학습 내용:**

- Proxmox VE 클러스터 구성
- Ceph 분산 스토리지
- pfSense 네트워크 보안
- Terraform 인프라 자동화

---

## 🔗 강의자료

| 주제                         | GitHub Repository                                      |
| ---------------------------- | ------------------------------------------------------ |
| **AWS Technical Essentials** | https://github.com/masungil70/Aws-Technical-Essentials |
| **Infra (Proxmox)**          | https://github.com/masungil70/infra                    |
| **FlaskApp (MSA)**           | https://github.com/masungil70/FlaskApp                 |
| **Docker & Kubernetes**      | https://github.com/masungil70/docker-kubernetes        |
| **ArgoCD Exam**              | https://github.com/masungil70/argocd_exam              |
| **Ansible**                  | https://github.com/masungil70/ansible                  |

---
