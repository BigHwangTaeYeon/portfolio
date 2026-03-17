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
- 감정 분석
- 상담 리포트 생성
- 일기 기록

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
| **배포 / 운영** | Docker Compose, Nginx 리버스 프록시 |
| **관리자 시스템** | 회원·토큰·설문·입장문 관리 API |
| **데이터 보호** | 이메일/채팅 암호화 |
| **AI 연동** | Python FastAPI HTTP/SSE 통신 |
| **AI 서비스** | Gemini 기반 상담 Agent 구성 |

---

# 2. 기술 스택

### Backend

| 기술 | 선택 이유 |
|-----|-----|
| **Spring Boot 3.5** | 안정적인 백엔드 API 서버 구축 |
| **Java 21** | 최신 LTS 기반 성능 및 기능 활용 |
| **Spring Security** | 인증·인가 관리 |

---

### AI / Streaming

| 기술 | 역할 |
|-----|-----|
| **Python FastAPI** | AI 서버 |
| **SSE** | 스트리밍 응답 전달 |
| **Gemini API** | 상담 및 분석 AI |

---

### Database

| 기술 | 역할 |
|-----|-----|
| **PostgreSQL** | 메인 데이터베이스 |
| **pgvector** | Long-term Memory 벡터 검색 |
| **Redis** | Short Memory 캐시 |

---

### DevOps

| 기술 | 역할 |
|-----|-----|
| **Docker Compose** | 서비스 컨테이너 관리 |
| **Nginx** | Reverse Proxy |
| **Cloudflare Tunnel** | 외부 접속 |

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
| **pgvector 사용** | Long-term Memory 임베딩 검색 |
| **Redis 사용** | 세션 기반 Short Memory 관리 |

---

### 4.3 Agent 파이프라인 (Conversation Loop)

상담 요청 시 **Emotion → Safety → Therapy → Recommendation** 순으로 4개 Agent가 순차 실행됩니다.

| Agent | 입력 | 출력 | 역할 |
|------|------|------|------|
| EmotionAgent | user_message | emotion, confidence, risk_level | 감정·위험 수준 분석 |
| SafetyAgent | user_message, emotion_risk | risk_level, crisis_message | 위험 발언 규칙 감지, 위기 연락처 안내 |
| TherapyAgent | summary, recent_messages, memories, risk_level | answer | Persona 기반 공감 상담 응답 |
| RecommendationAgent | emotion, risk_level, user_message, therapy_reply | recommendations | 행동·자기인식·생활 추천 |

---

# 5. 트러블슈팅

## 5.1 스트리밍 3-hop 파이프라인 정합성 문제

### 문제

Python → Spring → Client  
**3단계 스트리밍 파이프라인에서 데이터 유실 가능**

Client가 `done` 이벤트 이전에 연결을 종료하면  
assistant 메시지가 DB에 저장되지 않는 문제가 발생.

---

### 원인

`saveStreamResult()`가  
`done 이벤트`에서만 호출되도록 구현되어 있었음.

Client disconnect 시 `done` 이벤트 미수신.

---

### 해결

- **user 메시지 먼저 저장**
- assistant 메시지는 스트리밍 완료 후 저장
- 실패 시 **불완전 세션 허용**

선택적 개선

- Python 완료 시 Spring callback API 호출

---

## 5.2 Spring + Python 이원화 아키텍처 트랜잭션 문제

### 문제

Spring과 Python이 **서로 다른 서비스**이기 때문에  
원자적 트랜잭션 보장이 불가능.

---

### 해결

- 사용자 메시지는 **먼저 저장**
- AI 실패 시 **assistant 데이터만 없음**
- **Eventual Consistency 전략 채택**

---

## 5.3 AI 호출 타임아웃과 블로킹 문제

### 문제

비스트리밍 AI 호출이

```
WebClient + .block()
```

으로 구현되어 **Spring 스레드가 블로킹**되는 문제 발생.

---

### 해결

- 스트리밍: `bodyToFlux + subscribe`
- WebClient **timeout 설정**
- 비동기 처리로 전환 예정

---

# 6. AI 트러블슈팅

## 6-1 EphemeralStreamStore: Redis 대신 인메모리 선택

| 항목 | 내용 |
|------|------|
| **상황** | 익명 채팅 컨텍스트(메시지 목록)를 120초 동안 보관해야 함. Redis vs ConcurrentHashMap 선택 |
| **선택** | ConcurrentHashMap 기반 인메모리 저장 |
| **이유** | 1) 데이터 수명이 짧고 1회성(getAndRemove) 2) 동시 접속이 많지 않아 JVM 힙으로 충분 3) Redis는 연결·네트워크 오버헤드 대비 이점 적음 4) 단일 인스턴스로 운영 중이라 공유 저장소 불필요 |
| **트레이드오프** | 수평 스케일아웃 시 인스턴스별 저장이라 streamKey가 다른 인스턴스로 라우팅되면 "Invalid stream key" 가능. 이 경우 Redis/공유 저장소로 전환 필요 |
| **배운 점** | 수명이 짧고 공유가 필수가 아닌 데이터는 인메모리가 단순·빠름. 스케일 전략에 맞춰 저장소를 선택 |

---

## 6-2 Nginx SSE 버퍼링으로 인한 실시간 전달 지연

| 항목 | 내용 |
|------|------|
| **문제** | Nginx 프록시 뒤에서 SSE 스트리밍이 한 번에 몰아서 도착하거나, 토큰 단위 실시간 전달이 되지 않음 |
| **원인** | Nginx 기본값 `proxy_buffering on`으로 응답을 버퍼링. 버퍼가 가득 차거나 요청이 끝나기 전까지 클라이언트로 전달하지 않음 |
| **해결** | `proxy_buffering off` 설정. SSE 응답 헤더에 `X-Accel-Buffering: no` 추가 |
| **코드** | nginx.conf, counsel-chat StreamingResponse headers |
| **배운 점** | 리버스 프록시 뒤에서 SSE/스트리밍을 쓸 때는 버퍼링 끄기가 기본 설정 |

---

## 6-3 클라이언트 끊김 시 Python 호출 취소 미연결

| 항목 | 내용 |
|------|------|
| **상황** | 사용자가 탭을 닫거나 네트워크가 끊겨 SseEmitter 쪽은 종료되지만, WebClient→Python HTTP 연결은 별도 |
| **현재 동작** | Client disconnect 시 Spring SseEmitter는 종료되나, WebClient 구독은 별도 스레드에서 유지. Python은 끝까지 스트리밍하고 done 이벤트 전송 |
| **의도** | done 수신 시 saveStreamResult로 DB 저장이 우선. 클라이언트는 이미 떠났어도 데이터 정합성은 유지 |
| **트레이드오프** | 끊긴 클라이언트를 위해 Python·Gemini 리소스를 추가로 소모. 대신 불완전 세션(assistant 미저장)이 줄어듦 |
| **대안** | SseEmitter.onCompletion/onTimeout 시 WebClient 구독 취소(dispose)를 연결하면 리소스는 절약되나, done 미수신으로 assistant 저장이 누락될 수 있음 |
| **배운 점** | "정합성 우선 vs 리소스 절약" 정책을 명확히 한 뒤 disconnect 시 동작을 결정해야 함 |

---

## 6-4 Redis Short Memory 캐시 일관성(Cache-Aside)

| 항목 | 내용 |
|------|------|
| **구조** | Redis가 miss면 DB에서 로드 후 Redis에 세팅. appendAndTrim으로 Redis 갱신 |
| **일관성** | 단일 인스턴스: DB가 source of truth. 다중 인스턴스: 모두 같은 Redis를 사용하므로 Redis가 공유 source |
| **고려사항** | DB 직접 수정/마이그레이션 시 Redis가 오래된 값 유지. TTL(7일)로 자연 만료되거나, 명시적 invalidation 필요 |
| **선택** | 상담 메시지는 관리자가 직접 수정하는 경우가 거의 없어, TTL 기반 만료만으로 수용 |
| **배운 점** | Cache-Aside에서는 source of truth와 갱신·만료 정책을 명확히 정의해야 함 |

---

💡 **GitHub 포트폴리오용으로 실제 서비스까지 운영 중인 프로젝트입니다.**
