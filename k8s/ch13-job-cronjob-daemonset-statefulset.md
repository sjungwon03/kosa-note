# Chapter 13: Job, CronJob, DaemonSet, StatefulSet

## 1. Job

### 1.1 정의
- 시작과 끝이 있는 **일회성 작업**
- 성공 완료 후 종료

### 1.2 vs Deployment
| Job | Deployment |
|-----|------------|
| 일회성 작업 | 계속 실행 서비스 |
| Exit Code 0 → 완료 | Running 유지 |
| 백업, ETL, 이메일 발송 | 웹 서버, API 서버 |

### 1.3 작동 방식
```
Job 생성 → Pod 생성 → 컨테이너 실행
성공 (Exit 0) → Job 완료
실패 (Exit != 0) → 재시도 (backoffLimit)
```

### 1.4 주요 설정
| 설정 | 설명 |
|------|------|
| **completions** | 성공 완료 필요 파드 수 (기본 1) |
| **parallelism** | 동시 실행 파드 수 (기본 1) |
| **backoffLimit** | 재시도 횟수 (기본 6) |
| **activeDeadlineSeconds** | 최대 실행 시간 |

### 1.5 restartPolicy
| Job | Deployment |
|-----|------------|
| OnFailure / Never | Always |

### 1.6 YAML 예제
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: hello-world-job
spec:
  template:
    spec:
      containers:
      - name: hello-world-container
        image: busybox:latest
        command: ["/bin/sh", "-c", "echo 'Hello, K8s Job!' && sleep 5"]
      restartPolicy: OnFailure
```

### 1.7 명령어
```bash
kubectl apply -f my-job.yaml
kubectl get jobs
kubectl get pods
kubectl logs <pod-name>
```

---

## 2. CronJob

### 2.1 정의
- 지정된 시간 스케줄에 따라 **Job을 주기적으로 생성**

### 2.2 사용 사례
| 사례 | 설명 |
|------|------|
| 정기 백업 | 매일 새벽 2시 DB 백업 |
| 주기 리포트 | 매주 월요일 9시 리포트 생성 |
| 데이터 동기화 | 1시간마다 외부 API 동기화 |
| 인증서 갱신 | 만료 전 확인 및 갱신 |

### 2.3 schedule (Cron 표현식)
```
분(0-59) 시(0-23) 일(1-31) 월(1-12) 요일(0-6)
```

| 예시 | 설명 |
|------|------|
| `*/1 * * * *` | 매 1분 |
| `0 17 * * 1-5` | 평일 17시 |
| `0 4 1 * *` | 매월 1일 4시 |

### 2.4 concurrencyPolicy
| Policy | 설명 |
|--------|------|
| **Allow** (기본) | 동시 실행 허용 |
| **Forbid** | 이전 Job 실행 중 → 새 Job 건너뜀 |
| **Replace** | 이전 Job 중단 → 새 Job 교체 |

### 2.5 History Limit
| 설정 | 설명 |
|------|------|
| **successfulJobsHistoryLimit** | 성공 Job 보관 수 (기본 3) |
| **failedJobsHistoryLimit** | 실패 Job 보관 수 (기본 1) |

### 2.6 YAML 예제
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: history-limit-example
spec:
  schedule: "*/1 * * * *"
  successfulJobsHistoryLimit: 2
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      backoffLimit: 0
      template:
        spec:
          containers:
          - name: success-or-fail-container
            image: busybox
            command:
            - /bin/sh
            - -c
            - |
              if [ $(($RANDOM % 2)) -eq 0 ]; then
                echo "Job Succeeded!"
                exit 0
              else
                echo "Job Failed!"
                exit 1
              fi
          restartPolicy: OnFailure
```

---

## 3. DaemonSet

### 3.1 정의
- **모든 노드**에 파드의 복사본이 하나씩 실행
- 노드 추가 → 파드 자동 배포
- 노드 삭제 → 파드 자동 정리

### 3.2 사용 사례
| 사례 | 설명 |
|------|------|
| 로그 수집 | Fluentd, Logstash, Promtail |
| 노드 모니터링 | Prometheus Node Exporter |
| 네트워크 플러그인 | CNI (Calico, Flannel) |
| 스토리지 데몬 | GlusterFS, Ceph |

### 3.3 YAML 예제
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: daemonset-example
spec:
  selector:
    matchLabels:
      name: my-daemonset-example
  template:
    metadata:
      labels:
        name: my-daemonset-example
    spec:
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: daemonset-example
        image: busybox
        args: ["tail", "-f", "/dev/null"]
        resources:
          limits:
            cpu: 100m
            memory: 200Mi
```

### 3.4 Master 노드 파드 실행
- Master 노드는 기본 Taint 설정 → 파드 스케줄링 안 됨
- `tolerations` 설정으로 Master 노드에도 실행 가능

---

## 4. StatefulSet

### 4.1 정의
- **상태 유지(Stateful)** 앱 관리
- DB, 메시지 큐, 분산 파일 시스템

### 4.2 vs Deployment
| Deployment | StatefulSet |
|------------|-------------|
| Stateless | Stateful |
| 랜덤 이름 (nginx-xxx) | 순서 있는 이름 (web-0, web-1) |
| 공유 볼륨 | 각 Pod 고유 볼륨 |
| 병렬 스케일링 | 순차 스케일링 |
| 웹 서버 | 데이터베이스 |

### 4.3 핵심 특징
| 특징 | 설명 |
|------|------|
| **고유 이름** | `<StatefulSet명>-<순서>` (web-0, web-1) |
| **고유 볼륨** | 각 Pod에 고유 PVC 연결 |
| **순차 생성** | 0 → 1 → 2 순서대로 생성 |
| **순차 삭제** | 역순 삭제 (2 → 1 → 0) |

### 4.4 Headless Service
- clusterIP: None → Pod별 DNS 제공
- `web-0.statefulset-svc.default.svc.cluster.local`

### 4.5 YAML 예제
```yaml
apiVersion: v1
kind: Service
metadata:
  name: statefulset-svc
spec:
  clusterIP: None
  selector:
    app: pod-nginx
  ports:
    - port: 80
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: statefulset
spec:
  serviceName: statefulset-svc
  replicas: 3
  selector:
    matchLabels:
      app: pod-nginx
  template:
    metadata:
      labels:
        app: pod-nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
```

### 4.6 volumeClaimTemplates
- 각 Pod에 고유 PVC 자동 생성
```yaml
volumeClaimTemplates:
- metadata:
    name: webserver-files
  spec:
    accessModes: ["ReadWriteOnce"]
    storageClassName: local-path
    resources:
      requests:
        storage: 100Mi
```

---

## 5. 요약

| 오브젝트 | 특징 | 사용 사례 |
|----------|------|-----------|
| **Job** | 일회성 작업 | 백업, ETL |
| **CronJob** | 주기적 Job | 정기 백업, 리포트 |
| **DaemonSet** | 모든 노드 1개 | 로그, 모니터링 |
| **StatefulSet** | 상태 유지 | DB, 메시지 큐 |