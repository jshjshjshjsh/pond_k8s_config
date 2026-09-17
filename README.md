# 🌊 Pond 2.0 Infrastructure (Pond-Cluster)

본 레포지토리는 **Pond 2.0 프로젝트**의 쿠버네티스(Kubernetes) 기반 인프라스트럭처 및 애플리케이션 배포를 관리하는 GitOps 저장소입니다.

ArgoCD를 활용한 App-of-Apps 패턴을 기반으로 인프라, 서비스, 워크로드를 체계적으로 배포하며, 고가용성 데이터베이스, 자동화된 시크릿 관리, 그리고 완벽한 관측성(Observability) 스택을 포함하고 있습니다.

## 🏛 Architecture Overview


<img width="8889" height="4506" alt="image (1)" src="https://github.com/user-attachments/assets/03024529-f28d-47b2-804b-d2df4fd8570a" />
*(아키텍처 구성도: 외부 트래픽 제어부터 GitOps 파이프라인, 모니터링, 데이터베이스까지 클러스터의 전체 흐름을 보여줍니다.)* 

## 🚀 Key Features & Tech Stack

### 1. GitOps & CI/CD

* **ArgoCD (App-of-Apps Pattern):** `root-app`을 통해 `infrastructure`, `services`, `workloads` 3개의 레이어로 나누어 순차적(Sync-wave)으로 클러스터 리소스를 자동 배포합니다.
* **Argo Rollouts (Canary Deployment):** 백엔드(`pond-back`) 배포 시 트래픽을 20%로 인입 후 대기(Pause), 이후 수동 승인 시 50%를 거쳐 100%로 안전하게 무중단 배포를 수행합니다.

### 2. High Availability Database

* **Percona XtraDB Cluster (PXC):** MySQL 8.0 기반의 2-Node HA(고가용성) 클러스터로 구성되어 있으며, HAProxy를 통해 애플리케이션에 단일 엔드포인트를 제공합니다.
* **Automated Off-site Backup:** 매일 새벽 3시(KST)에 데이터베이스 백업을 수행하며, Cloudflare R2(S3 호환 스토리지)로 안전하게 자동 전송 및 보관됩니다.
* **Redis:** 캐싱 및 분산 락 처리를 위한 인메모리 데이터 저장소로 연동되어 있습니다.

### 3. Secret Management & Security

* **HashiCorp Vault & External Secrets Operator (ESO):** Vault를 통해 중요 자격 증명을 중앙 관리하고, ESO를 활용하여 클러스터 내의 K8s Secret으로 자동 동기화합니다.
* **Sealed Secrets:** Git 저장소에 안전하게 시크릿(DB 비밀번호, Cloudflare 토큰 등)을 커밋하기 위해 Bitnami Sealed Secrets로 암호화하여 관리합니다.
* **Kyverno (Cluster Policy):** `default` 네임스페이스 사용을 원천 차단하는 등 클러스터 내부 거버넌스 및 보안 정책을 강제합니다.

### 4. Observability (모니터링 및 관측성)

* **Kube-Prometheus-Stack:** 노드, 파드, 애플리케이션의 메트릭을 수집하고 Alertmanager를 통해 알람을 구성합니다.
* **OpenTelemetry (OTel) Collector:** `pond-back` 애플리케이션의 Trace 및 Metric 데이터를 수집하여 각 백엔드(Jaeger, Prometheus)로 라우팅합니다.
* **Jaeger & Loki:** MSA 환경의 분산 트레이싱(Jaeger)과 애플리케이션 중앙 집중식 로깅(Loki+Promtail)을 지원합니다.

### 5. Networking & Traffic Control

* **Cloudflare Tunnel (`cloudflared`):** 외부로 포트를 개방하지 않고 Zero Trust 기반의 안전한 터널링을 통해 인그레스 트래픽을 처리합니다.
* **NGINX Ingress Controller:** 클러스터 내부의 L7 라우팅을 담당하며 벡엔드(`pond.theponds.work/api`)와 프론트엔드(`pond.theponds.work/`)로 트래픽을 분기합니다.

---

## 📂 Repository Structure

```text
├── argocd/
│   ├── root-app.yaml          # GitOps의 진입점 (Root Application)
│   ├── layers/                # Sync-wave 기반 3단계 배포 계층 (infra -> services -> workloads)
│   └── apps/                  # 각 컴포넌트별 ArgoCD Application 정의 (vault, prometheus, mysql 등)
├── pond-back/                 # Backend (Spring Boot) K8s Helm Chart (Argo Rollouts, HPA 등 포함)
├── pond-front/                # Frontend K8s Helm Chart
├── database/                  # PXC 기반 고가용성 MySQL 클러스터 및 백업 설정 매니페스트
├── infrastructure/            # NGINX Ingress, Cloudflare Tunnel 등 기반 네트워크 리소스
├── secrets/                   # Sealed Secrets 및 External Secrets 연동 정의 파일
├── policies/                  # Kyverno 클러스터 보안 정책 정의 (e.g., block-default-ns.yaml)
└── ingress/                   # pond-front 및 pond-back을 외부로 노출하는 Ingress 라우팅 룰

```

## 🛠 Getting Started

### Prerequisites

* Kubernetes 클러스터 (Kind, EKS 등)
* `kubectl` 및 `argocd` CLI 도구 설치
* Cloudflare Tunnel 토큰, Vault Unseal Key 등 초기 마스터 시크릿

### Bootstrap Installation

클러스터에 처음 인프라를 배포할 때는 ArgoCD를 설치한 후, 단 하나의 Root Application만 배포하면 모든 인프라가 선언된 순서에 맞게 자동으로 구축됩니다.

```bash
# 1. Root Application 배포
kubectl apply -f argocd/root-app.yaml

# 2. ArgoCD 대시보드에서 동기화 상태 확인
kubectl port-forward svc/argocd-server -n argocd 8080:443

```
