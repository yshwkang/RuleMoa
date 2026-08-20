# 강남대 규정집 챗봇 아키텍처

전체적으로는 **크롤링 → 구조화 저장 → RAG(검색증강생성) 기반 챗봇** 파이프라인입니다. 각 단계별 기술 스택을 정리합니다.

## 1. 크롤링 (Crawling)

사이트 특성 파악이 먼저입니다. 대학 규정집은 대개 게시판형 정적 HTML(첨부 HWP/PDF를 뷰어로 보여주는 경우도 많음)이라 JS 렌더링이 필요 없을 가능성이 높습니다.

- **정적 HTML**: Python `requests` + `BeautifulSoup4` (가장 간단, 추천 시작점)
- **JS 렌더링이 필요한 경우**: `Playwright` (Selenium보다 최신, 속도/안정성 우위)
- **PDF/HWP 첨부파일이 실제 본문인 경우**: `pdfplumber`/`PyMuPDF` (PDF), `pyhwp` 또는 변환 서비스 (HWP) — 실무에서 이 부분이 의외로 까다로울 수 있음
- **구조 보존이 핵심**: 규정집은 장(章)-절(節)-조(條)-항(項) 계층 구조를 가지므로, 단순 텍스트 추출이 아니라 **이 계층을 유지하며 파싱**해야 나중에 검색 정확도와 "제 O조 O항" 같은 출처 인용이 가능해짐
- **스케줄링**: 규정은 자주 안 바뀌므로 `APScheduler` 또는 작업 스케줄러로 주 1회 정도면 충분. 대규모가 아니면 Airflow 같은 오케스트레이션은 과함
- **변경 감지**: 문서별 해시값을 저장해두고 달라진 것만 재처리 → 불필요한 재임베딩 비용 절감

## 2. DB 저장

**PostgreSQL + pgvector** 조합을 추천. 별도 벡터DB 인프라(Milvus, Weaviate 등)를 운영할 규모가 아니라면, 하나의 DB로 관계형 데이터와 벡터 검색을 동시에 처리하는 게 운영 부담이 훨씬 적음.

- **호스팅: Supabase 무료 티어**로 확정 (Postgres + pgvector 네이티브 지원, 만료 없는 무료 플랜). 자세한 이유는 [5. 배포 아키텍처](#5-배포-아키텍처) 참고.

- **원문/메타데이터 테이블**: 규정명, 조항 번호, 원문 텍스트, 최종 개정일, 원본 URL, 크롤링 시각
- **청킹(Chunking)**: 조 단위(또는 항 단위)로 분할 — 임의의 글자 수로 자르면 문맥이 깨져 검색 품질이 떨어짐
- **임베딩**: Gemini Embedding API 무료 티어로 확정 (자세한 이유는 [3. 챗봇 (RAG)](#3-챗봇-rag) 참고)
- **벡터 컬럼**: pgvector extension으로 임베딩 저장, 코사인 유사도로 검색

## 3. 챗봇 (RAG)

```
질문 → 임베딩 → pgvector 유사도 검색(top-k 조항) → LLM에 컨텍스트로 전달 → 답변 생성
```

- **LLM/임베딩: Google Gemini API 무료 티어**로 확정 (비용 발생 방지가 최우선 요구사항)
  - 생성: Gemini 2.5 Flash / Flash-Lite
  - 임베딩: Gemini Embedding API
  - 임베딩까지 같은 공급자로 통일해 별도 유료 임베딩(Voyage AI 등) 비용 발생 여지를 없앰
- **무료 사용량 초과 시 정책: 서비스 중단** — 유료 전환이나 타 공급자(Groq 등) 폴백 없이, 한도 초과 시 챗봇 응답을 중단하고 사용자에게 안내 메시지 표시
  - 백엔드에서 Gemini API의 429(rate limit/quota exceeded) 응답을 감지해 "일일/분당 무료 사용량을 초과했습니다. 잠시 후 다시 시도해주세요" 같은 메시지를 반환하도록 구현
  - 재시도·유료 폴백 로직은 넣지 않음 (의도적으로 비용이 절대 발생하지 않게 설계)
- **프레임워크**: LangChain/LlamaIndex는 선택 사항. 이 정도 규모(단일 도메인, 단순 RAG)면 Gemini SDK로 직접 구현하는 게 오히려 커스터마이징이 쉽고 디버깅도 편함
- **프롬프트 설계의 핵심**:
  - "검색된 조항 내용에 근거해서만 답변하고, 반드시 조항 번호를 인용할 것"
  - 근거가 없으면 "관련 규정을 찾을 수 없다"고 답하도록 지시 (hallucination 방지가 규정 챗봇에서는 특히 중요)
- **출처 표시**: 답변에 "학칙 제O조" 식으로 원문 링크까지 같이 보여주면 학생이 직접 검증 가능 → 신뢰도 확보

## 4. 웹앱 (백엔드/프론트엔드)

| 계층 | 추천 | 이유 |
|---|---|---|
| 백엔드 API | FastAPI (Python) | 크롤러와 언어 통일, 비동기 처리 용이, `/chat`, `/admin/crawl` 등 엔드포인트 구성 |
| 프론트엔드 | Next.js (React) | 정식 서비스용. 빠른 프로토타입만 필요하면 Streamlit도 대안 |
| 배포 | Vercel(프론트) + Render(백엔드/크롤러) | 상세는 [5. 배포 아키텍처](#5-배포-아키텍처) 참고 |

## 5. 배포 아키텍처

배포 플랫폼별 실행 모델 차이를 고려해 역할을 분리한다.

| 계층 | 배포 위치 | 비고 |
|---|---|---|
| 프론트엔드 (Next.js) | **Vercel** | Vercel이 가장 잘하는 영역 그대로 유지 |
| 챗봇 API (FastAPI) | **Render** (Web Service) | 상시 구동으로 DB 커넥션 풀을 안정적으로 유지 |
| 크롤러 | **Render** (Cron Job) | 325개 규정 순회 + 요청 간 딜레이 → 장시간 작업, 서버리스 타임아웃에 부적합 |
| DB (PostgreSQL + pgvector) | **Supabase 무료 티어** | pgvector 네이티브 지원, 무료 플랜 만료 없음 |

**FastAPI를 Vercel이 아니라 Render에 두는 이유**: Next.js API 라우트에서 바로 Postgres에 연결하는 방법도 있지만, Vercel 서버리스 함수는 요청마다 인스턴스가 뜨기 때문에 DB 커넥션이 급증해 커넥션 풀이 고갈되는 문제가 흔함. Render의 상시 구동 FastAPI는 SQLAlchemy 커넥션 풀을 안정적으로 유지할 수 있어 이 문제를 피할 수 있음. Next.js는 이 FastAPI를 HTTPS로 호출하는 클라이언트 역할만 담당.

**DB를 Supabase로 정한 이유**: Render 무료 PostgreSQL은 일정 기간 후 만료되는 정책이 있어 장기 운영에 부적합. Supabase는 pgvector를 네이티브로 지원하고 무료 티어가 만료되지 않아 학생 프로젝트 규모의 장기 운영에 적합. 백엔드(Render)에서는 Supabase 연결 문자열로 접속.

**크롤러**: Render Cron Job으로 주기 실행. 아웃바운드 HTTP 제약 없어 `app.kangnam.ac.kr` 호출에 문제 없음. 크롤링한 데이터는 Supabase Postgres에 직접 upsert.

## 요약 스택

```
크롤링:  Python (requests/BeautifulSoup) → Render Cron Job
저장:    PostgreSQL + pgvector → Supabase 무료 티어
임베딩:  Gemini Embedding API (무료 티어)
LLM:     Gemini API 무료 티어 (2.5 Flash / Flash-Lite)
백엔드:  FastAPI → Render Web Service
프론트:  Next.js → Vercel

정책:   무료 사용량 초과 시 유료 전환/폴백 없이 서비스 응답 중단
```

## 다음 단계

가장 먼저 확인해야 할 것은 실제 규정집 사이트 URL의 HTML 구조입니다. URL이 확보되면 정적/동적 여부와 조항 구조를 확인하여 크롤러 프로토타입부터 구현합니다.

---

## 대상 사이트 구현 가능성 조사 결과

대상 URL: `https://app.kangnam.ac.kr/knumis/mo_open/index.jsp`

**결론: 구현 가능성 매우 높음.** 오래된 프레임 기반 JSP 시스템(EUC-KR 인코딩)이지만, 실제 데이터는 명확한 POST 엔드포인트로 분리되어 있어 크롤링이 수월함.

### 사이트 구조

`index.jsp` → 프레임셋(`rules.jsp` → `rulesC.jsp` + `rulesL1.jsp`/`rulesL2.jsp` 등 중첩 프레임) 구조. 최상위 검색 폼(`rulesC.jsp`)이 로드 시 자동으로 목록 조회 요청을 트리거하는 구조이며, 실제로는 아래 두 엔드포인트만 직접 호출하면 됨.

**1) 전체 규정 목록 조회**
```
POST https://app.kangnam.ac.kr/knumis/mo_open/rulesL1.jsp
body: gubn=null&save_gubn=&rglt_cont=&rglt_cont2=
```
- 한 번의 요청으로 **전체 325개 규정** 목록 반환
- 각 항목에 `(편, 장, 호)` 식별자 포함 (예: `1,1,1` = 제1편-제1장-1호 = 학교법인강남학원정관)
- 규정명 검색(`rglt_cont2`), 본문 검색(`rglt_cont`) 파라미터도 지원

**2) 개별 규정 전문 조회**
```
POST https://app.kangnam.ac.kr/knumis/mo_open/rulesL2.jsp
body: gubn=null&save_gubn=2&rglt_pyon={편}&rglt_jang={장}&rglt_bnho={호}&rglt_cont=&rglt_alls=Y
```
- `rglt_alls=Y` 파라미터로 해당 규정의 **전체 조항을 한 번에** 반환
- 편/장/절/조 계층이 HTML 구조로 명확히 구분됨: 장/절 제목은 `<font size="4"/"3">`, 조항은 `<tr id="rowN">` + `<font style='font-weight:bold'>제N조</font>` + 본문
- 실제 검증: "학교법인강남학원정관" 전문을 이 방식으로 수집, 제1조부터 조항별로 정확히 파싱됨
- 최종 개정일자도 함께 포함 (예: `[2026.07.27 개정]`)

### 크롤링이 수월한 이유

- **인증/세션 불필요**: 쿠키는 트래킹용(`WMONID`)만 존재, 로그인 없이 접근 가능 (`mo_open` = 공개 경로로 추정)
- **JS 렌더링 불필요**: Playwright 없이 Python `requests`만으로 충분
- **요청량이 가볍다**: 325개 규정 × 요청 1회 = 총 325회 POST로 전체 규정집 수집 가능
- **robots.txt 없음** (404) — 명시적 제한은 없으나 요청 간 딜레이(0.5~1초 권장)로 예의 유지

### 주의사항

- **EUC-KR 인코딩**: `response.encoding = 'euc-kr'` 명시적 지정 필요 (자동 감지 시 깨짐)
- **별표/별지 첨부파일**: 서식류는 `rules_download.jsp`로 별도 다운로드(한글/PDF) — 챗봇 핵심 답변엔 우선순위 낮음, 필요시 후속 작업으로 분리
- **개정이력**: `rule_hist.jsp` 팝업으로 별도 제공 — 기본 크롤링 범위에서 제외 가능

### 크롤러 구현 방향 (확정)

```
1. rulesL1.jsp POST → 325개 규정의 (편,장,호) 식별자 + 규정명 목록 확보
2. 각 (편,장,호)에 대해 rulesL2.jsp POST (rglt_alls=Y) → 전문 텍스트 수집
3. EUC-KR 디코딩 후 BeautifulSoup으로 파싱
   - 장/절 제목: <font size="4">, <font size="3">
   - 조항: <tr id="rowN"> 단위로 조번호/제목/본문 분리
4. 요청 간 0.5~1초 딜레이
5. DB에 규정명, 편/장/절/조 계층, 조항 텍스트, 최종개정일, 원본 식별자(pyon/jang/bnho) 저장
```
