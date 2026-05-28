# Newstagram 프로젝트 포트폴리오

## 1. 프로젝트 한 줄 소개

**Newstagram**은 여러 언론사의 RSS 뉴스 기사를 수집하고, 기사 임베딩과 사용자 행동 로그를 기반으로 실시간 이슈, 개인 맞춤 기사, 자연어 프롬프트 기반 뉴스 검색을 제공하는 지능형 뉴스 큐레이션 서비스입니다.

사용자는 흩어진 언론사 사이트를 직접 방문하지 않아도 최신 이슈를 기간별로 확인할 수 있고, 본인의 클릭 패턴이나 직접 입력한 문장에 가까운 뉴스를 추천받을 수 있습니다.

## 2. 내 역할

| 구분 | 담당 내용 |
| --- | --- |
| PM | 프로젝트 기획, 주요 기능 우선순위 결정, 전체 아키텍처 방향성 결정, API/화면 흐름 조율, 일정 및 리스크 관리 |
| Backend | RSS 기사 수집, 기사 정규화 및 DB 저장, AI 임베딩 파이프라인, PgVector 기반 유사도 검색, Spring Batch/Quartz 기반 자동화, 실패 대비 재시도 및 로그 설계 |
| Frontend | Vue 기반 라우팅/페이지 구조 설계, 백엔드 API 연동, 기사 피드/프롬프트 검색/개인 추천 화면 구현, 인증 토큰 처리 및 사용자 상태 관리 |

## 3. 프로젝트 목적

뉴스 소비자는 매일 여러 언론사와 포털을 오가며 동일한 이슈를 반복적으로 확인합니다. 이 과정에서 정보 탐색 시간이 길어지고, 관심사 기반으로 뉴스를 정리하기 어렵다는 문제가 있습니다.

Newstagram은 다음 문제를 해결하는 것을 목표로 했습니다.

- 여러 언론사의 뉴스를 한곳에서 통합 조회
- 실시간, 일간, 주간 단위로 현재 주목받는 이슈 파악
- 사용자의 기사 클릭 로그를 분석해 개인화 뉴스 추천
- 사용자가 입력한 자연어 문장과 유사한 기사 검색
- RSS 수집, 임베딩, 클러스터링을 자동화해 운영 부담 최소화

## 4. 주요 기능

### 4-1. 핫 이슈 뉴스 피드

여러 언론사에서 수집한 기사 중 유사한 내용을 하나의 이슈 그룹으로 묶고, 기간별로 많이 다뤄지는 이슈를 제공합니다.

- `REALTIME`, `DAILY`, `WEEKLY` 기간 구분
- 기사 임베딩 기반 유사 기사 클러스터링
- 클러스터 크기와 대표 벡터 기준으로 이슈 순위화
- 그룹별 기사 중 일부를 랜덤 노출해 같은 이슈 안에서도 다양한 기사 제공
- Redis look-aside 캐싱으로 반복 조회 비용 감소

### 4-2. 개인 맞춤 뉴스 피드

사용자가 클릭한 기사 로그를 기반으로 사용자의 관심 벡터를 갱신하고, 해당 벡터와 유사한 기사를 추천합니다.

- 기사 클릭 이벤트 수집
- Kafka 기반 비동기 로그 처리
- 최근 클릭 로그 30개를 기준으로 사용자 선호 벡터 계산
- 시간 감쇠 가중치를 적용해 최근 행동의 영향도를 높임
- PgVector 유사도 검색으로 개인 추천 기사 조회

### 4-3. 자연어 프롬프트 뉴스 검색

사용자가 원하는 뉴스 주제를 문장으로 입력하면, 문장의 의미와 가까운 기사를 검색합니다.

예시:

- "요즘 KBO에서 기아 타이거즈 관련한 소식이 궁금해."
- "최근 인공지능 산업 동향 알려줘."

구현 방식:

- Komoran 형태소 분석으로 핵심 키워드 추출
- 날짜 표현을 감지해 검색 기간 설정
- LLM을 활용해 관련 카테고리 추론
- 검색 문장을 임베딩으로 변환
- PgVector의 코사인 거리 기반 검색
- 검색 이력 저장, 조회, 삭제 지원

### 4-4. 콜드 스타트 대응

신규 사용자는 클릭 로그가 없어 개인화 추천이 어렵기 때문에, 초기 설문에서 선택한 카테고리를 기반으로 초기 인터랙션 로그를 생성합니다.

- 관심 카테고리 선택
- 선택 카테고리별 최신 기사 조회
- 초기 사용자 행동 로그 생성
- 사용자 선호 벡터 생성
- 이후 개인 추천 피드로 연결

### 4-5. 회원 및 인증

- 일반 회원가입/로그인
- Google OAuth2 로그인
- JWT Access Token/Refresh Token 구조
- Redis 기반 Refresh Token 저장 및 검증
- 휴대폰 인증, 이메일 찾기, 비밀번호 재설정
- Vue Axios interceptor 및 토큰 갱신 처리

## 5. 전체 시스템 구조

![전체 시스템 아키텍처](newstagram_back-main/docs/images/FullSystemArchitecture.png)

### Backend 멀티 모듈 구조

```text
newstagram_back-main/
├── api-server/          # 사용자 요청을 처리하는 메인 REST API 서버
├── logging-server/      # Kafka 기반 사용자 행동 로그 처리 서버
├── rss-collector/       # RSS 수집, 기사 임베딩, 클러스터링 배치 서버
├── newstagram-domain/   # 공통 도메인 엔티티 및 유틸 모듈
├── sql/                 # DB 초기화 스크립트
└── docker-compose.yml   # PostgreSQL/PgVector, Redis, Kafka 로컬 환경
```

### Frontend 구조

```text
newstagram_front-main/
├── src/api/             # 백엔드 API 호출 모듈
├── src/router/          # Vue Router 라우팅
├── src/stores/          # Pinia 상태 관리
├── src/pages/           # 화면 단위 Vue 컴포넌트
├── src/components/      # 공통 Header, Navi, Footer 등
├── vite.config.js       # Vite 설정 및 API 프록시
├── Dockerfile           # Vue 빌드 후 Nginx 배포
└── nginx.conf           # SPA 라우팅 대응 Nginx 설정
```

## 6. 기술 스택

### Backend

| 영역 | 기술 |
| --- | --- |
| Language | Java 17 |
| Framework | Spring Boot 3.5.8 |
| Build | Gradle |
| API | Spring Web, Spring Validation, Swagger/OpenAPI |
| Persistence | PostgreSQL, PgVector, Spring Data JPA, MyBatis |
| Cache | Redis |
| Messaging | Kafka, Kafka DLT |
| Auth | Spring Security, JWT, OAuth2 Client |
| Batch | Spring Batch, Quartz |
| RSS/Parsing | Rome, Rome Modules, Jsoup |
| NLP/AI | OpenAI-compatible Embedding API, GPT 계열 LLM API, Komoran |
| ML/Clustering | Smile DBSCAN |
| Infra | Docker, Docker Compose |

### Frontend

| 영역 | 기술 |
| --- | --- |
| Framework | Vue 3 |
| Build | Vite |
| Routing | Vue Router |
| State | Pinia |
| HTTP | Axios |
| UI | Component-based Vue SFC, responsive layout |
| Deploy | Docker multi-stage build, Nginx |

## 7. 데이터베이스 설계

![ERD](newstagram_back-main/docs/images/ERD.png)

### 주요 테이블

| 테이블 | 역할 |
| --- | --- |
| `users` | 사용자 계정, 인증 정보, 사용자 선호 임베딩 저장 |
| `news_categories` | 뉴스 카테고리 |
| `news_sources` | 언론사/뉴스 소스 |
| `rss_feeds` | 언론사 및 카테고리별 RSS 피드 URL |
| `articles` | 수집된 기사, 본문, 썸네일, 발행일, 1536차원 임베딩 |
| `period_recommendations` | 기간별 클러스터링 결과와 이슈 랭킹 |
| `user_interaction_logs` | 기사 클릭 및 초기 설문 기반 사용자 행동 로그 |
| `recommendation_logs` | 추천 결과 추적용 로그 |
| `system_job_logs` | RSS 수집, 임베딩, 클러스터링 배치 실행 로그 |

### PgVector 활용

기사와 사용자 선호도를 벡터로 저장하기 위해 PostgreSQL에 PgVector 확장을 사용했습니다.

- `articles.embedding`: 기사 본문/제목 기반 임베딩
- `users.preference_embedding`: 사용자 행동 로그 기반 선호 벡터
- 벡터 차원: `1536`
- 유사도 검색: PgVector 거리 연산자 `<=>` 활용

## 8. 화면 및 시스템 흐름

### 주요 화면

| 화면 | 설명 |
| --- | --- |
| 메인 화면 | 실시간/일간/주간 핫 이슈 피드 제공 |
| 로그인/회원가입 화면 | 일반 로그인, 회원가입, 소셜 로그인 진입 |
| 프롬프트 검색 화면 | 사용자가 입력한 자연어 문장을 기반으로 기사 검색 |
| 개인 추천 화면 | 사용자 선호 벡터 기반 추천 기사 제공 |
| 설문 화면 | 신규 사용자 콜드 스타트 해결을 위한 관심 카테고리 선택 |

참고 이미지:

![메인 화면](newstagram_back-main/docs/images/Main.png)

![로그인 화면](newstagram_back-main/docs/images/Login.png)

![회원가입 화면](newstagram_back-main/docs/images/SignUp.png)

![프롬프트 검색 화면](newstagram_back-main/docs/images/SearchPrompt.png)

### Use Case

![Use Case](newstagram_back-main/docs/images/UseCase.png)

### 주요 시퀀스 다이어그램

| 흐름 | 다이어그램 |
| --- | --- |
| 소셜 로그인 | ![Social Login](newstagram_back-main/docs/images/SocialLogin.png) |
| 일반 로그인 | ![General Login](newstagram_back-main/docs/images/GeneralLogin.png) |
| 문자 인증 | ![Letter Authentication](newstagram_back-main/docs/images/LetterAuthentication.png) |
| 이메일 인증 | ![Email Authentication](newstagram_back-main/docs/images/EmailAuthentication.png) |
| 공용 로깅 서비스 | ![Logging Public Service](newstagram_back-main/docs/images/LoggingPublicService.png) |
| 콜드 스타트 | ![Cold Start](newstagram_back-main/docs/images/ColdStart.png) |
| 기사 클릭 로깅 | ![Click Article Logging](newstagram_back-main/docs/images/ClickArticleLogging.png) |
| 이슈 기사 조회 | ![Issue Article Inquiry](newstagram_back-main/docs/images/IssueArticleInquiry.png) |
| 프롬프트 검색 | ![Prompt Search](newstagram_back-main/docs/images/SearchPrompt.png) |
| RSS 기사 수집 | ![RSS Article Collection](newstagram_back-main/docs/images/RssArticleCollection.png) |
| 클러스터링 | ![Clustering](newstagram_back-main/docs/images/Clustering.png) |

## 9. 내가 구현한 Backend 핵심 내용

### 9-1. RSS 기사 수집 파이프라인

담당 경로:

- `rss-collector/src/main/java/com/newstagram/rss/service/RssArticleServiceImpl.java`
- `rss-collector/src/main/java/com/newstagram/rss/batch/NewsSourceItemWriter.java`
- `rss-collector/src/main/java/com/newstagram/rss/batch/RssBatchConfig.java`

구현 내용:

- 활성화된 RSS Feed 목록 조회
- Rome `SyndFeedInput`으로 RSS XML 파싱
- RSS entry를 내부 `Article` 모델로 변환
- 제목, 본문, 설명, URL, 작성자, 발행일, 카테고리, 언론사 정보 매핑
- RSS enclosures, MediaRSS metadata, HTML `<img>` 태그에서 썸네일 추출
- tracking pixel, beacon, logger URL 필터링
- 기사 URL unique constraint 기반 중복 저장 방지
- feed 단위 처리 결과를 성공/스킵/실패로 구분

수집 흐름:

```text
RSS Feed 목록 조회
→ Feed URL 요청
→ RSS Entry 파싱
→ 기사 본문/설명 정규화
→ 썸네일 추출
→ Article 변환
→ DB insert
→ 중복 URL은 skip
→ system_job_logs 기록
```

### 9-2. 기사 임베딩 파이프라인

담당 경로:

- `rss-collector/src/main/java/com/newstagram/rss/service/ArticleVectorServiceImpl.java`
- `rss-collector/src/main/java/com/newstagram/rss/batch/ArticleEmbeddingItemWriter.java`

구현 내용:

- 임베딩이 없는 기사만 조회
- 기사 제목과 본문을 결합해 임베딩 입력 텍스트 생성
- 제목 정규화로 언론사 접두어, 괄호, 불필요한 태그 제거
- 본문이 긴 경우 최대 길이 제한으로 토큰 비용 방어
- batch size를 적용해 여러 기사를 묶어 임베딩 API 호출
- `text-embedding-3-large` 모델 사용
- `1536` 차원 벡터 생성
- PgVector literal 형식으로 변환해 DB 저장
- 응답 개수 불일치, 빈 임베딩, API 오류 등 예외 상황 로깅

임베딩 입력 예시:

```text
TITLE: 정규화된 기사 제목

CONTENT: 기사 본문 일부
```

임베딩 저장 흐름:

```text
미임베딩 기사 조회
→ 제목/본문 정규화
→ batch 단위 AI 임베딩 API 호출
→ 응답 index 기준 정렬
→ PgVector literal 변환
→ articles.embedding 업데이트
→ 성공/실패 결과 기록
```

### 9-3. Spring Batch 기반 자동화

담당 경로:

- `rss-collector/src/main/java/com/newstagram/rss/batch/RssBatchConfig.java`
- `rss-collector/src/main/java/com/newstagram/rss/service/RssBatchService.java`

Batch Job 구성:

```text
rssMasterJob
├── rssPerSourceStep
│   ├── NewsSourceItemReader
│   └── NewsSourceItemWriter
└── embeddingPerSourceStep
    ├── NewsSourceItemReader
    └── ArticleEmbeddingItemWriter
```

설계 포인트:

- 언론사 source 단위로 chunk 처리
- RSS 수집과 임베딩을 독립 step으로 분리
- `ThreadPoolTaskExecutor`로 병렬 처리
- core/max pool size `7`로 여러 언론사 수집 병렬화
- 매 실행마다 `runId` JobParameter를 부여해 반복 실행 가능
- Job 실행 결과로 `jobId`, `status`, `startTime`, `endTime` 반환

### 9-4. Quartz 기반 스케줄링

담당 경로:

- `rss-collector/src/main/java/com/newstagram/rss/config/QuartzConfig.java`
- `rss-collector/src/main/java/com/newstagram/rss/batch/RssBatchQuartzJob.java`

스케줄:

```text
cron: 0 0 0,6,9,12,15,18,21 * * ?
timezone: Asia/Seoul
```

실행 시각:

- 00:00
- 06:00
- 09:00
- 12:00
- 15:00
- 18:00
- 21:00

스케줄 동작:

```text
Quartz Trigger
→ RssBatchQuartzJob 실행
→ rssMasterJob 실행
→ RSS 기사 수집
→ 기사 임베딩
→ REALTIME 클러스터링 실행
→ 00시에는 DAILY 클러스터링 실행
→ 월요일 00시에는 WEEKLY 클러스터링 실행
```

운영 안정성:

- `@DisallowConcurrentExecution`으로 동일 Job 중복 실행 방지
- 실행 시각은 `Asia/Seoul` 기준으로 계산
- RSS Batch 실패 시 `JobExecutionException`으로 Quartz에 실패 전파

### 9-5. 실패 대비 및 운영 로그

담당 경로:

- `NewsSourceItemWriter.java`
- `ArticleEmbeddingItemWriter.java`
- `ClusteringOrchestratorService.java`
- `system_job_logs` 테이블

실패 대비 전략:

- RSS feed 단위 최대 재시도 `2회`
- 임베딩 source 단위 최대 재시도 `2회`
- 클러스터링 period 단위 최대 재시도 `2회`
- 실패해도 전체 파이프라인이 무한 중단되지 않도록 `Failed` 또는 `Skipped`로 상태 기록
- 실행 시작/종료 시각, 처리 건수, 재시도 횟수, 마지막 오류 메시지 저장

로그 예시로 관리한 항목:

- `jobName`
- `runDate`
- `status`
- `message`
- `itemsProcessed`
- `retryCount`
- `startedAt`
- `endedAt`
- `feedId`

이 설계로 단순히 배치를 실행하는 수준이 아니라, 운영 중 어떤 feed나 period에서 실패했는지 추적할 수 있도록 했습니다.

### 9-6. 클러스터링 기반 핫 이슈 생성

담당 경로:

- `rss-collector/src/main/java/com/newstagram/rss/clustering/service/ClusteringService.java`
- `rss-collector/src/main/java/com/newstagram/rss/clustering/service/DbscanService.java`
- `rss-collector/src/main/java/com/newstagram/rss/clustering/service/ClusteringOrchestratorService.java`

구현 내용:

- 기간별 기사 조회
- DB에 저장된 PgVector literal을 `double[]`로 변환
- Smile DBSCAN으로 유사 기사 군집화
- 코사인 거리 기반 커스텀 distance 적용
- noise cluster 제거
- 클러스터별 medoid 대표 벡터 계산
- 클러스터 크기가 큰 순서로 이슈 랭킹 부여
- 각 기사와 대표 벡터 사이의 거리 score 저장

클러스터링 파라미터:

```text
eps = 0.4
minPts = 3
distance = cosine distance
```

클러스터링 흐름:

```text
기간 계산
→ 해당 기간 기사 조회
→ embedding literal 파싱
→ DBSCAN 수행
→ noise 제거
→ cluster별 article index 그룹핑
→ cluster별 medoid 계산
→ cluster size 기준 ranking
→ period_recommendations 저장
```

이슈 랭킹 기준:

1. 같은 이슈로 묶인 기사 수가 많을수록 상위
2. 같은 이슈 안에서는 medoid와의 거리를 score로 저장
3. 하나의 사건을 여러 언론사가 다루는 정도를 기준으로 핫 이슈를 판단

### 9-7. 기간별 이슈 조회 API

담당 경로:

- `api-server/src/main/java/com/newstagram/api/article/service/HotIssueService.java`
- `api-server/src/main/java/com/newstagram/api/article/controller/ArticleController.java`

주요 API:

```http
GET /api/article/hot-issues/{periodType}/{cursor}
```

구현 내용:

- `REALTIME`, `DAILY`, `WEEKLY` periodType 지원
- 현재 기간의 클러스터링 결과 조회
- 현재 기간 결과가 없으면 이전 기간 결과로 fallback
- group ranking 기준 cursor pagination
- group별 노출 기사 랜덤 선택
- 기사 상세는 Redis look-aside 캐싱 적용
- 응답에 `hasNext`, `nextCursor` 포함

캐싱 전략:

- 핫 이슈 결과 key: `hot-issue:{periodType}:{periodStart}:{cursor}`
- 기사 상세 key: `article:{articleId}`

### 9-8. 프롬프트 검색 API

담당 경로:

- `api-server/src/main/java/com/newstagram/api/article/service/SearchService.java`
- `api-server/src/main/java/com/newstagram/api/article/controller/SearchController.java`
- `api-server/src/main/java/com/newstagram/api/article/repository/ArticleRepository.java`

주요 API:

```http
POST /api/v1/search
GET /api/v1/search/history
DELETE /api/v1/search/history
PUT /api/v1/search/history
GET /api/v1/search/preference
```

검색 파이프라인:

```text
사용자 자연어 입력
→ Komoran 형태소 분석
→ 검색 키워드 추출
→ 날짜 표현 분석
→ LLM 카테고리 분석
→ 검색어 임베딩 생성
→ PgVector 후보 기사 조회
→ 키워드 포함 여부 필터링
→ 최신순 정렬
→ pagination
→ 검색 이력 저장
```

기술적 포인트:

- `text-embedding-3-small` 모델로 검색어 임베딩 생성
- `gpt-4o-mini` 모델로 관련 카테고리 분석
- `@Cacheable`로 검색 결과와 키워드 임베딩 캐싱
- authenticated search는 strict threshold `0.80` 적용
- 후보 기사 limit을 크게 잡고 메모리에서 추가 필터링
- 검색 history는 사용자별 최신 이력 중심으로 관리

### 9-9. 사용자 행동 로그 및 개인화 추천

담당 경로:

- `api-server/src/main/java/com/newstagram/api/logging/aspect/ClickTrackingAspect.java`
- `api-server/src/main/java/com/newstagram/api/logging/model/service/KafkaProducerService.java`
- `logging-server/src/main/java/com/newstagram/logging/feature/articleclick/ArticleClickConsumer.java`
- `logging-server/src/main/java/com/newstagram/logging/feature/articleclick/UserPreferenceService.java`

로그 처리 흐름:

```text
Frontend 기사 클릭
→ POST /api/logging/click
→ @CollectLog AOP intercept
→ userId, articleId, sessionId, userAgent, ipAddress 수집
→ Kafka topic log.article.click 발행
→ logging-server consumer 수신
→ user_interaction_logs 저장
→ 사용자 선호 벡터 업데이트
```

사용자 선호 벡터 업데이트 방식:

- 최근 클릭 로그 30개 조회
- 각 클릭 기사 임베딩 조회
- 클릭 시점이 최근일수록 높은 가중치 적용
- 시간 가중치: `exp(-0.02 * hoursDiff)`
- weighted average로 사용자 preference vector 계산
- `users.preference_embedding` 갱신

이 설계를 통해 사용자가 뉴스를 볼수록 추천 결과가 점진적으로 개인화되도록 만들었습니다.

### 9-10. Kafka DLT와 비동기 로그 안정성

담당 경로:

- `logging-server/src/main/java/com/newstagram/logging/global/config/KafkaConsumerConfig.java`
- `logging-server/src/main/java/com/newstagram/logging/feature/deadletter/DeadLetterEventListener.java`

구현 내용:

- 클릭 로그 topic: `log.article.click`
- 클릭 로그 DLT: `log.article.click.DLT`
- 설문 로그 topic: `log.survey.submit`
- 설문 로그 DLT: `log.survey.submit.DLT`
- consumer 오류 발생 시 fixed backoff 적용
- 30초 간격, 최대 5회 재시도
- 모든 재시도 실패 시 Dead Letter Topic으로 이동
- DLT listener에서 최종 실패 메시지 로깅

## 10. 내가 구현한 Frontend 핵심 내용

### 10-1. 전체 Vue 앱 구조

담당 경로:

- `src/App.vue`
- `src/router/index.js`
- `src/components/Header.vue`
- `src/components/Navi.vue`
- `src/components/Footer.vue`

구현 내용:

- 로그인 여부에 따라 레이아웃 표시 분기
- Header, Side Navigation, Content, Footer 구조 설계
- 모바일에서 side navigation drawer 처리
- route meta 기반 특정 페이지 layout 숨김 처리
- 전역 프롬프트 입력을 prompt 검색 페이지로 연결
- 다크/라이트 테마 상태 적용

### 10-2. API 연동 구조

담당 경로:

- `src/api/index.js`
- `src/api/refreshToken.js`
- `src/api/HomeApi.js`
- `src/api/PromptApi.js`
- `src/api/MyApi.js`
- `src/api/LogApi.js`
- `src/api/UserApi.js`

구현 내용:

- Axios instance 생성
- `.env` 기반 API URL 관리
- 요청 interceptor에서 Authorization token 자동 주입
- 401 응답 시 logout 및 로그인 페이지 이동
- API 모듈을 기능별로 분리해 화면 컴포넌트와 결합도 낮춤
- 주요 API 호출 전 refresh token 검증 수행

### 10-3. 핫 이슈 화면

담당 경로:

- `src/pages/home/Home.vue`
- `src/api/HomeApi.js`
- `src/stores/homePeriodStore.js`

구현 내용:

- `REALTIME`, `DAILY`, `WEEKLY` 탭 전환
- 선택 기간 상태를 Pinia store로 유지
- cursor 기반 더보기 로딩
- 기사 카드 리스트 렌더링
- 썸네일 없을 때 placeholder 처리
- 기사 클릭 시 로그 API 호출
- iframe modal로 기사 원문 표시
- 일부 iframe 제한 언론사는 새 창 안내 처리

### 10-4. 프롬프트 검색 화면

담당 경로:

- `src/pages/prompt/Prompt.vue`
- `src/api/PromptApi.js`
- `src/stores/promptStore.js`

구현 내용:

- route query `q`를 감지해 검색 실행
- 검색 결과 pagination 및 더보기
- 검색 이력 조회
- 검색 이력 localStorage fallback
- 검색 이력 삭제 모드
- 검색 결과 기사 클릭 로그 연동
- 기사 원문 modal 표시

### 10-5. 개인 추천 화면

담당 경로:

- `src/pages/my/My.vue`
- `src/api/MyApi.js`
- `src/components/Navi.vue`

구현 내용:

- `/v1/search/preference` API와 연동
- 사용자 선호 벡터가 없는 경우 설문 페이지로 유도
- 개인 추천 기사 infinite loading
- 기사 클릭 로그 연동
- 추천 결과 empty/loading/error 상태 처리

### 10-6. 인증 및 사용자 상태 관리

담당 경로:

- `src/stores/user.js`
- `src/api/UserApi.js`
- `src/api/refreshToken.js`

구현 내용:

- Access Token, Refresh Token localStorage 저장
- 로그인 상태 computed 관리
- 일반 로그인, 소셜 로그인 결과 저장
- 앱 초기 진입 시 localStorage에서 사용자 상태 복원
- 만료 토큰 검증 및 갱신
- 인증 실패 시 사용자 상태 초기화 및 로그인 페이지 이동

### 10-7. 배포 대응

담당 경로:

- `Dockerfile`
- `nginx.conf`
- `vite.config.js`

구현 내용:

- Node 20 기반 build stage
- Nginx 기반 production stage
- build argument로 API URL 주입
- SPA routing을 위한 `try_files $uri $uri/ /index.html`
- Vite dev server proxy로 로컬 백엔드 API 연동

## 11. PM으로서 진행한 의사결정

### 11-1. 기능 우선순위 결정

초기 요구사항을 사용자 가치와 구현 리스크 기준으로 나누고, 우선순위를 정했습니다.

상위 우선순위:

- RSS 기사 수집
- 유사 기사 그룹핑
- 사용자 클릭 로그 저장
- 사용자 관심사 기반 기사 추천
- 회원가입/로그인

중간 우선순위:

- 카테고리별 기사 필터링
- 실시간/일간/주간 이슈 구분
- 사용자 입력 문장 기반 기사 추천
- 대표 기사 선정

하위 우선순위:

- 프로필 정보 수정
- 기사 조회 기록 삭제

### 11-2. 아키텍처 의사결정

단일 서버에 모든 기능을 넣으면 배치, 로그 처리, 사용자 API가 서로 영향을 줄 수 있다고 판단했습니다. 그래서 서버를 다음처럼 분리했습니다.

- `api-server`: 사용자 요청과 화면 API 처리
- `rss-collector`: RSS 수집, 임베딩, 클러스터링처럼 무거운 백그라운드 작업 처리
- `logging-server`: Kafka 기반 비동기 로그 처리
- `newstagram-domain`: 공통 엔티티와 period 계산 로직 공유

이 구조는 다음 장점이 있습니다.

- RSS 수집 실패가 사용자 API 전체 장애로 번지는 위험 감소
- 로그 수집과 개인화 계산을 비동기로 처리해 API 응답 지연 방지
- 배치 서버만 별도로 스케일하거나 스케줄 변경 가능
- 공통 도메인 모듈로 엔티티 중복 최소화

### 11-3. 데이터 흐름 결정

핵심 데이터 흐름을 두 축으로 나눴습니다.

기사 생산 파이프라인:

```text
RSS 수집
→ Article 저장
→ Article 임베딩 저장
→ 기간별 클러스터링
→ Hot Issue API 제공
```

개인화 파이프라인:

```text
기사 노출
→ 사용자 클릭
→ Kafka 로그 발행
→ 로그 저장
→ 사용자 선호 벡터 업데이트
→ 개인 추천 API 제공
```

### 11-4. API 계약과 화면 흐름 조율

프론트엔드와 백엔드가 동시에 개발될 수 있도록 API 단위를 화면 기준으로 정리했습니다.

- 메인 피드: 기간별 hot issue API
- 프롬프트 페이지: 검색 API, 검색 이력 API
- My 페이지: 개인 추천 API
- 기사 클릭: logging API
- 설문 페이지: 카테고리 조회, 설문 제출, 임베딩 초기화 여부 확인
- 인증 페이지: 회원가입, 로그인, 토큰 검증, 비밀번호 재설정

### 11-5. 리스크 관리

프로젝트에서 가장 큰 리스크는 RSS 수집 실패, 외부 AI API 실패, Kafka consumer 실패였습니다.

대응 방식:

- RSS feed 단위 재시도와 실패 로그
- 임베딩 source 단위 재시도와 skip 처리
- 클러스터링 period 단위 재시도와 skip 처리
- Kafka consumer fixed backoff 재시도
- 최종 실패 메시지를 DLT로 이동
- 배치 실행 결과를 `system_job_logs`로 추적

## 12. 주요 API 정리

### 기사/이슈 API

| Method | Endpoint | 설명 |
| --- | --- | --- |
| `GET` | `/api/article/hot-issues/{periodType}/{cursor}` | 기간별 핫 이슈 기사 조회 |
| `GET` | `/api/article/detail/{id}` | 기사 상세 조회 |
| `GET` | `/api/article/home-issues` | 기본 홈 이슈 조회 |

### 검색/추천 API

| Method | Endpoint | 설명 |
| --- | --- | --- |
| `POST` | `/api/v1/search` | 자연어 프롬프트 기반 기사 검색 |
| `POST` | `/api/v1/search/test` | 비인증 테스트 검색 |
| `GET` | `/api/v1/search/history` | 검색 이력 조회 |
| `DELETE` | `/api/v1/search/history` | 검색 이력 삭제 |
| `PUT` | `/api/v1/search/history` | 검색 이력 수정 |
| `GET` | `/api/v1/search/preference` | 사용자 선호 벡터 기반 추천 기사 조회 |

### 로그/설문 API

| Method | Endpoint | 설명 |
| --- | --- | --- |
| `POST` | `/api/logging/click` | 기사 클릭 로그 수집 |
| `GET` | `/api/survey/categories` | 초기 설문 카테고리 조회 |
| `POST` | `/api/survey/submit` | 초기 설문 제출 |
| `GET` | `/api/survey/users/embedding` | 사용자 선호 임베딩 존재 여부 조회 |

### 인증 API

| Method | Endpoint | 설명 |
| --- | --- | --- |
| `POST` | `/api/auth/login` | 일반 로그인 |
| `POST` | `/api/auth/token` | Access Token 검증 및 재발급 |
| `POST` | `/api/auth/refresh` | Refresh Token 기반 토큰 재발급 |
| `POST` | `/api/auth/logout` | 로그아웃 |
| `POST` | `/api/auth/password/reset-request` | 비밀번호 재설정 요청 |
| `POST` | `/api/auth/password/reset` | 비밀번호 재설정 |
| `POST` | `/api/auth/signup/phone-verification/request` | 회원가입 휴대폰 인증 요청 |
| `POST` | `/api/auth/signup/phone-verification/verify` | 회원가입 휴대폰 인증 검증 |

## 13. 핵심 기술적 고민과 해결

### 고민 1. RSS 수집 실패가 전체 파이프라인을 멈추지 않게 하기

RSS 피드는 외부 언론사 서버에 의존하기 때문에 언제든 응답 실패, XML 파싱 실패, 네트워크 오류가 발생할 수 있습니다.

해결:

- feed 단위로 처리 단위를 쪼갬
- feed별 재시도 횟수 관리
- 실패한 feed는 실패 로그를 남기고 다음 feed 처리
- 성공/실패 상태를 `system_job_logs`에 기록

### 고민 2. 외부 AI 임베딩 API 비용과 안정성 관리

기사 본문 전체를 그대로 임베딩하면 토큰 비용이 커지고, 응답 시간이 증가할 수 있습니다.

해결:

- 제목과 본문을 결합하되 본문 길이 제한 적용
- batch size를 적용해 API 호출 횟수 최적화
- 응답 index 기준 정렬로 기사와 임베딩 매칭 안정화
- API 4xx, body empty, data empty, count mismatch를 분리해 처리

### 고민 3. 같은 이슈를 어떻게 묶을 것인가

단순 키워드 기반 그룹핑은 표현이 조금만 달라져도 같은 사건을 놓칠 수 있습니다.

해결:

- 기사 의미를 임베딩 벡터로 표현
- 코사인 거리 기반 DBSCAN 사용
- 클러스터 수를 미리 정하지 않아도 되는 방식 선택
- noise를 제거해 이슈성이 약한 기사 제외
- 클러스터 크기를 기준으로 이슈 중요도 판단

### 고민 4. 개인화 추천을 즉시 반영하면서도 API 응답을 늦추지 않기

사용자가 기사를 클릭할 때마다 선호 벡터를 계산하면 사용자 API 응답이 느려질 수 있습니다.

해결:

- 클릭 API는 Kafka에 이벤트 발행
- logging-server에서 비동기로 로그 저장 및 벡터 갱신
- 사용자 API와 개인화 계산을 분리
- 실패 시 Kafka retry와 DLT로 보강

### 고민 5. 신규 사용자에게 추천할 데이터가 없는 문제

처음 가입한 사용자는 클릭 로그가 없으므로 개인 추천을 제공할 수 없습니다.

해결:

- 초기 관심 카테고리 설문 제공
- 선택 카테고리별 최신 기사 기반 초기 인터랙션 로그 생성
- 기존 개인화 로직을 그대로 사용해 초기 선호 벡터 생성

### 고민 6. 프론트엔드와 백엔드 동시 개발

화면과 API가 동시에 변하면 개발 충돌이 잦아집니다.

해결:

- 화면 단위로 API 모듈 분리
- `HomeApi`, `PromptApi`, `MyApi`, `LogApi`, `UserApi`로 책임 분리
- route와 store를 기준으로 페이지 상태 관리
- 공통 axios instance와 token refresh 로직을 중앙화

## 14. 프로젝트에서 드러나는 내 강점

### 14-1. 데이터 파이프라인 설계 역량

RSS 수집, AI 임베딩, 벡터 저장, 클러스터링, API 제공까지 이어지는 데이터 흐름을 직접 구현했습니다. 단순 CRUD가 아니라 외부 데이터와 AI 모델, 배치 처리, 검색 API를 연결하는 end-to-end 파이프라인을 설계했습니다.

### 14-2. 실패를 전제로 한 백엔드 구현

외부 RSS, AI API, Kafka consumer처럼 실패 가능성이 큰 지점을 고려해 재시도, skip, DLT, job log를 구현했습니다. 운영 관점에서 어떤 작업이 언제 실패했는지 추적할 수 있도록 설계했습니다.

### 14-3. AI 기능의 제품화 경험

AI 모델을 단순 호출하는 데서 끝내지 않고, 사용자의 실제 기능으로 연결했습니다.

- 기사 임베딩 → 유사 기사 클러스터링
- 사용자 클릭 로그 → 선호 벡터 생성
- 자연어 검색어 → 임베딩 검색
- LLM 카테고리 분석 → 검색 정확도 보완

### 14-4. Frontend와 Backend를 함께 보는 시야

API 응답 구조, cursor pagination, token refresh, 검색 이력, 기사 클릭 로그까지 화면과 서버의 연결 지점을 직접 구현했습니다. 덕분에 API 설계가 실제 사용자 흐름에 맞게 이어지도록 조율할 수 있었습니다.

### 14-5. PM으로서의 제품 판단

프로젝트 전체에서 어떤 기능을 먼저 만들고, 어떤 구조로 나누며, 어떤 실패를 대비해야 하는지 결정했습니다. 팀 프로젝트에서 기술 구현뿐 아니라 제품 방향과 개발 우선순위를 함께 책임졌습니다.

## 15. 면접에서 설명하기 좋은 구현 포인트

### 질문: RSS 수집은 어떻게 자동화했나요?

Quartz가 정해진 시간에 `RssBatchQuartzJob`을 실행하고, 이 Job이 Spring Batch의 `rssMasterJob`을 실행합니다. Batch는 RSS 수집 step과 임베딩 step으로 나뉘며, source 단위로 병렬 처리됩니다. 수집 후에는 실시간 클러스터링을 실행하고, 00시에는 일간 클러스터링, 월요일 00시에는 주간 클러스터링까지 실행합니다.

### 질문: 기사 중복은 어떻게 방지했나요?

기사 URL을 unique key로 관리하고, insert 시 conflict가 발생하면 skip 처리했습니다. RSS 특성상 같은 기사가 반복해서 들어올 수 있기 때문에 URL 기준 중복 제거가 가장 안정적이라고 판단했습니다.

### 질문: 핫 이슈는 어떻게 계산했나요?

기사를 임베딩 벡터로 변환한 뒤 DBSCAN으로 의미적으로 가까운 기사들을 묶었습니다. 같은 이슈를 여러 언론사가 다루면 클러스터 크기가 커지므로, 클러스터 크기 기준으로 이슈 랭킹을 만들었습니다. 각 클러스터 내부에서는 medoid를 계산해 대표 벡터와 기사 간 거리를 score로 저장했습니다.

### 질문: 개인화 추천은 어떻게 동작하나요?

사용자가 기사를 클릭하면 클릭 로그가 Kafka로 발행됩니다. logging-server가 이 메시지를 소비해 로그를 저장하고, 최근 클릭 로그 30개를 기준으로 사용자의 선호 벡터를 갱신합니다. 이때 오래된 클릭보다 최근 클릭의 영향이 더 크도록 시간 감쇠 가중치를 적용했습니다. 이후 추천 API는 사용자 선호 벡터와 기사 임베딩의 거리로 유사 기사를 조회합니다.

### 질문: AI 모델은 어디에 사용했나요?

세 군데에 사용했습니다.

- 기사 제목/본문을 임베딩해 유사 기사 클러스터링에 사용
- 사용자 검색 문장을 임베딩해 의미 기반 검색에 사용
- LLM으로 검색 문장의 관련 카테고리를 추론해 검색 범위를 보정

### 질문: 실패 처리는 어떻게 했나요?

RSS 수집, 임베딩, 클러스터링 모두 재시도 횟수를 두고 실패 상태를 로그로 남겼습니다. Kafka consumer는 fixed backoff로 재시도하고, 끝까지 실패하면 DLT로 이동합니다. Batch Job은 `system_job_logs`에 상태, 처리 건수, 재시도 횟수, 오류 메시지를 기록합니다.

### 질문: PM으로서 가장 중요하게 본 것은 무엇인가요?

기능을 많이 만드는 것보다, 사용자가 뉴스를 발견하는 흐름이 자연스럽게 이어지는 것을 중요하게 봤습니다. 그래서 메인 핫 이슈, 프롬프트 검색, 개인 추천을 각각 독립된 기능이 아니라 하나의 뉴스 탐색 경험으로 연결하려고 했습니다. 기술적으로도 RSS 수집, 임베딩, 클러스터링, 로그 기반 추천이 끊기지 않는 데이터 흐름을 우선순위로 잡았습니다.

## 16. 이력서용 요약 문장

### 한 줄 버전

RSS 뉴스 수집, AI 임베딩, DBSCAN 클러스터링, Kafka 기반 행동 로그 분석을 결합한 개인화 뉴스 큐레이션 서비스 Newstagram에서 PM 및 Fullstack 개발을 담당했습니다.

### Backend 중심 버전

Spring Batch와 Quartz를 활용해 RSS 기사 수집, AI 임베딩, 기간별 클러스터링을 자동화하고, PgVector 기반 유사도 검색과 Kafka 기반 사용자 행동 로그 파이프라인을 구현했습니다.

### Frontend 중심 버전

Vue 3, Pinia, Vue Router, Axios 기반으로 핫 이슈 피드, 자연어 프롬프트 검색, 개인 추천 피드 화면을 구현하고 백엔드 API 및 인증 토큰 흐름을 연동했습니다.

### PM 중심 버전

프로젝트 PM으로서 기능 우선순위, 데이터 파이프라인 구조, API 계약, 화면 흐름, 실패 대응 전략을 결정하고 RSS 수집부터 개인화 추천까지 이어지는 전체 제품 흐름을 설계했습니다.

## 17. 포트폴리오 핵심 문단

Newstagram은 단순 뉴스 목록 서비스가 아니라, 외부 RSS 데이터 수집부터 AI 임베딩, 클러스터링, 사용자 행동 로그 분석, 개인화 추천까지 하나의 자동화 파이프라인으로 연결한 프로젝트입니다. 저는 PM과 Fullstack 개발자로 참여해 프로젝트의 기능 우선순위와 아키텍처를 결정하고, 백엔드에서는 RSS 기사 수집과 임베딩, Spring Batch/Quartz 기반 스케줄링, DBSCAN 클러스터링, Kafka 기반 로그 처리와 개인화 추천 흐름을 구현했습니다. 프론트엔드에서는 Vue 기반 페이지 구조와 API 연동을 담당해 실시간/일간/주간 이슈 피드, 프롬프트 검색, 개인 추천 화면이 실제 사용자 흐름으로 이어지도록 만들었습니다.

특히 이 프로젝트에서 가장 집중한 부분은 "AI 기능을 서비스 기능으로 안정적으로 연결하는 것"이었습니다. 기사 임베딩은 단순 저장이 아니라 유사 기사 클러스터링과 의미 기반 검색에 활용했고, 사용자 클릭 로그는 Kafka를 통해 비동기로 처리하여 선호 벡터를 갱신했습니다. 또한 RSS 수집과 임베딩, 클러스터링 과정에는 재시도와 상태 로그를 추가해 외부 API나 RSS feed 실패에도 운영자가 실패 지점을 추적할 수 있도록 설계했습니다.

## 18. 성과 및 배운 점

### 성과

- RSS 수집, 임베딩, 클러스터링으로 이어지는 자동화 데이터 파이프라인 구축
- 의미 기반 기사 검색과 기간별 이슈 클러스터링 구현
- 사용자 행동 로그 기반 개인화 추천 구조 구현
- Kafka와 DLT를 활용한 비동기 로그 처리 안정성 확보
- Vue 기반 사용자 화면과 백엔드 API 연동
- PM 역할로 기능 우선순위와 전체 개발 흐름 조율

### 배운 점

- AI API는 호출 자체보다 입력 정규화, 비용 관리, 실패 처리, 후속 데이터 구조가 더 중요하다는 점
- 배치 파이프라인은 성공 경로보다 실패 경로를 먼저 설계해야 운영 가능성이 높아진다는 점
- 추천 서비스는 데이터 수집, 로그 처리, 벡터 계산, 조회 API가 모두 연결되어야 사용자 가치가 생긴다는 점
- Fullstack 개발에서는 API 명세와 화면 상태 관리가 초기에 맞춰져야 개발 속도가 안정된다는 점
- PM 역할에서는 기술적 완성도와 사용자 경험 사이의 우선순위를 계속 조율해야 한다는 점

## 19. 추가로 기입하면 좋은 정보

아래 항목은 코드와 README에서 명확히 확인되지 않은 정보라, 최종 포트폴리오 제출 전에 실제 값으로 채우면 좋습니다.

| 항목 | 기입 예시 |
| --- | --- |
| 프로젝트 기간 | 2026.01 ~ 2026.02 |
| 팀 규모 | 6명 |
| 담당 비율 | PM 30%, Backend 50%, Frontend 20% |
| 배포 URL | 실제 서비스 URL |
| GitHub URL | 저장소 URL |
| 시연 영상 | YouTube/Google Drive 링크 |
| 수집 언론사 수 | 실제 RSS feed 수 |
| 일 평균 수집 기사 수 | 실제 운영/테스트 수치 |
| 임베딩 처리량 | batch 1회 평균 처리 기사 수 |
| 성능 개선 수치 | 캐싱 적용 전후 응답 시간 등 |
