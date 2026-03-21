# 🧠 AI 심리상담 서비스 포트폴리오

AI 챗봇과 대화를 통해 감정을 정리하고 심리적 지원을 받을 수 있는  
**웹 기반 심리 상담 서비스**의 백엔드 시스템을 설계하고 개발했습니다.

🌐 **Service**  
[https://onoff-m.com](https://onoff-m.com)

---

# 📑 목차

- [1. 프로젝트 개요](#1-프로젝트-개요)
- [2. 기술 스택](#2-기술-스택)
- [3. 시스템 아키텍처](#3-시스템-아키텍처)
- [4. ERD / 도메인 설계](#4-erd--도메인-설계)
- [5. 백엔드 트러블슈팅](#5-트러블슈팅)
- [6. AI 트러블슈팅](#6-AI-트러블슈팅)

---

# 1. 프로젝트 개요

### 🧾 서비스 소개

AI 챗봇과 대화를 통해 감정을 정리하고  
**심리적 지원과 상담 리포트를 제공하는 웹 서비스**

- AI 상담 채팅
- 감정 분석 및 위험 감지
- 상담 리포트 자동 생성
- 일기 기록
- 쿠폰 및 토큰 기반 이용권 관리
- FCM 푸시 알림

---

### 📅 프로젝트 기간
```
2025.01 ~ 현재 (진행 중)
```

---

### 👨‍💻 개발 범위

| 영역 | 내용 |
|-----|-----|
| **Backend 설계** | Spring Boot 기반 REST API, JPA 엔티티·Repository 설계 |
| **인증 / 인가** | OAuth2 소셜 로그인(카카오·네이버·구글), JWT 인증 |
| **배포 / 운영** | Docker Compose, Nginx 리버스 프록시, Cloudflare Tunnel |
| **관리자 시스템** | 회원·토큰·쿠폰·설문·입장문 관리 API (React 기반 관리자 페이지 연동) |
| **데이터 보호** | 이메일/채팅 데이터 암호화 |
| **AI 연동** | Python FastAPI HTTP/SSE 스트리밍 통신 |
| **AI 서비스** | LLM 기반 멀티 Agent 상담 파이프라인 설계 및 구현 |
| **클라이언트** | Next.js 웹 프론트, Flutter 모바일 앱과의 API 연동 |

---

# 2. 기술 스택

### Backend

| 기술 | 선택 이유 |
|-----|-----|
| **Spring Boot 3.5** | 안정적인 백엔드 API 서버 구축 |
| **Java 21** | 최신 LTS 기반 성능 및 기능 활용 |
| **Spring Security** | 인증·인가 관리 |
| **Spring WebFlux (WebClient)** | Python AI 서버와의 비동기 SSE 스트리밍 통신 |

---

### AI / Streaming

| 기술 | 역할 |
|-----|-----|
| **Python FastAPI** | AI 서버 (멀티 Agent 오케스트레이션) |
| **SSE (Server-Sent Events)** | 토큰 단위 실시간 스트리밍 응답 전달 |
| **LLM API** | 상담 응답 생성 및 감정 분석 |

---

### Database

| 기술 | 역할 |
|-----|-----|
| **PostgreSQL** | 메인 데이터베이스 |
| **pgvector** | Long-term Memory 벡터 검색 |
| **Redis** | Short Memory 세션 캐시 |

---

### DevOps

| 기술 | 역할 |
|-----|-----|
| **Docker Compose** | 멀티 서비스 컨테이너 관리 |
| **Nginx** | Reverse Proxy |
| **Cloudflare Tunnel** | 외부 접속 및 도메인 연결 |

---

# 3. 시스템 아키텍처

### 전체 구조

![architecture](mermaid-diagram-2026-03-15-151606.png)

---

# 4. ERD / 도메인 / Agent 파이프라인 설계

### 4.1 핵심 엔티티 관계

![erd](mermaid-diagram-2026-03-15-151128.png)

---

### 4.2 설계 판단 근거

| 설계 | 이유 |
|-----|-----|
| **메시지 저장소 PostgreSQL 선택** | Redis 메모리 부담, MongoDB 아키텍처 복잡성 증가 |
| **pgvector 사용** | Long-term Memory 임베딩 검색 (별도 벡터 DB 없이 PostgreSQL 확장으로 처리) |
| **Redis 사용** | 세션 기반 Short Memory 관리, TTL 7일 기반 자연 만료 전략 |

---

### 4.3 Agent 파이프라인 (Conversation Loop)

상담 요청 시 **Emotion → Safety → Therapy → Recommendation** 순으로 4개 Agent가 순차 실행됩니다.

| Agent | 입력 | 출력 | 역할 |
|------|------|------|------|
| EmotionAgent | user_message | emotion, confidence, risk_level | 감정·위험 수준 분석 |
| SafetyAgent | user_message, emotion_risk | risk_level, crisis_message | 위험 발언 규칙 감지, 위기 연락처 안내 |
| TherapyAgent | summary, recent_messages, memories, risk_level | answer | Persona 기반 공감 상담 응답 |
| RecommendationAgent | emotion, risk_level, user_message, therapy_reply | recommendations | 행동·자기인식·생활 추천 |

각 Agent는 독립적으로 책임을 분리하여 설계했으며, Python `orchestrator.py`에서 전체 실행 흐름을 조율합니다.

---

# 5. 트러블슈팅

## 5.1 스트리밍 3-hop 파이프라인 정합성 문제

### 문제

Python → Spring → Client  
**3단계 스트리밍 파이프라인에서 클라이언트 조기 종료 시 데이터 유실 가능**

Client가 `done` 이벤트 이전에 연결을 종료하면  
assistant 메시지가 DB에 저장되지 않는 문제가 발생.

---

### 원인

`saveStreamResult()`가  
`done 이벤트`에서만 호출되도록 구현되어 있었음.

Client disconnect 시 `done` 이벤트 미수신.

---

### 해결

- **user 메시지 먼저 저장** → 요청 자체는 항상 기록
- assistant 메시지는 스트리밍 완료(`done`) 후 저장
- 실패 시 **불완전 세션 허용 (Eventual Consistency 전략 채택)**

선택적 개선

- Python 완료 시 Spring callback API 호출로 저장 보장 가능

---

## 5.2 Spring + Python 이원화 아키텍처 트랜잭션 문제

### 문제

Spring과 Python이 **서로 다른 프로세스 서비스**이기 때문에  
원자적 트랜잭션 보장이 불가능.

---

### 해결

- 사용자 메시지는 **먼저 저장** (Spring 트랜잭션 내)
- AI 처리 실패 시 **assistant 응답 데이터만 없는 상태**로 허용
- **Eventual Consistency 전략 채택** → 정합성 우선, 완전한 원자성 대신 부분 실패 허용 설계

---

## 5.3 AI 호출 타임아웃과 블로킹 문제

### 문제

비스트리밍 AI 호출이
```
WebClient + .block()
```

으로 구현되어 **Spring 스레드가 블로킹**되는 문제 발생.  
AI 응답 지연 시 스레드 자원이 점유되어 처리량 저하 우려.

---

### 해결

- 스트리밍 호출: `bodyToFlux + subscribe` 방식으로 전환
- WebClient **timeout 설정** 적용으로 무한 대기 방지
- 비스트리밍 호출도 비동기 처리로 전환 예정

---

# 6. AI 트러블슈팅

## 6-1 익명 채팅 컨텍스트 임시 저장소 선택 문제

### 문제

AI 응답 스트리밍 중, **익명 채팅의 메시지 컨텍스트를 약 120초간 임시 보관**해야 했습니다.

Redis와 인메모리(ConcurrentHashMap) 중 어떤 저장소를 선택할지 결정이 필요했습니다.

---

### 해결

**인메모리(ConcurrentHashMap) 방식을 선택했습니다.**

- 데이터 수명이 120초로 매우 짧고, 조회 즉시 삭제되는 1회성 데이터
- 현재 단일 서버로 운영 중이라 인스턴스 간 공유 저장소가 불필요
- Redis는 네트워크 연결 비용 대비 실제 이점이 없음

> ⚠️ **트레이드오프**: 서버를 여러 대로 확장할 경우, 요청이 다른 서버로 라우팅되면 저장된 컨텍스트를 찾지 못할 수 있습니다. 이 경우 Redis로 전환이 필요합니다.

---

## 6-2 클라이언트 연결 종료 시 AI 호출이 계속 유지되는 문제

### 문제

사용자가 탭을 닫거나 네트워크가 끊기면,
**클라이언트와의 연결(SseEmitter)은 종료**되지만
**Python AI 서버로의 HTTP 연결은 그대로 유지**됩니다.

클라이언트가 이미 없는데도 AI가 계속 응답을 생성하며 서버 자원을 소모하는 상황이 발생했습니다.

---

### 해결

**AI 호출을 중단하지 않고 끝까지 완료하는 방향을 선택했습니다.**

- AI 응답이 완료(`done` 이벤트)되면 DB 저장까지 보장
- 클라이언트가 없어도 **상담 내용이 유실되지 않음**

> ⚠️ **트레이드오프**: 이미 떠난 클라이언트를 위해 AI 리소스를 추가 소모합니다.
> AI 호출을 즉시 취소하면 리소스는 절약되지만, DB 저장이 누락되어 불완전한 상담 세션이 남을 수 있습니다.
> **"데이터 정합성 우선"** 정책으로 현재 방식을 유지했습니다.

---

## 6-3 Redis 캐시와 DB 데이터 불일치 문제

### 문제

Short Memory(최근 대화 내역)를 Redis에 캐싱하는 구조에서,
**DB 데이터가 직접 수정될 경우 Redis에는 이전 값이 그대로 남는** 불일치 문제가 발생할 수 있습니다.

---

### 해결

**TTL(7일) 기반 자연 만료 전략을 채택했습니다.**

- Redis miss 시 DB에서 불러와 Redis에 다시 저장 (Cache-Aside 패턴)
- 상담 메시지는 관리자가 직접 수정하는 경우가 거의 없기 때문에, 7일 후 자동 만료만으로도 충분하다고 판단

> ⚠️ **트레이드오프**: DB를 직접 수정하거나 마이그레이션을 실행할 경우, TTL 만료 전까지 Redis가 오래된 데이터를 반환할 수 있습니다. 이 경우 명시적으로 Redis 키를 삭제(invalidation)해야 합니다.

💡 **기획부터 배포·운영까지 전 과정을 직접 담당한 프로젝트로, 현재 실서비스로 운영 중입니다.**
