# UGH (Unsolved Guideline for HGU) - 전체 시스템 아키텍처

한동대 학생들의 백준 미풀이 문제를 추적하고, Q&A 게시판과 랭킹 기능을 제공하는 웹 서비스.

---

## 1. 전체 시스템 구조도

```
                          ┌────────────────┐
                          │   solved.ac    │
                          │   (외부 API)    │
                          └───────┬────────┘
                                  │ HTTP GET (문제/유저 데이터)
                                  v
┌─────────────────────────────────────────────────────────┐
│              데이터 파이프라인 (Python)                    │
│                                                         │
│  ┌──────────┐   메세지 발행   ┌──────────┐              │
│  │  Broker  │ ──────────────> │ RabbitMQ │              │
│  │(Producer)│                 │  (Queue) │              │
│  └──────────┘                 └─────┬────┘              │
│       │                             │ 메세지 소비        │
│       │ API vs DB 비교              v                   │
│       │                       ┌──────────┐              │
│       │                       │ Consumer │              │
│       │                       │(Subscriber)│            │
│       └───────────────────────┴─────┬────┘              │
└─────────────────────────────────────┼───────────────────┘
                                      │ INSERT/UPDATE
                                      v
                              ┌──────────────┐
    ┌────────────────────────>│    MySQL     │
    │         JPA             │  (Database)  │
    │                         └──────────────┘
    │
┌───┴──────────────────────────────────────────────────────┐
│                백엔드 (Spring Boot 3.3)                    │
│                                                           │
│  Controller → Service → Repository → DB                   │
│                                                           │
│  ┌─────────┐  ┌───────┐  ┌───────┐  ┌──────────────────┐│
│  │ OAuth2  │  │ Redis │  │ SMTP  │  │    Thymeleaf     ││
│  │(G/K/N)  │  │(인증코드)│ │(Gmail)│  │   (프론트엔드)    ││
│  └─────────┘  └───────┘  └───────┘  └──────────────────┘│
└──────────────────────────┬───────────────────────────────┘
                           │ HTML/AJAX
                           v
                    ┌──────────────┐
                    │   브라우저    │
                    └──────────────┘
```

---

## 2. 백엔드 (hguAPIs - Spring Boot)

**기술 스택:** Spring Boot 3.3.0, Java 17, JPA/Hibernate, Spring Security, OAuth2

### 2.1 도메인 구조

| 도메인 | 패키지 | 설명 |
|--------|--------|------|
| User | `com.unsolved.hgu.user` | 사용자 계정, 인증, OAuth2 |
| Problem | `com.unsolved.hgu.problem` | 백준 문제 목록, 난이도 필터링 |
| Question | `com.unsolved.hgu.question` | Q&A 게시판 질문 |
| Answer | `com.unsolved.hgu.answer` | Q&A 게시판 답변 |
| UserInfo | `com.unsolved.hgu.userinfo` | 유저 통계, 랭킹 |
| UserSolved | `com.unsolved.hgu.usersolved` | 유저별 풀이 문제 추적 |
| Email | `com.unsolved.hgu.email` | 이메일 인증 |

### 2.2 계층 구조 (Controller → Service → Repository)

```
UserController          → UserService          → UserRepository
UserSelfController      → UserService          → UserRepository
QuestionController      → QuestionService      → QuestionRepository
AnswerController        → AnswerService        → AnswerRepository
ProblemController       → ProblemService        → ProblemRepository
UserInfoController      → UserInfoService       → UserInfoRepository
EmailController (REST)  → EmailService          → Redis
```

### 2.3 엔티티 관계도

```
SiteUser ──1:N──> Question ──1:N──> Answer
    │                 │                │
    │ (voter M:N)     │ (voter M:N)    │ (voter M:N)
    │                 │                │
    └── M:N ──> Problem ──M:N──> Tag
                    │
                    └──1:N──> UserInfoProblemSolved <──N:1── UserInfo
                                  (복합 키: problemId + username)
```

- **SiteUser**: 사용자 계정 (사이트/OAuth2)
- **Problem**: 백준 문제 (id = 백준 문제 번호, level 1~30)
- **UserInfo**: solved.ac에서 동기화된 유저 통계 (username이 PK)
- **UserInfoProblemSolved**: UserInfo와 Problem의 조인 테이블

### 2.4 인증

| 방식 | 설명 |
|------|------|
| 사이트 로그인 | 이메일 인증 → BCrypt 비밀번호 → Spring Security |
| Google OAuth2 | sub, email, name |
| Kakao OAuth2 | id, nickname, email |
| Naver OAuth2 | id, nickname, email |

### 2.5 주요 엔드포인트

| 경로 | 기능 |
|------|------|
| `/problem` | 문제 목록 (난이도/키워드 필터링) |
| `/question/list` | Q&A 게시판 (5개 필드 검색) |
| `/unsolved-hgu/ranking` | 유저 랭킹 (기여도/풀이수) |
| `/user/signup`, `/user/login` | 회원가입, 로그인 |
| `/user/detail/{username}` | 마이페이지 |
| `/user/emails/*` | 이메일 인증 (REST API) |

---

## 3. 프론트엔드 (Thymeleaf)

### 3.1 템플릿 구조

```
layout.html (마스터 레이아웃)
├── navbar.html (네비게이션)
├── footer.html (푸터)
├── form_errors.html (폼 에러 표시)
│
├── home.html          # 메인 페이지 (난이도별 카드, Lottie 애니메이션)
├── problem.html       # 문제 목록 (검색, 난이도 필터, 저장 버튼)
├── question_list.html # Q&A 목록 (검색, 페이지네이션)
├── question_detail.html # 질문 상세 (답변, 추천)
├── ranking.html       # 유저 랭킹 (기여도/풀이수 정렬)
├── mypage.html        # 마이페이지 (작성글, 저장 문제)
├── login_form.html    # 로그인 (OAuth2 버튼 포함)
├── signup_form.html   # 회원가입 (이메일 인증)
└── privacy_policy.html, terms.html # 약관
```

### 3.2 AJAX 호출

| 엔드포인트 | 기능 | 방식 |
|-----------|------|------|
| `POST /user/emails/verification-request` | 이메일 인증 코드 발송 | JSON |
| `POST /user/emails/verification` | 인증 코드 확인 | JSON |
| `POST /user/save/problem/{id}` | 문제 저장/해제 토글 | JSON |
| `POST /user/update/username` | 닉네임 변경 | JSON |

---

## 4. 데이터 파이프라인 (Python)

**기술 스택:** Python, RabbitMQ (pika), MySQL (mysql-connector), requests

### 4.1 모듈 구조

```
server/
├── broker.py              # Producer - 갱신 대상 선정 → 메세지 발행
├── consumer.py            # Consumer - 메세지 소비 → DB 갱신
├── api_controller.py      # solved.ac API 호출 (페이지네이션 처리)
├── data_transformer.py    # API/DB 데이터 비교 및 변환
├── db_controller.py       # MySQL CRUD (INSERT IGNORE, DUPLICATE UPDATE)
├── rabbit_mq_controller.py # RabbitMQ 연결/발행/소비/ACK
├── selenium_croller.py    # 백준 문제 크롤러 (독립 스크립트)
└── utility.py             # config 로딩, 유틸리티 함수
```

### 4.2 Broker (Producer) 동작

```
1. DataTransformer.choose_user() 호출
   ├── DB에서 유저 목록 조회
   ├── solved.ac API에서 유저 목록 조회
   └── 차집합 계산: (API 유저) - (DB 유저) = 갱신 대상

2. 갱신 대상마다 메세지 생성
   └── 형식: {username: solved_count}

3. RabbitMQ 큐에 메세지 발행
   └── Durable 큐, Persistent 메세지
```

### 4.3 Consumer (Subscriber) 동작

```
1. RabbitMQ에서 메세지 수신 (pop_queue)
   └── prefetch_count=1, auto_ack=False

2. 메세지 파싱 → username, solved_count 추출

3. 스킵 판단 (_should_skip)
   ├── DB의 풀이 수 == 메세지의 풀이 수 → 스킵
   └── 다르면 → API 호출하여 DB 갱신

4. DB 갱신
   ├── solved.ac API에서 유저의 풀이 문제 목록 조회
   └── INSERT IGNORE로 user_solved 테이블에 삽입

5. ACK 전송 → 메세지 큐에서 제거

6. 속도 제한: API 호출 500회 도달 시 중단
```

### 4.4 메세지 형식

```json
// RabbitMQ 메세지 본문
{"alice": 42}
// → username: "alice", solved_count: 42
```

### 4.5 DB 삽입 전략

| 전략 | 용도 |
|------|------|
| INSERT IGNORE | 풀이 문제 삽입 (중복 무시) |
| INSERT...ON DUPLICATE KEY UPDATE | 유저 정보 갱신 (있으면 업데이트) |

---

## 5. 전체 데이터 흐름

```
[solved.ac API]
      │
      │ (1) Broker: 유저 목록 조회
      v
[DataTransformer] ── API 유저 vs DB 유저 비교
      │
      │ (2) 갱신 대상 메세지 발행
      v
[RabbitMQ Queue] ── {username: solved_count}
      │
      │ (3) Consumer: 메세지 소비
      v
[solved.ac API] ── 유저별 풀이 문제 조회
      │
      │ (4) INSERT IGNORE
      v
[MySQL]
  ├── user_info 테이블 (유저 통계)
  ├── user_info_problem_solved 테이블 (풀이 기록)
  └── problem, tag 테이블 (문제 정보)
      │
      │ (5) JPA/Hibernate 조회
      v
[Spring Boot]
  ├── ProblemService: 문제 필터링 (난이도, 키워드, 풀이 여부)
  ├── UserInfoService: 랭킹 계산 (기여도, 풀이수)
  └── QuestionService: Q&A 검색
      │
      │ (6) Thymeleaf 렌더링
      v
[브라우저] ── 문제 목록, 랭킹, Q&A, 마이페이지
```

---

## 6. 인프라 의존성

| 서비스 | 용도 | 기본 설정 |
|--------|------|-----------|
| MySQL | 메인 데이터베이스 | localhost:3306 |
| Redis | 이메일 인증 코드 저장 (TTL 5분) | localhost:6379 |
| RabbitMQ | 파이프라인 메세지 큐 | localhost |
| Gmail SMTP | 이메일 인증 발송 | smtp.gmail.com:587 |
| Google OAuth2 | 소셜 로그인 | - |
| Kakao OAuth2 | 소셜 로그인 | - |
| Naver OAuth2 | 소셜 로그인 | - |

---

## 7. 설정 파일

| 파일 | 위치 | 설명 |
|------|------|------|
| `application.properties` | hguAPIs/src/main/resources/ | DB, JPA, SMTP, Redis 설정 |
| `application-oauth.properties` | hguAPIs/src/main/resources/ | OAuth2 클라이언트 설정 |
| `application-prod.properties` | hguAPIs/src/main/resources/ | 운영 환경 설정 |
| `config.json` | pipeline-unsolved-list-APIs-project/ | API URL, DB, RabbitMQ 설정 |
