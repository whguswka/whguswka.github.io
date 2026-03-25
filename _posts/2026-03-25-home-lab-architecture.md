---
title: "홈랩 시스템 아키텍처 -- 4대 서버로 구성한 AI/ML 인프라"
excerpt: "K3s 클러스터와 DGX Spark 2노드로 구성한 홈랩 인프라의 전체 아키텍처를 소개합니다."
categories:
  - Infrastructure
tags:
  - Kubernetes
  - K3s
  - DGX Spark
  - vLLM
  - Kubeflow
  - Home Lab
toc: true
toc_sticky: true
---

## 들어가며

업무에서 사용하는 기술들을 직접 구축하고 운영해보기 위해 홈랩을 만들었다. Kubernetes 클러스터 운영, ML 파이프라인 자동화, LLM 서빙 등 실무에서 접하는 인프라를 축소된 규모로 재현하는 것이 목표였다.

현재 4대의 서버로 구성되어 있으며, 크게 두 개의 클러스터로 나뉜다.

- **K3s 클러스터** (2노드): 컨테이너화된 서비스 운영, ML 파이프라인 실행
- **DGX Spark 클러스터** (2노드): LLM 모델 서빙 전담

이 글에서는 전체 구성과 각 구성요소의 역할을 정리한다.

---

## 하드웨어 구성

### 서버 목록

| 구분 | IP | 역할 | CPU | RAM | GPU | 스토리지 |
|------|----|------|-----|-----|-----|----------|
| K3s Master + Worker | 192.168.0.156 | Control Plane, 서비스 호스팅 | AMD Threadripper PRO 7965WX (24C/48T) | 251 GB | RTX A6000 (48GB) + RTX 4090 (24GB) | NVMe 5.2TB |
| K3s Worker | 192.168.0.87 | GPU 워크로드 전담 | AMD Ryzen 9 3900X (12C/24T) | 125 GB | RTX 3090 (24GB) | NVMe 1.7TB + HDD 3.6TB |
| DGX Spark #1 (Head) | 192.168.0.228 | vLLM API 서버 | ARM Cortex-X925/A725 (20C) | 119 GB | NVIDIA GB10 (Blackwell) | NVMe 931GB |
| DGX Spark #2 (Worker) | 192.168.0.237 | vLLM 분산 추론 노드 | ARM Cortex-X925/A725 (20C) | 119 GB | NVIDIA GB10 (Blackwell) | NVMe 916GB |

DGX Spark는 NVIDIA에서 출시한 개인용 AI 워크스테이션이다. CPU와 GPU가 통합 메모리 아키텍처(UMA)를 사용하여, 두 대를 합치면 약 240GB의 메모리를 LLM 서빙에 활용할 수 있다.

### K3s 클러스터 노드 역할

156 서버가 Control Plane과 Worker를 겸하고, 87 서버가 GPU 전용 Worker로 참여한다. GPU Time-Slicing을 적용하여 87 서버의 RTX 3090 하나를 가상 2개로 분할, Ollama 임베딩 서비스와 ML 파이프라인 GPU 학습이 동시에 실행 가능하도록 했다.

---

## 소프트웨어 스택

각 구성요소를 역할별로 분류하면 다음과 같다.

### 컨테이너 오케스트레이션

- **K3s**: 경량 Kubernetes 배포판. 홈랩 규모에서 kubeadm보다 설치와 관리가 간편하다.
- **Longhorn**: Kubernetes 네이티브 분산 스토리지. PersistentVolume을 자동 복제한다.
- **Rancher**: 클러스터 관리 UI. 노드 상태, 워크로드, 이벤트를 웹에서 확인할 수 있다.

### CI/CD

- **GitLab**: Self-hosted Git 저장소. 모노레포(ai-agent-scm) 구조로 여러 서비스를 관리한다.
- **GitLab Runner + Kaniko**: Docker 데몬 없이 컨테이너 이미지를 빌드한다. `feature/*` 브랜치에서 MR을 생성하고 main에 merge하면 자동으로 빌드, Harbor에 push, Kubernetes 배포까지 진행된다.
- **Harbor**: 컨테이너 이미지 레지스트리. Trivy 기반 취약점 스캔이 내장되어 있다.

### ML 파이프라인

- **Kubeflow Pipelines v2**: ML 워크플로우를 DAG로 정의하고 스케줄 실행한다. 데이터 수집, 모델 학습, 추론, 결과 평가까지의 파이프라인을 관리한다.
- **TensorBoard**: 모델 학습 메트릭 시각화.

### 데이터베이스

- **PostgreSQL**: 주식 시세, 환율, 거시경제 지표 등 수집 데이터의 중앙 저장소.
- **Teradata Vantage Express**: 업무에서 사용하는 Teradata 환경을 로컬에 재현. QEMU 가상머신에서 실행되며, `teradatasql`(SQL 드라이버)과 `teradataml`(DataFrame 기반 in-database analytics) 두 가지 접근 방식을 테스트하는 데 사용된다.
- **CouchDB**: 문서 동기화용.
- **Elasticsearch**: 로그 수집 및 검색.

### LLM 서빙

- **vLLM**: DGX Spark 2노드에서 200B+ 규모 모델을 tensor parallel(tp=2)로 서빙한다. OpenAI 호환 API를 제공하여 OpenWebUI, OpenCode 등 클라이언트에서 동일한 인터페이스로 접근할 수 있다.
- **Ollama**: 경량 모델과 임베딩용. K3s 클러스터 내에서 Pod로 실행된다.
- **OpenWebUI**: LLM 채팅 웹 인터페이스.

### 모니터링

- **Prometheus**: 메트릭 수집. Node Exporter, DCGM Exporter 등으로 CPU/RAM/GPU/디스크 지표를 스크래핑한다.
- **Grafana**: 대시보드. Kubeflow Pipeline 실행 시 GPU 사용률, Pod 메모리 등을 실시간으로 확인할 수 있는 커스텀 대시보드를 구성했다.
- **DCGM Exporter**: NVIDIA GPU 메트릭(사용률, 온도, 전력, 메모리) 전용 수집기.

### 인증 및 접근 제어

- **Keycloak**: SSO(Single Sign-On) 서버. Grafana, Rancher 등 서비스에 통합 로그인을 제공하기 위해 구축했다.

---

## 네트워크 구성

```
                     Internet
                        |
                    [ 공유기 ]
                        |
          ──────────────┼──────────────────
          |             |                 |
   ┌──────────┐  ┌──────────┐     ┌──────────────┐
   │   .156   │  │   .87    │     │  DGX Spark   │
   │  Master  │──│  Worker  │     │  Cluster     │
   │  +Worker │  │  (GPU)   │     │              │
   │          │  │          │     │ .228 (Head)  │
   │ K3s      │  │ K3s      │     │ .237 (Worker)│
   │ Cluster  │  │ Cluster  │     │              │
   └──────────┘  └──────────┘     └──────────────┘
        |              |                 |
        └──────────────┘          QSFP link-local
         Flannel CNI              169.254.0.0/16
         (Overlay Network)        (GPU 직접 통신)
```

### 서비스 노출 방식

Ingress Controller 대신 **NodePort** 방식을 기본으로 사용한다. 30000~32767 범위에서 포트를 할당하며, 현재 약 28개의 NodePort 서비스가 운영 중이다.

외부 접근이 필요한 서비스는 공유기의 포트포워딩을 통해 노출한다. 방화벽은 공유기 포트포워딩(1차)과 UFW(2차)의 이중 구조를 사용한다. Kubernetes NodePort는 UFW INPUT 규칙을 우회하는 특성이 있어, 공유기 포트포워딩이 외부 접근의 실질적인 제어 수단이 된다.

### DGX Spark 클러스터 통신

DGX Spark 두 대는 QSFP 케이블로 직접 연결되어 있다. link-local 주소(169.254.0.0/16)를 사용하여 클러스터 내부 통신을 한다. vLLM은 Ray를 분산 실행 백엔드로 사용하며, tensor parallel 2로 모델을 양쪽 노드에 분할 배치한다.

---

## 주요 프로젝트 요약

이 인프라 위에서 운영 중인 프로젝트들이다. 각각은 별도의 글에서 상세히 다룰 예정이다.

### Stock Prediction ML Pipeline

Teradata의 데이터 로드 방식(`teradatasql` vs `teradataml`)과 컴퓨트 장치별(CPU vs GPU) ML 학습을 비교 테스트하기 위한 파이프라인이다. Kubeflow Pipelines v2로 자동화되어 있으며 데이터 수집, 동기화, 모델 학습, 추론, 결과 평가까지의 전체 워크플로우가 포함된다.

### LLM 서빙

DGX Spark 2노드에서 200B 이상 규모의 모델을 vLLM으로 서빙한다. Qwen3-235B AWQ에서 시작하여 Qwen3.5-122B FP8, 현재는 MiniMax-M2.5-AWQ(229B MoE)를 192K 컨텍스트로 운영 중이다. Tool calling과 reasoning parser가 활성화되어 있다.

### Agent Task Hub (ATH)

여러 AI 코딩 에이전트가 동일한 코드베이스에서 동시에 작업할 때 충돌을 방지하기 위한 시스템이다. 작업 등록, 지식 공유, 파일 락 등의 기능을 MCP(Model Context Protocol) 서버로 제공하여 에이전트 간 협업을 조율한다.

### 기타

- **Service Portal**: Kubernetes 서비스 현황을 한눈에 볼 수 있는 웹 대시보드.
- **Log Analyzer**: RAG 기반 로그 분석 및 오류 해결 시스템. Streamlit 대시보드로 제공.
- **Teradata Vector PoC**: Teradata에서 벡터 검색 기능을 테스트하는 개념 증명.
- **Android Docling Scanner**: 카메라로 문서를 스캔하고 OCR 처리하여 텍스트를 추출하는 앱.

---

## 마무리

이 글에서는 홈랩의 전체 구조를 요약했다. 이후 글에서는 각 프로젝트의 설계 의도, 구현 과정, 운영하면서 겪은 문제들을 구체적으로 다룰 예정이다.
