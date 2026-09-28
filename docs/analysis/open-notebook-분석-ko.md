# Open Notebook 전수조사 분석 정리 (한국어)

> 작성일: 2026-09-28
> 대상 저장소: [bmshin94/open-notebook](https://github.com/bmshin94/open-notebook)
> 원본(upstream): [lfnovo/open-notebook](https://github.com/lfnovo/open-notebook)
> 공식 사이트: https://www.open-notebook.ai
> 분석 기준 버전: v1.14.0 (MIT License)

---

## 목차

1. [프로젝트 개요](#1-프로젝트-개요)
2. [쉬운 설명 (비유 중심)](#2-쉬운-설명-비유-중심)
3. [자주 묻는 질문 7가지](#3-자주-묻는-질문-7가지)
4. [수익화 아이디어](#4-수익화-아이디어)
5. [참고 링크 모음](#5-참고-링크-모음)

---

## 1. 프로젝트 개요

### 1-1. 한 줄 요약

Google NotebookLM을 오픈소스로 구현한 **프라이버시 중심 · 자체 호스팅형 AI 연구 비서**.

### 1-2. 아키텍처 (3-Tier)

```
브라우저 (:8502 docker / :3000 dev)
      |
[1층] Next.js 16 + React 19 프론트엔드
      |  /api/* 내부 프록시
[2층] FastAPI 백엔드 (:5055)
      |
[3층] SurrealDB (:8000)  - 그래프 DB + 벡터 검색 내장

   + surreal-commands 워커 : 팟캐스트 / 임베딩 / 소스처리 비동기 잡
```

**기동 순서 (중요)**

```bash
make database      # 1. SurrealDB
make api           # 2. FastAPI (스키마 마이그레이션 자동 실행)
make worker-start  # 3. 워커 (없으면 백그라운드 작업이 영원히 대기)
make frontend      # 4. Next.js
# 또는
make start-all / make status / make stop-all
```

### 1-3. 규모

| 항목 | 수치 |
|---|---|
| Python 코드 | 약 33,000줄 |
| TypeScript/TSX | 약 43,000줄 |
| 문서(md) | 75개 |
| API 라우터 | 21개 |
| pytest 테스트 파일 | 58개 |
| 지원 UI 언어 | 14개 (한국어 **미지원**) |

### 1-4. 폴더 구조

| 경로 | 설명 |
|---|---|
| `api/` | FastAPI 서버. `main.py`에 21개 라우터 등록 |
| `api/routers/sources.py` | 최대 파일(44KB). PDF/URL/영상/오디오 수집 |
| `api/credentials_service.py` | API 키 암호화 저장 (36KB) |
| `api/auth.py` | 단일 비밀번호 Bearer 미들웨어 (타이밍 공격 방어 포함) |
| `open_notebook/graphs/` | LangGraph 워크플로우 5종: `chat`, `ask`, `source`, `transformation`, `source_chat` |
| `open_notebook/ai/` | 18개+ AI 제공자 추상화 (esperanto 기반) |
| `open_notebook/domain/` | DDD 도메인 모델 |
| `open_notebook/database/` | SurrealQL 마이그레이션 (API 기동 시 자동 적용) |
| `commands/` | 백그라운드 잡: podcast / embedding / source |
| `prompts/` | Jinja 프롬프트 템플릿 9개 |
| `frontend/src/` | Next.js App Router + Zustand + TanStack Query + Tailwind v4 + shadcn/ui |
| `tests/` | 보안 테스트(TOCTOU, path containment, SSRF) 포함 58개 파일 |
| `docs/` | `0-START-HERE` ~ `7-DEVELOPMENT` 번호 체계 문서 |
| `.claude/`, `.agents/`, `.codex/` | AI 에이전트용 스킬/에이전트 정의 |

### 1-5. 핵심 기능

1. **Notebook / Source / Note 3단 구조** — 프로젝트 / 입력물 / 산출물
2. **Chat vs Ask**
   - Chat: 선택한 소스 **전문**을 LLM에 전달 (`graphs/chat.py`)
   - Ask: **RAG** — 검색전략 수립 → 최대 5개 병렬 검색 → 최종 합성 (`graphs/ask.py`)
3. **Transformations** — 커스텀 프롬프트로 요약/인사이트 추출
   - 보안: 사용자 프롬프트를 Jinja 템플릿 *소스*로 컴파일하지 않음 (GHSA-f35w-wx37-26q7 대응)
4. **팟캐스트 생성** — 1~4명 화자, Episode/Speaker Profile 커스텀 (NotebookLM은 2명 고정)
5. **하이브리드 검색** — SurrealDB 내장 풀텍스트 + 벡터(코사인 유사도)

### 1-6. AI 제공자 (18개+)

OpenAI, Anthropic, Google GenAI, Vertex AI, Groq, Ollama, oMLX, Perplexity, ElevenLabs,
Deepgram, Azure OpenAI, Mistral, DeepSeek, Cohere, Voyage, xAI, OpenRouter,
DashScope(Qwen), MiniMax, Novita, PayPerQ, OpenAI-Compatible(LM Studio 등)

> `esperanto` 라이브러리로 추상화되어 **코드 수정 없이 UI에서 교체** 가능.
> Ollama/LM Studio 사용 시 **API 비용 0원 + 데이터 100% 로컬**.

### 1-7. NotebookLM 과의 비교

| 항목 | Open Notebook | Google NotebookLM |
|---|---|---|
| 데이터 위치 | 내 서버 | Google 클라우드 |
| AI 제공자 | 18개+ | Google 모델만 |
| 팟캐스트 화자 | 1~4명 커스텀 | 2명 고정 |
| API | 전체 REST 공개 | 없음 |
| 배포 | Docker/클라우드/로컬 | Google 호스팅만 |
| 비용 | AI 사용량만 | 무료티어 + 구독 |
| 인용(Citation) | 기본 수준 | 더 정교함 |

---

## 2. 쉬운 설명 (비유 중심)

### 2-1. 요리 비유

| 요리 | Open Notebook |
|---|---|
| 장바구니 | **Notebook** (프로젝트 폴더) |
| 재료 | **Source** (PDF, 유튜브, 웹페이지) |
| 손질 | **Transformation** (요약, 핵심추출) |
| 완성 요리 | **Note** (내 인사이트) |
| 요리사 | **AI 모델** (교체 가능) |
| "이걸로 뭐 만들지?" | **Chat / Ask** |
| 요리 방송 | **팟캐스트 생성** |

### 2-2. Chat vs Ask 쉽게

책 100권이 있다고 가정하면,

- **Chat** = "이 3권만 읽고 얘기하자" → 3권 통째로 AI에게 전달
  - 맥락 완벽 / 비싸고 많이 못 넣음
- **Ask** = "100권 중 관련 페이지만 찾아서 답해줘" → AI가 알아서 검색(RAG)
  - 자료 많아도 OK, 저렴 / 가끔 놓침

### 2-3. 벡터 검색이란

"강아지"로 검색했을 때 "반려견", "멍멍이" 문서도 찾아주는 것.
글자가 아니라 **의미**로 찾는 검색.

### 2-4. 활용 시나리오

1. 유튜브 강의 10개 + PDF 5개를 노트북에 투입
2. 자동으로 텍스트 추출 → 청킹 → 임베딩 (워커가 처리)
3. "React 성능 최적화 부분만 정리해줘" → **Ask**
4. 답변을 **Note**로 저장
5. 이동 중 듣고 싶으면 **팟캐스트** 변환
6. Claude Desktop에서 **MCP**로 내 노트북 검색

---

## 3. 자주 묻는 질문 7가지

### Q1. 설치 및 사용법

**방법 A — Docker (권장, 약 2분)**

```bash
curl -o docker-compose.yml \
  https://raw.githubusercontent.com/lfnovo/open-notebook/main/docker-compose.yml
# docker-compose.yml 에서 OPEN_NOTEBOOK_ENCRYPTION_KEY 를 비밀 문자열로 변경 (필수)
docker compose up -d
# 15~20초 후 http://localhost:8502
```

초기 설정: **Models → 제공자 선택 → + Add Configuration → API 키 입력 →
Test → Sync Models → Auto-Assign Defaults**

**방법 B — 소스 빌드 (개발용)**

```bash
git clone https://github.com/bmshin94/open-notebook
cd open-notebook
uv sync
cd frontend && npm install && cd ..
make start-all
```

**포트**: 프론트 3000(dev)/8502(docker) · API 5055 · DB 8000
**API 문서**: http://localhost:5055/docs (Swagger 자동 생성)
**무료 운영**: `examples/docker-compose-ollama.yml` (Ollama, API 비용 0원)

---

### Q2. 플러그인 / 스킬 / MCP 중 무엇인가?

**셋 다 아님. 독립 실행형 웹 애플리케이션(제품)이다.**

| 구분 | 여부 |
|---|---|
| 플러그인 | 아님 (그 자체가 앱) |
| Claude Skill | 아님 (단, 저장소 내 `.claude/skills/`에 개발용 스킬 존재) |
| MCP | **본체는 아니지만, 별도 MCP 서버가 존재** |

```
[Claude Desktop] --MCP--> [open-notebook-mcp] --REST--> [Open Notebook 본체]
```

- MCP 서버 저장소: https://github.com/Epochal-dev/open-notebook-mcp
- PyPI: https://pypi.org/project/open-notebook-mcp
- MCP 레지스트리: https://registry.modelcontextprotocol.io ("open-notebook" 검색)

설정 예시 (`claude_desktop_config.json`):

```json
{
  "mcpServers": {
    "open-notebook": {
      "command": "uvx",
      "args": ["open-notebook-mcp"],
      "env": {
        "OPEN_NOTEBOOK_URL": "http://localhost:5055",
        "OPEN_NOTEBOOK_PASSWORD": "your_password_here"
      }
    }
  }
}
```

MCP 제공 기능: 노트북 CRUD, 소스 추가/조회, 노트 생성, 채팅 세션, 벡터/텍스트 검색, 모델 설정.

---

### Q3. API 토큰이 필요한가?

두 종류의 "토큰"을 구분해야 한다.

**(1) AI 제공자 API 키 — 사실상 필수 (단, 우회 가능)**

| 옵션 | 비용 |
|---|---|
| OpenAI / Anthropic / Google | 종량제 |
| Groq | 무료 티어 있음 |
| **Ollama / LM Studio / oMLX** | **완전 무료, 키 불필요** |

저장 방식: UI(Manage → Models) 입력 → **암호화되어 DB 저장**.
암호화 키는 `OPEN_NOTEBOOK_ENCRYPTION_KEY` 환경변수(필수).
`.env`에 직접 키를 넣는 방식은 **deprecated**.

**(2) Open Notebook 접근 비밀번호 — 선택**

```bash
OPEN_NOTEBOOK_PASSWORD=secret
```

- 미설정 시 **인증 자체가 비활성화**
- 설정 시 모든 요청에 `Authorization: Bearer <비밀번호>` 필요
- `secrets.compare_digest`로 타이밍 공격 방어

> **주의**: `AGENTS.md`에 명시된 대로 CORS는 전면 개방이고 인증은 단일 비밀번호다.
> 이는 **개발용 기본값이지 프로덕션 하드닝이 아니다.** 외부 공개 시 리버스 프록시 +
> HTTPS + 추가 인증이 필요하다.

---

### Q4. 왜 GitHub에서 유명한가?

1. **타이밍** — NotebookLM이 카테고리를 대중화한 직후 등장
2. **프라이버시** — "기밀문서를 Google에 올릴 수 없다"는 니즈 직격
3. **벤더 락인 탈출** — 18개+ 제공자 지원
4. **팟캐스트가 원조보다 우수** — 1~4명 화자 + 스크립트 제어
5. **문서/운영 성숙도** — 문서 75개, README 8개 언어 번역, VISION.md,
   ADR/PDR 의사결정 기록 체계, Discord + GitHub Discussions
6. **외부 노출** — Trendshift 배지(repositories/14536), 전용 웹사이트,
   메인테이너의 X 홍보, 설치 도우미 CustomGPT
7. **최신 기술 스택** — FastAPI + Next.js 16 + React 19 + LangGraph + SurrealDB
8. **MIT 라이선스** — 상업적 이용 자유

> 스타 수 등 정량 지표는 원본 저장소 조회 권한이 없어 미확인. 위 분석은 저장소
> 내부의 배지 · 문서 · 커뮤니티 흔적 기반 추론이다.

---

### Q5. 로컬 에이전트 구축에 도움이 되는가?

**세 가지 레벨로 도움이 된다.**

**레벨 1 — 에이전트의 "장기 기억"으로 사용 (가장 실용적)**

```
[내 에이전트] --MCP--> [Open Notebook] = 영구 메모리 + RAG 엔진
```

- 문서 수집 → 파싱 → 청킹 → 임베딩 자동
- 벡터 + 풀텍스트 하이브리드 검색 내장
- REST API 전면 개방, MCP 서버 이미 존재

> `VISION.md`의 Horizon에 **"Agents operating Open Notebook"** 클러스터가 공식
> 방향으로 명시되어 있다 (이슈 #878, #693, #973).

**레벨 2 — 아키텍처 레퍼런스**

| 패턴 | 참고 파일 |
|---|---|
| LangGraph 상태머신 | `open_notebook/graphs/*.py` |
| RAG (전략→병렬검색→합성) | `graphs/ask.py` |
| 멀티 프로바이더 추상화 | `open_notebook/ai/provider_registry.py` |
| 비동기 잡 큐 | `commands/` + surreal-commands |
| 프롬프트 템플릿 관리 | `prompts/*.jinja` + ai-prompter |
| 자격증명 암호화 | `utils/encryption.py` |
| 에러 분류/재시도 | `utils/error_classifier.py` |

**레벨 3 — 포크해서 에이전트 백엔드로 개조** (MIT라 가능)

**한계**

| 한계 | 설명 |
|---|---|
| 단일 사용자 | 멀티유저는 검토 단계 (#712) |
| 툴 실행 없음 | 행동하는 에이전트 프레임워크가 아니라 **지식 저장소** |
| 인증 약함 | 비밀번호 1개 |
| 무거움 | SurrealDB + 워커 + 프론트 = 최소 3프로세스 |

---

### Q6. 수익화 아이디어가 있는가?

있다. 상세 내용은 [4장](#4-수익화-아이디어) 참고.
요약: 매니지드 호스팅 / 한국어 현지화 / 버티컬 특화 / 팟캐스트 단독 SaaS /
엔터프라이즈 애드온 / 에이전트 메모리 인프라 / 교육·컨설팅.

---

### Q7. React나 PHP로 만들 수 있는가?

**React — 이미 React다. 100% 가능.**

프론트엔드가 Next.js 16 + React 19 + TypeScript.
백엔드는 그대로 두고 `frontend/`만 교체하거나, Vite+React / React Native로
새로 작성해도 된다.

```
[내 React 앱] --REST--> [Open Notebook API :5055] --> [SurrealDB]
```

**PHP — 3가지 레벨**

| 레벨 | 방식 | 난이도 | 추천도 |
|---|---|---|---|
| 1 | PHP를 **클라이언트**로 (Laravel + Guzzle로 REST 호출) | ★☆☆☆☆ | ★★★★★ |
| 2 | PHP로 **백엔드 전체 재작성** | ★★★★★ | ★☆☆☆☆ |
| 3 | **하이브리드** (PHP=웹/결제/권한, Python=AI엔진) | ★★☆☆☆ | ★★★★★ |

레벨 2가 비추천인 이유 — Python 생태계 의존이 깊다:

| 필요 기능 | Python | PHP |
|---|---|---|
| LangGraph | 공식 지원 | 없음 |
| Esperanto (18 제공자) | 있음 | 각 SDK 개별 구현 |
| content-core (PDF/영상 파싱) | 있음 | 제한적 |
| podcast-creator | 있음 | 없음 |
| SurrealDB 드라이버 | 공식 | 커뮤니티 |
| 비동기 워커 | asyncio | Swoole/Queue 필요 |

Laravel 클라이언트 예시:

```php
$response = Http::withToken(env('ON_PASSWORD'))
    ->post('http://localhost:5055/api/search', [
        'query' => '리액트 성능 최적화',
        'search_type' => 'vector',
    ]);
```

**최종 권장**

| 목표 | 방법 |
|---|---|
| 빠르게 내 서비스 출시 | 프론트만 React로 재작성 (백엔드 재사용) |
| PHP 팀 | Laravel + Open Notebook API 하이브리드 |
| 학습 목적 | `graphs/`, `api/routers/` 정독 |
| 전체 재작성 | 비권장 |

---

## 4. 수익화 아이디어

> **라이선스 전제**: MIT이므로 상업적 이용 · 수정 · 비공개 재배포가 모두 합법.
> 조건은 **저작권 표시와 라이선스 사본 포함** 하나뿐이다.
> 다만 상표(이름/로고)는 자체 브랜드로 대체하는 것이 안전하다.

### 4-1. 매니지드 호스팅 SaaS

Docker/서버/API키 설정이 어려운 사용자를 위한 클릭 한 번 인스턴스 제공.

| 플랜 | 가격 | 내용 |
|---|---|---|
| Free | 0원 | 노트북 1개, 소스 20개, 월 팟캐스트 1회 |
| Pro | 월 15,000원 | 무제한 노트북, 소스 500개, 팟캐스트 10회 |
| Team | 월 50,000원/인 | 공유 노트북, SSO, 우선 지원 |
| Private Cloud | 월 50만원~ | 전용 인스턴스, 커스텀 도메인 |

- 난이도 ★★★☆☆ / 잠재력 ★★★★☆
- 과제: 멀티테넌시(원본은 단일 사용자) → **고객당 컨테이너 격리**로 우회,
  오히려 프라이버시 셀링포인트로 활용
- 리스크: AI API 비용이 마진 잠식 → **BYOK(내 키 가져오기) 옵션 필수**

### 4-2. 한국어 완전 현지화 버전 (진입 기회)

`frontend/src/lib/locales/`에 14개 언어가 있으나 **한국어(ko-KR)가 없다.**
(en, pt-BR, zh-CN, zh-TW, ja, ru, bn-IN, de, es, fr, it, pl, tr, ca)

**A. 오픈소스 기여 경로** — 한국어 번역 PR → 공식 컨트리뷰터 → 포트폴리오/인지도

**B. 한국 특화 상용 버전**

| 기능 | 근거 |
|---|---|
| 한국어 UI/프롬프트 최적화 | 기본 |
| HWP(한글) 파일 지원 | 한국 전용 수요 = 원본이 만들지 않음 |
| 공공데이터/국가법령정보 커넥터 | 공공기관 수요 |
| 토스/카카오페이 결제 | 국내 결제 |
| 네이버 클로바/카카오 AI 연동 | 국내 제공자 |
| 한국어 TTS 최적화 팟캐스트 | 품질 차별화 |

- 가격: 개인 월 9,900원 / 팀 월 39,000원 / 공공·대학 연 300만~1,000만원
- 난이도 ★★☆☆☆ / 잠재력 ★★★★☆

### 4-3. 버티컬 특화 SaaS

| 버티컬 | 차별 기능 | 가격 |
|---|---|---|
| **학술** | arXiv/PubMed 수집, BibTeX/Zotero 연동, 논문 간 모순점 탐지, 문헌리뷰 초안 | 학생 월 5,900원 / 연구실 월 10만원 |
| **법률** | 판례·계약서 파서, **온프레미스 필수**, 리스크 조항 탐지 | 변호사 월 15만원 / 로펌 연 2,000만원~ |
| **의료** | 개인정보보호법·HIPAA 대응(100% 로컬), 가이드라인 요약 | 병원 단위 계약 |
| **기업 지식관리** | 위키/노션/컨플루언스 커넥터, 온보딩 봇 | 100인 기업 연 1,000만원~ |

- 난이도 ★★★★☆ / 잠재력 ★★★★★ (법률이 단가 최고)

### 4-4. 팟캐스트 생성기 단독 서비스

팟캐스트 엔진만 분리한 단일 목적 SaaS.

```
문서/URL 업로드 → 1~4명 화자 팟캐스트 → MP3 다운로드
```

- 타겟: 콘텐츠 크리에이터, 마케터, 사내교육 담당, 뉴스레터 운영자, 접근성 서비스
- 가격: 무료 3회 → 10회 9,900원 → 50회 39,000원 → 무제한 월 99,000원 / API 건당 500원
- 장점: 범위가 좁아 **MVP 2~4주**, 결과물이 청각적이라 바이럴 유리
- 난이도 ★★☆☆☆ / 잠재력 ★★★★☆ → **빠른 수익화 1순위**

### 4-5. 엔터프라이즈 애드온 (Open-Core)

| 애드온 | 근거 | 가격 |
|---|---|---|
| 멀티유저 + RBAC | 원본은 단일 사용자 (#712 검토 단계) | 연 500만원 |
| SSO (SAML/OIDC) | 기업 필수 | 연 300만원 |
| 감사 로그 | 컴플라이언스 | 연 200만원 |
| 사용량/비용 대시보드 | AI 비용 관리 | 연 150만원 |
| 커넥터 팩 (Slack/Notion/Jira/Drive) | 워크플로우 통합 | 연 300만원 |
| SLA 지원 | 장애 대응 | 연 1,000만원 |

- 난이도 ★★★★☆ / 잠재력 ★★★★★
- 핵심: 멀티유저가 아직 **공백** 상태라는 점이 기회

### 4-6. AI 에이전트 메모리 인프라

```
개발자의 AI 에이전트 --MCP/API--> [내 서비스] = 관리형 리서치 메모리
```

- 가격: 문서 1,000개당 월 $10 / 검색 1,000건당 $1 / 임베딩 GB당 $5
- 근거: `VISION.md` Horizon의 "Agents operating Open Notebook" (#878, #693, #973)
- 난이도 ★★★★★ / 잠재력 ★★★★★ (경쟁: Mem0, Zep, Letta 등)

### 4-7. 교육 · 컨설팅 · 콘텐츠

- 기업 온프레미스 구축 대행: 건당 300만~1,000만원
- 커스터마이징 개발: 시간당 15만원
- 강의/전자책: "프라이빗 AI 리서치 시스템 구축" 49,000원
- 유튜브/블로그 → 제휴/광고/스폰서
- 난이도 ★☆☆☆☆ / 잠재력 ★★★☆☆ (초기 자본 0원, 마케팅 채널로도 활용)

### 4-8. 추천 로드맵

**개인 · 빠른 수익화**

```
1개월: 팟캐스트 생성기 단독 SaaS MVP 출시      (4-4)
2개월: 한국어 번역 PR 기여 → 컨트리뷰터 확보    (4-2 A)
3개월: 유튜브/블로그로 트래픽 확보              (4-7)
6개월: 한국 특화 상용 버전으로 확장             (4-2 B)
```

**팀 · B2B**

```
1단계: 버티컬 선택 (법률 = 단가 최고)          (4-3)
2단계: 온프레미스 구축 컨설팅으로 현금흐름 확보  (4-7)
3단계: 멀티유저/SSO 애드온 제품화               (4-5)
```

**수익성 vs 난이도**

```
수익성
  ↑
높 | 애드온(4-5)          법률버티컬(4-3)
   | 에이전트인프라(4-6)
중 | 팟캐스트(4-4)  한국어(4-2)   호스팅(4-1)
   |
낮 | 교육(4-7)
   +--------------------------------→ 난이도
     쉬움                    어려움
```

가성비 스윗스팟: **팟캐스트(4-4) + 한국어(4-2)**

### 4-9. 주의사항

1. MIT라서 경쟁자도 동일하게 할 수 있다 → 해자는 기술이 아니라 **브랜드·유통·고객관계**
2. AI API 비용 관리 필수 → **BYOK 옵션** 없으면 마진 증발
3. 원본 프로젝트가 경쟁자가 될 수 있다 → **그들이 하지 않을 영역**(한국 특화, 특정 산업)에 집중
4. 프라이버시 마케팅은 지키지 못하면 독 → 실제 로컬/온프레미스로 구현
5. 저작권 고지 의무 준수 (LICENSE 포함, 크레딧 표기)
6. B2B 영업 사이클은 6개월~1년 → 현금흐름 대비 필요

---

## 5. 참고 링크 모음

### 저장소

| 항목 | URL |
|---|---|
| **이 저장소 (포크)** | https://github.com/bmshin94/open-notebook |
| **원본 (upstream)** | https://github.com/lfnovo/open-notebook |
| MCP 서버 | https://github.com/Epochal-dev/open-notebook-mcp |
| Esperanto (AI 제공자 추상화) | https://github.com/lfnovo/esperanto |

### 공식 채널

| 항목 | URL |
|---|---|
| 웹사이트 | https://www.open-notebook.ai |
| Discord | https://discord.gg/37XJPXfz2w |
| GitHub Issues | https://github.com/lfnovo/open-notebook/issues |
| GitHub Discussions | https://github.com/lfnovo/open-notebook/discussions |
| 메인테이너 X | https://x.com/lfnovo |
| 설치 도우미 CustomGPT | https://chatgpt.com/g/g-68776e2765b48191bd1bae3f30212631-open-notebook-installation-assistant |

### 배포 / 패키지

| 항목 | URL |
|---|---|
| Docker Hub | `lfnovo/open_notebook:v1-latest` |
| GHCR | `ghcr.io/lfnovo/open-notebook` |
| MCP PyPI | https://pypi.org/project/open-notebook-mcp |
| MCP 레지스트리 | https://registry.modelcontextprotocol.io |

### 저장소 내부 문서

| 항목 | 경로 |
|---|---|
| 시작하기 | `docs/0-START-HERE/index.md` |
| 설치 가이드 | `docs/1-INSTALLATION/index.md` |
| 핵심 개념 | `docs/2-CORE-CONCEPTS/index.md` |
| 사용자 가이드 | `docs/3-USER-GUIDE/index.md` |
| AI 제공자 | `docs/4-AI-PROVIDERS/index.md` |
| MCP 연동 | `docs/5-CONFIGURATION/mcp-integration.md` |
| 보안 설정 | `docs/5-CONFIGURATION/security.md` |
| 트러블슈팅 | `docs/6-TROUBLESHOOTING/quick-fixes.md` |
| 아키텍처 | `docs/7-DEVELOPMENT/architecture.md` |
| 변경 레시피 | `docs/7-DEVELOPMENT/change-playbooks.md` |
| API 레퍼런스 | `docs/7-DEVELOPMENT/api-reference.md` |
| 의사결정 기록 | `docs/7-DEVELOPMENT/decisions/` |
| 제품 비전 | `VISION.md` |
| 에이전트 규칙 | `AGENTS.md` |

### 로컬 실행 시 접속 주소

| 서비스 | 주소 |
|---|---|
| 프론트엔드 (Docker) | http://localhost:8502 |
| 프론트엔드 (dev) | http://localhost:3000 |
| API | http://localhost:5055 |
| **API 문서 (Swagger)** | http://localhost:5055/docs |
| SurrealDB | http://localhost:8000 |

---

*이 문서는 저장소 전체를 조사한 뒤 한국어로 정리한 분석 노트입니다.*
