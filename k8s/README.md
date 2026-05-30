# Kubernetes(k8s) 학습 노트

[docker-kubernetes 강의자료](https://github.com/masungil70/docker-kubernetes) 문서를 기반으로 정리한 학습 노트입니다.

---

## 챕터별 학습 노트

| 챕터       | 파일                                                                              | 주요 내용                                                    |
| ---------- | --------------------------------------------------------------------------------- | ------------------------------------------------------------ |
| **Ch05**   | [쿠버네티스 기본](ch05-쿠버네티스-기본.md)                                        | K8s 정의, 핵심 기능, Docker Swarm vs K8s, 기본 용어          |
| **Ch06**   | [핵심 오브젝트](ch06-핵심-오브젝트.md)                                            | Pod, ReplicaSet, Deployment, Service                         |
| **Ch07**   | [Namespace, ConfigMap, Secret](ch07-namespace-configmap-secret.md)                | Namespace, ConfigMap, Secret                                 |
| **Ch08**   | [Ingress](ch08-ingress.md)                                                        | Ingress, Ingress Controller, TLS, Let's Encrypt              |
| **Ch09**   | [Volume & PV/PVC](ch09-volume-pv-pvc.md)                                          | Volume, emptyDir, hostPath, PV, PVC, StorageClass            |
| **Ch10**   | [보안 (RBAC)](ch10-보안-rbac.md)                                                  | 인증/인가, RBAC, ServiceAccount                              |
| **Ch11-1** | [리소스 관리](ch11-1-리소스-관리.md)                                              | 리소스 단위, requests/limits, QoS, LimitRange, ResourceQuota |
| **Ch11-2** | [스케줄링](ch11-2-스케줄링.md)                                                    | 파드 생성 과정, Node Label, Affinity, Taints/Tolerations     |
| **Ch11-3** | [Pod 생명주기 & HPA](ch11-3-pod-생명주기-hpa.md)                                  | Pod Phase, Probe, HPA                                        |
| **Ch12**   | [CRD & Operator](ch12-crd-operator.md)                                            | CRD, Custom Resource, Controller, Operator                   |
| **Ch13**   | [Job, CronJob, DaemonSet, StatefulSet](ch13-job-cronjob-daemonset-statefulset.md) | Job, CronJob, DaemonSet, StatefulSet                         |
| **Ch14**   | [모니터링](ch14-모니터링.md)                                                      | Prometheus, Grafana, Helm, ArgoCD                            |

---
