# Chapter 6: Terraform & Proxmox 연동

## 1. Terraform 개요

### 1.1 정의
- **IaC (Infrastructure as Code)** 도구
- 코드로 인프라 정의 → 자동 배포

### 1.2 Proxmox + Terraform
- VM 생성, 설정, 삭제 → 코드로 자동화
- 일관된 인프라 관리

---

## 2. Proxmox 서버 설정

### 2.1 Role 생성
1. 데이터센터 → 권한 → 역할 → 생성
2. Role ID: TerraformProv
3. 권한 체크:
```
Datastore.AllocateSpace
Datastore.Audit
VM.Allocate
VM.Audit
VM.Clone
VM.Config.CDROM
VM.Config.Cloudinit
VM.Config.CPU
VM.Config.Disk
VM.Config.Memory
VM.Config.Network
VM.PowerMgmt
```

### 2.2 User 생성
1. 데이터센터 → 권한 → 사용자 → 추가
2. 사용자 이름: terraform-prov
3. Realm: Proxmox VE authentication
4. 암호: 설정

### 2.3 Permission 할당
1. 데이터센터 → 권한 → 사용자 권한 추가
2. 경로: / (루트)
3. 사용자: terraform-prov@pve
4. 역할: TerraformProv

### 2.4 API Token 생성
1. 데이터센터 → 권한 → API 토큰 → 추가
2. 사용자: terraform-prov@pve
3. Token ID: terraform-token
4. 권한 분리: 체크 해제

**⚠️ 중요: Token ID와 Secret 즉시 복사 저장!**

---

## 3. Terraform 설치

### 3.1 Windows 수동 설치
1. HashiCorp 다운로드 페이지 접속
2. Windows amd64 버전 다운로드
3. 압축 해제 → terraform.exe
4. C:\Program Files\Terraform 폴더로 이동
5. PATH 환경 변수에 추가

### 3.2 설치 확인
```bash
terraform --version
```

---

## 4. VM 템플릿 준비

### 4.1 Cloud 이미지 다운로드
```bash
ssh root@proxmox-server

qm create 9000 --name ubuntu-2204-template

wget https://cloud-images.ubuntu.com/releases/jammy/release/jammy-server-cloudimg-amd64.img

qm importdisk 9000 jammy-server-cloudimg-amd64.img local-lvm

qm set 9000 --scsihw virtio-scsi-pci --scsi0 local-lvm:vm-9000-disk-0

qm template 9000
```

---

## 5. Terraform 구성 파일

### 5.1 main.tf
```hcl
terraform {
  required_providers {
    proxmox = {
      source  = "telmate/proxmox"
      version = "3.0.2-rc06"
    }
  }
}

provider "proxmox" {
  pm_api_url          = var.proxmox_api_url
  pm_api_token_id     = var.proxmox_api_token_id
  pm_api_token_secret = var.proxmox_api_token_secret
  pm_tls_insecure     = true
}

variable "proxmox_api_url" {
  type        = string
  sensitive   = true
}

variable "proxmox_api_token_id" {
  type        = string
  sensitive   = true
}

variable "proxmox_api_token_secret" {
  type        = string
  sensitive   = true
}

variable "vm_name" {
  type = string
}

variable "proxmox_node" {
  type = string
}

variable "template_name" {
  type = string
}

variable "ssh_public_key" {
  type      = string
  sensitive = true
}

resource "proxmox_vm_qemu" "ubuntu_vm" {
  name        = var.vm_name
  target_node = var.proxmox_node
  clone       = var.template_name
  full_clone  = true

  os_type = "cloud-init"
  agent   = 1

  cpu {
    cores   = 2
    sockets = 1
  }

  memory  = 2048
  scsihw  = "virtio-scsi-pci"
  boot    = "order=scsi0"

  disk {
    slot    = "scsi0"
    size    = "32G"
    type    = "disk"
    storage = "local-lvm"
  }

  disk {
    slot    = "ide2"
    type    = "cloudinit"
    storage = "local-lvm"
  }

  network {
    id     = 0
    model  = "virtio"
    bridge = "vmbr0"
  }

  ipconfig0 = "ip=dhcp"
  ciuser    = "kosa"
  cipassword = "kosa1004"
  sshkeys   = var.ssh_public_key
}
```

### 5.2 terraform.tfvars
```hcl
proxmox_api_url          = "https://192.168.0.3:8006/api2/json"
proxmox_api_token_id     = "terraform-prov@pve!terraform-token"
proxmox_api_token_secret = "e3f31683-xxx"

vm_name       = "kosa-vm-01"
proxmox_node  = "masungil1"
template_name = "ubuntu-2204-template"
ssh_public_key = "ssh-rsa AAAAB3..."
```

---

## 6. Terraform 실행

### 6.1 초기화
```bash
terraform init
# Provider 다운로드
```

### 6.2 실행 계획 확인
```bash
terraform plan
# 변경 사항 미리 확인 (실제 변경 없음)
```

### 6.3 적용
```bash
terraform apply
# → yes 입력 → VM 생성
```

### 6.4 삭제
```bash
terraform destroy
# → Terraform으로 생성한 리소스 모두 삭제
```

---

## 7. Cloud-Init Snippets

### 7.1 Snippets 활성화
- 데이터센터 → Storage → local → 수정 → Snippets 추가

### 7.2 qemu-guest-agent.yml 생성
```bash
tee /var/lib/vz/snippets/qemu-guest-agent.yml <<EOF
#cloud-config
runcmd:
 - apt update
 - apt install -y qemu-guest-agent
 - systemctl start qemu-guest-agent
EOF
```

### 7.3 main.tf에 추가
```hcl
cicustom = "vendor=local:snippets/qemu-guest-agent.yml"
```

---

## 8. Terraform 학습 단계

| 단계 | 내용 |
|------|------|
| Step 1 | 기본 VM 생성 |
| Step 2 | 변수 사용 |
| Step 3 | Cloud-Init 설정 |
| Step 4 | 네트워크 설정 |
| Step 5 | 디스크 설정 |
| Step 6 | Multiple VM |
| Step 7 | Module 사용 |
| Step 8 | Ceph 연동 |

---

## 9. 요약

| 구성 | 설명 |
|------|------|
| **Terraform** | IaC 도구 |
| **Provider** | telmate/proxmox |
| **API Token** | Proxmox 접속용 |
| **VM Template** | Cloud 이미지 기반 |
| **terraform apply** | 인프라 배포 |
| **terraform destroy** | 리소스 삭제 |