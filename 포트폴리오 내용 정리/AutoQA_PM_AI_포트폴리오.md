# AutoQA 포트폴리오 정리

> 담당 역할: PM / AI  
> 프로젝트: AI 기반 웹 QA 자동화 SaaS  
> 기간: 2026.04.06 ~ 2026.06.02  
> 팀 규모: 6명  
> 서비스 URL: https://www.autoqa.site

---

## 1. 프로젝트 한 줄 소개

AutoQA는 사용자가 테스트 대상 URL과 자연어 요구사항을 입력하면, 웹 페이지를 자동으로 탐색하고, AI가 실행 가능한 QA 시나리오를 생성한 뒤, Playwright 기반 브라우저 자동 실행 결과를 실시간 리포트로 제공하는 AI 기반 웹 QA 자동화 플랫폼입니다.

기존 QA 자동화는 QA 엔지니어가 직접 테스트 케이스를 작성하고, 개발자가 Playwright/Cypress 같은 E2E 스크립트를 구현해야 했습니다. AutoQA는 이 과정을 `웹 분석 -> AI 시나리오 생성 -> 시나리오 검토/수정 -> 브라우저 자동 실행 -> 결과 리포트/Jira 이슈화` 흐름으로 연결해, 비개발자도 URL과 자연어만으로 QA 자동화를 시작할 수 있게 만드는 것을 목표로 했습니다.

---

## 2. 내가 맡은 역할 요약

저는 이 프로젝트에서 PM과 AI 파트를 담당했습니다.

PM으로서는 서비스의 문제 정의, 핵심 사용자 흐름, 파이프라인 구조, 팀별 역할 분담, 기술 의사결정, 단계별 우선순위 조정을 주도했습니다. 단순히 기능 목록을 나누는 수준이 아니라, “AI가 생성한 결과를 실제 브라우저에서 실행 가능한 품질로 만들기 위해 어떤 데이터 계약과 검증 절차가 필요한가”를 중심으로 전체 시스템의 경계를 설계했습니다.

AI 담당자로서는 웹 분석 산출물을 학습 데이터로 전환하는 데이터 수집/정제 구조를 만들고, Gemma4 기반 초기 실험부터 Qwen3-8B 기반 production fine-tune 파이프라인까지 발전시켰습니다. 또한 SFT, DPO, 평가, vLLM 기반 추론, guided JSON 출력 제약, mode collapse 진단 및 보정 계획까지 AI 모델 개발 전 과정을 직접 수행했습니다.

### 핵심 담당 범위

| 구분 | 담당 내용 |
|---|---|
| PM | 서비스 문제 정의, 핵심 기능 우선순위화, 팀별 책임 분리, QA 파이프라인 설계 방향 결정 |
| PM | SQS/Redis/S3 기반 비동기 파이프라인의 역할 경계 정의 |
| PM | AI 생성 결과를 실행 가능한 QA suite로 만들기 위한 contract, validator, retry/fixer 정책 수립 |
| PM | 데모 흐름, 사용자 시나리오, 서비스 차별점, 기술 발표 구조 정리 |
| AI | 웹 분석 결과 기반 학습 데이터 수집 및 정제 |
| AI | Gemma4 LoRA/QLoRA 초기 학습 실험 및 OOM 문제 분석 |
| AI | Qwen3-8B 전환 의사결정 및 v2/v3 fine-tune 파이프라인 구축 |
| AI | SFT/DPO 데이터셋 생성, teacher data 보강, contract-aware negative pair 생성 |
| AI | Vertex AI 기반 학습 job, checkpoint/GCS sync, preemption 대응 구조 설계 |
| AI | vLLM/guided_json 기반 production 추론 구조 및 평가 지표 설계 |
| AI | NAVER mode collapse 등 모델 실패 모드 진단 및 보정 계획 수립 |

---

## 3. 문제 정의

웹 서비스 QA 자동화의 가장 큰 병목은 “테스트할 기능을 사람이 이해하고, 케이스를 쓰고, 자동화 코드로 옮기는 과정”입니다. 특히 일반적인 웹 서비스는 화면 구조가 매번 다르고, 로그인 여부에 따라 접근 가능한 페이지가 달라지며, 버튼이나 링크가 명확한 HTML semantic tag로 되어 있지 않은 경우도 많습니다.

AutoQA는 다음 문제를 해결하고자 했습니다.

1. QA 시나리오 작성 시간이 길다.
2. 테스트 자동화 코드를 직접 작성할 수 있는 인력이 제한적이다.
3. DOM 구조와 실제 사용자 행동 사이의 간극 때문에 자동화 스크립트가 쉽게 깨진다.
4. AI가 자연어로 테스트 케이스를 만들어도 실제 Playwright 실행 계약을 만족하지 못하면 쓸 수 없다.
5. QA 실행 결과가 실패했을 때, 실패 원인을 추적하고 리포트/Jira 이슈로 넘기는 과정이 번거롭다.

따라서 프로젝트의 핵심 목표는 “그럴듯한 테스트 설명”을 만드는 것이 아니라, Playwright Worker가 실제 Chromium 브라우저에서 실행할 수 있는 구조화 QA suite JSON을 안정적으로 생성하는 것이었습니다.

---

## 4. 서비스 핵심 흐름

```text
사용자 입력
  - 테스트 대상 URL
  - 테스트 목적/요구사항
  - 선택적으로 로그인 정보

        |
        v

1. Playwright 기반 웹 분석
  - BFS 사이트 탐색
  - DOM/상호작용 후보 추출
  - 스크린샷/어노테이션 수집
  - site-summary.json, analysis-summary.json 생성

        |
        v

2. AI QA 시나리오 생성
  - 웹 분석 산출물 입력
  - Qwen3-8B fine-tuned 모델 추론
  - QA Execution Contract를 따르는 suite JSON 생성

        |
        v

3. 시나리오 검증/보정
  - nodeId 유효성 검사
  - step type/matcher/signal whitelist 검사
  - 잘못된 시나리오 자동 retry/fix/drop

        |
        v

4. 사용자 검토/편집
  - 생성된 시나리오 확인
  - 자연어로 추가/수정/삭제

        |
        v

5. Playwright QA 실행
  - 실제 Chromium 브라우저 실행
  - step별 상태/스크린샷/로그 기록
  - self-healing locator 및 fallback

        |
        v

6. 결과 리포트
  - 통과/실패/부분 성공 요약
  - 실패 step 스크린샷
  - S3 artifact
  - Jira 이슈 발급
```

---

## 5. 기술 스택

| 영역 | 기술 |
|---|---|
| Frontend | React, TypeScript, Vite, Tailwind CSS, Zustand, WebSocket, Three.js |
| Backend | Spring Boot, Spring Security, JWT, JPA, Redis, PostgreSQL, WebSocket |
| Playwright Worker | Node.js, Express, Playwright, AWS SDK, ioredis |
| AI Server / Worker | Python, FastAPI, vLLM, Transformers, PEFT, QLoRA, DPO |
| Model | Gemma4 실험, Qwen3-8B production 후보 |
| Queue / Storage | AWS SQS, S3, SNS, MinIO, LocalStack |
| Infra | Docker, Docker Compose, AWS EC2, ECS Fargate, ECS GPU Worker, ECR, RDS, CloudWatch, Terraform |
| CI/CD | Jenkins, Docker image build/push, ECS redeploy |
| Test / Load | Playwright, k6 |

---

## 6. PM 관점에서 설계한 핵심 의사결정

### 6.1 AI와 실행기를 분리한 하이브리드 구조

초기 기획에서 가장 중요하게 본 점은 AI에게 모든 판단을 맡기지 않는 것이었습니다. AI는 시나리오 후보를 생성하는 데 강하지만, 실제 QA 실행의 신뢰성은 규칙 기반 실행기와 검증기가 보장해야 합니다.

그래서 구조를 다음처럼 분리했습니다.

| 역할 | 책임 |
|---|---|
| AI 모델 | 분석 결과를 바탕으로 QA suite JSON 생성 |
| Backend validator | suite contract 검증, nodeId 검증, retry/fix/drop 판단 |
| Playwright Worker | 실제 브라우저 실행, locator resolution, assertion, artifact 생성 |
| Redis/WebSocket | 사용자에게 실시간 진행 상태 전달 |
| SQS result queue | terminal event의 source of truth |

이 결정 덕분에 AI가 잘못된 JSON을 생성하더라도 바로 실행되지 않고, Backend 검증과 fixer를 통과한 결과만 사용자 검토 단계로 넘어가도록 만들 수 있었습니다.

### 6.2 SQS와 Redis의 역할 분리

비동기 작업 처리와 실시간 UX 이벤트를 같은 메시징 수단으로 처리하면 장애 추적과 재처리가 어려워진다고 판단했습니다. 그래서 SQS와 Redis를 명확히 분리했습니다.

| 메시징 수단 | 사용 목적 |
|---|---|
| SQS | 분석, AI 생성, QA 실행, 결과 이벤트의 비동기 작업 분배 |
| Redis Pub/Sub | 사용자 화면에 보여줄 실시간 진행 이벤트 |
| Redis 상태 키 | 현재 pipeline state snapshot 저장 |
| S3 | 큰 분석 결과, QA suite, 실행 리포트, 스크린샷, 영상 저장 |

SQS는 at-least-once delivery를 전제로 retry와 idempotency를 설계했고, Redis/WebSocket은 화면 갱신용으로만 사용했습니다. 이 구조는 “화면 이벤트가 누락되어도 최종 상태는 SQS result queue와 Redis state로 복구 가능해야 한다”는 원칙을 세운 결과입니다.

### 6.3 3단계 표준 QA 파이프라인 결정

초기에는 페이지별 분석 작업을 더 잘게 쪼갤 수 있었지만, 프로젝트 기간과 안정성을 고려해 표준 파이프라인을 3단계로 정의했습니다.

```text
PAGE_ANALYSIS
  Backend -> Playwright Worker
  URL 1개를 받아 사이트 탐색 + 요소 분석

SCENARIO_GENERATE
  Backend -> AI Worker
  분석 결과를 받아 QA suite 생성

QA_EXECUTE
  Backend -> Playwright Worker
  QA suite 전체 실행
```

이 결정은 팀 구현 범위를 명확히 하고, SQS payload와 result event 계약을 단순화했습니다. 대용량 데이터는 SQS에 직접 넣지 않고 S3 URL만 전달하는 방식으로 통일했습니다.

### 6.4 site-summary 우선 정책

AI가 단일 페이지 정보만 보고 시나리오를 생성하면 사이트 전체 맥락을 놓치기 쉽습니다. 그래서 AI 입력은 `scenario-brief.json` 하나만으로는 생성할 수 없고, 반드시 `site-summary.json`을 먼저 보도록 정책을 잡았습니다.

`site-summary.json`은 사이트 전체 기능 패턴, 시나리오 클러스터, 자동 생성 가능 후보, 재검토 필요 후보를 요약합니다. 이를 통해 AI가 단순히 눈앞의 DOM 후보를 클릭하는 것이 아니라, 사이트 전체 기능 분포를 보고 smoke/regression/e2e/content/auth/external/capture 같은 QA 범주를 균형 있게 생성하도록 유도했습니다.

### 6.5 생성 가능 후보 게이팅

AI가 아무 nodeId나 만들거나, 컨테이너 div를 클릭 대상으로 삼으면 Playwright 실행은 실패합니다. 그래서 분석 단계에서 각 action candidate에 `autoScenarioEligible`과 confidence score를 부여하고, AI는 safe candidate만 사용하도록 contract를 정의했습니다.

대표 정책은 다음과 같습니다.

| 정책 | 내용 |
|---|---|
| nodeId 제한 | scenario step의 `targetRef.nodeId`는 같은 page brief에 존재하는 후보만 사용 |
| 페이지 정합성 | scenario의 `goto.url`과 target node가 수집된 페이지 path가 일치해야 함 |
| 자동 생성 금지 | `avoid_auto_generation` 패턴은 자동 시나리오 생성 금지 |
| 인증 분리 | anonymous/authenticated scenario의 후보와 세션을 분리 |
| 민감 행위 금지 | 결제, 삭제, 권한 변경, 민원 제출 등 destructive action 생성 금지 |
| popup 처리 | 새 탭 이동은 current page URL이 아니라 popup signal로 검증 |
| form 처리 | form 자체를 click하지 않고 input fill + Enter 또는 submit button 사용 |

이 정책은 AI 모델 학습 데이터, 프롬프트, validator, Playwright Runner 방어 로직에 모두 반영했습니다.

### 6.6 사용자 경험 우선순위

PM 관점에서는 “AI가 뒤에서 오래 걸리는 작업”을 사용자가 기다릴 수 있도록 실시간 가시성을 주는 것이 중요했습니다. 그래서 다음 UX를 주요 요구사항으로 잡았습니다.

1. 분석 중인 페이지가 실시간으로 화면에 나타날 것
2. 페이지별 스크린샷을 S3 presigned URL로 즉시 확인할 수 있을 것
3. AI 생성 단계와 QA 실행 단계가 끊겨 보이지 않을 것
4. step 단위 실행 상태와 실패 이유를 볼 수 있을 것
5. 최종 리포트에서 실패 스크린샷과 요약 지표를 바로 확인할 수 있을 것
6. 실패 시나리오를 Jira 이슈로 발급할 수 있을 것

---

## 7. 시스템 아키텍처

```text
Frontend
  React / TypeScript
  URL 입력, 진행 화면, 시나리오 검토, 실행 리포트
        |
        | REST / WebSocket
        v
Backend
  Spring Boot
  인증, 프로젝트/리포트 관리, QA pipeline state machine
  SQS 발행, result queue 소비, Redis/WebSocket 중계
        |
        | SQS
        +--------------------------+
        |                          |
        v                          v
Playwright Worker              AI Worker
  PAGE_ANALYSIS                 SCENARIO_GENERATE
  QA_EXECUTE                    Qwen3-8B 추론
  Chromium 실행                 QA suite 생성
        |                          |
        +------------+-------------+
                     |
                     v
                  S3 / Redis
  final-report.json, site-summary.json, QA suite, suite-report.json,
  screenshots, videos, realtime events
```

---

## 8. AI 파트 전체 구조

AI 파트의 목표는 `site-summary.json + analysis-summary.json`을 입력으로 받아 Playwright Worker가 실행할 수 있는 QA suite JSON을 생성하는 모델을 만드는 것이었습니다.

일반 LLM에게 “이 사이트 테스트해줘”라고 요청하면 그럴듯한 테스트 설명은 만들 수 있지만, AutoQA에서 필요한 출력은 훨씬 엄격했습니다.

### AI 출력이 만족해야 하는 조건

1. JSON이 완전히 파싱 가능해야 한다.
2. suite/scenario/step 구조가 schema를 만족해야 한다.
3. step type, matcher, signal, capture kind가 whitelist 안에 있어야 한다.
4. `targetRef.nodeId`는 분석 결과에 실제 존재해야 한다.
5. nodeId의 page prefix와 scenario 시작 URL이 일치해야 한다.
6. popup, navigation, select, input maxlength 같은 Playwright 실행 계약을 지켜야 한다.
7. 단순 `waitFor`만 반복하지 않고 click/fill/expect/capture/scroll/multi-page flow를 다양하게 생성해야 한다.
8. 한국어 사이트와 글로벌 사이트 모두에서 안정적으로 동작해야 한다.
9. 긴 입력과 긴 출력 suite를 처리할 수 있어야 한다.

---

## 9. AI 데이터 수집 및 전처리

### 9.1 웹 분석 산출물 기반 데이터 수집

AI 학습 데이터는 단순 텍스트 QA 데이터가 아니라, 실제 웹 분석 결과와 실행 가능한 QA suite를 쌍으로 구성해야 했습니다. 이를 위해 Playwright 분석 결과에서 다음 파일들을 학습 입력으로 활용했습니다.

| 파일 | 역할 |
|---|---|
| `final-report.json` | 페이지별 DOM, action candidate, screenshot, 분석 상태 |
| `analysis-summary.json` | 여러 페이지의 page brief 집계 |
| `scenario-brief.json` | 페이지 단위 시나리오 후보 요약 |
| `site-summary.json` | 사이트 전체 기능 패턴과 시나리오 클러스터 요약 |
| QA scenario JSON | 모델이 생성해야 하는 정답 suite |

웹 분석은 BFS 기반으로 사이트를 탐색하고, URL 정규화, DOM 후보 수집, trigger probing, screenshot annotation, auth/anonymous 분리 등을 수행했습니다. AI 입장에서는 이 원본 데이터를 그대로 넣으면 context가 너무 길어지기 때문에, 학습 가능한 compact representation으로 줄이는 것이 핵심 과제였습니다.

### 9.2 입력 압축

v3 데이터 분석 결과, raw `site-summary.json + analysis-summary.json`은 대부분 그대로 학습하기 어려운 수준으로 컸습니다.

| 항목 | 관찰 |
|---|---|
| 139개 사이트 중 132개 | 10K tokens 초과 |
| 139개 사이트 중 107개 | 30K tokens 초과 |
| raw token 중앙값 | 약 164K tokens |
| 최대 raw token | 약 2.14M tokens |

따라서 학습 입력을 compact format으로 다시 설계했습니다.

핵심 전략은 다음과 같습니다.

1. 전체 HTML을 버리고 AI에게 필요한 기능 후보만 유지
2. nodeId, tag, role, purpose, importance, text, targetUrl 중심으로 축약
3. key 이름을 짧게 줄여 token 사용량 절감
4. 페이지별 후보 수 cap 적용
5. `autoScenarioEligible`과 confidence score를 유지해 생성 가능 후보 구분
6. `functionalPaths`와 scenario cluster 정보를 보존해 기능 커버리지 유지

그 결과 v3에서는 입력을 약 16배 압축해, median 9~10K tokens 수준의 compact input을 만들었습니다. 이 결정은 GPU 메모리 문제와 모델 품질 문제를 동시에 줄인 가장 중요한 전처리 개선이었습니다.

### 9.3 데이터 정제

초기 v3 데이터는 약 139개 사이트, 14K+ 시나리오 규모였습니다. 하지만 모든 시나리오가 학습에 적합한 것은 아니었습니다.

정제 기준은 다음과 같았습니다.

| 정제 항목 | 처리 |
|---|---|
| `avoid_auto_generation` 패턴 | 자동 생성 학습에서 제외, DPO rejected 자산으로 보존 |
| `autoScenarioEligible=false` target | 학습 제외 |
| page-target path mismatch | path-template 정규화 |
| deprecated category naming | 카테고리 명명 통일 |
| nodeId hallucination 집중 사이트 | drop 또는 재생성 후보로 분리 |
| thin data 사이트 | 별도 격리 |
| `.bak` 구버전 시나리오 | DPO chosen/rejected 자산으로 활용 |

v3 정제 결과는 다음과 같이 정리했습니다.

| 데이터 구분 | 규모 |
|---|---:|
| 원본 v3 시나리오 | 약 14,614개 |
| clean 시나리오 | 약 12,709개 |
| avoided 시나리오 | 약 1,522개 |
| ineligible 시나리오 | 약 330개 |
| DPO negative pair | 약 7,791 pair |

---

## 10. 모델 개발 히스토리

### 10.1 1차: Gemma4 기반 LoRA/QLoRA 실험

초기에는 `google/gemma-4-E4B-it` 기반으로 Vertex AI에서 LoRA/QLoRA fine-tuning을 시도했습니다.

초기 3단계 학습 목표는 다음과 같았습니다.

| 단계 | 목적 | 데이터 | 결과 |
|---|---|---:|---|
| Stage 1 | 일반 웹 액션 이해 | Mind2Web 약 917 train / 55 val | 웹 액션 sequence JSON 생성 적응 |
| Stage 2 | 페이지 분석 결과 기반 QA suite 생성 | Gmarket 중심 reference QA | contract 기반 suite 생성 학습 |
| Stage 3 | 범용 QA scenario catalog 학습 | general scenario 약 1,800 train / 200 val | 도메인 독립 QA archetype 보강 |

이 단계에서 `train.py`, `dataset.py`, `config.py`, `prepare_mind2web_data.py`, `vertex_jobs/*.yaml.template`, `scripts/render_yamls.py` 등을 구성했습니다.

### 10.2 Gemma4 OOM과 구조 문제

Gemma4 E4B로 32K context 학습을 시도하면서 A100 40GB 환경에서 반복적인 OOM이 발생했습니다. 단순히 `max_seq_length`를 낮추는 것으로는 해결되지 않았고, 실제 병목은 긴 입력/출력에서 발생하는 attention/logits 메모리였습니다.

적용한 개선은 다음과 같습니다.

1. LoRA target을 `all-linear`에서 language model attention/MLP 계층으로 제한
2. vision/audio tower에 LoRA가 붙어 device map을 깨뜨리는 문제 제거
3. SDPA 사용
4. Liger fused linear cross entropy 적용 시도
5. SwiGLU, chunked CE, gradient checkpointing, KV cache 비활성화
6. site-summary와 page brief를 compact format으로 압축
7. full context 대신 QA contract와 eligible candidate 중심 입력으로 재설계

이 실험에서 얻은 가장 큰 결론은 “학습 코드 최적화보다 데이터 표현 설계가 더 중요하다”는 것이었습니다. 모델이 봐야 할 정보가 정제되지 않으면 GPU 메모리도 부족하고, 출력 품질도 안정화되지 않았습니다.

### 10.3 2차: Qwen3-8B로 전환

Gemma4 실험 이후 production 후보를 Qwen3-8B로 전환했습니다.

전환 이유는 다음과 같습니다.

1. 한국어와 구조화 JSON 생성에서 Qwen3-8B가 더 유리하다고 판단
2. Gemma4 multimodal architecture로 인한 LoRA/device map 리스크 제거
3. vLLM 기반 serving과 긴 출력 평가로 이어가기 쉬움
4. AutoQA의 핵심 사용처가 한국어 웹 서비스 QA였음

v2에서는 Qwen3-8B + QLoRA Stage A SFT를 구성하고, thinking-aware/mixed 데이터, compact contract prompt, raw output 평가 로그를 만들었습니다.

중요한 진단은 “validator pass rate 0%가 모델 실패가 아니라 평가/추론 인프라 한계일 수 있다”는 점이었습니다. HF Transformers 기반 평가에서 긴 suite JSON이 `max_new_tokens` 제한과 속도 문제에 걸렸고, 실제 raw output은 schema를 상당 부분 따르는 경우가 있었습니다.

이 경험을 바탕으로 v3에서는 `vLLM`, `guided_json`, `max_new_tokens=20K`, strict/partial JSON extractor, contract metrics를 먼저 설계했습니다.

### 10.4 3차: v3 production fine-tune 파이프라인

v3는 production을 목표로 한 Qwen3-8B 단일 트랙 fine-tune 파이프라인입니다.

전체 흐름은 다음과 같습니다.

```text
원본 사이트 분석 데이터
  -> minimal cleansing
  -> compact input 생성
  -> LLM teacher로 고품질 QA suite 보강
  -> rule-based DPO negative 생성
  -> Stage A SFT
  -> Stage C DPO
  -> Stage D vLLM guided_json 추론
```

v3의 핵심 설계는 다음과 같습니다.

| 설계 | 내용 |
|---|---|
| Track A 단독 | Qwen3-8B 단일 production 후보 |
| Facet-conditional training | `core_flows`, `interactive`, `cross_cutting` facet hint 사용 |
| Chunking | 긴 suite 출력이 max sequence를 넘지 않도록 분할 |
| Max sequence | 32K context 학습 |
| Max new tokens | 긴 suite 생성을 위해 20K 출력 대응 |
| QLoRA | NF4 기반 parameter-efficient tuning |
| Weighted loss | teacher/full/chunk 등 샘플 중요도 반영 |
| NEFTune | embedding noise로 generalization 개선 |
| Schema-aware loss mask | JSON 구조 토큰에 대한 학습 안정화 |
| DPO always-run | SFT만으로 막기 어려운 contract violation을 preference로 학습 |
| Guided JSON | production 추론에서 JSON schema 강제 |

---

## 11. v3 학습 데이터 구성

v3 SFT 데이터는 clean data와 teacher data를 함께 사용했습니다.

대표 split은 다음과 같습니다.

| Split | rows | 구성 |
|---|---:|---|
| train | 1,968 | clean 1,632 / teacher 336 |
| val | 212 | clean 92 / teacher 120 |
| test | 708 | clean 300 / teacher 408 |

facet 분포는 `core_flows`, `interactive`, `cross_cutting`으로 나누었습니다. 이 구조는 같은 site input에서 서로 다른 part output을 학습할 때 발생하는 ambiguity를 줄이기 위한 결정이었습니다.

기존 데이터는 사이트당 3개 part 파일로 구성되어 있었습니다.

| Part | Facet | 주요 카테고리 |
|---|---|---|
| part1 | core_flows | smoke, regression, e2e |
| part2 | interactive | auth, content, form, data variable, press key, select/check |
| part3 | cross_cutting | capture compare, deferred dynamic, external navigation |

이 3개 part를 하나로 합치면 출력 길이가 너무 길어지고, 같은 input에 여러 output이 대응되는 문제가 생겼습니다. 그래서 part를 별도 학습 row로 유지하되 `[FACET]` hint를 입력에 추가했습니다.

---

## 12. SFT 학습 구현

Stage A SFT는 Qwen3-8B + QLoRA 기반으로 구현했습니다.

학습 스크립트의 주요 기능은 다음과 같습니다.

| 기능 | 설명 |
|---|---|
| QLoRA | 4-bit NF4 quantization + bf16 compute |
| LoRA target | attention/MLP projection 중심 |
| weighted loss | `meta.sample_weight`를 loss에 반영 |
| gradient checkpointing | 32K context 메모리 절감 |
| resume auto-detect | latest checkpoint 자동 탐지 |
| GCS sync callback | checkpoint와 final adapter를 GCS에 동기화 |
| preemption 대응 | FLEX_START 중단 시 GCS checkpoint에서 재시작 |
| NEFTune | embedding noise alpha 적용 |
| schema-aware loss mask | JSON 구조 토큰 학습 안정화 |

Stage A 학습 결과는 다음과 같습니다.

| 항목 | 결과 |
|---|---|
| Vertex job | `v3-stage-a-sft` |
| 학습 시간 | 약 12시간 27분 |
| epoch | 2.0 |
| train loss | 0.6776 |
| final adapter 크기 | 약 348 MiB |
| best HP config | learning rate 2e-4, LoRA r 32, dropout 0.10 |

HP sweep는 4개 trial로 구성했고, 여러 차례 OOM/평가 문제를 겪은 뒤 최종적으로 train loss 기준으로 best config를 선택했습니다. 특히 eval 단계에서 logits tensor가 과도하게 커지는 문제가 있어, 본 학습 중 eval을 비활성화하고 generation 기반 별도 평가를 수행하는 방식으로 전환했습니다.

---

## 13. DPO 학습 구현

Stage C DPO는 contract-aware negative pair를 사용해 모델이 만들면 안 되는 패턴을 명시적으로 학습시키는 단계입니다.

SFT만으로는 “좋은 출력”을 학습할 수 있지만, 다음과 같은 “나쁜 출력”을 확실히 피하게 만들기는 어려웠습니다.

1. 존재하지 않는 nodeId 사용
2. `avoid_auto_generation` 패턴에서 시나리오 생성
3. page-target path mismatch
4. 컨테이너 node 클릭
5. popup을 current page URL 변경으로 검증
6. `autoScenarioEligible=false` 후보 사용
7. valueTemplate 또는 fill 값 오류

이를 위해 chosen/rejected pair를 구성하고, Stage A adapter 위에서 DPO를 수행했습니다.

| 항목 | 내용 |
|---|---|
| DPO pair | 약 7,791 pair |
| 입력 형식 | `{prompt, chosen, rejected}` JSONL |
| beta | 0.1 |
| learning rate | SFT보다 작은 DPO용 LR |
| adapter | Stage A LoRA adapter를 로드해 continued fine-tuning |
| GCS sync | 학습 중 checkpoint와 final adapter 업로드 |

Stage C DPO 결과는 다음과 같습니다.

| 항목 | 결과 |
|---|---|
| Vertex job | `v3-qwen3-stage-c-dpo-v2` |
| 상태 | 성공 |
| train runtime | 약 9,380초 |
| train loss | 0.1680 |
| final sync | 11 objects / 348.2 MiB 업로드 |

---

## 14. AI 평가 기준

모델 평가는 단순 train loss가 아니라 “실행 가능한 QA suite를 생성했는가”를 기준으로 설계했습니다.

주요 평가 지표는 다음과 같습니다.

| 지표 | 의미 |
|---|---|
| validator pass rate | QA suite가 schema와 contract를 통과하는 비율 |
| parse fail rate | JSON 파싱 실패율 |
| truncation rate | 출력이 잘려 suite가 완성되지 않은 비율 |
| contract violation rate | whitelist 밖 step/matcher/signal 사용률 |
| nodeId hallucination rate | 입력 분석 결과에 없는 nodeId 참조 비율 |
| step type diversity | waitFor만 반복하지 않고 다양한 step을 생성하는지 |
| category coverage | auth/search/navigation/content/capture 등 범주 다양성 |
| multi-page coverage | 여러 페이지를 엮는 flow 생성 여부 |
| avoid_auto_generation rate | 자동 생성 금지 패턴에서 생성한 비율 |
| latency | production 환경에서 suite 생성 시간 |

v3 production 목표는 다음과 같이 잡았습니다.

| 지표 | 목표 |
|---|---:|
| validator pass rate | 92% 이상 |
| nodeId hallucination rate | 3% 이하 |
| contract violation rate | 1% 이하 |
| mojibake 출력 | 0% |
| avoid_auto_generation 생성률 | 0.5% 이하 |
| L40S latency | 30 시나리오 기준 15초 이하 |

---

## 15. 추론 파이프라인 구현

AI 추론 구조는 초기에는 브라우저, 로컬 FastAPI 프록시, Vertex worker를 연결하는 방식으로 설계했고, 이후 production에서는 SQS 기반 AI Worker와 vLLM serving 구조로 확장했습니다.

### 15.1 초기 Vertex worker 구조

```text
Browser UI
  -> Local FastAPI proxy
  -> GCS queue
  -> Vertex Custom Job worker
  -> GCS results
  -> Local proxy polling
  -> Browser result
```

이 구조의 장점은 로컬에 GPU가 없어도 GCP 인증만 있으면 Vertex worker를 통해 모델 추론을 테스트할 수 있다는 점이었습니다. worker는 모델을 메모리에 유지한 채 queue를 polling하고, idle timeout이 지나면 자동 종료하도록 설계했습니다.

### 15.2 production 추론 구조

production 방향은 다음과 같습니다.

```text
Backend
  -> SQS AI queue
  -> AI GPU Worker
  -> S3에서 site-summary / analysis-summary 다운로드
  -> compact input / prompt 구성
  -> Qwen3-8B fine-tuned model 추론
  -> guided_json으로 schema 제약
  -> suite JSON S3 업로드
  -> SQS result queue로 SCENARIO_GENERATE_DONE 발행
  -> Backend validator / fixer / READY_TO_EXECUTE 전이
```

AI Worker는 다음을 책임집니다.

1. 분석 산출물 다운로드
2. site-summary와 scenario brief slim 처리
3. QA contract prompt 구성
4. 모델 추론
5. `<think>` 블록 제거 및 JSON 추출
6. normalization/post-processing
7. suite JSON 저장
8. result event 발행

### 15.3 guided JSON

v3 Stage D에서는 vLLM의 guided JSON 기능을 사용해 출력이 `suite_schema.json`을 따르도록 강제하는 구조를 설계했습니다. 이는 production에서 parse fail과 contract violation을 줄이는 핵심 장치입니다.

SFT/DPO가 모델의 생성 성향을 학습시키는 단계라면, guided JSON은 추론 시점에 구조적 실패를 줄이는 안전장치입니다.

---

## 16. QA Scenario Contract 설계

AI 모델이 생성해야 하는 출력은 단순 자연어가 아니라 QA 실행 계약을 따르는 JSON입니다.

대표 구조는 다음과 같습니다.

```json
{
  "suiteId": "SITE-2026-001",
  "title": "사이트 핵심 QA Suite",
  "description": "핵심 사용자 흐름과 회귀 검증",
  "analysisContext": {
    "analysisJobId": "job-id",
    "finalUrl": "https://example.com",
    "pageTitle": "Example"
  },
  "environment": {
    "baseURL": "https://example.com",
    "locale": "ko-KR",
    "viewport": { "width": 1920, "height": 1080 }
  },
  "defaults": {
    "timeoutMs": 10000,
    "retries": 1
  },
  "scenarios": [
    {
      "scenarioId": "EXAMPLE-SMOKE-001",
      "title": "메인 페이지 핵심 링크 이동 확인",
      "priority": "P1",
      "authMode": "anonymous",
      "preconditions": [
        { "stepId": "pre-1", "type": "goto", "url": "/" }
      ],
      "steps": [
        {
          "stepId": "s1",
          "type": "click",
          "targetRef": { "nodeId": "p001_example_:node-12" },
          "expectedSignals": [
            { "type": "urlChangedOptional", "required": false }
          ]
        }
      ]
    }
  ]
}
```

Contract에서 특히 신경 쓴 부분은 다음입니다.

| 항목 | 이유 |
|---|---|
| `authMode` | 로그인 전/후 분석 결과와 실행 세션을 분리하기 위함 |
| `targetRef.nodeId` | Playwright가 분석한 실제 요소를 안정적으로 찾기 위함 |
| `expectedSignals` | click 이후 URL 변화, popup, DOM 변화 등을 명시하기 위함 |
| `flows` | 반복 step 묶음을 재사용하기 위함 |
| `dataSets` | 동일 시나리오를 여러 입력값으로 실행하기 위함 |
| `capture` | 화면 텍스트/값/screenshot을 저장하고 후속 assertion에 활용하기 위함 |
| `reviewNeeded` / `skippedReason` | 자동 생성하면 위험한 후보를 사람이 검토하도록 넘기기 위함 |

---

## 17. 모델 실패 모드와 해결

### 17.1 긴 출력 평가 실패

v2/v3 초기에 validator pass rate가 0%로 나오는 문제가 있었습니다. 처음에는 모델이 실패한 것처럼 보였지만, raw output을 확인한 결과 `scenarios`와 `scenarioId`가 실제로 생성되어 있었습니다.

원인은 다음이었습니다.

1. 긴 suite JSON이 완전히 닫히지 않는 경우가 있음
2. extractor가 첫 번째 balanced object만 잡아 `analysisContext` 일부를 suite로 오인
3. HF Transformers 기반 generation이 너무 느리고 긴 출력에 불리함
4. max_new_tokens가 부족하면 JSON brace가 닫히기 전에 잘림

해결한 방법은 다음과 같습니다.

1. `scenarios` 키를 포함한 suite 후보를 우선 추출
2. strict JSON이 아니어도 완성된 scenario object를 부분 추출
3. `partial_scenarios_rate`, `non_strict_json_rate` 지표 추가
4. max_new_tokens를 20K로 확장
5. vLLM 기반 평가/추론으로 전환

### 17.2 OOM 문제

A100 40GB에서 32K context 학습을 하며 여러 번 OOM이 발생했습니다.

해결을 위해 다음을 적용했습니다.

1. input compact format 도입
2. max sequence를 실측 token 분포에 맞게 조정
3. gradient checkpointing 사용
4. paged_adamw_8bit optimizer 사용
5. torch empty cache step 적용
6. PyTorch CUDA allocator 설정 조정
7. eval 중 logits 생성 OOM을 피하기 위해 generation 평가를 별도 분리
8. checkpoint 저장 주기를 줄여 preemption 손실 최소화

### 17.3 NAVER mode collapse

Stage C 이후 NAVER 추론에서 mode collapse가 관찰되었습니다.

관찰된 실패는 다음과 같습니다.

| 항목 | 실패 내용 |
|---|---|
| category collapse | 60개 시나리오가 거의 모두 `auth_entry` 계열 |
| step type collapse | step이 전부 `waitFor` 위주 |
| nodeId collapse | 참조 nodeId가 2개 수준으로 축소 |
| page collapse | 여러 페이지 입력 중 `/`만 사용 |
| reasoning duplication | reasoning 문장이 반복 |

진단한 원인은 다음과 같습니다.

1. greedy decoding `temperature=0.0`이 반복 attractor를 강화
2. `_slim_site_summary`에서 `avoid_auto_generation` 패턴을 너무 일찍 제거해 카테고리 후보가 사라짐
3. teacher data 자체의 waitFor 비중이 높음
4. DPO가 안전하고 단순한 출력만 선호하도록 작용했을 가능성

보정 계획은 단계별 off-ramp 방식으로 세웠습니다.

| Phase | 목표 | 조치 |
|---|---|---|
| Phase 1 | 학습 없이 추론 보정 | temperature/top_p/repetition penalty, guided_json, summary filter 완화 |
| Phase 2 | Stage A vs Stage C 비교 | DPO가 collapse를 악화했는지 진단 |
| Phase 3 | 데이터 보강 + mini SFT | self-generated positive, 수작업 search/capture/multi-page 예제 추가 |
| Phase 4 | 게이트 평가 | 다양성, nodeId, category coverage, validator 회귀 확인 |
| Phase 5 | 추가 재학습 | 부족 카테고리 중심으로 1회 보강 |

이 경험은 모델 품질을 단순 loss로 판단하면 안 되고, 실제 생성 다양성과 contract 통과율을 함께 봐야 한다는 교훈을 남겼습니다.

---

## 18. Backend와 AI 연결부

AI가 생성한 suite는 Backend에서 바로 사용자에게 노출하지 않고 검증 절차를 거칩니다.

Backend의 QA pipeline orchestrator는 다음 흐름을 관리합니다.

```text
PAGE_ANALYSIS_DONE
  -> SCENARIO_GENERATING 상태 전이
  -> AI queue로 SCENARIO_GENERATE 발행

SCENARIO_GENERATE_DONE
  -> suiteS3Url 저장
  -> QaSuiteValidationService 검증
  -> 실패 시 AI 재요청 또는 GMS fixer
  -> 검증 통과 시 자연어 설명 캐싱
  -> READY_TO_EXECUTE 상태 전이

QA_EXECUTE_DONE
  -> suite-report.json 다운로드
  -> DB 저장
  -> COMPLETED / FAILED 상태 확정
  -> credential secret 정리
```

특히 AI 생성 결과가 검증 실패했을 때의 정책을 중요하게 설계했습니다.

1. 1~2차 실패: AI worker 재요청
2. 마지막 시도: 실패한 scenario와 validation error, 사용 가능한 element 후보를 기반으로 GMS fixer 호출
3. fixer 후에도 실패한 scenario는 drop fallback
4. 남은 scenario가 있으면 전체 실패보다 부분 성공을 우선
5. 최종적으로 READY_TO_EXECUTE 상태에서는 validation error 흔적을 제거해 프론트 혼란 방지

이 구조는 AI 출력의 불확실성을 제품 안정성으로 흡수하기 위한 장치입니다.

---

## 19. Playwright 실행과 AI 출력의 접점

AI가 만든 suite는 Playwright Worker의 실행 엔진으로 전달됩니다. 실행기는 다음 기능을 지원합니다.

| 기능 | 설명 |
|---|---|
| goto/click/fill/press/select/check/expect/capture/waitFor | 기본 step 실행 |
| flow expansion | 반복 step 묶음 재사용 |
| dataSet expansion | 데이터드리븐 QA 실행 |
| target resolution | nodeId 기반 요소 매핑 |
| self-healing locator | 실패 시 CSS/aria/text 등 fallback |
| expected signal | URL 변경, popup, DOM 변경, element visible 등 관찰 |
| artifact upload | screenshot, report, failure artifact S3 업로드 |
| realtime event | step/scenario 진행 상태 Redis publish |
| partial success | popup/document download 등 애매한 결과를 방어적으로 분류 |

AI 관점에서 중요한 점은, 모델 출력이 실행기의 계약과 정확히 맞아야 한다는 것입니다. 그래서 학습 데이터, prompt, schema, validator, runner 방어 로직을 한 세트로 설계했습니다.

---

## 20. 프로젝트 주요 기능

### 20.1 URL 기반 자동 웹 분석

- BFS 기반 사이트 탐색
- 최대 200페이지까지 분석 가능
- URL 정규화로 중복 페이지 병합
- 익명/인증 이중 크롤
- DOM 후보, 링크 관계, 스크린샷, trigger 결과 수집

### 20.2 AI QA 시나리오 생성

- Qwen3-8B fine-tuned 모델 사용
- 분석 결과 기반 QA suite JSON 생성
- smoke/regression/e2e/interactive/cross-cutting 등 범주 생성
- 자연어 요구사항 기반 custom scenario 반영
- contract validator를 통한 안전성 검증

### 20.3 시나리오 검토/편집

- 생성된 시나리오 실행 전 확인
- 자연어로 시나리오 추가/수정/삭제
- step 단위 재시도 및 보정 흐름 지원

### 20.4 실제 브라우저 QA 실행

- Playwright 기반 Chromium 실행
- step별 실행 상태 추적
- screenshot/video/report artifact 저장
- UI 반응 + 네트워크 요청/응답 검증
- console/network error matcher 지원

### 20.5 실시간 진행 표시

- Redis Pub/Sub -> Backend WebSocket -> Frontend
- 분석, AI 생성, 실행 단계별 이벤트 스트리밍
- 페이지 그래프, step 진행률, 실패 이유 표시

### 20.6 결과 리포트와 Jira 이슈화

- 통과/부분 성공/실패 summary
- failure screenshot
- flaky signal
- step duration
- Jira issue 1-click 발급
- 실패 step screenshot attachment

---

## 21. 정량적 성과

| 항목 | 성과 |
|---|---:|
| 분석 대상 사이트 규모 | 약 139개 사이트 |
| 원본 QA 시나리오 규모 | 약 14,614개 |
| 정제 후 clean 시나리오 | 약 12,709개 |
| DPO negative pair | 약 7,791개 |
| SFT train rows | 1,968 rows |
| SFT val rows | 212 rows |
| SFT test rows | 708 rows |
| input compression | median 기준 약 16배 압축 |
| Stage A SFT train loss | 0.6776 |
| Stage C DPO train loss | 0.1680 |
| Stage A adapter size | 약 348 MiB |
| HP sweep best config | lr 2e-4, LoRA r 32, dropout 0.10 |
| production 목표 validator pass rate | 92% 이상 |
| production 목표 nodeId hallucination | 3% 이하 |
| production 목표 contract violation | 1% 이하 |

---

## 22. 내가 한 PM 의사결정 상세

### 22.1 “AI 생성”이 아니라 “실행 가능한 QA 자동화”를 목표로 재정의

프로젝트 초기에 AI가 QA 시나리오를 생성하는 것만으로는 제품 가치가 부족하다고 판단했습니다. 실제 사용자는 생성된 문서를 원하는 것이 아니라, 브라우저에서 실행되고 결과가 남는 자동화 QA를 원합니다.

그래서 제품 목표를 다음처럼 재정의했습니다.

```text
Before:
  AI가 테스트 케이스를 생성한다.

After:
  AI가 Playwright 실행 계약을 만족하는 QA suite를 생성하고,
  실제 브라우저에서 실행한 결과까지 리포트로 제공한다.
```

이 정의가 이후 모든 의사결정의 기준이 되었습니다.

### 22.2 팀별 책임 경계 설정

팀원이 6명이었기 때문에, 각자의 구현 범위가 충돌하지 않도록 경계를 명확히 했습니다.

| 영역 | 책임 |
|---|---|
| Frontend | 입력/진행/검토/리포트 화면 |
| Backend | 인증, pipeline orchestration, 상태 관리, SQS/Redis/S3 연동 |
| Playwright | 웹 분석, QA 실행, artifact 생성 |
| AI | 시나리오 생성 모델, 데이터셋, 학습/추론 |
| Infra | AWS, ECS, EC2, Terraform, Jenkins |

특히 AI와 Playwright의 경계가 중요했습니다. AI는 실행 코드를 만들지 않고 suite JSON만 만들며, Playwright는 suite JSON을 해석해 실행하는 구조로 분리했습니다.

### 22.3 기술 리스크 우선순위화

프로젝트 리스크를 다음 순서로 봤습니다.

1. AI 출력이 실행 불가능할 위험
2. 분석 데이터가 너무 커서 모델 입력에 들어가지 않을 위험
3. Worker 간 비동기 pipeline이 꼬일 위험
4. 실시간 진행 상태와 실제 terminal state가 불일치할 위험
5. GPU 비용과 cold start가 커질 위험

그래서 validator, compact input, SQS result queue, Redis state, zero-idle worker를 우선 설계했습니다.

### 22.4 비용과 운영성을 고려한 zero-idle 구조

Playwright Worker와 AI GPU Worker는 항상 켜두기에는 비용 부담이 컸습니다. 그래서 작업이 없을 때는 worker를 0대에 가깝게 유지하고, 요청 시 기동하는 구조를 선택했습니다.

| Worker | 운영 전략 |
|---|---|
| Playwright Worker | ECS Fargate, SQS 메시지 기반 scale out |
| AI GPU Worker | GPU EC2 / Vertex worker, idle timeout, prebaked AMI |

AI는 모델 load 시간이 길기 때문에, worker를 완전히 매번 새로 띄우는 구조와 idle 유지 비용 사이에서 균형을 잡아야 했습니다. 초기 Vertex worker는 30분 idle timeout으로 비용을 줄였고, production에서는 GPU worker 예열/AMI 전략으로 cold start를 줄이는 방향을 잡았습니다.

---

## 23. 어려웠던 점과 해결 과정

### 23.1 “그럴듯한 JSON”과 “실행 가능한 JSON”의 차이

가장 어려웠던 점은 모델이 생성한 JSON이 문법적으로 맞아 보여도 실제 실행에는 실패할 수 있다는 점이었습니다.

예를 들어 다음과 같은 문제가 있었습니다.

1. nodeId가 실제 분석 결과에 없음
2. `/NOTICE` 페이지의 nodeId를 `/`에서 클릭하려고 함
3. popup 링크인데 current page URL 변경을 기대함
4. form을 클릭 대상으로 사용함
5. select option label을 value로 착각함
6. input maxlength를 고려하지 않음

이를 해결하기 위해 AI prompt만 수정하지 않고, 다음 계층을 모두 보강했습니다.

1. 분석 데이터에 confidence와 eligibility 추가
2. generation contract 문서화
3. 학습 데이터 정제
4. DPO negative 구성
5. schema validator 작성
6. Backend fixer/retry/drop fallback
7. Playwright Runner 방어 처리

### 23.2 긴 입력/긴 출력 문제

웹 분석 결과는 LLM 학습 데이터로 쓰기에 너무 컸고, QA suite 출력도 일반적인 chat completion보다 훨씬 길었습니다.

해결 전략은 다음과 같습니다.

| 문제 | 해결 |
|---|---|
| raw input이 너무 큼 | compact input format 설계 |
| output suite가 너무 김 | facet split + chunking |
| 평가 중 JSON이 잘림 | max_new_tokens 20K |
| extractor가 잘못된 JSON을 선택 | `scenarios` 포함 suite 우선 추출 |
| HF generation이 느림 | vLLM 평가/추론으로 전환 |

### 23.3 GPU 학습 안정성

Vertex AI FLEX_START는 비용이 낮지만 preemption 가능성이 있습니다. 긴 학습에서 중단되면 시간을 잃기 때문에 checkpoint 전략이 중요했습니다.

해결한 부분은 다음과 같습니다.

1. save_steps를 줄여 손실 구간 축소
2. save_total_limit 조정
3. GCS sync callback retry
4. resume_from_checkpoint auto 탐지
5. GCS latest checkpoint를 local로 prefetch
6. restartJobOnWorkerRestart와 auto requeue 전략 고려

---

## 24. 포트폴리오에서 강조할 수 있는 역량

### PM 역량

1. AI 기능을 제품의 핵심 사용자 가치로 연결
2. 불확실한 AI 출력을 안정적인 서비스 pipeline 안에 흡수
3. 팀별 책임 경계를 명확히 분리
4. SQS/Redis/S3 기반 비동기 아키텍처 의사결정
5. 사용자 경험을 고려한 실시간 이벤트 설계
6. 비용과 운영성을 고려한 worker scale 전략
7. validation/retry/fixer/drop fallback 같은 제품 안정성 정책 설계

### AI 역량

1. 웹 분석 데이터 기반 instruction tuning 데이터셋 설계
2. Gemma4 -> Qwen3-8B 모델 전환 의사결정
3. LoRA/QLoRA, SFT, DPO fine-tuning 구현
4. 긴 context 학습을 위한 compact input 설계
5. contract-aware negative pair 생성
6. Vertex AI 학습 job, GCS artifact, checkpoint sync 운영
7. vLLM/guided_json 기반 구조화 출력 추론 설계
8. validator pass, hallucination, diversity 등 실행 중심 평가 지표 설계
9. mode collapse 진단 및 데이터/추론/학습 관점의 보정 계획 수립

### Backend/Infra 이해 역량

1. SQS 기반 at-least-once pipeline 설계 이해
2. Redis Pub/Sub과 WebSocket 실시간 이벤트 구조 이해
3. S3 artifact 중심 대용량 데이터 전달 구조 이해
4. AWS ECS/Fargate/GPU worker 운영 구조 이해
5. CI/CD, Docker image, Terraform 기반 운영 흐름 이해

---

## 25. 이력서용 요약 문장

### 짧은 버전

AutoQA 프로젝트에서 PM과 AI를 담당하며, URL 기반 웹 분석 결과를 Qwen3-8B fine-tuned 모델로 Playwright 실행 가능한 QA suite JSON으로 변환하는 AI QA 자동화 파이프라인을 설계·구현했습니다. 139개 사이트, 14K+ 시나리오 데이터를 수집/정제하고, QLoRA SFT와 contract-aware DPO, vLLM guided JSON 추론 구조를 구축했습니다.

### 상세 버전

AI 기반 웹 QA 자동화 SaaS AutoQA에서 PM과 AI 파트를 담당했습니다. PM으로서 `PAGE_ANALYSIS -> SCENARIO_GENERATE -> QA_EXECUTE` 비동기 파이프라인, SQS/Redis/S3 역할 분리, AI 출력 validation/retry/fixer 정책을 설계했습니다. AI 담당자로서 웹 분석 산출물 기반 학습 데이터 수집 및 compact input 변환, Gemma4 LoRA 실험, Qwen3-8B QLoRA SFT, 7,791개 contract-aware DPO pair 학습, Vertex AI 학습 job/GCS checkpoint sync, vLLM guided JSON 추론 구조를 구현했습니다.

### 성과 중심 버전

웹 분석 JSON을 실제 Playwright 실행 가능한 QA suite JSON으로 변환하는 Qwen3-8B 기반 모델 파이프라인을 구축했습니다. 139개 사이트에서 14,614개 QA 시나리오를 수집하고, 약 16배 입력 압축, 12,709개 clean scenario 정제, 7,791개 DPO negative pair 생성, Stage A SFT train loss 0.6776, Stage C DPO train loss 0.1680을 달성했습니다. 모델 평가는 단순 loss가 아니라 validator pass rate, nodeId hallucination, contract violation, step diversity 등 실행 가능성 중심 지표로 설계했습니다.

---

## 26. 면접 답변용 핵심 스토리

### Q. 이 프로젝트에서 가장 중요한 기술적 도전은 무엇이었나요?

가장 큰 도전은 AI가 생성한 결과를 “그럴듯한 테스트 케이스”가 아니라 “실제 Playwright에서 실행 가능한 QA suite JSON”으로 만드는 것이었습니다. 이를 위해 단순 prompt engineering에 의존하지 않고, 분석 데이터 정제, 생성 contract, schema validator, DPO negative pair, guided JSON 추론, Backend retry/fixer/drop fallback까지 여러 계층으로 안정성을 확보했습니다.

### Q. 왜 Qwen3-8B로 전환했나요?

초기에는 Gemma4 E4B로 LoRA/QLoRA 학습을 시도했지만, 32K context 학습에서 A100 40GB OOM이 반복되었고 multimodal architecture로 인한 LoRA/device map 리스크도 있었습니다. AutoQA는 한국어 사이트와 구조화 JSON 생성 품질이 중요했기 때문에 Qwen3-8B가 더 적합하다고 판단했습니다. 이후 vLLM 기반 긴 출력 추론과 guided JSON으로 이어가기 쉬운 점도 전환 이유였습니다.

### Q. AI 학습 데이터는 어떻게 만들었나요?

Playwright 분석 산출물인 `site-summary.json`, `analysis-summary.json`, `scenario-brief.json`과 사람이/규칙 기반으로 만든 QA scenario suite를 연결해 학습 데이터를 만들었습니다. raw 분석 결과는 너무 커서 그대로 쓸 수 없었기 때문에, nodeId, role, purpose, confidence, functionalPaths, scenario cluster 중심의 compact input으로 약 16배 압축했습니다. 이후 avoid_auto_generation, ineligible target, nodeId hallucination 등을 정제하고, rejected 사례는 DPO negative pair로 활용했습니다.

### Q. 모델 평가는 어떻게 했나요?

train loss만으로는 실제 실행 가능성을 알 수 없기 때문에, validator pass rate, parse fail rate, truncation rate, contract violation rate, nodeId hallucination rate, step type diversity, category coverage, multi-page coverage를 함께 봤습니다. 특히 모델이 JSON을 길게 생성하다가 잘리는 문제를 분리하기 위해 strict JSON뿐 아니라 partial scenario extraction 지표도 추가했습니다.

### Q. PM으로서 가장 중요한 결정은 무엇이었나요?

AI와 실행기의 책임을 분리한 것입니다. AI는 suite JSON을 생성하지만, 실제 실행 가능성은 Backend validator와 Playwright Runner가 보장하도록 했습니다. 또한 SQS는 비동기 작업 처리, Redis/WebSocket은 실시간 UX, S3는 대용량 artifact 저장으로 역할을 분리했습니다. 이 구조 덕분에 AI 출력의 불확실성을 제품 안정성 안으로 흡수할 수 있었습니다.

---

## 27. 최종 회고

AutoQA에서 가장 크게 배운 점은 AI 제품은 모델 하나만으로 완성되지 않는다는 것입니다. 실제 사용자에게 가치 있는 AI 기능을 만들려면 데이터 수집, 전처리, 학습, 평가, 추론, 검증, fallback, UX, 운영 비용까지 모두 하나의 시스템으로 설계해야 합니다.

특히 이 프로젝트는 LLM이 자유롭게 자연어를 생성하는 문제가 아니라, 외부 실행기와 계약을 맞춰야 하는 구조화 생성 문제였습니다. 그래서 모델 성능을 높이는 것만큼이나 contract 설계, compact input, validator, DPO negative, guided decoding, retry/fixer 정책이 중요했습니다.

저는 이 프로젝트를 통해 PM으로서는 AI 기능을 제품 흐름과 팀 구조 안에 녹이는 경험을 했고, AI 담당자로서는 실제 서비스에서 동작하는 fine-tuned LLM 파이프라인을 데이터부터 추론까지 끝까지 설계하고 구현하는 경험을 했습니다.

