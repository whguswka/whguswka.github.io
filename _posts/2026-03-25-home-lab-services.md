---
title: "홈랩 시스템 아키텍처 (2) -- 서비스 구성과 개발 배경"
excerpt: "홈랩 인프라 위에서 운영 중인 서비스들을 사용자용, 관리용, 데이터/ML 파이프라인으로 나누어 소개합니다."
categories:
  - Infrastructure
tags:
  - Kubernetes
  - Service Portal
  - Analysis Portal
  - Multi Agent
  - Kubeflow
  - Home Lab
toc: true
toc_sticky: true
---

## 개요

[이전 글](/infrastructure/home-lab-architecture/)에서 홈랩의 하드웨어와 소프트웨어 스택을 다뤘다. 이 글에서는 그 인프라 위에서 운영 중인 서비스들을 소개한다.

서비스는 크게 세 가지로 나뉜다.

1. **사용자용 서비스**: 분석 환경, AI 도구, 검색 엔진 등 일반 사용자가 직접 접속하여 사용하는 서비스
2. **관리용 서비스**: 서비스 상태 모니터링, 에이전트 작업 관리 등 운영/관리 목적의 서비스
3. **데이터 수집 및 ML 파이프라인**: 백그라운드에서 자동 실행되는 데이터 수집기와 ML 워크플로우

---

## 사용자용 서비스

### Analysis Portal (분석 포털)

Kubeflow 기반 JupyterLab 환경을 통합 관리하는 포털이다. 사용자가 웹에서 CPU/Memory/GPU 스펙과 노트북 이미지 종류(Base, DataScience, Full)를 선택하여 분석 환경을 요청하면, 관리자가 승인하고, 승인 후 Kubeflow Notebook CRD가 자동으로 생성되는 구조다.

이 포털을 별도로 개발한 이유는 Kubeflow 기본 UI만으로는 사용자 관리, 자원 통제, DB 연동 등이 부족했기 때문이다. 주요 기능은 다음과 같다:

- **환경 라이프사이클 관리**: 요청 - 승인 - 생성 - 실행/중지/재시작 - 삭제
- **PostgreSQL 연동**: 사용자 생성 시 PostgreSQL 계정 자동 동기화, 스키마별 권한 관리 UI, 권한 매트릭스 뷰(전체 사용자 DB/스키마 권한을 한눈에 확인)
- **VectorDB 관리**: Elasticsearch, CouchDB 사용자 계정 자동 동기화 및 권한 관리
- **DB 메타데이터 관리**: 테이블/칼럼 설명(COMMENT) 인라인 편집, 미등록 테이블 하이라이트
- **공유 리소스 폴더**: NFS 기반. Admin은 읽기/쓰기, 일반 사용자는 읽기 전용
- **사용자 가이드**: 빠른 시작, GitLab 사용법, AI API 연동, PostgreSQL 사용법, FAQ를 탭형으로 제공

기술 스택은 FastAPI + Jinja2 Templates + HTMX이다. React 같은 SPA 프레임워크 대신 HTMX(Server-Driven UI)를 선택한 이유는, 관리자 도구 특성상 복잡한 클라이언트 상태 관리보다는 서버 렌더링이 적합하다고 판단했기 때문이다.

### JupyterLab 노트북 환경

분석 포털에서 생성된 JupyterLab에는 다음이 기본으로 포함된다:

- **jupyter-ai (Jupyternaut)**: 노트북 내에서 AI 코딩 어시스턴트를 사용할 수 있는 확장. 기본 모델은 `qwen3-coder-next-ko`(한국어 응답 지원)이며, 내부 Ollama 서버에 연결된다.
- **임베딩 서비스 연동**: 87 서버에서 운영되는 `ollama-embed-87`(nomic-embed-text, 4096 dims)에 자동 연결
- **PostgreSQL 접근**: 분석 포털에서 부여받은 권한에 따라 `analytics_shared`, `stock_analytics` 등의 DB에 접근 가능
- **GitLab 연동**: LDAP 공통 계정으로 내부 GitLab에 SSH/HTTPS push/pull 가능

### Langflow


AI 워크플로우를 시각적으로 구성할 수 있는 로우코드 플랫폼이다. 분석 포털 사이드바에서 바로 접근할 수 있으며, `AUTO_LOGIN=true`로 별도 인증 없이 동작한다(내부 네트워크 전용).

배포한 이유는 CoLLM 활용 경험이 없는 사용자들도 GUI로 LLM 체인을 구성하고 RAG 파이프라인을 테스트할 수 있게 하기 위해서다. 내부에서 Ollama(임베딩/소형 모델), vLLM(대형 LLM), Elasticsearch, CouchDB와 연동할 수 있도록 환경변수를 사전 설정해두었다.

RAG Vector Store로는 Elasticsearch(대규모), Chroma DB(프로토타이핑), CouchDB Vector Store(커스텀 컴포넌트, 코사인 유사도 기반) 세 가지 옵션을 제공한다.

### OpenWebUI


LLM 채팅 웹 인터페이스. ChatGPT와 유사한 UI로 내부 LLM에 접근할 수 있다. vLLM(DGX Spark)과 Ollama 모두의 모델에 접근 가능하며, OpenAI 호환 API를 통해 동일한 인터페이스로 사용한다. 외부에서도 포트포워딩을 통해 접근 가능하도록 열어두었다.

### SearXNG


프라이버시 중심의 메타 검색 엔진이다. 여러 검색 엔진의 결과를 집계하여 보여주며, 사용자 추적이 없다. AI 에이전트들이 웹 검색 기능이 필요할 때 외부 서비스(Google API 등)에 의존하지 않고 내부에서 검색할 수 있도록 배포했다.

### ComfyUI


Stable Diffusion 기반 이미지 생성 노드 에디터. Worker 서버의 GPU에서 실행된다.

---

## 관리용 서비스

### Service Portal (서비스 포털)


모든 내부 서비스를 한곳에서 관리하고 접근할 수 있는 통합 대시보드다. 현재 28개 이상의 서비스가 등록되어 있다.

이 서비스를 만든 배경은, 서비스 수가 늘어나면서 어떤 서비스가 어느 포트에서 돌아가는지, 정상 동작하고 있는지, 어떤 사용자가 접근 권한이 있는지를 일일이 확인하는 것이 어려워졌기 때문이다. 주요 기능은 다음과 같다:

- **실시간 헬스체크**: 30초 간격으로 모든 서비스의 상태와 응답 시간을 측정
- **서비스 제어**: 관리자가 웹에서 서비스를 실행/중지/재시작/업데이트 (kubectl scale, rollout 기반)
- **K8s 자동 디스커버리**: 클러스터에서 미등록 워크로드를 자동 탐지하고 원클릭으로 포털에 등록
- **전체 토폴로지 시각화**: 28개 서비스를 카테고리별(Infrastructure, Database, DevOps, Monitoring, AI/ML, Tools) 레이어로 배치한 관계도
- **클러스터 오버뷰**: 노드별 CPU/Memory/GPU 사용량 게이지, 네임스페이스별 Pod 분포
- **감사 로그**: 서비스 제어, 권한 변경, 레지스트리 수정 등 모든 관리 작업 기록
- **알림 에스컬레이션**: L1(알림) - L2(자동 재시작) - L3(긴급 알림) 3단계 자동 대응. 인프라 핵심 서비스 9개는 보호 목록으로 자동 재시작 대상에서 제외

기술 스택은 FastAPI(Backend) + React 18 + TypeScript + Tailwind CSS(Frontend)이다. 분석 포털과 달리 React SPA를 선택한 이유는, 헬스체크 결과의 실시간 갱신, 토폴로지 시각화, 복잡한 필터링/정렬 등 인터랙티브한 요소가 많아 클라이언트 사이드 렌더링이 적합하다고 판단했기 때문이다.

LDAP/Keycloak SSO 인증을 사용하며, 관리자/일반 사용자 권한에 따라 노출되는 기능이 다르다. MCP(Model Context Protocol) 서버로도 운영되어 AI 에이전트가 서비스 레지스트리를 프로그래밍 방식으로 조회/등록할 수 있다.

### Agent Task Hub (ATH)


여러 AI 코딩 에이전트(Antigravity, Claude Code, OpenCode, OpenClaw)가 동일한 코드베이스에서 동시에 작업할 때 충돌을 방지하기 위해 직접 설계하고 구현한 시스템이다.

개발 동기는 다음과 같다. 실제로 여러 에이전트에 동시에 작업을 시키다 보면 동일한 파일을 동시에 수정하거나, 이미 완료된 작업을 중복으로 수행하거나, 한 에이전트가 발견한 문제를 다른 에이전트가 모르는 상황이 빈번하게 발생했다. 이를 해결하기 위해 작업 등록/추적, 지식 공유, 파일 잠금의 세 가지 핵심 기능을 갖춘 중앙 허브를 만들었다.

주요 기능:

- **작업 관리**: 에이전트가 작업 시작 전에 Task를 등록하고, 진행 상태와 결과를 기록. 부모-자식 Task 연결로 관련 작업 추적
- **지식 베이스**: 에이전트가 작업 중 발견한 사실, 결정 사항, 에러 해결법, 패턴 등을 등록. 다른 에이전트가 검색하여 활용
- **파일 잠금**: 에이전트가 특정 파일 수정 전에 잠금을 획득하여 동시 수정 충돌 방지. 5분 주기로 만료된 잠금 자동 정리
- **포커스 관리**: 현재 각 에이전트가 어떤 작업 영역에 집중하고 있는지 공유
- **대시보드**: 에이전트 상태, 작업 현황, 지식 베이스, 활동 로그를 한눈에 볼 수 있는 웹 UI

FastAPI + SQLite(WAL 모드)로 구현했다. 별도의 DB 서버를 두지 않은 이유는 단일 프로세스만 접근하므로 동시성 문제가 없고, 인프라를 경량하게 유지하기 위해서다.

각 에이전트는 자신의 설정 파일(GEMINI.md, CLAUDE.md, AGENTS.md 등)에 ATH 규칙이 삽입되어 있어, 작업 시작 시 자동으로 Task를 등록하고 완료 시 상태를 업데이트하도록 강제된다. CLI(`ath` 명령)와 MCP 서버 두 가지 방식으로 접근할 수 있다.

### 로그 분석기


Streamlit 기반 웹 대시보드로, 시스템 로그를 수집하고 RAG 기반으로 오류를 분석/해결하는 시스템이다.

- 5분 간격으로 시스템 메트릭(CPU, 메모리, zombie 프로세스, 헤비 유저)을 자동 수집
- sentence-transformers 기반 벡터 임베딩으로 유사 로그/해결 문서 검색
- PII(개인식별정보) 자동 마스킹
- 오류 해결 문서를 태그 기반으로 등록/검색/매칭

---

## 데이터 수집기 및 ML 파이프라인

### 주식 데이터 수집기

Kubeflow Pipeline Recurring Run으로 자동 실행되는 데이터 수집기들이다:

| 수집기 | 대상 | 스케줄 |
|--------|------|--------|
| KR Stock Collector | 한국 주식 시세 (yfinance) | 매일 |
| US Stock Collector | 미국 주식 시세 (yfinance) | 매일 |
| Exchange Rate Collector | 한국수출입은행 환율 API | 매일 |
| Index Collector | 글로벌 지수 (VIX 등) | 매일 |
| Market Cap Collector | 미국 시가총액 | 매일 |
| Stock Data Sync | PostgreSQL - Teradata 동기화 | 매일 |

수집된 데이터는 PostgreSQL에 적재되고, Stock Data Sync 파이프라인이 이를 Teradata로 동기화한다.

### Stock Prediction ML Pipeline

4가지 조합의 ML 파이프라인이 Kubeflow에서 실행된다:

| 파이프라인 | 데이터 접근 | 컴퓨트 |
|-----------|------------|--------|
| teradatasql-cpu | teradatasql (SQL 드라이버) | CPU |
| teradatasql-gpu | teradatasql (SQL 드라이버) | GPU |
| teradataml-cpu | teradataml (in-DB analytics) | CPU |
| teradataml-gpu | teradataml (in-DB analytics) | GPU |

이 파이프라인의 주된 목적은 주가 예측 모델 자체보다는, Teradata의 두 가지 데이터 로드 방식(`teradatasql` vs `teradataml`)의 차이와 CPU/GPU 간 학습 성능을 비교 테스트하는 것이다. 주가 예측은 이를 위한 실제 데이터 기반 워크로드로 사용되고 있다. 이 부분은 별도의 글에서 상세히 다룰 예정이다.

### Docling Extractor API


문서(PDF, 이미지 등)에서 텍스트를 추출하는 REST API. Android OCR 앱과 연동하여 카메라로 스캔한 문서의 텍스트를 Markdown 형태로 반환한다.

### Teradata Vector PoC


Teradata에서 벡터 검색 기능을 테스트하는 개념 증명. Streamlit 대시보드로 문서 등록, 벡터 저장소 관리, 임베딩 구조 확인, 벡터 검색, 대시보드 시각화 다섯 가지 기능을 제공한다.

---

## 서비스 간 연결 관계

```mermaid
graph LR
    subgraph UserServices["사용자용 서비스"]
        AP[Analysis Portal]
        JL[JupyterLab<br/>Notebooks]
        LF[Langflow]
        OW[OpenWebUI]
        SX[SearXNG]
    end

    subgraph AdminServices["관리용 서비스"]
        SP[Service Portal]
        ATH[Agent Task Hub]
    end

    subgraph DataML["데이터/ML"]
        KFP[Kubeflow Pipelines]
        COL[Data Collectors]
        STK[Stock Prediction<br/>Dashboard]
    end

    subgraph Infra["인프라"]
        PG[(PostgreSQL)]
        TD[(Teradata)]
        ES[(Elasticsearch)]
        OL[Ollama]
        VLLM[vLLM<br/>DGX Spark]
        LDAP[OpenLDAP]
        KC[Keycloak]
    end

    AP --> JL
    AP --> PG
    AP --> ES
    AP --> LDAP
    LF --> OL
    LF --> VLLM
    OW --> VLLM
    OW --> OL
    SP --> LDAP
    SP --> KC
    SP --> PG
    ATH -.-> SP
    KFP --> PG
    KFP --> TD
    COL --> PG
    STK --> TD

    style AP fill:#1a5276,stroke:#2e86c1,color:#fff
    style SP fill:#1a5276,stroke:#2e86c1,color:#fff
    style ATH fill:#4a235a,stroke:#7d3c98,color:#fff
    style VLLM fill:#5c1a5c,stroke:#9b2d9b,color:#fff
```

---

## 마무리

현재 진행 중이거나 계획 중인 작업도 있다:

- **Keycloak SSO 전면 통합**: Grafana, Rancher, Analysis Portal, Service Portal 등 모든 서비스에 SSO 로그인 적용
- **GitLab CI/CD 모노레포 확대**: 현재 Service Portal만 연결된 CI/CD 파이프라인을 다른 서비스로 확장
- **Agent Chat Hub**: AI 에이전트와의 대화를 웹 인터페이스로 통합하는 서비스 개발 중

이후 글에서는 각 프로젝트별로 설계 의도, 구현 과정, 트러블슈팅 경험을 더 깊이 다룰 예정이다.
