# AI 심리상담 앱 백엔드 포트폴리오
https://onoff-m.com/

---

## 목차

1. [프로젝트 개요](#1-프로젝트-개요)
2. [기술 스택](#2-기술-스택)
3. [시스템 아키텍처](#3-시스템-아키텍처)
4. [ERD / 도메인 설계](#4-erd--도메인-설계)
5. [트러블슈팅](#5-트러블슈팅)

---

## 1. 프로젝트 개요

### 서비스 한 줄 소개

AI 챗봇과 대화를 통해 감정을 정리하고 심리적 지원을 받을 수 있는 웹 기반 상담 서비스.

### 프로젝트 기간

- 2025년 1월 ~ 현재 (진행 중)

### 기여 범위

| 영역 | 내용 |
|------|------|
| **백엔드 설계/구현** | Spring Boot 기반 REST API, JPA 엔티티·Repository, 서비스 레이어 설계 |
| **인증/인가** | OAuth2 소셜 로그인(카카오·네이버·구글), JWT 발급·검증, 관리자 ID/PW 로그인(BCrypt) |
| **배포/운영** | Docker Compose, Nginx 리버스 프록시, PostgreSQL init-db 스크립트 |
| **관리자 기능** | 회원·채팅·일기·리포트·설문·토큰·쿠폰·입장문·인사말 CRUD, KPI 대시보드 API 설계 |
| **데이터 보호** | 이메일 AES-256 암호화, HMAC 검색, 상담 데이터 소유권 검증 |
| **AI 연동** | Python HTTP/SSE 호출, 컨텍스트(Short/Summary/Long-term Memory) 수집·전달 |
| **AI 서비스 개발** | Python FastAPI + Gemini 연동, Agent 오케스트레이션(Emotion→Safety→Therapy→Recommendation), 리포트/인사말 생성 API |

---

## 2. 기술 스택

| 영역 | 기술 | 선택 이유 |
|------|------|-----------|
| **Backend** | Spring Boot 3.5, Java 21 | LTS, WebFlux(WebClient)로 비동기 HTTP/SSE 호출 |
| **Database** | PostgreSQL 16 + pgvector | 관계형 + 벡터 검색(Long-term Memory) 단일 DB로 운영 |
| **캐시** | Redis | Short Memory(세션별 최근 메시지) TTL 관리 |
| **인증** | Spring Security, OAuth2 Client, JWT(jjwt) | 소셜 로그인 + API용 Stateless JWT |
| **AI 연동** | WebClient, SSE | counsel-chat(Python) HTTP/스트리밍 호출 |
| **배포** | Docker Compose, Nginx | 로컬·운영 환경 통일, 리버스 프록시로 API·OAuth2 라우팅 |
| **암호화** | AES-256-GCM, HMAC-SHA256 | 이메일 암호화, 검색용 HMAC |

---

## 3. 시스템 아키텍처

### 전체 구조

```mermaid
flowchart TB
    subgraph Client["클라이언트"]
        FE[Frontend :9880]
        Admin[Admin :9882]
    end

    subgraph MainNginx["메인 Nginx :80 (9880)"]
        N1[location / → frontend]
        N2[location /api/ → backend]
    end

    subgraph AdminNginx["Admin Nginx :9882 (9882)"]
        A1[location / → SPA]
        A2[location /api/ → backend]
    end

    subgraph Backend["백엔드"]
        Spring[Spring Boot :9881]
    end

    subgraph Data["데이터"]
        PG[(PostgreSQL)]
        Redis[(Redis)]
    end

    subgraph AI["AI 서비스"]
        Chat[counsel-chat]
        Python[Python FastAPI :8000]
    end

    Ext[Google Gemini API]

    FE --> MainNginx
    Admin --> AdminNginx
    MainNginx --> Spring
    AdminNginx --> Spring
    Spring --> PG
    Spring --> Redis
    Spring --> Chat
    Chat --> Python
    Python --> Ext
```

---

## 4. ERD / 도메인 설계

### 핵심 엔티티 관계

```mermaid
erDiagram
    users ||--o{ chat_sessions : owns
    users ||--o{ diaries : writes
    users ||--o{ reports : has
    users ||--o{ token_usage_logs : has
    users ||--o{ user_memories : has
    users ||--o{ conversation_analysis : has
    users ||--o{ recommendations : has
    users ||--o{ survey_feedback : has
    chat_sessions ||--o{ chat_messages : contains
    chat_sessions ||--o{ conversation_summary : has
    chat_sessions ||--o{ token_usage_logs : logs
    chat_sessions ||--o{ conversation_analysis : has
    chat_sessions ||--o{ survey_feedback : for
    chat_sessions ||--o{ recommendations : has
    chat_messages ||--o| emotion_analysis : has
    chat_messages ||--o{ token_usage_logs : logs

    users {
        bigint id PK
        text email
        varchar email_hmac
        varchar provider
        varchar provider_id
        varchar status
    }

    chat_sessions {
        bigint id PK
        bigint user_id FK
        varchar title
        int message_count
        boolean deleted
    }

    chat_messages {
        bigint id PK
        bigint session_id FK
        varchar role
        text content
        int token_used
    }

    emotion_analysis {
        bigint id PK
        bigint message_id FK
        varchar emotion
        double confidence
        varchar risk_level
    }

    conversation_summary {
        bigint id PK
        bigint session_id FK
        text summary
    }

    conversation_analysis {
        bigint id PK
        bigint user_id FK
        bigint chat_room_id FK
        varchar emotion
        double emotion_score
        varchar topic
        text summary
    }

    reports {
        bigint id PK
        bigint user_id FK
        varchar type
        date period_start
        date period_end
        text content
    }

    diaries {
        bigint id PK
        bigint user_id FK
        varchar title
        text content
    }

    user_memories {
        bigint id PK
        bigint user_id FK
        text memory_text
        vector embedding
    }

    token_usage_logs {
        bigint id PK
        bigint user_id FK
        bigint chat_room_id FK
        bigint message_id FK
        varchar chat_type
        varchar agent_name
        int total_tokens
        timestamp created_at
    }

    recommendations {
        bigint id PK
        bigint user_id FK
        bigint session_id FK
        varchar type
        varchar title
        text description
    }

    survey_feedback {
        bigint id PK
        bigint user_id FK
        bigint chat_room_id FK
        varchar positive_feedback
        varchar negative_feedback
        text additional_comment
    }
```

### 설계 판단 근거

| 설계 | 이유 |
|------|------|
| **세션·메시지 분리** | 세션 단위 목록/삭제, 메시지 단위 페이지네이션·스트리밍 분리 |
| **리포트 독립 엔티티** | 일일/주간별 기간 고정, AI 생성 본문 저장, 동일 기간 재생성 시 upsert |
| **일기·상담 이력 분리** | 일기는 사용자 작성, 상담은 AI 대화 기반. 감정 타임라인은 emotion_history로 통합 |
| **user_memories (pgvector)** | Long-term Memory 검색을 위해 임베딩 벡터 저장, counsel-chat /embed 호출 결과 활용 |
| **conversation_analysis** | 상담별 감정·주제·요약을 리포트 생성 시 집계용으로 분리 저장 |

---

## 5. 트러블슈팅

---
