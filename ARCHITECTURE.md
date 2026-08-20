# 아키텍처: 강남대 규정집 챗봇 (RuleMoa)

> 개발 착수 전 전체 시스템 구조를 정리한 문서. 기술 스택/배포 결정은 [PLANNING.md](PLANNING.md), 기능 요구사항은 [PRD.md](PRD.md) 참고.

## 1. 전체 시스템 구성도

```mermaid
graph TB
    subgraph EXT["외부"]
        KNU["강남대 규정집 사이트<br/>app.kangnam.ac.kr"]
        GEMINI["Google Gemini API<br/>(생성 + 임베딩, 무료 티어)"]
    end

    subgraph RENDER["Render"]
        CRON["크롤러<br/>Cron Job (Python)"]
        API["챗봇 백엔드<br/>FastAPI Web Service"]
    end

    subgraph SUPABASE["Supabase"]
        DB[("PostgreSQL + pgvector<br/>무료 티어")]
    end

    subgraph VERCEL["Vercel"]
        WEB["Next.js<br/>프론트엔드 + 조항별 뷰페이지(ISR)"]
    end

    STUDENT(["학생 브라우저"])

    KNU -- "주기적 POST 크롤링" --> CRON
    CRON -- "규정/조항 upsert" --> DB
    CRON -- "조항 임베딩 요청" --> GEMINI
    CRON -- "재수집 후 revalidate 호출" --> WEB

    STUDENT -- "입학연도 + 질문" --> WEB
    WEB -- "/chat 요청" --> API
    API -- "질문 임베딩" --> GEMINI
    API -- "유사도 검색" --> DB
    API -- "답변 생성 요청" --> GEMINI
    API -- "답변 + 근거 + 정확도%" --> WEB
    WEB -- "결과 렌더링" --> STUDENT

    STUDENT -- "규정 링크 클릭" --> WEB
    WEB -- "조항 조회(캐시 미스 시)" --> DB
```

## 2. 컴포넌트 역할

| 컴포넌트 | 배포 위치 | 역할 |
|---|---|---|
| 크롤러 | Render Cron Job | 규정집 주기 재수집, EUC-KR 디코딩, 편/장/절/조 구조 파싱, DB upsert |
| 챗봇 백엔드 | Render Web Service (FastAPI) | 질문 임베딩, pgvector 검색, LLM 호출, 확인 과정/학부 매칭/정확도 산출 |
| DB | Supabase (Postgres + pgvector) | 규정·조항·학부/학과 데이터, 임베딩 벡터 |
| 프론트엔드 | Vercel (Next.js) | 입력 UI, 응답 렌더링, 조항별 뷰페이지(ISR) |
| LLM/임베딩 | Google Gemini API | 임베딩 생성, 답변 생성 |

## 3. 핵심 흐름

### 3-1. 크롤링 파이프라인 (배치)

```mermaid
sequenceDiagram
    participant Cron as Render Cron Job
    participant KNU as 강남대 규정집
    participant Gemini as Gemini API
    participant DB as Supabase DB
    participant Web as Vercel(Next.js)

    Cron->>KNU: POST rulesL1.jsp (전체 목록)
    KNU-->>Cron: 325개 규정 (편,장,호)
    loop 각 규정
        Cron->>KNU: POST rulesL2.jsp (rglt_alls=Y)
        KNU-->>Cron: 규정 전문 HTML
        Cron->>Cron: EUC-KR 디코딩, 장/절/조 파싱
        Cron->>Gemini: 조항 텍스트 임베딩 요청
        Gemini-->>Cron: 임베딩 벡터
        Cron->>DB: upsert(규정, 조항, 임베딩)
    end
    Cron->>Web: on-demand revalidate 호출
```

### 3-2. 챗봇 질의응답 (기본/예외 플로우)

```mermaid
sequenceDiagram
    participant S as 학생
    participant Web as Next.js
    participant API as FastAPI
    participant Gemini as Gemini API
    participant DB as Supabase DB

    S->>Web: 입학연도 + 질문
    Web->>API: /chat 요청
    API->>Gemini: 질문 임베딩
    Gemini-->>API: 벡터
    API->>DB: pgvector 유사도 검색
    DB-->>API: 관련 조항 후보

    alt 학부/학과별로 답변이 갈림
        API-->>Web: 확인 과정(이유 설명 + 소속 요청)
        Web-->>S: 소속 입력 요청
        S->>Web: 학부/학과 입력 (또는 "전공"→재입력 유도)
        Web->>API: 재요청(소속 포함)
        API->>DB: 직제규정 목록과 유사도 매칭
        opt 매칭 불확실
            API-->>Web: 후보 최대 3개 제시
            Web-->>S: 후보 확인 요청
            S->>Web: 확정
            Web->>API: 확정된 소속으로 재요청
        end
    end

    alt 관련 규정 0개
        API-->>Web: "답변할 수 없음" + 재질문 유도 안내
        Web-->>S: 안내 표시
    else 관련 규정 있음
        API->>Gemini: 컨텍스트 + 질문으로 답변 생성
        Gemini-->>API: 답변
        API-->>Web: 답변요약 + 정확도% + 규정목록(최대5) + 링크
        Web-->>S: 결과 렌더링
    end
```

### 3-3. 조항별 뷰페이지 (ISR, 온디맨드)

```mermaid
sequenceDiagram
    participant S as 학생
    participant Web as Vercel(Next.js)
    participant DB as Supabase DB

    S->>Web: /rules/{규정명}/{조번호} 클릭
    alt 캐시 있음
        Web-->>S: 캐시된 페이지 즉시 응답
    else 캐시 없음(최초 요청)
        Web->>DB: 규정명 + 조번호로 조회
        DB-->>Web: 조항 데이터
        Web->>Web: 렌더링 후 캐시
        Web-->>S: 페이지 응답
    end
```

## 4. 데이터 모델 (초안)

```mermaid
erDiagram
    REGULATIONS ||--o{ ARTICLES : contains
    DEPARTMENTS

    REGULATIONS {
        int id PK
        string name
        string slug
        date last_revised
        string source_ref "원본 편,장,호"
    }
    ARTICLES {
        int id PK
        int regulation_id FK
        string article_no
        text content
        vector embedding
    }
    DEPARTMENTS {
        int id PK
        string name
        string type "학부 또는 학과"
    }
```

정식 스키마(컬럼 타입, 인덱스 등)는 구현 단계에서 확정.

## 5. 배포 경계 요약

- **Vercel**: Next.js만 배포 (프론트엔드 + 조항별 뷰페이지)
- **Render**: FastAPI(상시 구동, DB 커넥션 풀 유지) + 크롤러(Cron Job)
- **Supabase**: Postgres + pgvector, 무료 티어
- **Google**: Gemini API, 무료 티어 (초과 시 서비스 응답 중단)

각 선택의 근거는 [PLANNING.md §5 배포 아키텍처](PLANNING.md#5-배포-아키텍처) 참고.

## 6. 다음 개발 순서 (제안)

1. Supabase 프로젝트 생성 + pgvector 익스텐션 활성화 + 스키마 마이그레이션
2. 크롤러 프로토타입 (rulesL1/rulesL2 파싱 → 로컬 검증)
3. 임베딩 파이프라인 연동 (Gemini Embedding API) + DB upsert
4. FastAPI 챗봇 API (검색 + 생성 + 확인 과정/학부 매칭 로직)
5. Next.js 프론트엔드 (입력 폼 + 응답 렌더링)
6. 조항별 뷰페이지 (ISR)
7. Render/Vercel/Supabase 배포 연결 + e2e 테스트
