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

## 6-1 EphemeralStreamStore: Redis 대신 인메모리 선택

| 항목 | 내용 |
|------|------|
| **상황** | 익명 채팅 컨텍스트(메시지 목록)를 120초 동안 보관해야 함. Redis vs ConcurrentHashMap 선택 |
| **선택** | ConcurrentHashMap 기반 인메모리 저장 |
| **이유** | 1) 데이터 수명이 짧고 1회성(getAndRemove) 2) 동시 접속이 많지 않아 JVM 힙으로 충분 3) Redis는 연결·네트워크 오버헤드 대비 이점 없음 4) 단일 인스턴스로 운영 중이라 공유 저장소 불필요 |
| **트레이드오프** | 수평 스케일아웃 시 인스턴스별 저장이라 streamKey가 다른 인스턴스로 라우팅되면 "Invalid stream key" 발생 가능. 이 경우 Redis/공유 저장소로 전환 필요 |
| **배운 점** | 수명이 짧고 공유가 필수가 아닌 데이터는 인메모리가 단순·빠름. 스케일 전략에 맞춰 저장소를 선택해야 함 |

---

## 6-2 클라이언트 끊김 시 AI 호출 취소 미연결 문제

| 항목 | 내용 |
|------|------|
| **상황** | 사용자가 탭을 닫거나 네트워크가 끊겨 SseEmitter 쪽은 종료되지만, WebClient→Python HTTP 연결은 별도로 유지됨 |
| **현재 동작** | Client disconnect 시 Spring SseEmitter는 종료되나, WebClient 구독은 별도 스레드에서 유지. Python은 끝까지 스트리밍하고 done 이벤트 전송 |
| **의도** | done 수신 시 saveStreamResult로 DB 저장 보장 우선. 클라이언트가 이미 떠났어도 데이터 정합성은 유지 |
| **트레이드오프** | 끊긴 클라이언트를 위해 AI 리소스를 추가 소모. 대신 불완전 세션(assistant 미저장) 발생이 줄어듦 |
| **대안** | SseEmitter.onCompletion/onTimeout 시 WebClient 구독 취소(dispose)를 연결하면 리소스는 절약되나, done 미수신으로 assistant 저장이 누락될 수 있음 |
| **배운 점** | "정합성 우선 vs 리소스 절약" 정책을 명확히 정의한 뒤 disconnect 시 동작을 결정해야 함 |

---

## 6-3 Redis Short Memory 캐시 일관성(Cache-Aside)

| 항목 | 내용 |
|------|------|
| **구조** | Redis miss 시 DB에서 로드 후 Redis에 세팅. appendAndTrim으로 Redis 갱신 |
| **일관성** | 단일 인스턴스: DB가 source of truth. 다중 인스턴스: 모두 같은 Redis를 사용하므로 Redis가 공유 source |
| **고려사항** | DB 직접 수정/마이그레이션 시 Redis가 오래된 값을 유지할 수 있음. TTL(7일)로 자연 만료되거나, 명시적 invalidation 필요 |
| **선택** | 상담 메시지는 관리자가 직접 수정하는 경우가 거의 없어, TTL 기반 만료만으로 수용 가능하다고 판단 |
| **배운 점** | Cache-Aside에서는 source of truth와 갱신·만료 정책을 명확히 정의해야 함 |

---

💡 **기획부터 배포·운영까지 전 과정을 직접 담당한 프로젝트로, 현재 실서비스로 운영 중입니다.**
