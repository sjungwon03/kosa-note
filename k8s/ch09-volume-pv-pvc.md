# Chapter 9: Volume & PV/PVC & 동적 프로비저닝

## 1. Volume 필요성

### 1.1 컨테이너 파일 시스템의 한계
- 컨테이너 파일 시스템은 **일시적(Ephemeral)**
- 컨테이너 재시작/삭제 시 데이터 손실
- 컨테이너 간 파일 공유 불가

### 1.2 Volume의 역할
- 파드의 생명주기에 맞춰 데이터 보존
- 컨테이너 재시작해도 데이터 유지
- 파드 내 컨테이너 간 파일 공유

---

## 2. emptyDir

### 2.1 정의
- 파드 생성 시 **빈 디렉터리** 생성

### 2.2 특징
| 특징 | 설명 |
|------|------|
| 생성 시점 | 파드가 노드에 할당될 때 |
| 생명주기 | 파드가 존재하는 동안만 |
| 데이터 보존 | 컨테이너 재시작해도 보존 |
| 파드 삭제 시 | 데이터 삭제 |

### 2.3 사용 목적
- 임시 데이터 저장 (Scratch Space)
- 파드 내 컨테이너 간 파일 공유 (**사이드카 패턴**)

### 2.4 예제: 사이드카 패턴
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-example
spec:
  volumes:
  - name: shared-data
    emptyDir: {}
  containers:
  - name: main-app
    image: busybox
    command: ["/bin/sh", "-c", "while true; do echo $(date) > /data/time.log; sleep 5; done"]
    volumeMounts:
    - name: shared-data
      mountPath: /data
  - name: sidecar
    image: busybox
    command: ["/bin/sh", "-c", "while true; do cat /shared/time.log; sleep 5; done"]
    volumeMounts:
    - name: shared-data
      mountPath: /shared
```

---

## 3. hostPath

### 3.1 정의
- 노드의 파일 시스템을 직접 마운트

### 3.2 사용 목적
| 목적 | 설명 |
|------|------|
| 노드 모니터링 | `/var/log`, `/proc`, `/sys` 접근 |
| 노드 관리 | Docker 소켓 `/var/run/docker.sock` 접근 |

### 3.3 ⚠️ 주의사항
- **보안 위험**: 컨테이너 해킹 → 노드 장악 가능
- **이식성 저하**: 특정 노드에 종속
- **프로덕션 사용 금지**: 학습/테스트만

### 3.4 예제
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-example
spec:
  volumes:
  - name: host-logs
    hostPath:
      path: /var/log
      type: Directory
  containers:
  - name: log-reader-container
    image: busybox
    command: ["/bin/sh", "-c", "while true; do ls -la /mnt/host-logs; sleep 5; done"]
    volumeMounts:
    - name: host-logs
      mountPath: /mnt/host-logs
      readOnly: true
```

---

## 4. PersistentVolume (PV)

### 4.1 정의
- 클러스터에서 프로비저닝된 **스토리지 리소스**
- NFS, AWS EBS, GCE PD, Azure Disk 등 추상화

### 4.2 주요 속성
| 속성 | 설명 |
|------|------|
| capacity | 스토리지 크기 (예: 5Gi) |
| accessModes | 접근 모드 (RWO, ROX, RWX) |
| volumeMode | Filesystem (기본) 또는 Block |
| persistentVolumeReclaimPolicy | Retain, Delete, Recycle |
| storageClassName | StorageClass 연결 |

### 4.3 Access Modes
| 모드 | 설명 |
|------|------|
| **ReadWriteOnce (RWO)** | 단일 노드에서만 읽기/쓰기 (블록 스토리지) |
| **ReadOnlyMany (ROX)** | 여러 노드에서 읽기만 |
| **ReadWriteMany (RWX)** | 여러 노드에서 읽기/쓰기 (NFS) |

### 4.4 Reclaim Policy
| Policy | 설명 |
|--------|------|
| Retain | 수동 회수, 데이터 보존 |
| Delete | PV와 스토리지 삭제 |
| Recycle | 데이터 삭제 후 재사용 (deprecated) |

---

## 5. PersistentVolumeClaim (PVC)

### 5.1 정의
- 사용자가 PV에 대해 **요청**하는 리소스
- "10GB 읽기/쓰기 가능한 스토리지 필요"

### 5.2 주요 속성
| 속성 | 설명 |
|------|------|
| accessModes | PV와 호환되는 접근 모드 |
| resources.requests.storage | 필요한 스토리지 크기 |
| storageClassName | 특정 StorageClass 지정 |

---

## 6. PV/PVC 바인딩 기준

### 6.1 바인딩 과정
1. Pending 상태의 PVC 감지
2. Available 상태의 PV 감지
3. 조건 비교 → 바인딩

### 6.2 바인딩 기준
| 기준 | 설명 |
|------|------|
| **Capacity** | PV 용량 ≥ PVC 요청 용량 |
| **Access Modes** | PVC 요청 모드를 PV가 지원 |
| **Volume Mode** | Filesystem 또는 Block 일치 |
| **StorageClass** | 동적 프로비저닝의 핵심 |
| **Selector** | 특정 레이블 PV 선택 |

---

## 7. StorageClass & 동적 프로비저닝

### 7.1 StorageClass 정의
- PV를 동적으로 프로비저닝하는 **템플릿**
- 어떤 프로비저너, 어떤 파라미터 사용

### 7.2 동적 프로비저닝 과정
```
PVC 생성 → StorageClass 확인 → 프로비저너 실행 → PV 자동 생성 → 바인딩
```

### 7.3 예제: local-path-provisioner
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage-class
provisioner: rancher.io/local-path
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

### 7.4 volumeBindingMode
| Mode | 설명 |
|------|------|
| Immediate | PVC 생성 시 즉시 PV 생성 |
| WaitForFirstConsumer | 파드가 스케줄링될 때까지 지연 |

### 7.5 PVC 생성
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-dynamic-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: local-storage-class
```

### 7.6 설치
```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml
kubectl get pods -n local-path-storage
```

---

## 8. PV/PVC 사용 예제 (MariaDB)

### 8.1 Secret 생성
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mariadb-secret-hostpath
type: Opaque
data:
  MARIADB_ROOT_PASSWORD: MTAwNA==
```

### 8.2 PV 생성 (hostPath)
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mariadb-pv
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /mnt/data/mariadb
    type: DirectoryOrCreate
```

### 8.3 PVC 생성
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mariadb-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

### 8.4 Deployment 생성
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mariadb-deployment-hostpath
spec:
  selector:
    matchLabels:
      app: mariadb-hostpath
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mariadb-hostpath
    spec:
      nodeName: k8s-node1
      containers:
        - name: mariadb
          image: mariadb:10.5
          env:
            - name: MARIADB_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mariadb-secret-hostpath
                  key: MARIADB_ROOT_PASSWORD
          ports:
            - containerPort: 3306
          volumeMounts:
            - name: mariadb-persistent-storage
              mountPath: /var/lib/mysql
      volumes:
        - name: mariadb-persistent-storage
          persistentVolumeClaim:
            claimName: mariadb-pvc
```

---

## 9. Best Practices

| Practice | 설명 |
|----------|------|
| hostPath 사용 금지 | 프로덕션에서 위험 |
| StorageClass 사용 | 동적 프로비저닝으로 자동화 |
| StatefulSet + volumeClaimTemplates | DB 등 상태 유지 앱 |
| ReadWriteOnce | DB는 단일 파드만 쓰기 |
| RBAC | PV/PVC 접근 제어 |