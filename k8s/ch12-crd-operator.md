# Chapter 12: CRD & Custom Resource Controller

## 1. 커스텀 리소스 (Custom Resources)

### 1.1 정의
- K8s API를 확장하여 **사용자 정의 오브젝트 타입** 생성
- Pod, Deployment 같은 내장 리소스 외에 새로운 리소스 만들기

### 1.2 CustomResourceDefinition (CRD)
- 커스텀 리소스를 정의하는 오브젝트
- 새 리소스의 이름, 버전, 스키마 명세

### 1.3 역할
- 커스텀 리소스 자체는 **데이터 저장만**
- 동작은 없음 → 컨트롤러 필요

---

## 2. CRD 예제

### 2.1 CRD 정의
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: greetings.kosa.go.kr
spec:
  group: kosa.go.kr
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                message:
                  type: string
                recipient:
                  type: string
              required: ["message", "recipient"]
  scope: Namespaced
  names:
    plural: greetings
    singular: greeting
    kind: Greeting
    shortNames: ["greet"]
```

### 2.2 커스텀 리소스 인스턴스
```yaml
apiVersion: kosa.go.kr/v1
kind: Greeting
metadata:
  name: hello-world
spec:
  message: "Hello"
  recipient: "World"
```

### 2.3 명령어
```bash
kubectl apply -f greeting-crd.yaml
kubectl get crd
kubectl apply -f my-greeting.yaml
kubectl get greetings
kubectl get greet
```

---

## 3. 컨트롤러 (Controllers)

### 3.1 정의
- "원하는 상태"와 "실제 상태"를 일치시키는 **제어 루프**

### 3.2 작동 원리
```
1. Watch (감시)   → 리소스 변경 감시
2. Compare (비교) → 원하는 상태 vs 실제 상태
3. Reconcile (조정) → 실제 상태를 원하는 상태로 만들기
```

### 3.3 내장 컨트롤러
| 컨트롤러 | 역할 |
|----------|------|
| Deployment Controller | ReplicaSet/Pod 생성 관리 |
| Service Controller | 로드밸런서/네트워크 규칙 생성 |
| ReplicaSet Controller | Pod 수 유지 |

---

## 4. Operator 패턴

### 4.1 정의
- **커스텀 리소스 + 커스텀 컨트롤러** = Operator
- 운영 지식을 코드로 캡슐화하여 자동화

### 4.2 작동 방식
```
1. 사용자가 Greeting 리소스 생성
2. Greeting 컨트롤러가 감지
3. spec 필드 읽기 (message, recipient)
4. 실제 작업 수행 (로그 출력, 이메일 발송)
5. status 필드 업데이트
```

### 4.3 Operator 예시
- PostgreSQL Operator (CrunchyData)
- MySQL Operator (Oracle)
- Prometheus Operator
- Etcd Operator

---

## 5. RBAC 설정

### 5.1 컨트롤러용 RBAC
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: greeting-controller-sa
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: greeting-controller-role
rules:
- apiGroups: ["kosa.go.kr"]
  resources: ["greetings"]
  verbs: ["get", "list", "watch", "create", "update", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: greeting-controller-binding
subjects:
- kind: ServiceAccount
  name: greeting-controller-sa
roleRef:
  kind: ClusterRole
  name: greeting-controller-role
```

---

## 6. Operator 장점

| 장점 | 설명 |
|------|------|
| **확장성** | K8s를 모든 앱 관리 플랫폼으로 확장 |
| **운영 자동화** | 배포, 스케일링, 백업, 복구 자동화 |
| **선언적 관리** | "무엇" 원하는지 선언 → "어떻게"는 Operator |
| **일등 시민** | kubectl, RBAC 등 모든 기능 통합 |
| **플랫폼 구축** | DB-as-a-Service, CI/CD-as-a-Service |

---

## 7. 컨트롤러 배포

### 7.1 Deployment 예제
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: greeting-controller
spec:
  replicas: 1
  selector:
    matchLabels:
      app: greeting-controller
  template:
    metadata:
      labels:
        app: greeting-controller
    spec:
      serviceAccountName: greeting-controller-sa
      containers:
      - name: controller
        image: greeting-controller:latest
```

---

## 8. 요약

| 개념 | 역할 |
|------|------|
| **CRD** | 새 리소스 타입 정의 |
| **Custom Resource** | CRD 기반 데이터 저장 |
| **Controller** | 상태 감시 + 조정 |
| **Operator** | CR + Controller 결합 (운영 자동화) |