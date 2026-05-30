# Chapter 6: Galera Cluster (MariaDB)

## 6.1 Galera Cluster 개요

### 정의
- **MariaDB Galera Cluster**: Multi-master 동기 복제 클러스터
- 모든 노드가 **Read/Write 가능**
- 데이터 **실시간 동기화**

### 특징
| 특징 | 설명 |
|------|------|
| **Multi-Master** | 모든 노드 쓰기 가능 |
| **동기 복제** | 데이터 즉시 동기화 |
| **자동 장애 조치** | 노드 장애 시 자동 제외 |
| **확장성** | 노드 추가로 성능 향상 |

---

## 6.2 Ceph + Kubernetes + Galera 구조

### 전체 아키텍처
```
MariaDB Galera Cluster (3 Pods)
    ↓
PVCs (Individual RBD per Pod)
    ↓
RBD CSI Driver
    ↓
Ceph RBD Pool
    ↓
Ceph Cluster (3 nodes)
```

### Ceph Cluster 구성
```
ceph1 = MON + MGR + OSD + RGW (core)
ceph2 = MON + OSD
ceph3 = MON + OSD
```

---

## 6.3 Ceph RBD 설정

### RBD Pool 생성
```bash
ceph osd pool create kube_rbd 32
ceph osd pool set kube_rbd size 3
rbd pool init kube_rbd
```

### Kubernetes용 User 생성
```bash
ceph auth get-or-create client.k8s \
  mon 'profile rbd' \
  osd 'profile rbd' \
  mgr 'profile rbd' \
  -o /etc/ceph/ceph.client.k8s.keyring
```

### Key 추출
```bash
ceph auth get-key client.k8s
# 출력: AQDJvPhpUQ2VIxAAZwcc/FCU3gs2L71i5tMQQw==
```

---

## 6.4 Kubernetes 설정

### Secret 생성
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: ceph-secret
  namespace: ceph-storage
type: Opaque
data:
  userID: azhz  # base64("k8s")
  userKey: QVFEdk5QdHB3cTVlRUJBQTIyZWk2dVp4dTRDL01RUW9rR1c2ekE9PQ==
```

### StorageClass 생성
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ceph-rbd
provisioner: rbd.csi.ceph.com
parameters:
  clusterID: 36edd2cc-4621-11f1-9004-a7cd2b580b3e
  pool: kube_rbd
  imageFormat: "2"
  imageFeatures: layering
  
  csi.storage.k8s.io/provisioner-secret-name: ceph-secret
  csi.storage.k8s.io/provisioner-secret-namespace: ceph-storage
  csi.storage.k8s.io/node-stage-secret-name: ceph-secret
  csi.storage.k8s.io/node-stage-secret-namespace: ceph-storage

reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: Immediate
```

---

## 6.5 Galera Cluster 배포

### galera-values.yaml
```yaml
replicaCount: 3

rootUser:
  password: "root_password"

db:
  name: kosa_db
  user: kosa
  password: "kosa1004"

galera:
  name: kosa-galera-cluster
  replication:
    user: repl_user
    password: "repl_password"

initdbScripts:
  init_schema.sql: |
    CREATE TABLE IF NOT EXISTS kosa_db.items (
      id INT AUTO_INCREMENT PRIMARY KEY,
      name VARCHAR(255) NOT NULL
    ) ENGINE=InnoDB;

persistence:
  enabled: true
  storageClass: "ceph-rbd"
  size: 10Gi

architecture: galera
```

### Helm 설치
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

helm install mariadb bitnami/mariadb-galera \
  -n ceph-storage \
  -f galera-values.yaml
```

---

## 6.6 설치 스크립트 (install.sh)

```bash
# 1. Namespace 생성
kubectl create ns ceph-csi
kubectl create ns ceph-storage

# 2. Secret 적용
kubectl apply -f secret.yaml

# 3. CSI Driver 설치
helm repo add ceph-csi https://ceph.github.io/csi-charts
helm install ceph-csi-rbd ceph-csi/ceph-csi-rbd -n ceph-csi

# 4. ConfigMap 적용
kubectl apply -f configmap.yaml

# 5. StorageClass 생성
kubectl apply -f storageclass.yaml

# 6. MariaDB Galera 설치
helm install mariadb bitnami/mariadb-galera \
  -f galera-values.yaml -n ceph-storage

# 7. 상태 확인
kubectl get all -n ceph-storage
kubectl get pvc -n ceph-storage
```

---

## 6.7 Galera Cluster 구성

### 3 Pods 구조
```
mariadb-0 → PVC: data-mariadb-0 (Ceph RBD 10Gi)
mariadb-1 → PVC: data-mariadb-1 (Ceph RBD 10Gi)
mariadb-2 → PVC: data-mariadb-2 (Ceph RBD 10Gi)
```

### 복제 사용자
```
user: repl_user
password: repl_password
```

### SST (State Snapshot Transfer)
- 노드 간 데이터 전송 방식
- 새 노드 추가 시 전체 데이터 복사

---

## 6.8 클러스터 운영

### 상태 확인
```bash
kubectl get pods -n ceph-storage
kubectl get pvc -n ceph-storage
kubectl exec -it mariadb-0 -n ceph-storage -- mysql -u root -p
```

### MySQL 접속
```sql
SHOW STATUS LIKE 'wsrep_%';
-- wsrep_cluster_size: 3
-- wsrep_cluster_status: Primary
-- wsrep_connected: ON
```

### 삭제
```bash
bash clean.sh
```