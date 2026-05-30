# Chapter 8: Ingress

## 1. Ingress란?

### 1.1 정의
- 클러스터 외부 HTTP/HTTPS 요청을 내부 서비스로 연결하는 **관문(Gateway)**
- Nginx, Traefik, HAProxy 등의 Ingress Controller 필요

### 1.2 NodePort/LoadBalancer의 한계
| 한계 | 설명 |
|------|------|
| NodePort | 예쁘지 않은 주소 (`http://<노드IP>:<포트>`) |
| LoadBalancer | 서비스마다 로드밸런서 할당 → 비용 부담 |

### 1.3 Ingress의 장점
| 장점 | 설명 |
|------|------|
| 단일 진입점 | 하나의 IP로 여러 서비스 노출 |
| URL 라우팅 | `/api` → A 서비스, `/web` → B 서비스 |
| 가상 호스팅 | `aaa.com` → A, `bbb.com` → B |
| TLS 처리 | HTTPS 인증서 중앙 관리 |

---

## 2. 핵심 구성 요소

### 2.1 Ingress Resource
- YAML로 라우팅 규칙 정의
- "어떤 주소로 들어온 요청을 어떤 서비스로 보낼지"

### 2.2 Ingress Controller
- 실제 작업을 수행하는 **두뇌**
- 클러스터 내에서 실행되는 파드
- Ingress Resource가 있어야만 동작

---

## 3. Ingress Controller 설치

### 3.1 NGINX Ingress Controller
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.1/deploy/static/provider/cloud/deploy.yaml
```

### 3.2 설치 확인
```bash
kubectl get pods --namespace=ingress-nginx -w
kubectl get service --namespace=ingress-nginx
```

### 3.3 온프레미스 설정
- LoadBalancer가 pending인 경우 → NodePort로 변경
```bash
kubectl edit svc ingress-nginx-controller -n ingress-nginx
# type: LoadBalancer → NodePort

kubectl -n ingress-nginx patch svc ingress-nginx-controller \
  -p '{"spec":{"externalTrafficPolicy":"Cluster"}}'
```

---

## 4. 경로 기반 라우팅 예제

### 4.1 백엔드 애플리케이션 배포
```yaml
# apple-app
apiVersion: apps/v1
kind: Deployment
metadata:
  name: apple-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: apple
  template:
    metadata:
      labels:
        app: apple
    spec:
      containers:
      - name: apple-container
        image: masungil/http-echo
        env:
        - name: ECHO_TEXT
          value: "이것은 사과 애플리케이션입니다!"
---
apiVersion: v1
kind: Service
metadata:
  name: apple-service
spec:
  selector:
    app: apple
  ports:
  - port: 8000
```

### 4.2 Ingress Resource 생성
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: "nginx"
  rules:
  - http:
      paths:
      - path: /apple
        pathType: Prefix
        backend:
          service:
            name: apple-service
            port:
              number: 8000
      - path: /banana
        pathType: Prefix
        backend:
          service:
            name: banana-service
            port:
              number: 8000
```

### 4.3 Rewrite Target
- `nginx.ingress.kubernetes.io/rewrite-target: /`
- `/apple` 요청을 `/`로 전달 (앱이 `/`만 응답)

### 4.4 테스트
```bash
kubectl get service --namespace=ingress-nginx
curl http://<INGRESS_IP>/apple
curl http://<INGRESS_IP>/banana
```

---

## 5. TLS (HTTPS) 적용

### 5.1 인증서 및 Secret 생성
```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
   -keyout tls.key -out tls.crt -subj "/CN=my-app.com"

kubectl create secret tls tls-secret --key tls.key --cert tls.crt
rm tls.key tls.crt
```

### 5.2 TLS Ingress YAML
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress-tls
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: "nginx"
  tls:
  - hosts:
    - my-app.com
    secretName: tls-secret
  rules:
  - host: my-app.com
    http:
      paths:
      - path: /apple
        pathType: Prefix
        backend:
          service:
            name: apple-service
            port:
              number: 8000
```

### 5.3 HTTPS 테스트
```bash
curl --resolve my-app.com:443:<INGRESS_IP> https://my-app.com/apple -k
```

---

## 6. Let's Encrypt (무료 인증서)

### 6.1 Certbot 설치
```bash
sudo snap install core; sudo snap refresh core
sudo apt-get remove certbot
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot
```

### 6.2 인증서 발급
```bash
sudo certbot --nginx -d masungil.shop -d www.masungil.shop
```

### 6.3 자동 갱신 테스트
```bash
sudo certbot renew --dry-run
```

---

## 7. 도메인 연결 (가비아 + AWS)

### 7.1 AWS EC2 Elastic IP 할당
- EC2 인스턴스에 고정 IP 할당

### 7.2 AWS Route 53 설정
1. 호스팅 영역 생성 (도메인 이름 입력)
2. A 레코드 생성 (Elastic IP 연결)
3. NS 레코드 값 복사

### 7.3 가비아 네임서버 변경
- 가비아 → 도메인 관리 → 네임서버 설정
- AWS Route 53의 NS 값 입력

---

## 8. ALB (Application Load Balancer) 연동

### 8.1 구성
- EC2 → 대상 그룹 → ALB → Route 53 → 가비아

### 8.2 ACM 인증서
- AWS Certificate Manager에서 인증서 요청
- Route 53에서 레코드 생성하여 인증
- ALB 리스너에 인증서 추가