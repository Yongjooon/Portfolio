# A601 프로젝트 포트폴리오

## 1. 프로젝트 한 줄 소개

**A601은 AI 판사가 지배하는 우주선을 배경으로, WebRTC 영상/음성 대화와 WebSocket 실시간 게임 진행을 결합한 SF 서바이벌 마피아 게임입니다.**

플레이어는 연구원, 외계인, 시스템 엔지니어, 영상 엔지니어, 밀항자 역할 중 하나를 배정받고, 낮에는 토론과 변론을 통해 외계인을 추리하며, 밤에는 각 직업의 능력을 사용해 생존과 승리를 겨룹니다. 기존 마피아 게임의 투표 구조를 AI 판결 구조로 바꾸어, 사용자의 발언과 변론을 AI가 분석하고 처형 대상을 결정하는 것이 핵심 차별점입니다.

<img src="docs/app.png" alt="A601 서비스 이미지" width="800">

## 2. 프로젝트 개요

| 항목 | 내용 |
|---|---|
| 프로젝트명 | Project A601 |
| 서비스 유형 | 모바일 환경을 고려한 웹 기반 실시간 마피아 게임 |
| 장르 | SF 세계관 기반 서바이벌 / 소셜 디덕션 / 마피아 게임 |
| 개발 기간 | 2026.01.06 ~ 2026.02.13 |
| 개발 인원 | 4명 |
| 담당 역할 | PM, Frontend, 실시간 UX 구현 |
| 핵심 기술 | React, Vite, LiveKit 기반 WebRTC, STOMP WebSocket, Spring Boot, Redis, FastAPI, OpenAI API |
| 주요 기능 | 회원/인증, 방 생성/입장, 대기방, 실시간 영상/음성, 실시간 채팅, 직업 배정, 낮/밤/AI 판결 단계, 직업별 능력, 게임 결과 화면 |

## 3. 기획 배경

마피아 게임은 플레이어 간 커뮤니케이션의 밀도가 게임 재미를 결정합니다. 하지만 온라인 환경에서는 음성/영상 연결, 실시간 상태 동기화, 화면 전환, 사망자 처리, 직업별 권한 제어가 조금만 어긋나도 몰입이 크게 깨집니다.

A601은 이 문제를 다음 방향으로 해결했습니다.

- 플레이어 간 직접 대화는 WebRTC 기반 영상/음성으로 제공
- 게임 상태, 직업 능력, 페이즈 전환, 방 상태는 WebSocket으로 실시간 동기화
- AI 판결이라는 세계관 장치를 통해 단순 투표 대신 “변론 제출 → AI 분석 → 판결 연출” 흐름 제공
- 낮/밤/판결/결과 화면마다 색, 소리, 영상, 애니메이션을 다르게 구성해 게임 몰입감 강화
- PM 관점에서는 개발 기간이 짧은 팀 프로젝트였기 때문에 기능 우선순위와 실시간 안정성을 중심으로 범위를 관리

## 4. 내가 맡은 역할

### PM

프로젝트 전반의 의사결정과 협업 기준 정리를 담당했습니다.

- 프로젝트 콘셉트와 핵심 게임 루프 정리
- 기능 우선순위 결정 및 개발 범위 조율
- Jira 이슈 구조, Git 브랜치 전략, 커밋 컨벤션, 코드 컨벤션 수립
- 화면/기능 흐름을 기준으로 프론트엔드, 백엔드, AI 서버 간 계약 정리
- 게임 진행 중 발생 가능한 예외 상황 정리
  - 방장 퇴장
  - 게임 중 연결 끊김
  - 사망자 처리
  - AI 판결 응답 지연
  - 역할별 능력 중복 사용
  - 페이즈 전환 중 UI race condition
- 일정 내 완성을 위해 필수 기능과 몰입 요소의 우선순위 조정

### Frontend

실제 사용자가 경험하는 대부분의 화면과 실시간 게임 UX를 구현했습니다.

- React + Vite 기반 화면 구조 구현
- 로그인, 메인 로비, 방 목록, 방 생성/수정/입장, 게임 대기방, 인게임 단계 화면 구현
- LiveKit 기반 WebRTC 영상/음성 연결 화면 구현
- STOMP/SockJS WebSocket 연결 및 메시지 처리 흐름 구현
- Zustand 기반 방/게임 상태 관리 구조 구현
- 낮/밤/AI 판결/결과/엔딩 단계별 UI와 애니메이션 구현
- 직업별 능력 사용 UI 구현
- 사운드, BGM, 효과음, 영상 연출을 통한 몰입형 UX 구현
- 모바일 및 다양한 화면 크기 대응을 위한 고정 캔버스 + 스케일 방식 반응형 처리

## 5. 기술 스택

| 영역 | 사용 기술 |
|---|---|
| Frontend | React, Vite, React Router, Zustand, TailwindCSS, Ant Design, Axios, STOMP.js, SockJS, LiveKit Client, LiveKit React Components, Framer Motion, GSAP, Three.js/OGL |
| WebRTC | OpenVidu v3 계열의 LiveKit 기반 미디어 서버, `livekit-client`, `@livekit/components-react` |
| Realtime | Spring WebSocket, STOMP, SockJS, SimpMessagingTemplate |
| Backend | Spring Boot, Java 17, JPA, Redis, MySQL, Spring Security, JWT, OAuth2, Swagger |
| AI | FastAPI, Python 3.11, OpenAI API |
| Infra | Docker Compose, Nginx, Jenkins, LiveKit Server, Redis Commander, Dozzle |

참고: 코드 기준으로 WebRTC 미디어 서버는 LiveKit SDK와 LiveKit Server를 직접 사용합니다. OpenVidu v3가 LiveKit 기반 구조를 사용하기 때문에 포트폴리오에서는 **OpenVidu v3/LiveKit 기반 WebRTC 구현**으로 정리했습니다.

## 6. 시스템 아키텍처

<img src="docs/architecture.png" alt="A601 시스템 아키텍처" width="800">

### 전체 구조

A601은 Frontend, Backend, AI Server, Redis, MySQL, LiveKit Server로 구성됩니다.

- Frontend는 사용자 화면, WebRTC 연결, WebSocket 구독/발행, 게임 상태 렌더링을 담당합니다.
- Backend는 REST API, WebSocket 메시지 처리, 방/게임 상태 관리, 역할 배정, 페이즈 제어, AI 서버 호출을 담당합니다.
- Redis는 실시간 방 상태, 플레이어 상태, 액션, WebSocket 세션 정보, AI 변론 데이터 저장에 사용됩니다.
- MySQL은 회원, 게임 기록, 통계 등 영속 데이터 저장에 사용됩니다.
- LiveKit Server는 WebRTC 기반 영상/음성 룸과 publish/subscribe 권한 제어를 담당합니다.
- AI Server는 제출된 변론을 OpenAI API로 분석하고 처형 대상과 판결 근거를 반환합니다.

### 실시간 처리 방식

A601은 실시간 기능을 두 계층으로 분리했습니다.

1. **WebRTC 계층**
   - 영상/음성 송수신
   - LiveKit Room 입장
   - 사용자별 camera/microphone publish 제어
   - 밤 페이즈에서 외계인만 원격 영상/음성을 볼 수 있도록 subscribe 권한 제어
   - 사망자는 송신을 차단하고 관전자 상태로 전환

2. **WebSocket 계층**
   - 방 입장/퇴장
   - 준비 상태
   - 방장 변경
   - 게임 시작
   - 페이즈 전환
   - 직업별 능력 사용
   - AI 판결 결과
   - 플레이어 사망
   - 게임 종료

이렇게 분리한 이유는 영상/음성 데이터는 WebRTC가 적합하고, 게임 상태와 이벤트는 서버 권위적인 WebSocket 메시지가 적합하기 때문입니다.

## 7. 핵심 게임 흐름

### 7.1 방 생성 및 입장

사용자는 메인 로비에서 공개방 또는 비공개방을 생성하고 입장할 수 있습니다.

- 방 정원은 6명 또는 8명으로 제한
- 비공개방은 PIN 기반 입장
- 방 제목 중복 체크
- 방 상태는 `WAITING`, `FULL`, `INGAME`으로 관리
- 입장 성공 시 Backend가 LiveKit 토큰과 LiveKit URL을 발급
- Frontend는 발급받은 token/url을 Zustand에 저장하고 WebSocket 연결 후 로비로 이동

구현 흐름은 다음과 같습니다.

```text
방 입장 클릭
→ REST API로 방 참가 요청
→ Backend에서 Redis 방 상태 확인
→ LiveKit 토큰 발급
→ Frontend에서 LiveKit 연결 정보 저장
→ STOMP WebSocket 연결
→ /topic/room/{roomId} 구독
→ 대기방 화면 진입
```

### 7.2 대기방

대기방은 게임 시작 전 커뮤니케이션과 준비 상태 관리를 담당합니다.

- LiveKit 기반 실시간 영상/음성 표시
- 카메라 ON/OFF
- 기기 설정 모달
- 준비/준비 취소
- 방장만 게임 시작 가능
- 모든 인원이 준비하고 정원이 찼을 때 게임 시작 가능
- 강퇴, 방장 변경, 방 나가기 처리
- WebSocket 연결이 끊기면 안전하게 방에서 이탈 처리

<img src="docs/lobby.png" alt="A601 대기방" width="800">

### 7.3 직업 배정

게임 시작 시 Backend가 플레이어 수에 따라 역할 풀을 만들고 무작위 배정합니다.

6인 게임:

- 외계인 1명
- 시스템 엔지니어 1명
- 영상 엔지니어 1명
- 연구원 3명

8인 게임:

- 외계인 2명
- 시스템 엔지니어 1명
- 영상 엔지니어 1명
- 밀항자 1명
- 연구원 3명

Frontend는 `GAME_START` 개인 메시지를 받아 내 역할을 저장하고, 역할 배정 화면을 보여줍니다. 역할별 이미지, 승리 조건, 능력 설명을 함께 제공하여 사용자가 즉시 자신의 목표를 이해할 수 있도록 구성했습니다.

<img src="docs/role.gif" alt="직업 배정 화면" width="800">

### 7.4 낮 단계

낮 단계는 플레이어들이 서로 토론하고 의심 대상을 추리하는 단계입니다.

- 모든 생존자가 영상/음성 및 채팅으로 소통 가능
- 사망자는 채팅/송신 제한
- 타이머 표시
- AI 변론 시작 10초 전 안내 오버레이 표시
- 밀항자는 낮에 단 한 번 즉시 처형 능력 사용 가능
- 외계인 팀 동료 표시
- 사망자는 모든 역할 정보를 볼 수 있는 관전자 UI 제공

<img src="docs/day.gif" alt="낮 화면" width="800">

### 7.5 AI 변론 단계

낮 토론이 끝나면 AI 판결을 위한 변론 단계로 전환됩니다.

사용자는 두 가지 방식으로 변론을 제출할 수 있습니다.

- **탄원서 제출**: 텍스트로 변론 작성
- **심판 직접 변론**: 음성 인식 기반 변론 작성

타이머가 종료되면 작성 중인 텍스트 또는 음성 인식 결과가 자동 제출됩니다. 제출하지 않은 사용자는 기본 변론 또는 기권 처리되어 게임 진행이 멈추지 않도록 설계했습니다.

AI 서버는 생존자들의 변론을 분석하고, 가장 의심스러운 플레이어와 판결 근거 3문장을 반환합니다. Backend는 해당 플레이어를 처형 처리하고, Frontend는 판결 대기 연출 후 결과 화면을 보여줍니다.

<img src="docs/pleading.gif" alt="AI 변론 화면" width="800">

<img src="docs/judgment.gif" alt="AI 판결 화면" width="800">

### 7.6 밤 단계

밤 단계는 직업별 능력 사용과 정보 비대칭이 핵심입니다.

- 외계인은 제거할 대상을 선택
- 외계인이 2명인 경우, 둘이 같은 대상을 선택해야 최종 제거가 성립
- 시스템 엔지니어는 보호할 대상을 선택
- 영상 엔지니어는 의심 대상의 외계인 여부를 확인
- 연구원과 밀항자는 밤이 지나가기를 기다림
- 사망자는 관전자 역할로 액션 결과를 볼 수 있음
- 밤 종료 10초 전 안내 오버레이 표시

LiveKit 권한은 페이즈에 맞춰 서버에서 제어합니다.

- 낮: 생존자는 모두 publish/subscribe 가능
- 밤: 생존자 중 외계인만 subscribe 가능
- 사망자: publish 불가, subscribe 가능

이 구조를 통해 밤에는 외계인끼리만 카메라/음성 정보를 공유할 수 있고, 다른 생존자는 화면상 고립감을 느끼도록 UX를 설계했습니다.

<img src="docs/night.gif" alt="밤 화면" width="800">

### 7.7 직업별 능력

#### 외계인

- 밤마다 제거할 대상을 선택
- 1차 선택 시 대상 타일에 경고/스캔 연출 표시
- 같은 대상을 다시 선택하면 확정 상태로 표시
- 8인 게임에서 외계인이 2명일 경우, 살아있는 외계인 모두가 같은 대상을 선택해야 제거 성공

<img src="docs/alien.gif" alt="외계인 능력 사용" width="800">

#### 시스템 엔지니어

- 밤마다 한 명을 보호
- 보호 대상은 파란 쉴드 연출로 표시
- 외계인의 제거 대상과 보호 대상이 일치하면 사망 방지
- 선택 후 변경 불가하도록 UI 상호작용 잠금

<img src="docs/sys.gif" alt="시스템 엔지니어 능력 사용" width="800">

#### 영상 엔지니어

- 밤마다 한 명을 조사
- 조사 결과는 외계인/인간 결과 이미지로 표시
- 조사 결과는 본인과 관전자에게 전달
- 한 번 사용 후 중복 사용 방지

<img src="docs/cam.gif" alt="영상 엔지니어 능력 사용" width="800">

#### 밀항자

- 낮에 단 한 번 원하는 대상 즉시 처형
- 처형 후 짧은 결과 연출을 보여주고 밤 단계로 전환
- 즉시 처형이 게임 흐름을 바꾸는 변수로 작동하도록 설계

<img src="docs/milhangza.gif" alt="밀항자 능력 사용" width="800">

### 7.8 게임 종료

승리 조건은 서버에서 계속 검사합니다.

- 인간 승리: 모든 외계인 사망
- 외계인 승리: 살아있는 외계인 수가 인간 수 이상

게임 종료 시 모든 플레이어의 역할을 공개하고, 승리 진영에 따라 다른 영상과 결과 화면을 보여줍니다. 결과 화면 이후에는 다음 게임을 위해 방 상태, ready 상태, 역할, 생존 상태, LiveKit 권한을 초기화합니다.

## 8. 실시간 WebRTC 구현 정리

### 8.1 LiveKit/OpenVidu v3 기반 WebRTC 연결

WebRTC 구현은 LiveKit Room을 중심으로 설계했습니다.

Frontend:

- `LiveKitRoom`으로 room 연결
- `RoomAudioRenderer`로 원격 오디오 렌더링
- `useParticipants` 기반 참여자 목록 렌더링
- `useLocalParticipant`로 로컬 카메라 제어
- `useIsSpeaking`, `useTrackMutedIndicator`로 발화/음소거 상태 표시
- 사망자 또는 CCTV 결과 상황에서는 video track을 detach하고 이미지/결과 화면으로 대체

Backend:

- LiveKit API Key/Secret 기반 토큰 발급
- RoomJoin grant 부여
- 게임 페이즈와 생존 여부에 따라 participant permission 업데이트
- 사망자 publish 차단
- 게임 종료 후 publish/subscribe 권한 복구

### 8.2 WebRTC와 게임 규칙 결합

일반 화상 채팅과 달리 A601의 영상/음성은 게임 규칙과 직접 연결됩니다.

- 죽은 플레이어는 말할 수 없도록 송신 차단
- 밤에는 외계인만 다른 외계인의 영상을 볼 수 있도록 구독 권한 제한
- 로비로 돌아오면 카메라/마이크 자동 복구
- WebSocket 연결이 끊기면 방 이탈 또는 게임 중 사망 처리
- 사망자는 전체 역할을 볼 수 있어 관전 재미를 유지

이렇게 WebRTC 권한을 게임 상태와 연결해 단순 화상 채팅이 아니라 게임 메커니즘의 일부로 만들었습니다.

## 9. WebSocket 구현 정리

### 9.1 STOMP/SockJS 기반 통신

Frontend는 `@stomp/stompjs`와 `sockjs-client`를 사용해 `/ws-stomp` 엔드포인트에 연결합니다.

연결 시 전달하는 정보:

- JWT Access Token
- roomId
- private room password

Backend는 STOMP CONNECT 단계에서 JWT를 검증하고, Principal에 userId를 저장합니다. SUBSCRIBE 성공 후 Redis에 WebSocket sessionId와 roomId/userId를 매핑하여 연결 종료 시 자동 정리할 수 있도록 했습니다.

### 9.2 토픽 구조

주요 구독 구조는 다음과 같습니다.

```text
/topic/room/{roomId}
  방 전체 브로드캐스트
  예: GAME_STATUS, GAME_PHASE, PLAYER_DEATH, AI_RESULT, GAME_OVER

/topic/room/{roomId}/{userId}
  개인 메시지
  예: GAME_START, ALIEN_PICK, VIDEO_VIEW, SYS_PICK, ALL_ROLES, HOST_CHANGED, KICKED

/app/room/ready/{roomId}
  준비 상태 변경

/app/rooms/start/{roomId}
  게임 시작 요청

/app/game/action/{role}/{roomId}
  직업별 액션 요청

/app/game/defense/success/{roomId}/{day}
  AI 변론 제출 성공 신호
```

### 9.3 서버 권위적 상태 관리

Frontend는 화면과 상호작용을 담당하지만, 핵심 판정은 Backend가 담당합니다.

- 게임 시작 가능 여부
- 역할 배정
- 페이즈 전환
- 외계인 제거 성립 여부
- 시스템 엔지니어 보호 성공 여부
- 영상 엔지니어 조사 결과
- 밀항자 처형 가능 여부
- AI 판결 대상 유효성
- 승리 조건

이 구조 덕분에 사용자가 UI를 조작하더라도 실제 게임 규칙은 서버 기준으로 일관되게 유지됩니다.

## 10. Frontend 구현 상세

### 10.1 화면 구조

주요 화면은 다음 흐름으로 구성했습니다.

```text
Login / Signup / Find ID / Find Password
→ Main Lobby
→ Room List / Create Room / Join Room
→ LobbyShell
→ StageRouter
→ LobbyStage / Role / Morning / Night / AI / Judgment / Result / Ending
```

`LobbyShell`은 실시간 게임 화면의 최상위 컨테이너입니다.

- LiveKit Room 연결
- STOMP WebSocket 연결
- 방 메타 정보 조회
- sessionStorage 기반 새로고침 복원
- WebSocket disconnect 감지
- 강퇴/나가기 이벤트 처리
- stage별 화면 라우팅
- 기기 설정 모달 연결
- 사망자 A/V 송신 차단
- 원격 오디오 구독 정책 적용

### 10.2 StageRouter

게임 단계는 문자열 stage로 관리하고, `StageRouter`에서 실제 컴포넌트로 분기했습니다.

```text
lobby      → 대기방
role       → 직업 배정
morning    → 낮 토론
ai         → AI 변론
waiting    → AI 판결 대기 연출
judgment   → AI 판결 결과
night      → 밤 능력 사용
pirate     → 밀항자 처형 결과
result     → 밤 사망 결과
ending     → 게임 종료
```

이 구조의 장점은 Backend에서 내려오는 `GAME_PHASE` 메시지를 Frontend stage로 매핑하기 쉽고, 각 단계의 UI/사운드/상호작용을 독립적으로 관리할 수 있다는 점입니다.

### 10.3 Zustand 상태 관리

방 입장 정보와 게임 상태를 분리했습니다.

- `roomSlice`
  - roomId
  - LiveKit URL
  - LiveKit token
  - 비공개방 joinPassword

- `lobbySlice`
  - roomInfo
  - userList
  - myUserId
  - myRole
  - currentStage
  - currentTurn
  - timer
  - deadUserIds
  - revealedRolesByUserId
  - judgeResult
  - gameOverWinnerTeam

- `inGameSlice`
  - 역할 이미지 variant
  - 룰/역할 모달 UI 상태

상태를 분리해 방 입장 정보, 실시간 게임 상태, UI 상태가 섞이지 않도록 했습니다.

### 10.4 영상 타일 UX

`VideoGrid`와 `UserCam`은 게임 몰입의 핵심 컴포넌트입니다.

구현한 상태 표현:

- 카메라 ON/OFF 아이콘
- 발화 중인 사용자 강조
- 준비 완료 상태
- 선택된 대상 강조
- 외계인 타겟 pending/confirmed 표시
- 시스템 엔지니어 보호 대상 표시
- CCTV 결과 표시
- 사망자 이미지 대체
- 관전자용 역할 뱃지 표시
- 외계인 동료 표시
- 닉네임 ellipsis 처리
- 비디오 track attach/detach 안정화

특히 `identity`, `userId`, `nickname`이 섞이는 실시간 환경에서 대상 매핑이 틀어지면 잘못된 플레이어에게 효과가 표시될 수 있기 때문에, `userList`를 기준으로 identity와 userId를 정규화하는 로직을 강화했습니다.

### 10.5 게임 단계별 UX

각 단계는 같은 컴포넌트를 재사용하되 색, 레이아웃, 사운드, 상호작용 정책을 다르게 적용했습니다.

- 로비: 붉은 SF 터미널 톤, 준비/시작 중심
- 낮: 밝은 주황/화이트 톤, 토론과 정보 확인 중심
- 밤: 보라/블랙 톤, 고립감과 긴장감 중심
- AI 변론: 붉은 시스템 터미널 톤, 재판/판결 느낌 강화
- 판결 결과: 글리치, 스캔라인, 타이핑 텍스트로 AI 판단 연출
- 엔딩: 승리 진영별 영상, 결과 공개, 역할 목록 표시

### 10.6 사운드와 영상 효과

몰입을 위해 사운드와 영상을 단순 배경이 아니라 상태 전환 신호로 사용했습니다.

- 로그인 BGM
- 메인 BGM
- 직업 배정 impact 효과음
- AI 대기/판결 noise loop
- 밤 단계 배경음
- 승리 진영별 엔딩 영상
- 결과 화면 타이핑 자막
- 외계인 선택/확정 연출
- 시스템 엔지니어 보호 쉴드 연출
- CCTV 결과 이미지

브라우저 autoplay 정책 때문에 BGM 재생이 막힐 수 있어, 사용자 입력 후 재생을 재시도하는 로직도 추가했습니다.

### 10.7 모바일/반응형 대응

게임 화면은 일반 웹 문서처럼 자연스럽게 줄바꿈되는 구조보다, “게임 보드”처럼 일정한 레이아웃을 유지하는 것이 중요했습니다.

그래서 AI 변론, 직업 배정, 판결, 엔딩 화면 등은 다음 방식으로 처리했습니다.

```text
고정 기준 캔버스 크기 설정
→ ResizeObserver로 현재 viewport 측정
→ 가로/세로 비율 중 작은 scale 계산
→ transform: scale(...) 적용
→ 내부 UI 비율 유지
```

이 방식으로 작은 화면에서도 버튼, 타이머, 영상 타일, 판결문, 결과 UI가 깨지지 않도록 했습니다.

## 11. AI 판결 기능

### 11.1 기능 목적

일반적인 마피아 게임은 플레이어 투표로 처형 대상을 정합니다. A601은 SF 세계관에 맞춰 중앙 AI가 플레이어의 변론을 분석하고 처형 대상을 결정하게 했습니다.

이 기능은 단순한 장식이 아니라 게임 루프의 핵심입니다.

```text
낮 토론
→ 변론 제출
→ AI 서버 분석
→ 판결 결과 수신
→ 대상 사망 처리
→ 판결 근거 연출
→ 밤 단계 전환
```

### 11.2 AI 서버 처리

AI 서버는 FastAPI로 구성했습니다.

- `/api/games/judgeCreate` 엔드포인트 제공
- roomId, day, 변론 목록을 입력으로 받음
- OpenAI API에 시스템 프롬프트와 사용자 변론 전달
- JSON 형식으로 처형 대상, confidence, 판결 근거 반환
- Backend는 반환된 닉네임이 실제 생존 플레이어인지 검증 후 사망 처리

### 11.3 안정성 처리

AI 판결은 외부 API 응답에 의존하기 때문에 다음 예외를 고려했습니다.

- 전원 제출 전 타이머가 종료되는 경우
- 일부 사용자가 변론을 제출하지 않는 경우
- AI 서버 응답 지연
- AI가 유효하지 않은 닉네임을 반환하는 경우
- 중복 판결 요청이 발생하는 경우

Backend에서는 Redis 기반 저장과 lock을 사용해 판결이 중복 실행되지 않도록 했고, 미제출자는 자동 기권 처리해 게임 진행이 멈추지 않도록 했습니다.

## 12. PM으로 진행한 의사결정

### 12.1 협업 컨벤션 수립

팀원이 같은 기준으로 개발할 수 있도록 `TeamConventionGuide.md`를 정리했습니다.

정리한 항목:

- FE/BE/AI 명명 규칙
- 주석 작성 기준
- Formatter 기준
- Git-Flow 기반 브랜치 전략
- 커밋 메시지 규칙
- Jira Epic/Story/Bug/Sub-task 구조
- 우선순위 기준
- 작업 상태 흐름

이를 통해 프로젝트 중반 이후에도 기능 추가, 버그 수정, 배포 작업이 같은 흐름으로 진행될 수 있도록 했습니다.

### 12.2 우선순위 결정 기준

개발 기간이 제한되어 있었기 때문에 다음 기준으로 의사결정했습니다.

1. 게임이 처음부터 끝까지 진행되는가
2. 실시간 연결이 안정적인가
3. 역할별 능력이 서버 판정과 UI에 일관되게 반영되는가
4. AI 판결이 실패해도 게임이 멈추지 않는가
5. 사용자가 현재 단계와 해야 할 행동을 즉시 이해할 수 있는가
6. 몰입을 높이는 사운드/영상/애니메이션을 핵심 흐름에 우선 적용할 수 있는가

### 12.3 기능 범위 조율

초기에는 여러 부가 기능 아이디어가 있었지만, 최종적으로는 “실시간 마피아 게임으로 완주 가능한 경험”을 최우선으로 두었습니다.

우선 구현한 기능:

- 방 생성/입장/대기
- WebRTC 영상/음성
- WebSocket 실시간 상태 동기화
- 직업 배정
- 낮/밤/AI 판결/엔딩까지 이어지는 전체 게임 루프
- 역할별 능력
- 사망자/관전자 처리
- 게임 결과 공개

추후 개선으로 남긴 기능:

- 상세 전적 분석
- 리플레이
- 고도화된 매칭
- 모바일 터치 UX 추가 개선
- AI 판결 품질 개선 및 판결 설명 강화

## 13. 주요 문제와 해결

### 문제 1. WebRTC 영상 권한과 게임 페이즈가 복잡하게 얽힘

단순 화상 채팅이라면 모든 사용자가 항상 publish/subscribe를 하면 됩니다. 하지만 A601에서는 낮/밤/사망 상태에 따라 누가 보고 들을 수 있는지가 달라져야 했습니다.

해결:

- Backend에서 게임 페이즈가 바뀔 때마다 LiveKit participant permission 업데이트
- 낮에는 생존자 모두 subscribe 가능
- 밤에는 외계인만 subscribe 가능
- 사망자는 publish 차단, subscribe 허용
- Frontend에서도 사망자일 경우 로컬 카메라/마이크를 강제로 OFF 처리
- 게임 종료 후 로비 복귀 시 권한과 A/V 상태 복구

결과:

- 영상/음성 자체가 게임 규칙에 맞게 동작
- 밤의 정보 비대칭과 사망자 관전 경험을 동시에 구현

### 문제 2. WebSocket 메시지 순서에 따라 화면이 잘못 전환될 수 있음

게임 단계가 빠르게 바뀌거나 AI 결과가 늦게 도착하면, Frontend stage가 먼저 넘어가거나 판결 화면이 생략될 위험이 있었습니다.

해결:

- `GAME_PHASE`와 `AI_RESULT`를 분리 처리
- `AI_RESULT`는 결과 저장만 하고 stage를 직접 바꾸지 않음
- `JUDGMENT` phase 도착 시 `waiting`을 최소 5초 노출한 뒤 `judgment`로 전환
- pending 중 들어온 다음 stage는 queue에 저장했다가 판결 연출 후 반영
- role intro 중 들어온 phase는 `pendingStage`로 저장

결과:

- AI 대기 연출, 판결 결과, 다음 단계 전환이 안정적으로 이어짐
- 사용자 입장에서 갑자기 화면이 튀는 느낌을 줄임

### 문제 3. 변론 제출과 AI 판결 트리거 동기화

모든 생존자가 변론을 제출해야 AI 판결을 요청할 수 있지만, 사용자가 제출하지 않거나 네트워크가 지연되면 게임이 멈출 수 있었습니다.

해결:

- Frontend에서 타이머 종료 시 자동 제출
- Backend에서 제출 수와 생존자 수를 비교
- 미제출자는 자동 기권 처리
- Redis lock으로 판결 중복 실행 방지
- AI 서버 호출 실패/응답 오류 시 에러 브로드캐스트

결과:

- 일부 사용자가 제출하지 않아도 게임 진행 가능
- AI 판결이 중복 실행되는 문제 방지

### 문제 4. 모바일 환경에서 게임 UI가 쉽게 깨짐

인게임 화면은 영상 타일, 채팅, 타이머, 버튼, 모달, 오버레이가 한 화면에 동시에 들어갑니다. 일반적인 반응형 레이아웃만으로는 작은 화면에서 버튼이 겹치거나 판결 UI가 깨지는 문제가 있었습니다.

해결:

- 게임 화면을 고정 캔버스처럼 설계
- ResizeObserver 기반 scale 계산
- 내부 UI 비율 유지
- 영상 타일은 grid/flex shrink 체인 강화
- 닉네임과 버튼 텍스트는 overflow/ellipsis 처리
- stage별 최소 너비와 padding 조정

결과:

- 다양한 화면 크기에서도 게임 보드 느낌을 유지
- UI가 깨지는 상황을 줄이고 몰입형 화면 구성을 유지

### 문제 5. 사망자 경험이 단순 이탈처럼 느껴질 수 있음

마피아 게임에서 사망자는 더 이상 승부에 직접 개입할 수 없지만, 완전히 아무것도 못 하면 게임에서 이탈하게 됩니다.

해결:

- 사망자는 카메라/마이크 송신 차단
- 채팅 입력 제한
- 전체 역할 공개
- 관전자에게 외계인 선택, CCTV 결과, 보호 대상 등 일부 액션 결과 전달
- 사망자 타일은 별도 이미지로 표시

결과:

- 게임 규칙은 유지하면서 관전 재미를 제공
- 사망 후에도 게임 흐름을 따라갈 수 있게 함

## 14. 화면 및 기능 산출물

### 로그인

<img src="docs/login.png" alt="로그인 화면" width="800">

### 마이페이지

<img src="docs/mypage.png" alt="마이페이지" width="800">

### 메인 로비

<img src="docs/main.png" alt="메인 로비" width="800">

### 게임 대기방

<img src="docs/lobby.png" alt="게임 대기방" width="800">

### AI 판결

<img src="docs/judgment.gif" alt="AI 판결" width="800">

### ERD

<img src="docs/erd.png" alt="ERD" width="800">

### Swagger API 문서

<img src="docs/swagger_1.png" alt="Swagger API 1" width="800">

## 15. 주요 구현 파일

### Frontend

| 파일 | 역할 |
|---|---|
| `frontend/src/pages/Lobby/LobbyShell.jsx` | LiveKit Room 연결, WebSocket 연결, stage 라우팅, 기기 설정, disconnect 처리 |
| `frontend/src/pages/Lobby/stages/StageRouter.jsx` | 게임 stage별 화면 분기와 사운드 제어 |
| `frontend/src/pages/Lobby/stages/MorningStage.jsx` | 낮 토론 화면, 밀항자 처형 UI, 채팅, 타이머 |
| `frontend/src/pages/Lobby/stages/NightStage.jsx` | 밤 능력 사용 화면, 외계인/엔지니어 액션 UI |
| `frontend/src/pages/Lobby/stages/AiStage.jsx` | AI 변론 방식 선택, 텍스트/음성 변론 제출 |
| `frontend/src/components/Lobby/VideoGrid.jsx` | 참여자 영상 그리드, 대상/사망/역할 상태 매핑 |
| `frontend/src/components/Lobby/UserCam.jsx` | 개별 영상 타일, 발화/음소거/사망/CCTV/보호/타겟 연출 |
| `frontend/src/api/services/socket.service.js` | STOMP/SockJS WebSocket 연결, 구독, 발행, disconnect 이벤트 |
| `frontend/src/api/services/lobby.service.js` | 방 전체/개인 토픽 메시지 처리 |
| `frontend/src/api/services/game.service.js` | 게임 이벤트 처리, 직업별 액션 발행, stage 전환 제어 |
| `frontend/src/store/slices/lobbySlice.js` | 게임 상태, stage, 판결 pending/queue, 사망자, 결과 상태 관리 |
| `frontend/src/utils/bgm.js` | 로그인/메인 BGM 제어와 autoplay retry |
| `frontend/src/utils/sfx.js` | impact, noise, night back loop 효과음 제어 |

### Backend

| 파일 | 역할 |
|---|---|
| `backend/src/main/java/com/project/config/WebSocketConfig.java` | STOMP WebSocket endpoint와 broker 설정 |
| `backend/src/main/java/com/project/common/websocket/StompHandler.java` | WebSocket JWT 인증, subscribe/disconnect 이벤트 처리 |
| `backend/src/main/java/com/project/service/RoomService.java` | 방 생성/입장/퇴장, LiveKit 토큰 발급, 세션 정리 |
| `backend/src/main/java/com/project/service/LiveKitService.java` | LiveKit 토큰 생성과 participant permission 제어 |
| `backend/src/main/java/com/project/service/GameService.java` | 게임 시작, 역할 배정, 개인별 GAME_START 전송 |
| `backend/src/main/java/com/project/service/GamePhaseService.java` | 페이즈 전환, 타이머, 승리 조건, 밤 결과 처리 |
| `backend/src/main/java/com/project/service/ActionService.java` | 직업별 능력 검증 및 액션 처리 |
| `backend/src/main/java/com/project/service/AiJudgeService.java` | 변론 저장, AI 판결 트리거, AI 서버 호출, 처형 처리 |
| `backend/src/main/java/com/project/service/RoomBroadcastService.java` | 방 전체/개인 메시지 브로드캐스트 |

### AI

| 파일 | 역할 |
|---|---|
| `ai/app/main.py` | FastAPI 앱 엔트리 |
| `ai/app/routers/judge.py` | AI 판결 API 라우터 |
| `ai/app/services/ai_judge.py` | OpenAI API 호출, 판결 프롬프트, JSON 응답 처리 |

### Infra/Docs

| 파일 | 역할 |
|---|---|
| `docker-compose.yml` | 로컬 개발용 MySQL, Redis, LiveKit, AI, 운영 도구 구성 |
| `docker-compose.prod.yml` | 운영 배포용 Backend, Frontend, AI 구성 |
| `Jenkinsfile` | FE/BE 빌드 및 Docker Compose 배포 파이프라인 |
| `livekit.yaml` | LiveKit 서버 포트 및 RTC 설정 |
| `TeamConventionGuide.md` | 팀 개발/협업 컨벤션 |
| `exec/포팅 메뉴얼.md` | 빌드/배포/외부 서비스 설정 문서 |

## 16. 프로젝트 결과

프로젝트를 통해 다음 기능을 하나의 게임 흐름으로 완성했습니다.

- 회원가입/로그인/Google OAuth 기반 인증
- 메인 로비 및 방 목록
- 공개/비공개 방 생성 및 참가
- 방장/준비 상태 관리
- WebRTC 영상/음성 기반 대기방 및 인게임 커뮤니케이션
- STOMP WebSocket 기반 실시간 게임 상태 동기화
- 6인/8인 역할 배정
- 낮/AI 변론/AI 판결/밤/결과/엔딩으로 이어지는 전체 게임 루프
- 외계인, 시스템 엔지니어, 영상 엔지니어, 밀항자 능력 구현
- 사망자/관전자 처리
- AI 판결 서버 연동
- 게임 종료 및 결과 공개
- Docker/Jenkins 기반 배포 문서화

## 17. 개인적으로 어필할 수 있는 부분

### PM 역량

- 제한된 기간 안에서 “완주 가능한 실시간 게임”을 최우선 목표로 설정
- 팀 컨벤션, Git 전략, Jira 흐름을 정리해 협업 비용 감소
- 기획, FE, BE, AI 간 계약이 필요한 지점을 먼저 정의
- 기능 욕심보다 핵심 게임 루프 완성에 집중
- 실시간 게임 특유의 예외 상황을 시나리오 단위로 정리

### Frontend 역량

- WebRTC와 WebSocket을 함께 사용하는 복합 실시간 화면 구현
- 서버 상태를 기반으로 stage를 안정적으로 전환하는 구조 구현
- 게임 규칙을 UI 권한, 영상 권한, 사운드, 애니메이션에 반영
- 사용자 행동이 명확하게 보이도록 역할별 상호작용 설계
- 작은 화면에서도 무너지지 않는 인게임 레이아웃 설계
- BGM, 효과음, 영상, 타이핑, 스캔라인, 글리치 등 몰입형 연출 구현

### 실시간 서비스 이해도

- WebRTC는 영상/음성 데이터, WebSocket은 게임 이벤트로 역할 분리
- JWT 기반 WebSocket 인증과 room/user topic 구조 설계
- Redis를 활용한 실시간 세션/방/액션 상태 관리 이해
- 연결 끊김, 중복 제출, 지연 응답 등 실시간 서비스의 경계 상황 처리

## 18. 이력서용 요약 문장

다음 문장들은 이력서나 자기소개서에 바로 활용할 수 있도록 정리한 내용입니다.

- 4인 팀 프로젝트에서 PM 및 Frontend를 담당하며, WebRTC 영상/음성과 WebSocket 실시간 이벤트를 결합한 SF 마피아 게임 A601을 개발했습니다.
- OpenVidu v3/LiveKit 기반 WebRTC를 활용해 실시간 영상/음성 룸을 구현하고, 게임 페이즈와 생존 여부에 따라 publish/subscribe 권한이 달라지는 구조를 설계했습니다.
- STOMP/SockJS WebSocket을 통해 방 입장, 준비 상태, 게임 시작, 페이즈 전환, 직업별 액션, AI 판결, 게임 종료 이벤트를 실시간으로 동기화했습니다.
- React와 Zustand를 기반으로 로비, 대기방, 낮, 밤, AI 변론, 판결, 엔딩으로 이어지는 전체 게임 화면과 상태 전환 구조를 구현했습니다.
- AI 판결 단계에서 텍스트/음성 변론 제출, 자동 제출, 판결 대기 연출, AI 결과 표시 UI를 구현해 기존 투표형 마피아와 다른 게임 경험을 설계했습니다.
- SF 세계관에 맞춘 BGM, 효과음, 영상, 타이핑, 글리치, 스캔라인, 역할별 타겟 효과를 적용해 게임 몰입도를 높였습니다.
- PM으로서 Jira 이슈 구조, Git 브랜치 전략, 커밋 컨벤션, 팀 코드 컨벤션을 정리하고, 기능 우선순위와 일정 범위를 조율했습니다.

## 19. 회고

가장 어려웠던 점은 “실시간”과 “게임 규칙”이 동시에 움직인다는 점이었습니다. 영상/음성 연결만 안정적이어도 부족하고, WebSocket 메시지만 정확해도 부족했습니다. 플레이어가 지금 살아있는지, 어떤 페이즈인지, 어떤 역할인지, 어떤 권한을 가져야 하는지가 모두 화면과 미디어 권한에 동시에 반영되어야 했습니다.

이 프로젝트를 통해 실시간 서비스에서 중요한 것은 단순히 데이터를 빠르게 주고받는 것이 아니라, **서버 기준의 상태를 신뢰할 수 있게 만들고, 클라이언트는 그 상태를 사용자 경험으로 자연스럽게 번역하는 것**이라는 점을 배웠습니다.

또한 PM으로서는 기능을 많이 넣는 것보다, 제한된 시간 안에서 팀이 같은 목표를 보고 움직이게 만드는 것이 중요하다는 점을 체감했습니다. 특히 A601처럼 FE, BE, AI, Infra가 모두 맞물리는 프로젝트에서는 의사결정 기준과 협업 규칙을 초기에 정리하는 것이 개발 속도와 완성도에 직접적인 영향을 준다는 것을 배웠습니다.

## 20. 관련 산출물

- [영상 포트폴리오](https://drive.google.com/file/d/1U3-hgDkIL4uVr8fKiIEmFyrDxVgn_6zn/view?usp=drive_link)
- [기능 명세서](https://wind-oval-efe.notion.site/2e6eb24de535818cb00ed97ea1a1d208?pvs=73)
- [API 명세서](https://wind-oval-efe.notion.site/API-2e2eb24de53581e4b3e4c9d9eee76863?pvs=74)
- [간트차트](https://docs.google.com/spreadsheets/d/1QTYnJbTTe41X3c6OtbqZKlY-3x4Qjuwpz5bJQpaAs_4/edit?usp=drive_link)
