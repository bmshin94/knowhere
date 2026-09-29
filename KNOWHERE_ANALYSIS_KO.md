# 📚 Knowhere 전수조사 & 분석 정리 노트

> 작성일: 2026-09-29
> 작성: 카리나 (Claude Code) ✨
> 분석 대상 레포지토리: **https://github.com/bmshin94/knowhere**
> 원본(업스트림) 레포지토리: **https://github.com/Ontos-AI/knowhere**

---

## 📑 목차

1. [프로젝트 정체 및 개요](#1-프로젝트-정체-및-개요)
2. [폴더 구조 전수조사](#2-폴더-구조-전수조사)
3. [핵심 기술: 듀얼 트랙 파싱](#3-핵심-기술-듀얼-트랙-파싱)
4. [전체 파이프라인 흐름](#4-전체-파이프라인-흐름)
5. [Agentic Retrieval & MCP](#5-agentic-retrieval--mcp)
6. [기술 스택 & 인프라](#6-기술-스택--인프라)
7. [쉽게 이해하기 (비유편)](#7-쉽게-이해하기-비유편)
8. [설치 및 사용법](#8-설치-및-사용법)
9. [플러그인? 스킬? MCP?](#9-플러그인-스킬-mcp)
10. [API 토큰 정리](#10-api-토큰-정리)
11. [GitHub에서 유명한 이유](#11-github에서-유명한-이유)
12. [로컬 에이전트 구축 활용도](#12-로컬-에이전트-구축-활용도)
13. [React / PHP 로 만들 수 있나?](#13-react--php-로-만들-수-있나)
14. [수익화 아이디어 10선](#14-수익화-아이디어-10선)
15. [실행 로드맵 & 리스크](#15-실행-로드맵--리스크)
16. [참고 링크 모음](#16-참고-링크-모음)

---

## 1. 프로젝트 정체 및 개요

한 줄 요약:

> **지저분한 비정형 문서(PDF, PPTX, DOCX, XLSX 등)를 AI 에이전트가 실제로 활용할 수 있는
> "탐색 가능한 영속 기억(Navigable Persistent Memory)"으로 변환하는 문서 파싱 + 검색 인프라**

공식 슬로건: `Prepare unstructured data for AI Agents`

| 항목 | 내용 |
|---|---|
| 원본 레포 | https://github.com/Ontos-AI/knowhere |
| 분석한 포크 | https://github.com/bmshin94/knowhere |
| 라이선스 | Apache 2.0 (상업적 이용 가능) |
| 언어 / 런타임 | Python 3.11+ , `uv` 기반 모노레포 워크스페이스 |
| 코드 규모 | Python 파일 802개 / 약 155,568 줄 |
| 문서 | Markdown 39개 (ADR 10편 포함) |
| 공식 사이트 | https://knowhereto.ai · https://docs.knowhereto.ai |
| 오픈소스 공개 | 2026년 5월 7일 (풀스택 공개) |

### 주요 버전 히스토리 (README 기준)

- **2026-09-08** — Retrieval 2.0: MapNav 고정 워크플로 → 에이전트 네이티브 코퍼스 기반으로 전환
- **2026-09** — Document Parsing 2.0: Vision + Text 듀얼 트랙 도입
- **2026-06-01** — 초장문 PDF(300~500p+) 및 도면집(atlas) 전용 파서 지원
- **2026-05-07** — 전체 스택 오픈소스화

---

## 2. 폴더 구조 전수조사

```
knowhere/
├── apps/
│   ├── api/                       # FastAPI 서버 (HTTP API + MCP 엔드포인트)
│   │   ├── app/api/v1/routes/     # api_key, billing, demo, documents, guest,
│   │   │                          # jobs, qstash_callbacks, retrieval,
│   │   │                          # s3_events, version, webhook, webhook_secrets
│   │   ├── app/api/v2/routes/     # documents, jobs, retrieval (Vision 트랙)
│   │   ├── app/mcp/               # ★ MCP 서버
│   │   │   ├── retrieval_server.py   # FastMCP 서버 생성 + retrieval.query 도구
│   │   │   └── dynamic_tools.py      # agent_tools REGISTRY → FastMCP 브릿지
│   │   ├── app/services/          # auth, billing, demo, document_ingestion,
│   │   │                          # documents, guest, jobs, rate_limit,
│   │   │                          # s3_events, webhook
│   │   ├── app/repositories/      # DB 접근 계층
│   │   ├── app/core/              # 미들웨어, 응답 포맷
│   │   ├── alembic/versions/      # DB 마이그레이션
│   │   └── tests/                 # contract / migrations / unit 테스트
│   │
│   └── worker/                    # Celery 워커 (무거운 파싱 작업 전담)
│       └── app/services/
│           ├── document_parser/   # assets, conversion, formats, orchestration,
│           │                      # profiling, providers, structure, support, tables
│           ├── page_memory/       # ★ Vision 트랙
│           │                      # page_renderer, page_plan, page_tagger,
│           │                      # skeleton_extractor, fine_hierarchy,
│           │                      # node_assembler, page_assets, memory_service
│           ├── document_agent/    # 문서 프로파일링 에이전트
│           │                      # coordinator, coarse_profile, calibration,
│           │                      # structure, visual, validators, trace
│           ├── document_ingestion/
│           ├── connect_builder/   # 청크 간 연결(connect_to) 구축
│           ├── workload/          # 과금용 워크로드 추정
│           └── common/
│
├── packages/shared-python/shared/ # API·워커 공용 코어 라이브러리
│   ├── services/retrieval/        # ★★ 검색 엔진 심장부
│   │   ├── agent_tools/           # corpus.* 도구 정의
│   │   │   ├── CORPUS_SCHEMA.md   # ★ 에이전트용 단일 진실 원천 스키마 문서
│   │   │   ├── registry.py        # 도구 레지스트리
│   │   │   └── tools/             # assets, grep, list_documents, neighbors,
│   │   │                          # node_filter, outline, read, recall
│   │   ├── agent_explore/         # 에이전트 탐색 하네스
│   │   │                          # bridge, budget, config, dispatch,
│   │   │                          # harness, ref_resolution
│   │   ├── search/ scoring/ graph/ hydration/ execution/ trace/ stats/
│   │   ├── publication_service.py # 파싱 결과 → DB 문서 발행
│   │   ├── cache_service.py       # 검색 캐시
│   │   └── serving_manifest.py    # 서빙 인덱스 세대 관리
│   ├── services/ai/               # LLM/VLM 프로바이더 추상화
│   ├── services/billing/          # credits_service, work_billing_service
│   ├── services/storage/ auth/ webhook/ telemetry/ redis/ quota/ jobs/
│   ├── models/database/ schemas/  # SQLAlchemy 모델 + Pydantic 스키마
│   └── core/config/               # ai, app, billing, celery, database,
│                                  # job, mineru, qstash, redis, retrieval, storage
│
├── deploy/
│   ├── local-dev/                 # docker-compose.dev.yml (Postgres+Redis+LocalStack)
│   │                              # start-dev.sh / stop-dev.sh
│   ├── docker/                    # Dockerfile.api, Dockerfile.worker
│   └── ecs/                       # AWS ECS 태스크 정의 + 렌더러 + 스모크 테스트
│
├── docs/
│   ├── adr/                       # 아키텍처 의사결정 기록 10편
│   │   ├── 0001 라우트/워커태스크를 어댑터로 유지
│   │   ├── 0002 타입드 워크플로 결과 사용
│   │   ├── 0003 검색 워크플로 정책 명시화
│   │   ├── 0004 익명 셀프호스팅 텔레메트리
│   │   ├── 0005 SSE 기반 검색 진행 스트리밍
│   │   ├── 0006 검색 서빙 인덱스 원자적 발행
│   │   ├── 0007 일관된 검색 서빙 세대
│   │   ├── 0008 서빙 인덱스 롤아웃 유지보수 윈도우
│   │   ├── 0009 맵 유닛 조회용 토큰 선행 커버링 인덱스
│   │   └── 0010 서빙 인덱스 통계 인플레이스 백필
│   ├── design/                    # 검색 인덱스·SSE·스냅샷 캐시 설계 문서
│   └── assets/                    # 배너, 벤치마크, 다이어그램 이미지
│
├── .github/workflows/             # pr-ci, codeql, secret-scan,
│                                  # build-images, manage-staging
├── scripts/                       # cleanup_macos_junk.sh, smoke_byok.sh
├── deprecated/mapnav/             # 구 MapNav 워크플로 (Retrieval 2.0 이전)
├── AGENTS.md                      # ★ 28KB 에이전트용 레포 설명서
├── CONTEXT.md                     # ★ 18KB 도메인 용어 사전
├── CLAUDE.md                      # 카리나 페르소나 정의 (이 포크에서 추가)
├── Makefile                       # make lint / lint-fix / typecheck / check
└── pyproject.toml                 # uv 워크스페이스 루트
```

---

## 3. 핵심 기술: 듀얼 트랙 파싱

기존 OCR/Document Intelligence 파이프라인은 "모든 요소를 완벽히 추출한 뒤에야 모델이 이해할 수 있다"는
전제를 깔고 간다. 그래서 읽기 순서, 레이아웃, 표, 숨은 텍스트 레이어에서 발생한 오류가 누적되면
모델 컨텍스트 전체가 신뢰할 수 없게 된다.

Knowhere는 **"완벽한 요소별 추출"을 검색의 전제조건에서 제외**했다.

| | **Vision Track** | **Text Track** |
|---|---|---|
| 대상 포맷 | `.pdf` `.pptx` (V2 Jobs API) | `.doc` `.docx` `.xls` `.xlsx` `.md` `.txt` `.html` `.htm` `.json` `.jpg` `.jpeg` `.png` |
| 처리 방식 | 페이지를 이미지로 렌더링 → 프론티어 비전 모델이 페이지 전체를 통째로 이해 | 원본의 정확한 텍스트와 네이티브 구조를 그대로 보존 |
| 청크 타입 | `page` (리프 섹션만 보유) | `text` (리프/비리프 모두 보유 가능) |
| 강점 | OCR/레이아웃 추출이 실패해도 요약·엔티티·계층으로 회수 가능 | 정밀도 최고, 처리 속도 빠름 |

**핵심 설계**: 두 트랙이 **동일한 chunk + metadata 스키마**로 수렴한다.
덕분에 저장, 계층 구축, 그래프 구축, 검색 단계는 원본 포맷에 독립적이다.

### 지원 포맷

- 지원: `.pdf` `.pptx` (Vision) / `.doc` `.docx` `.xls` `.xlsx` `.jpg` `.jpeg` `.png` `.md` `.txt` `.html` `.htm` `.json` (Text)
- 예정: `.epub` `.xml` `.mp4` `.mp3` `.skills.md`

---

## 4. 전체 파이프라인 흐름

```
① 업로드      POST /api/v2/jobs  (파일 또는 URL)
                ↓  S3 업로드 → Job 생성 → Celery 큐 전달
② 파싱        워커가 포맷 판별 → Vision / Text 트랙 라우팅
                ↓  페이지 렌더링, VLM 이해, 표·이미지 추출 및 요약
③ 정규화      통일된 chunk + metadata 스키마로 변환
                ↓  ZIP 결과 패키지 생성 (체크섬, 통계 포함)
④ 퍼블리케이션 DB 적재
                ↓  documents / document_sections / document_chunks
                ↓  graph_nodes / graph_edges (문서 간 관계 그래프)
⑤ 검색        POST /api/v1|v2/retrieval/query  또는  MCP /mcp 엔드포인트
```

### 코퍼스 데이터 모델 (CORPUS_SCHEMA.md 기준)

```
Namespace                       # 검색 격리 범위 (기본값: "default")
  └─ Document                   # document_id, source_file_name, parse_track
       └─ Section               # section_id, parent_section_id, section_path,
            │                   # section_level, summary
            └─ Chunk            # chunk_id, chunk_type, content, chunk_metadata
                                # chunk_type: text | page | image | table
```

핵심 규칙 (실제 스키마 문서에서 확인):

- 한 섹션은 **최대 1개의 본문 청크**(`text` 또는 `page`)를 가진다.
- `image` / `table` 청크는 시각적으로 속한 섹션의 자식이 **아니다**. DB 상으로는 문서의 합성 `Root`
  섹션 아래에 위치하고, 본문 청크의 `connect_to` 리스트로 연결된다 (단방향: 본문 → 자산).
- `page` 청크에서 같은 물리 페이지를 공유하는 리프 섹션은 `[SAME-AS <owner_path> p<page>]`
  마커를 가진다. 이건 미리보기가 아니라 **포인터**이므로 원본 소유 섹션에서 해석해야 한다.
- 문서 그래프는 현재 **문서 레벨 노드**와 무방향 `related` 엣지만 존재한다.
  엣지 점수는 타입드 엔티티 중첩을 우선하고, 없으면 TF-IDF 키워드 중첩으로 폴백한다.

---

## 5. Agentic Retrieval & MCP

### 기존 RAG vs Knowhere

| | 기존 RAG | Knowhere Agentic Retrieval |
|---|---|---|
| 방식 | 평면적 벡터 조회 → 고립된 스니펫 반환 | 에이전트가 섹션 트리와 문서 그래프를 직접 탐색 |
| 맥락 | 조각 간 관계 소실 | 문서 → 섹션 경로 → 페이지 번호까지 보존 |
| 출처 | 불명확 | 문서/섹션/페이지/렌더링 이미지까지 추적 가능 |
| 제어 | 고정 파이프라인 | 에이전트가 도구 호출 순서·깊이를 스스로 결정 |

### corpus.* 도구 세트

| 도구 | 역할 |
|---|---|
| `list_documents` | 네임스페이스 내 문서 목록 조회 |
| `outline` | 문서의 섹션 트리(목차) 조회 |
| `node_filter` | 구조 기반 섹션 필터링 |
| `grep` | 정확 일치 검색 (content trigram 인덱스 활용) |
| `recall` | 퍼지 / 의미 기반 회상 |
| `read` | 특정 섹션 전문 읽기 |
| `neighbors` | 문서 그래프 기반 관련 문서 탐색 |
| `assets` | 이미지 / 표 자산 조회 및 역방향 조회 |

### MCP 서버 구현

`apps/api/main.py`에서 FastMCP 앱을 `/mcp` 경로에 마운트한다.

```python
# apps/api/main.py
retrieval_mcp_server = create_retrieval_mcp_server(streamable_http_path="/mcp")
retrieval_mcp_app = retrieval_mcp_server.streamable_http_app()
```

```python
# apps/api/app/mcp/retrieval_server.py
server = FastMCP(
    "knowhere-retrieval",
    instructions=(... + load_corpus_schema_text()),
    stateless_http=True,
    transport_security=create_public_mcp_transport_security(),
)
register_corpus_tools(server, db_factory=db_factory)
```

- **인증**: HTTP `Authorization` 헤더 → `resolve_mcp_user_id()`
- **네임스페이스**: `x-knowhere-namespace` 헤더 → `resolve_mcp_namespace()`
- **도구 등록**: `dynamic_tools.py`가 `agent_tools.REGISTRY`의 JSON Schema를
  FastMCP `Tool` 객체로 직접 변환해 등록 (파이썬 함수 시그니처 왕복 없이 스키마를 그대로 노출)
- **레거시 도구**: `retrieval.query` (하위 호환용 원샷 검색)
- **응답 계약**: `evidence_text`, `referenced_chunks`, `decision_trace`

`CORPUS_SCHEMA.md` 하나를 **MCP 서버 instructions와 `agent_explore` 시스템 프롬프트에
동일하게 공급**하는 설계가 이 프로젝트의 핵심 아이디어 중 하나다.

### MCP 클라이언트 연결 예시

```jsonc
{
  "mcpServers": {
    "knowhere": {
      "type": "http",
      "url": "http://localhost:5005/mcp",
      "headers": {
        "Authorization": "Bearer <API_KEY>",
        "x-knowhere-namespace": "default"
      }
    }
  }
}
```

---

## 6. 기술 스택 & 인프라

| 구분 | 사용 기술 |
|---|---|
| 웹 프레임워크 | FastAPI 0.135.1, Uvicorn 0.34.3, Starlette 0.52.1 |
| DB | PostgreSQL (SQLAlchemy 2.0.42 async, Alembic 1.13.1) |
| 큐 / 캐시 | Redis 5.3.1, Celery 5.5.3 |
| 스토리지 | S3 호환 (개발: LocalStack, 알리바바 OSS 지원) |
| MCP | `mcp>=1.27.0` (FastMCP) |
| 결제 | Stripe 13.0.1 (크레딧 기반 과금) |
| 웹훅 | QStash 3.2.0 |
| 관측 | Logfire, Moesif, PostHog 익명 텔레메트리 |
| PDF 처리 | PyMuPDF 1.27.2, pymupdf4llm, pypdf 6.10.2 |
| Office 처리 | python-docx 1.2.0, python-pptx 1.0.2, openpyxl 3.1.2 |
| 표 추출 | tabula-py |
| OCR | rapidocr-onnxruntime |
| HTML | BeautifulSoup4, lxml, markdownify |
| 수치 연산 | numpy 2.2.6, pandas 2.3.1 |
| 토크나이징 | jieba (중국어) |
| LLM SDK | openai 1.93.3 (OpenAI 호환 인터페이스) |
| 에이전트 하네스 | cursor-sdk >= 1.0.31 |

### 지원 LLM / VLM 프로바이더

| 환경변수 | 프로바이더 |
|---|---|
| `DS_KEY` | DeepSeek (README 권장: `deepseek-v4-flash-vision-exp`) |
| `GPT_API_KEY` | OpenAI |
| `GLM_API_KEY` | Zhipu GLM |
| `ALI_API_KEYS` | 알리바바 Qwen |
| `ARK_API_KEY` | Volcengine |
| `MINERU_API_KEYS` | MinerU (V1 청크 기반 PDF 파이프라인 전용) |

### 생태계 레포지토리

| 레포 | 설명 |
|---|---|
| [knowhere](https://github.com/Ontos-AI/knowhere) | 백엔드 API + 워커 (이 레포) |
| [knowhere-dashboard](https://github.com/Ontos-AI/knowhere-dashboard) | 웹 UI |
| [knowhere-self-hosted](https://github.com/Ontos-AI/knowhere-self-hosted) | Docker Compose 셀프호스팅 스택 |
| [knowhere-python-sdk](https://github.com/Ontos-AI/knowhere-python-sdk) | 공식 Python SDK |
| [knowhere-node-sdk](https://github.com/Ontos-AI/knowhere-node-sdk) | 공식 Node.js SDK |

### 성능 벤치마크 (README 주장, 내부 평가 기준)

- 원본 문서 대비 **첫 시도 정확도 +36%**, **Recall +11%**
- 피드백 포함 시 정확도 **79%** (원본 문서 직접 투입은 약 53% 한계)
- 베이스라인: 원본 문서, Markitdown, Unstructured, MinerU 출력 직접 투입

---

## 7. 쉽게 이해하기 (비유편)

### 비유 1 — 신입사원 교육

- **기존 RAG**: 문서 1000장을 분쇄기에 갈아 조각으로 만들고, 질문할 때마다 조각 5장을 무작위로 던져줌.
  신입은 앞뒤 맥락을 알 수 없다.
- **Knowhere**: 문서를 제본하고 목차와 색인을 붙여 도서관처럼 정리한 뒤, 신입에게
  "검색대 / 목차 보기 / 해당 페이지 펼치기 / 관련 책 찾기" 도구를 쥐여준다.
  신입이 스스로 목차를 보고 필요한 챕터를 찾아간다.

### 비유 2 — 주방이 있는 식당

| 폴더 / 구성요소 | 식당 비유 | 역할 |
|---|---|---|
| `apps/api` | 홀 + 카운터 | 주문 접수, 인증, 결제, 결과 전달 |
| `apps/worker` | 주방 | 시간이 오래 걸리는 파싱 작업 수행 |
| Redis + Celery | 주문서 컨베이어 | 홀 → 주방 작업 전달 |
| PostgreSQL | 창고 장부 | 문서/섹션/청크/그래프 기록 |
| S3 | 냉장 창고 | 원본 파일 및 결과 패키지 보관 |
| `packages/shared-python` | 공용 레시피북 | API·워커가 공유하는 규칙 |
| `/mcp` | 배달앱 연동 창구 | 외부 AI 클라이언트의 주문 접수 |

### 비유 3 — 한 문장 요약

> **Knowhere = "AI를 위한 도서관 그 자체 + 도서관 사서"**
> 문서를 받아 정리해 꽂아두고, AI가 직접 찾아볼 수 있도록 도구까지 제공하는 시스템.

---

## 8. 설치 및 사용법

### 사전 준비물

- Python 3.11 이상
- `uv` (파이썬 패키지 매니저)
- Docker + `docker compose`

### 설치 절차

```bash
# 1) 워크스페이스 의존성 설치
uv sync --all-packages

# 2) 환경변수 파일 복사
cp apps/api/.env.example apps/api/.env
cp apps/worker/.env.example apps/worker/.env

# 3) .env 편집 (아래 필수 설정 참고)

# 4) 로컬 인프라 기동 (Postgres + Redis + LocalStack)
./deploy/local-dev/start-dev.sh

# 5) DB 마이그레이션
cd apps/api && uv run alembic upgrade heads

# 6) API 기동
cd apps/api && uv run main.py

# 7) 워커 기동 (별도 터미널)
cd apps/worker && uv run worker.py

# 8) API 전용 사용자/키 생성 (대시보드 미사용 시)
cd apps/api && uv run scripts/init_user.py --email you@example.com
```

> 대시보드를 함께 쓸 계획이면 `init_user.py` 대신 대시보드에서 회원가입할 것.

### 주요 환경변수

| 항목 | 설명 |
|---|---|
| `DATABASE_URL` | Postgres 연결 (기본 `postgresql+asyncpg://root:root123@localhost:5432/Knowhere`) |
| `REDIS_HOST` / `REDIS_PORT` | Redis 연결 |
| `CELERY_REDIS_URL` | Celery 브로커 |
| `S3_*` | S3 호환 스토리지 (개발은 LocalStack `:4566`) |
| `DS_KEY` / `GPT_API_KEY` / `GLM_API_KEY` / `ALI_API_KEYS` / `ARK_API_KEY` | LLM 키 (최소 1개 필수) |
| `MINERU_API_KEYS` | V1 PDF 파이프라인 사용 시에만 |
| `MAX_FILE_SIZE` | 기본 314572800 (300MB) |
| `MAX_PDF_PAGE_LIMIT` | 기본 200 페이지 |
| `OVERSIZED_PDF_SHARD_ENABLED` / `OVERSIZED_PDF_SOFT_LIMIT` | 초장문 PDF 샤딩 (기본 1500p) |
| `BILLING_ENABLED` | 과금 기능 on/off (개인 사용 시 `false` 권장) |
| `TELEMETRY_ENABLED` | 익명 텔레메트리 (기본 `true`, 끄려면 `false`) |
| `ENTITY_TYPES` | 기본 `person,location,organization` |

### 로컬 엔드포인트

| 서비스 | 주소 |
|---|---|
| API | http://localhost:5005 |
| OpenAPI 문서 (Swagger) | http://localhost:5005/docs |
| MCP 엔드포인트 | http://localhost:5005/mcp |
| LocalStack | http://localhost:4566 |
| PostgreSQL | localhost:5432 |
| Redis | localhost:6379 |

### 품질 검사 명령

```bash
make lint        # Ruff 린트
make lint-fix    # 안전한 자동 수정
make typecheck   # Pyright 타입 검사
make check       # 린트 + 타입 검사
```

### 더 쉬운 대안

1. **Docker Compose 통합 스택** — `knowhere-self-hosted` 레포 (API + 워커 + 대시보드)
2. **UI 포함 사용** — `knowhere-dashboard` 를 함께 기동
3. **설치 없이 사용** — knowhereto.ai 관리형 클라우드 (가입 시 $5 무료 크레딧)

---

## 9. 플러그인? 스킬? MCP?

**결론: 셋 다 아니다. "제품 수준의 독립 백엔드 서비스"이며, 그 안에 MCP 서버를 내장하고 있다.**

| 구분 | 해당 여부 | 설명 |
|---|---|---|
| Claude 플러그인 | 아님 | Claude Code 플러그인 규격 아님 |
| Claude 스킬 | 아님 | `SKILL.md` 등 스킬 구조 없음 |
| MCP 서버 | 내장함 | 전체가 MCP는 아니지만 `/mcp` 경로로 MCP 서버를 노출 |
| 독립 백엔드 서비스 | 정답 | FastAPI + Celery 워커 + PostgreSQL + Redis + S3 로 구성된 완전한 서버 |

정확한 표현: **"거대한 문서 처리 백엔드인데, 그 검색 기능을 MCP 프로토콜로도 개방한 시스템"**

---

## 10. API 토큰 정리

### (1) LLM 제공자 API 키 — 필수

문서 요약, 계층 추론, 페이지 이해, 이미지 설명에 모델이 필요하므로 최소 1개는 반드시 있어야 한다.
PDF/PPTX를 V2 Vision 트랙으로 처리하려면 **비전 지원 모델**이 필요하다.

> 참고: `openai` SDK(OpenAI 호환 인터페이스)를 사용하므로, Ollama/vLLM 등 OpenAI 호환 엔드포인트를
> 로컬에 띄워 연결하는 방식도 이론적으로 가능하다 → 완전 오프라인 구성의 실마리.

### (2) Knowhere 자체 API 키 — 필수

- 생성: `uv run scripts/init_user.py --email you@example.com`
- 사용: HTTP `Authorization` 헤더
- MCP 연결 시에도 동일한 헤더로 인증
- 관리 엔드포인트: `/api/v1/api_key`

### (3) 선택 사항

`STRIPE_SECRET_KEY`(결제), `QSTASH_TOKEN`(웹훅), `LOGFIRE_TOKEN`/`MOESIF_APPLICATION_ID`(관측),
`ILOVEAPI_*`(문서 변환 보조). 개인 사용 시 모두 비워도 무방하며 `BILLING_ENABLED=false` 로 설정하면 된다.

---

## 11. GitHub에서 유명한 이유

> 주의: 이 분석 세션은 `bmshin94/knowhere` 레포에만 접근 권한이 있어 **원본 레포의 정확한 스타 수는
> 확인하지 못했다.** 아래는 코드와 문서 근거에 기반한 인기 요인 분석이다.

1. **타이밍** — "에이전트에게 어떻게 컨텍스트를 줄 것인가"가 현재 AI 업계 최대 화두.
   README의 선언이 명확하다: *"We're not developing the next MinerU — we're building document
   memory infrastructure that agents can effectively consume."*
2. **진짜 풀스택 오픈소스** — 파싱, 그래프 구축, 검색, 과금, 웹훅까지 전부 Apache 2.0으로 공개.
3. **MCP 네이티브 설계** — 나중에 붙인 게 아니라 처음부터 MCP를 1급 인터페이스로 설계.
   `CORPUS_SCHEMA.md` 하나를 MCP instructions와 에이전트 프롬프트에 동일 공급.
4. **자극적인 벤치마크 수치** — 정확도 +36%, Recall +11%, 피드백 포함 79% vs 53% 한계.
5. **완비된 생태계** — 대시보드, 셀프호스팅 스택, Python/Node SDK 가 모두 별도 레포로 존재.
6. **높은 코드 품질** — ADR 10편, 28KB `AGENTS.md`, 18KB `CONTEXT.md`,
   CodeQL + 시크릿 스캔 CI, 계약 테스트.
7. **실질적 고통 해결** — "복잡한 PDF RAG 정확도가 안 나온다"는 보편적 문제에 대해
   "완벽 추출을 포기하고 비전 모델로 페이지를 통째로 이해한다"는 발상의 전환 제시.

---

## 12. 로컬 에이전트 구축 활용도

### 도움이 되는 점

1. **MCP 즉시 연결** — 서버 기동 후 Claude Code / Claude Desktop / Cursor MCP 설정에 등록하면
   `corpus.outline`, `corpus.grep`, `corpus.read` 등으로 문서 창고를 직접 탐색 가능.
2. **데이터 로컬 보관** — 원본 파일, 파싱 결과, DB 모두 자체 서버. 외부 호출은 LLM/VLM 추론 시점뿐.
3. **네임스페이스 격리** — 프로젝트별/고객사별 문서 분리 (`x-knowhere-namespace`).
4. **에이전트 설계 레퍼런스** — `agent_explore/`의 budget 관리, dispatch, ref_resolution 패턴은
   자체 로컬 에이전트 구현 시 그대로 참고 가능.

### 주의할 점

| 이슈 | 내용 |
|---|---|
| 완전 오프라인 아님 | 기본 구성은 외부 LLM API 호출. 오프라인화하려면 로컬 VLM을 OpenAI 호환으로 서빙해야 함 |
| 리소스 요구량 | Postgres + Redis + S3(LocalStack) + API + 워커 = 최소 5개 컨테이너 |
| VLM 비용 | 페이지 단위 비전 호출 → 장문 PDF는 비용 급증. `MAX_PDF_PAGE_LIMIT` 기본 200 |
| 러닝커브 | 약 15.5만 줄 규모. 내부 커스터마이징에는 학습 시간 필요 |

### 추천 도입 순서

1. `knowhere-self-hosted` Docker Compose 로 기동 → 샘플 문서 투입 → 품질 확인
2. Claude Code에 MCP 연결 → 실사용 검증
3. 저가 모델(DeepSeek 등)로 비용 최적화
4. 필요 시 `document_parser/formats/` 에 커스텀 파서 추가

---

## 13. React / PHP 로 만들 수 있나?

### (A) React로 프론트엔드 제작 — 가능

이 레포는 백엔드 전용이다. UI는 별도 레포이므로 React로 직접 구현하면 된다.

```
[React 앱] --HTTP--> [Knowhere API :5005]
  업로드 UI            POST /api/v2/jobs
  진행률 표시          GET  /api/v2/jobs/{job_id}   (SSE 스트리밍 지원, ADR-0005)
  문서 목록            GET  /api/v2/documents
  검색 UI              POST /api/v2/retrieval/query
```

`http://localhost:5005/docs` 에 OpenAPI 스펙이 제공되므로 `openapi-typescript` 등으로
타입스크립트 클라이언트를 자동 생성할 수 있다.

### (B) PHP로 프론트/BFF 제작 — 가능

```php
$response = Http::withHeaders([
    'Authorization'        => 'Bearer ' . env('KNOWHERE_API_KEY'),
    'x-knowhere-namespace' => 'default',
])->post('http://localhost:5005/api/v2/retrieval/query', [
    'query' => '2026년 매출 전망',
]);
```

API 키를 서버 측에서만 관리하는 BFF 구조가 보안상 권장된다.

### (C) Knowhere 자체를 React/PHP로 재구현 — 비권장

| 필요 기능 | Python | PHP | Node/JS |
|---|---|---|---|
| PDF 렌더링/파싱 | PyMuPDF (최상) | Imagick + poppler (보통) | pdf.js (보통) |
| DOCX/XLSX/PPTX | python-docx 등 (최상) | PhpSpreadsheet (보통) | 제한적 |
| 표 추출 | tabula-py (최상) | 사실상 없음 | 거의 없음 |
| OCR | rapidocr (최상) | Tesseract 래퍼 (보통) | tesseract.js (보통) |
| 비동기 워커 | Celery (최상) | Laravel Queue (양호) | BullMQ (양호) |
| 수치 연산 | numpy / pandas (최상) | 없음 | 제한적 |

### 권장 아키텍처

```
┌──────────────────┐
│   React 프론트엔드 │   직접 개발
└────────┬─────────┘
         │ HTTPS
┌────────▼─────────┐
│  PHP / Node BFF  │   인증·세션·비즈니스 로직 (선택)
└────────┬─────────┘
         │
┌────────▼──────────────────┐
│  Knowhere (Python) 그대로   │   문서 처리 엔진은 유지
│  API + Worker + MCP        │
└────────────────────────────┘
```

---

## 14. 수익화 아이디어 10선

### 법적 전제 (Apache 2.0)

| 가능 여부 | 항목 |
|---|---|
| 가능 | 상업적 이용, 수정, 재배포 |
| 가능 | 파생 코드의 클로즈드소스 유지 (GPL과 다름) |
| 가능 | 특허 사용 허가 포함 |
| 의무 | `LICENSE` + `NOTICE` 포함, 변경 사항 명시 |
| 의무 | "Knowhere" / "Ontos" 상표 무단 사용 금지 → 자체 브랜드로 리브랜딩 필요 |
| 권장 | 고객 데이터 보호 및 신뢰를 위해 `TELEMETRY_ENABLED=false` |

### TIER 1 — 진입장벽이 낮고 현실적

**① 국내 기업용 문서 AI 비서 SaaS**
- 원본 팀은 한국어 최적화가 약함 → 차별화 지점
- 국내 기업의 데이터 해외 반출 거부감 → 국내 리전 호스팅이 강력한 셀링포인트
- **HWP/HWPX 파서 추가 시 국내 독점급 경쟁력**
- 가격 예시: 월 9.9만원(50문서) / 29.9만원(300문서) / 99만원(무제한 + 온프레미스)
- 난이도 중 / 수익 잠재력 최상

**② 버티컬 특화 (업종 좁히기)**

| 버티컬 | 킬러 포인트 | 지불 의사 |
|---|---|---|
| 법무 / 계약서 | 조항별 페이지 단위 인용이 필수 → citation 강점 직결 | 매우 높음 |
| 건설 / 도면 | 300~500p 도면집 지원 기구현, Vision 트랙이 도면에 강함 | 높음 |
| 의료 / 논문 | 표·그래프 이해가 핵심 | 높음 |
| 제조 매뉴얼 | 현장 작업자 즉시 조회 | 중간 |
| IR / 재무 | 사업보고서 수백 페이지 표 추출 | 높음 |
| 교육 / 자격증 | 교재 기반 질의응답·문제 생성 | 중간 |

→ 추천: **법무 계약서** 또는 **건설 도면** (출처 증명이 핵심인 분야)

**③ MCP 마켓플레이스 진출**
- Claude Desktop / Claude Code / Cursor 사용자 대상 "내 문서 창고" MCP 구독 상품
- 사용자는 URL + API 키만 입력 → 설치 난이도 최소
- `/mcp` 엔드포인트가 이미 구현되어 개발 비용 최소
- 월 $9~19 구독, 개발자·연구자 타깃
- 난이도 하 / 수익 중

### TIER 2 — 기술력 기반 차별화

**④ 한국어 문서 파싱 특화 엔진 판매**
- HWP/HWPX 파서 (국내 공공기관 필수)
- 한국어 형태소 기반 청킹 (원본은 중국어용 `jieba` 사용)
- 한국식 표 양식(품의서, 결재란) 인식
- 공공기관 고시·공고문 구조 인식
- B2B 라이선스 형태로 고단가 가능

**⑤ 온프레미스 구축 + 유지보수 계약**

| 항목 | 금액 예시 |
|---|---|
| 초기 구축 | 3,000만 ~ 1억원 |
| 연간 유지보수 | 구축비의 15~20% |
| 커스텀 파서 개발 | 건당 500~2,000만원 |

금융·공공·방산은 클라우드 사용이 제한되어 온프레미스 수요가 확실하다.
난이도 상 (영업력 필요) / 수익 최상

**⑥ API 리셀링**
- 셀프호스팅 + 저가 LLM(DeepSeek) 조합으로 원가 절감
- "문서 1건당 N원" 종량제 API 판매
- Stripe 연동과 크레딧 시스템(`shared/services/billing/`)이 이미 구현되어 있어 재활용 가능

### TIER 3 — 크리에이티브 & 틈새

**⑦ AI 논문/교재 튜터** — 교재·논문 업로드 → 챕터별 요약, 퀴즈, 오답노트.
Vision 트랙의 수식·도표 이해가 킬러 기능. 대학생·자격증 준비생, 월 9,900원.

**⑧ 개인 지식 창고 (Second Brain)** — Notion/Obsidian 사용자 대상.
PDF·논문·스크랩을 모두 적재하고 MCP로 Claude 연결 → 내 자료에만 근거하는 개인 비서.

**⑨ RFP / 입찰 문서 분석기** — 공고문 수백 페이지에서 요건·평가기준·마감일 자동 추출.
사람이 3일 걸릴 작업을 10분으로 단축 → ROI 계산이 명확해 영업이 쉬움. 건당 과금.

**⑩ 교육 콘텐츠 수익화** — "Agentic RAG 실전 구축" 강의(인프런/유데미),
유튜브·블로그, 기술 컨설팅. 오픈소스 기여를 통한 개인 브랜딩 → 커리어 가치 상승.

### 최종 추천 순위

| 순위 | 아이디어 | 선정 이유 |
|---|---|---|
| 1위 | HWP 지원 + 한국어 특화 → 법무/건설 버티컬 SaaS | 진입장벽 + 명확한 페인포인트 + 높은 지불의사 |
| 2위 | MCP 구독 상품 | 개발 비용 최소, 빠른 시장 검증 가능 |
| 3위 | 온프레미스 구축 + 유지보수 | 단가 최고, 현금흐름 안정적 |

---

## 15. 실행 로드맵 & 리스크

### 6개월 로드맵

```
1개월차  셀프호스팅 구축 → 한국어 문서 100건 투입 → 품질 측정 및 베이스라인 확보
2개월차  HWP 파서 추가 + 한국어 청킹 개선 (기술 해자 구축)
3개월차  랜딩페이지 + React 데모 제작 → 잠재고객 10곳 인터뷰
4개월차  파일럿 3곳 무료 제공 → 피드백 반영 → 케이스 스터디 확보
5~6개월  유료 전환 + 가격 정책 확정 → 본격 영업
```

### 리스크 및 대응

| 리스크 | 대응 방안 |
|---|---|
| 원저작자의 라이선스 변경 (BSL 전환 등) | 현재 Apache 2.0 버전 포크 확보 + 커밋 해시 고정 |
| LLM API 비용 폭증 | 저가 모델 활용 + 결과 캐싱 + 페이지 수 제한 |
| 대형 벤더(MS, Google) 진입 | 버티컬 특화 + 한국어 최적화 + 온프레미스로 방어 |
| 오픈소스라 누구나 복제 가능 | 진짜 해자는 도메인 노하우 + 한국어 최적화 + 고객 관계 |

**핵심 인사이트**: 오픈소스 코드 자체는 해자가 아니다.
**업종 도메인 노하우, 한국어 최적화 축적, 고객 신뢰**가 실질적 방어선이다.

---

## 16. 참고 링크 모음

### GitHub 레포지토리

| 구분 | 주소 |
|---|---|
| **분석 대상 (포크)** | **https://github.com/bmshin94/knowhere** |
| **원본 (업스트림)** | **https://github.com/Ontos-AI/knowhere** |
| 웹 대시보드 | https://github.com/Ontos-AI/knowhere-dashboard |
| 셀프호스팅 스택 | https://github.com/Ontos-AI/knowhere-self-hosted |
| Python SDK | https://github.com/Ontos-AI/knowhere-python-sdk |
| Node.js SDK | https://github.com/Ontos-AI/knowhere-node-sdk |

### 공식 채널

| 구분 | 주소 |
|---|---|
| 웹사이트 | https://knowhereto.ai |
| 공식 문서 | https://docs.knowhereto.ai/ |
| GitHub Discussions | https://github.com/Ontos-AI/knowhere/discussions |
| GitHub Issues | https://github.com/Ontos-AI/knowhere/issues |

### 레포 내부 필독 문서

| 파일 | 내용 |
|---|---|
| `AGENTS.md` | 28KB 규모의 에이전트용 레포 설명서 (파이프라인 전 단계 상세) |
| `CONTEXT.md` | 18KB 도메인 용어 사전 |
| `packages/shared-python/shared/services/retrieval/agent_tools/CORPUS_SCHEMA.md` | 에이전트용 코퍼스 스키마 단일 진실 원천 |
| `docs/adr/README.md` | 아키텍처 의사결정 기록 10편 |
| `docs/design/` | 검색 인덱스, SSE 스트리밍, 스냅샷 캐시 설계 문서 |
| `docs/retrieval-document-scope.md` | 검색 문서 범위 가이드 |
| `CONTRIBUTING.md` | 브랜치 전략, 커밋 컨벤션, 리뷰 프로세스 |
| `LICENSE` / `NOTICE` | Apache 2.0 라이선스 원문 |

---

## 부록: 분석 요약 한 장

| 질문 | 답 |
|---|---|
| 이게 뭐야? | 비정형 문서를 AI 에이전트용 "탐색 가능한 기억"으로 바꾸는 파싱 + 검색 백엔드 |
| 언제 써? | 대량 사내 문서 검색, 복잡한 PDF/도면 처리, 출처 추적 필수 분야, 온프레미스 요구 환경 |
| 플러그인/스킬/MCP? | 독립 백엔드 서비스이며 MCP 서버를 내장 (`/mcp`) |
| 토큰 필요해? | LLM 제공자 키(필수) + Knowhere 자체 API 키(필수) |
| 유명한 이유? | 타이밍 + 진짜 풀스택 오픈소스 + MCP 네이티브 + 벤치마크 + 높은 코드 품질 |
| 로컬 에이전트에 도움? | 매우 도움됨. 단, 완전 오프라인은 로컬 VLM 서빙 필요 |
| React/PHP 가능? | 프론트/BFF는 가능. 엔진 재구현은 비권장 |
| 수익화? | 한국어·HWP 특화 버티컬 SaaS, MCP 구독, 온프레미스 구축이 유력 |

---

*이 문서는 저장소 전체를 직접 탐색(폴더 구조, 소스 코드, 설정 파일, 문서)하여 작성되었습니다.*
*벤치마크 수치 등 일부 내용은 원본 README의 주장을 인용한 것이며 독립적으로 검증되지 않았습니다.*
