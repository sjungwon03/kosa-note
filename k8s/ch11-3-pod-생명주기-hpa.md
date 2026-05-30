# Chapter 11-3: Pod 생명주기 & Probe & HPA

## 1. Pod 생명주기

### 1.1 Pod Phase (단계)
| Phase | 설명 |
|-------|------|
| **Pending** | 승인되었지만 컨테이너 생성 중 |
| **Running** | 노드에 할당, 컨테이너 실행 중 |
| **Succeeded** | 모든 컨테이너 성공 종료 (Exit Code 0) |
| **Failed** | 컨테이너 실패 종료 |
| **Unknown** | 노드 통신 문제 |

### 1.2 Container State
| State | 설명 |
|-------|------|
| **Waiting** | 이미지 다운로드, Init Container 실행 중 |
| **Running** | 정상 실행 중 |
| **Terminated** | 종료 (성공/실패/OOM) |

---

## 2. 생명주기 제어 메커니즘

### 2.1 Init Containers
- 앱 컨테이너 시작 전 **사전 작업** 담당
- 순차 실행 (0 → 1 → 2)
- 모두 성공 후 앱 컨테이너 시작

**사용 예:**
- 설정 파일 생성
- DB 스키마 마이그레이션
- 외부 서비스 대기

### 2.2 Lifecycle Hooks
| Hook | 실행 시점 | 용도 |
|------|-----------|------|
| **postStart** | 컨테이너 생성 직후 | 초기화 작업 |
| **preStop** | 컨테이너 종료 직전 | Graceful Shutdown |

**예제:**
```yaml
lifecycle:
  postStart:
    exec:
      command: ["/bin/sh", "-c", "echo 'Post Start'"]
  preStop:
    exec:
      command: ["/bin/sh", "-c", "nginx -s quit"]
```

---

## 3. Probes (프로브)

### 3.1 종류
| Probe | 역할 | 실패 시 동작 |
|-------|------|--------------|
| **livenessProbe** | "살아있는가?" | 컨테이너 재시작 |
| **readinessProbe** | "요청 처리 준비?" | 서비스에서 제외 |
| **startupProbe** | "시작되었는가?" | 다른 프로브 비활성화 |

### 3.2 livenessProbe
- Deadlock, 무한 루프 등 감지
- 실패 → 컨테이너 재시작

**예제:**
```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 80
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 3
```

### 3.3 readinessProbe
- 앱 시작은 되었지만 준비 중
- 실패 → 서비스 엔드포인트 제외 (트래픽 안 받음)

**예제:**
```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 80
  initialDelaySeconds: 5
  periodSeconds: 5
```

### 3.4 startupProbe
- 시작이 오래 걸리는 앱용
- 성공 전까지 liveness/readiness 비활성화

**예제:**
```yaml
startupProbe:
  httpGet:
    path: /startup
    port: 80
  failureThreshold: 30
  periodSeconds: 10
```

### 3.5 Probe 설정
| 필드 | 설명 |
|------|------|
| initialDelaySeconds | 첫 프로브 실행 전 대기 시간 |
| periodSeconds | 프로브 실행 주기 |
| failureThreshold | 실패 허용 횟수 |
| successThreshold | 성공 필요 횟수 |
| timeoutSeconds | 프로브 타임아웃 |

---

## 4. HPA (Horizontal Pod Autoscaler)

### 4.1 정의
- 메트릭(CPU, 메모리) 기반으로 **파드 수 자동 조절**
- 스케일 아웃/인 자동화

### 4.2 작동 방식
```
1. 메트릭 수집 (Metrics Server)
2. 현재값 vs 목표값 비교
3. 필요 복제본 수 계산
4. Deployment replicas 업데이트
```

**계산식:**
```
desiredReplicas = ceil[currentReplicas * (currentMetricValue / desiredMetricValue)]
```

### 4.3 메트릭 종류
| 종류 | 설명 |
|------|------|
| **리소스 메트릭** | CPU/메모리 사용률 |
| **커스텀 메트릭** | RPS, Queue 길이 |
| **외부 메트릭** | SQS, Kafka lag |

### 4.4 YAML 예제
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app-deployment
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

### 4.5 필수 조건
- 파드에 `resources.requests` 설정
- Metrics Server 설치

### 4.6 Metrics Server 설치
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# HTTPS 문제 해결
kubectl edit deployment metrics-server -n kube-system
# args에 --kubelet-insecure-tls 추가
```

### 4.7 HPA Behavior (고급 설정)
```yaml
behavior:
  scaleDown:
    stabilizationWindowSeconds: 300  # 5분 대기
    policies:
    - type: Percent
      value: 100
      periodSeconds: 60
  scaleUp:
    stabilizationWindowSeconds: 0    # 즉시 스케일 업
    policies:
    - type: Percent
      value: 100
      periodSeconds: 60
```

---

## 5. terminationGracePeriodSeconds

### 5.1 정의
- 파드 종료 시 Graceful Shutdown 대기 시간
- 기본값: 30초

### 5.2 종료 과정
```
1. preStop 훅 실행
2. SIGTERM 전송
3. terminationGracePeriodSeconds 대기
4. SIGKILL 강제 종료 (대기 후)
```

---

## 6. Best Practices

| Practice | 설명 |
|----------|------|
| readinessProbe 설정 | 트래픽 받기 전 준비 확인 |
| livenessProbe 설정 | Deadlock 감지 |
| startupProbe 설정 | 시작 오래 걸리는 앱 |
| preStop 훅 | Graceful Shutdown |
| HPA requests 설정 | CPU/메모리 기반 스케일링 필수 |
| stabilizationWindowSeconds | 스케일 다운 안정화 |