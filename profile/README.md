# 연금사수 — 퇴직연금 창구 상담 지원 시스템

> <!-- 과정명·기수·팀 소개 자리 -->
<img width="2816" height="1536" alt="연금사수 logo" src="https://github.com/user-attachments/assets/c235af21-ff46-476a-9f49-fea4ada70df8" />

## 📍 1. 프로젝트 개요

- **주제**: 실시간 STT·RAG 기반 은행 퇴직연금 창구 상담 지원 시스템 **연금사수**

- **기획 배경**

  퇴직연금은 **행원도 매번 확인해야 하는 상담**입니다. IRP 개설 하나에도 가입 자격·운용상품
  위험·수수료·디폴트옵션·세액공제 한도가 걸리고, 중도인출이면 법정 사유와 세금이 또 달라집니다.
  근거는 근퇴법·소득세법·금융소비자보호법과 상품설명서·표준계약서에 흩어져 있어, 헷갈리는
  대목이 상담마다 새로 나옵니다.

  부담이 걸리는 지점은 세 곳입니다.

  - **상담 중** — 헷갈리는 대목이 나와도 고객 앞에서 문서를 뒤질 수 없다
  - **상담 후** — 확인이 필요했던 것을 제대로 짚고 넘어갔는지 되돌아볼 방법이 없다
  - **본부** — 행원이 헷갈릴 때마다 전화가 오고, 같은 질문이 지점마다 반복된다.
    본부 담당자의 본업이 아닌데 응대 부담만 쌓인다

  연금사수는 셋을 각각 받칩니다. 상담 중에는 고객의 말에서 **확인이 필요한 대목을 키워드로 잡아**
  근거 문서를 바로 옆에 띄우고, 상담 후에는 전사문을 읽어 요약과 **짚었어야 할 항목의 점검
  결과**를 자동으로 만듭니다. 본부로 가던 문의는 **창구에서 먼저 걸러지고**, 그래도 반복되는
  질문은 「이번 주 질의 Top5」로 집계돼 본부가 **한 번만 답하면 사내 게시판을 통해 전 지점에
  공유**됩니다 — 1:1 전화 응대가 1:N 게시글로 바뀝니다.

  > <!-- 통계·근거 인용 자리 -->

- **기술 스택**
  - **AI / STT**: Whisper large-v3-turbo (Groq 추론 API), Silero VAD(ONNX), WeSpeaker ResNet34-LM(ONNX, 화자 구분)
  - **AI / RAG**: BGE-M3(1024차원, pgvector) + BM25(kiwipiepy), RRF 하이브리드 검색, Gemini(Structured Output·스트리밍)
  - **Backend**: FastAPI, SQLAlchemy async(psycopg3), Pydantic v2, WebSocket, SSE, asyncio 백그라운드 태스크
  - **Frontend**: React 19, Vite 8, React Router 7, AudioWorklet(PCM 스트리밍)
  - **Database**: Supabase(PostgreSQL) + pgvector(HNSW), Redis(전사문·키워드, TTL 4h)
  - **Infra**: Docker Compose, nginx(정적 파일 + `/api` 프록시 · WebSocket·SSE), Cloudflare Tunnel
  - **Test**: pytest + fakeredis + LLM 목업 — 외부 의존성 없이 **155개** 실행

---

## 📍 2. 아키텍처

### 2-1. 시스템 아키텍처

<!-- 이미지 자리: 시스템 아키텍처 -->
<img width="" height="" alt="연금사수 시스템 아키텍처" src="" />

프론트엔드(React)와 백엔드(FastAPI)가 각각 컨테이너로 뜨고 Supabase(Postgres+pgvector)·Redis와
연동됩니다. 배포 시 진입점은 **nginx 하나**로, 정적 파일과 `/api` 프록시(WebSocket·SSE 포함)를
같은 origin에서 서비스하므로 CORS가 없고 프론트에 서버 주소를 박지 않습니다.

| 레이어 | 구성 |
|---|---|
| **서비스** | Frontend(React+Vite) · Backend(FastAPI) — 각각 독립 컨테이너, nginx가 단일 진입점 |
| **데이터** | Supabase PostgreSQL(pgvector HNSW) · Redis(전사문·키워드, TTL 4시간) |
| **실시간 채널** | `WS /api/stt/stream`(PCM 업 → 자막·키워드 다운) · `POST /api/rag/query`(SSE 스트리밍) |
| **AI 엔진 격리** | STT는 `services/stt/`, RAG는 `services/rag/` 안에만 — 라우터가 엔진을 직접 import하지 않음 |
| **계층 규칙** | `api/`(HTTP) → `services/`(로직) → `db/`(쿼리). 요청 1건 = AsyncSession 1개 |
| **비동기 작업** | 상담 후 정리 · 질의 자동 적재 — 요청 세션과 분리된 전용 세션으로 fire-and-forget |
| **운영** | Docker Compose(개발/배포 분리) + Cloudflare Tunnel → `https://protect-desk.cloud` |

### 2-2. 상담 파이프라인

<!-- 이미지 자리: 상담 파이프라인 다이어그램 -->
<img width="" height="" alt="연금사수 상담 파이프라인" src="" />

한 건의 상담이 **세 단계**를 지나며 서로 다른 산출물을 남깁니다.

- **상담 중** — VAD가 발화를 자르고, 화자 구분이 행원/고객을 가르고, Whisper가 받아씁니다.
  고객 질문 한 턴이 끝날 때마다 검색 키워드를 최대 3개 뽑아 칩으로 띄우고, 행원이 칩을 누르면
  하이브리드 RAG가 근거 원문과 함께 답변 카드를 만듭니다.
- **상담 후** — 종료 API는 **202로 즉시 응답**하고, 백그라운드 태스크가 Redis 전사문 전체를 읽어
  요약 3항목(문의·안내·후속) + 상담 유형 + **필수 고지 체크리스트 판정**을 저장합니다. 누락이
  있으면 고객 보완 안내 문자를 그 자리에서 생성합니다(저장하지 않음).
- **지식 순환** — 창구의 AI 질의는 임베딩 유사도로 클러스터링돼 본부의 「이번 주 질의 Top5」가
  되고, 본부가 RAG로 답변 초안을 만들어 발행하면 행원 메인의 사내 게시판에 다시 나타납니다.

---

## 📍 3. 핵심 기술

| # | 기술 | 설명 |
|---|---|---|
| 1 | **실시간 STT + 화자 구분 (한 소켓)** | Silero VAD로 발화를 자르고 WeSpeaker ResNet34(int8 ONNX, 6.7MB)로 행원/고객을 판정한 뒤 Whisper로 받아씁니다. 화자 판정과 받아쓰기를 **동시에** 돌려 지연을 줄이고, torch 없이 numpy로 kaldi fbank를 직접 계산해 의존성을 6.7MB로 묶었습니다(SpeechBrain 원본은 4.7GB). |
| 2 | **하이브리드 RAG (Dense + BM25 → RRF)** | BGE-M3 dense 검색(pgvector HNSW)과 kiwipiepy 형태소 기반 BM25를 각각 top-20 뽑아 RRF(k=60)로 재정렬합니다. 코퍼스 전체 점수를 매길 수 없는 온라인 환경에 맞춰 **후보 합집합에만 RRF를 매기는 방식**으로 옮겼습니다. 임베딩 키가 없으면 BM25 단독 축소 모드로 자동 폴백합니다. |
| 3 | **체크리스트는 코드가 정하고 LLM은 판정만** | 항목을 LLM이 생성하면 상담마다 기준이 달라져 "빠뜨린 게 없나"라는 점검의 의미가 사라집니다. 유형 4종의 항목을 코드에 고정하고 LLM 응답을 그 목록에 맞춰 다시 세웁니다. **판정 없는 항목은 누락 처리** — 확인 안 된 항목을 "안내함"으로 두면 행원이 안 한 안내를 했다고 믿게 되기 때문입니다. |
| 4 | **조건부 UPDATE 멱등성 + 고착 방지 3중 장치** | 정리 작업 선점을 파이썬 if가 아니라 **DB의 조건부 UPDATE 한 방**으로 합니다(읽기→검사→쓰기를 나누면 동시 요청 둘이 모두 통과해 LLM이 두 번 돕니다). 태스크는 프로세스 메모리에·상태는 DB에 있는 구조라, 기동 시 회수 · 고아 재시작 허용 · 프론트 폴링 상한으로 `processing` 고착을 막습니다. |
| 5 | **질의 집계 → 지식 순환** | 창구 질문을 자동으로 클러스터링해 본부의 「이번 주 질의 Top5」로 올리고, 본부가 한 번 답하면 사내 게시판으로 전 지점에 공유됩니다(1:1 전화 응대 → 1:N 게시글). 답변 스트림과 분리된 태스크로 질문을 적재하고, 검색용 임베딩을 재사용해 기존 클러스터와 코사인 유사도를 비교합니다. 매칭용(0.85)과 검색용(0.3) 임계값·임베딩 컬럼을 목적별로 분리해, "짧은 질문 vs 긴 안내글" 비교로 유사도가 왜곡되는 것을 막았습니다. |

---

## 📍 4. 실험 결과

모델·전략은 전부 실측으로 골랐습니다. 실험 코드는 서비스와 분리돼 있고 **의존 방향은 항상
실험 → 서비스**입니다.

### 4-1. STT 모델 선정 — Whisper small / medium / large-v3-turbo

AI Hub KtelSpeech(금융 상담 전화) 고정 표본 50세션·1,590발화·141.2분 / RTX 3080
· [PROTOCOL.md](https://github.com/PROtect-withHang/PROtect/blob/main/experiments/stt_bench/PROTOCOL.md)

| 모델 | 오타 수 | CER | RTF | 발화 지연 p95 | VRAM |
|---|---|---|---|---|---|
| small (244M) | 8,283 | 15.96% | 0.114 | 1.35s | 1.5GB |
| medium (769M) | 5,953 | 11.47% | 0.214 | 2.59s | 4.4GB |
| **large-v3-turbo (809M)** | **3,228** | **6.22%** | **0.056** | **0.59s** | 4.9GB |

| 실험 | 코드 | 결과 |
|---|---|---|
| 3모델 × 2대 교차 벤치 | [`run_bench.py`](https://github.com/PROtect-withHang/PROtect/blob/main/experiments/stt_bench/run_bench.py) | **`large-v3-turbo` 채택** — medium 대비 오타 46%↓·속도 3.8배 |
| 연속 전사 검증 | [`run_longform.py`](https://github.com/PROtect-withHang/PROtect/blob/main/experiments/stt_bench/run_longform.py) | CER 6.22% → **5.51%** 로 오히려 개선(50세션 중 44개) → VAD를 과하게 쪼개지 않는 근거 |
| 앞 문맥 참고 on/off | 〃 | `condition_on_previous_text` 켜면 삽입 오류 31%↑ → **끄기** |
| 채점·비교 | [`score.py`](https://github.com/PROtect-withHang/PROtect/blob/main/experiments/stt_bench/score.py) · [`compare.py`](https://github.com/PROtect-withHang/PROtect/blob/main/experiments/stt_bench/compare.py) | 오타 수·CER·WER·SER·RTF·메모리 집계 (재추론 없이 재채점) |

> 배포 환경에 GPU가 없어 같은 모델을 외부 추론 API(Groq)로 호출합니다 —
> [`services/stt/groq_engine.py`](https://github.com/PROtect-withHang/PROtect/blob/main/backend/app/services/stt/groq_engine.py)

### 4-2. RAG — product 트랙 (퇴직연금 상품설명서, 표가 많은 PDF)

골든셋 48문항 / 전략 B 코퍼스 117청크 · [실험 기록](https://github.com/PROtect-withHang/PROtect/blob/main/backend/app/services/rag/experiments/README.md)

| # | 실험 | 코드 | 결과 |
|---|---|---|---|
| 1 | 청킹 전략 A/B/C/D 비교 | [`chunking/`](https://github.com/PROtect-withHang/PROtect/tree/main/backend/app/services/rag/experiments/chunking) · [`run_experiment.py`](https://github.com/PROtect-withHang/PROtect/blob/main/backend/app/services/rag/experiments/run_experiment.py) | R@5 — A(표 제외) 0.167 / B(평문) 0.542 / C(마크다운) 0.521 / D(라벨:값 행) 0.708 |
| 2 | 질문 증강 (역방향 HyDE) | [`question_augmentation.py`](https://github.com/PROtect-withHang/PROtect/blob/main/backend/app/services/rag/experiments/question_augmentation.py) | R@5 0.542 → **0.875** (LLM 호출 116회) |
| 3 | **제목 prefix 대조군** | [`title_prefix.py`](https://github.com/PROtect-withHang/PROtect/blob/main/backend/app/services/rag/experiments/title_prefix.py) | 제목 한 줄만으로 **0.833** (LLM 호출 **0회**) → **질문 증강 기각** |
| 4 | 청킹 × 제목 헤더 2×2 | [`chunking_header.py`](https://github.com/PROtect-withHang/PROtect/blob/main/backend/app/services/rag/experiments/chunking_header.py) | B+헤더 = D+헤더 R@5 0.833 동률, 강점 유형만 갈림 → **B+헤더 유지** |
| 5 | dense / sparse / hybrid | [`retrieval_hybrid.py`](https://github.com/PROtect-withHang/PROtect/blob/main/backend/app/services/rag/experiments/retrieval_hybrid.py) | 이 코퍼스에선 셋 다 R@5 동률, 섞으면 오히려 하락 |

> **가장 중요했던 발견 — 질문 증강의 이득은 "예상 질문"이 아니라 "제목 주입"이었습니다.**
> "제목 + 질문 증강"이 제목 단독과 전 지표에서 동일했고, 생성된 질문 220개 중 213개가 파일명 속
> 상품명을 그대로 포함하고 있었습니다. 청크 텍스트에 문서 제목이 빠져 있던 것이 진짜 원인이었고,
> LLM 116회로 사던 개선을 문자열 한 줄로 대체했습니다.
> 서비스 반영 → [`load/build_product_chunks.py`](https://github.com/PROtect-withHang/PROtect/blob/main/backend/app/services/rag/load/build_product_chunks.py)

### 4-3. RAG — lawpolicy 트랙 (법령 1 + 약관·계약서 10종, 총 16문서)

골든셋 30문항 · 조 526 + 항 1,436개

| 라운드 | 실험 | 코드 | 결과 |
|---|---|---|---|
| 1 | PDF → 청킹 → 검색·답변 평가 8단계 파이프라인 | [`chunking_granularity_filter/`](https://github.com/PROtect-withHang/PROtect/tree/main/backend/app/services/rag/experiments/chunking_granularity_filter) | 실제 PDF 11개 검증 중 파싱 버그 **10종** 수정(2단 컬럼 순서 뒤섞임이 최악), gold coverage 30/30 |
| 2 | 질문 증강 인덱싱 | [`question_augmentation_lawpolicy/`](https://github.com/PROtect-withHang/PROtect/tree/main/backend/app/services/rag/experiments/question_augmentation_lawpolicy) | 검색 MRR 0.623 → 0.652지만 **답변 정확도 0.767 → 0.700 하락** → **기각** |
| 3 | BM25 vs Dense vs Hybrid(RRF) | [`compare_retrieval_methods.py`](https://github.com/PROtect-withHang/PROtect/blob/main/backend/app/services/rag/experiments/retrieval_methods/compare_retrieval_methods.py) | lawpolicy는 BM25(MRR 0.673), product 골든셋은 Hybrid(0.859) 우세 → **Hybrid 채택** |
| 4 | 문서명 헤더 부착 | [`header_prefix_lawpolicy.py`](https://github.com/PROtect-withHang/PROtect/blob/main/backend/app/services/rag/experiments/retrieval_methods/header_prefix_lawpolicy.py) | 제도유형 계열에서 Dense MRR **0.274 → 0.712(2.6배)** → **채택** |
| 4-b | 헤더의 답변 품질 검증 | [`header_answer_eval_lawpolicy.py`](https://github.com/PROtect-withHang/PROtect/blob/main/backend/app/services/rag/experiments/retrieval_methods/header_answer_eval_lawpolicy.py) | 답변 정확도 **70.0% → 83.3%**, 5개 지표 전부 개선·악화 없음 |

> **검색 지표로 끝내지 않았습니다** — "RAG의 목적 지표는 Recall@5가 아니라 답변"이라는 원칙으로
> Round 2·4 모두 답변 품질을 따로 측정했고, 검색이 3배 개선돼도 답변 정확도는 그만큼 오르지 않는
> 갭을 확인했습니다. DB/DC/IRP 계열이 표준계약서를 공유해 조문 본문만으로 구분되지 않는다는
> 가설을 서브셋 집계로 직접 검증한 것이 Round 4입니다.
> 서비스 반영 → [`load/build_chunks.py`](https://github.com/PROtect-withHang/PROtect/blob/main/backend/app/services/rag/load/build_chunks.py) · [`services/rag/retrieval.py`](https://github.com/PROtect-withHang/PROtect/blob/main/backend/app/services/rag/retrieval.py)

---

## 📍 5. 모듈 구조

```
연금사수/
├── frontend/                      # React 19 + Vite 8
│   ├── public/pcm-worklet.js      # AudioWorklet — 마이크 → 16kHz mono PCM
│   └── src/
│       ├── api/                   # 서버 통신 전용 (fetch/WebSocket/SSE는 여기만)
│       ├── types/ hooks/          # 응답 타입 · useSttStream(WS) · useRagQuery(SSE)
│       └── pages/                 # TellerMain · Consultation · Summary · HqMain
│
├── backend/
│   ├── API.md                     # 구현 현황표 (명세는 Swagger가 단일 진실)
│   ├── app/
│   │   ├── main.py                # 앱 생성 + 라우터 등록 + lifespan(회수·예열)
│   │   ├── core/ api/ schemas/    # 설정 · 라우터 · Pydantic 요청/응답
│   │   ├── services/
│   │   │   ├── consultation/      # 상담 후 정리 — summarize · checklists · notice
│   │   │   ├── rag/               # retrieval · bm25 · embedding · llm
│   │   │   │   ├── load/          # PDF → 청킹 → 임베딩 → Supabase 적재
│   │   │   │   └── experiments/   # 청킹·검색 실험 (서비스가 import하지 않음)
│   │   │   ├── stt/               # VAD · 화자 구분 · Groq 엔진 · 지문 · 교정
│   │   │   ├── post/ question/    # 질의 집계 · 사내 게시판
│   │   │   ├── turns.py           # 턴 경계 판정 + 키워드 파이프라인
│   │   │   └── session_store.py   # Redis 전사문·키워드 (TTL 4시간)
│   │   └── db/                    # session · models · 기능별 데이터 함수 (async)
│   ├── supabase/schema.sql        # 테이블 + 데모 시드
│   └── tests/                     # pytest 155개 (fakeredis + LLM 목업)
│
├── experiments/stt_bench/         # Whisper 모델 비교 (PROTOCOL.md · manifest.csv · runs/)
├── docker-compose.yml             # 개발 — 5173 · 8000 · Redis (핫리로드)
└── docker-compose.prod.yml        # 배포 — web(nginx) + backend + redis + tunnel
```

**구조 규칙 3가지** — "API 응답 형태는 언제든 바뀐다", "기능을 유연하게 추가/삭제한다"를 지키기 위한 제약입니다.

1. 응답 형태는 `backend/app/schemas/`에만 정의하고, 프론트는 서버 호출을 `frontend/src/api/`로만
   모읍니다. 컴포넌트/페이지에서 fetch를 직접 부르지 않습니다.
2. `services/`는 SQLAlchemy를 직접 import하지 않고 `db/`만 호출합니다. 세션은 라우터에서
   `Depends(get_session)`으로 주입받아 db 계층까지 그대로 전달합니다.
3. RAG·STT의 외부 엔진은 각각 `services/rag/`·`services/stt/` **안에만** 숨깁니다. 기능 삭제는
   추가의 역순이면 끝납니다.

---

## 📍 6. 인프라 · 실행

| 환경 | 서비스 | 포트 | 역할 |
|---|---|---|---|
| 개발 | frontend (Vite dev) | 5173 | 핫리로드 UI |
| 개발 | backend (uvicorn --reload) | 8000 | API + Swagger(`/docs`) |
| 공통 | redis | (내부) | 전사문·키워드 세션 저장, TTL 4시간 |
| 배포 | web (nginx) | 8080 → 80 | 단일 진입점 — 정적 파일 + `/api` 프록시(WebSocket·SSE) |
| 배포 | backend | (내부) | 호스트에 포트를 열지 않음 |
| 배포 | tunnel (cloudflared) | — | Cloudflare Tunnel → `https://protect-desk.cloud` |
| 외부 | Supabase (PostgreSQL + pgvector) | — | 상담·지침·게시글·청크(1024차원 HNSW) |

```bash
# 개발 — Docker Desktop만 있으면 됨 (Node·Python 로컬 설치 불필요)
docker compose up --build

# 백엔드 테스트 — Redis·LLM 없이 동작
cd backend && pip install -r requirements.txt -r requirements-dev.txt && pytest tests/

# 배포
docker compose -f docker-compose.prod.yml up -d --build
```

```
인터넷 ─HTTPS─▶ Cloudflare ─▶ tunnel ─▶ web(nginx :80) ─▶ backend :8000 ─▶ redis
                                          ├─ /        프론트 정적 파일
                                          └─ /api/*   백엔드 프록시 (WebSocket · SSE)
```

처음엔 Vercel + Render를 계획했으나 **Render 무료 플랜은 GPU가 없고 15분 유휴 시 잠들어**
실시간 상담에 맞지 않아 로컬 PC + Cloudflare Tunnel로 바꿨습니다. 진입점이 하나라 프론트와 API가
같은 주소를 쓰므로 CORS가 없고, 터널 ID에 도메인이 묶여 있어 무엇을 껐다 켜도 주소가 그대로입니다.

---

## 📍 7. 화면

<!-- 이미지 자리: 행원 메인 -->
<img width="" height="" alt="행원 메인 — 상담 기록·본부 지침·사내 게시판" src="" />

<!-- 이미지 자리: 상담 중 화면 -->
<img width="" height="" alt="상담 중 — 실시간 자막·키워드 칩·RAG 답변 카드·AI 질의" src="" />

<!-- 이미지 자리: 상담 정리 화면 -->
<img width="" height="" alt="상담 정리 — 요약 3항목·필수 고지 점검·누락 안내 문자" src="" />

<!-- 이미지 자리: 본부 메인 -->
<img width="" height="" alt="본부 메인 — 이번 주 질의 Top5·대응 지침 등록·게시글 자동 작성" src="" />
