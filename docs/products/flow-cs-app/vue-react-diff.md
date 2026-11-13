---
title: Vue → React 달라진 점
parent: flow-cs-app
grand_parent: 제품
nav_order: 3
permalink: /products/flow-cs-app/vue-react-diff/
---

기준 일자 2026-11-13, 브랜치 `flow-counsel-react`. 저장소 안 파일은 경로로만 적었다.

## 한눈에

React 판은 브라우저가 아는 것을 줄였다. ai-advisor에 직접 붙지 않고, 토큰을 저장소에 두지 않고, 비밀값은 필요할 때 서버에서 받는다. 통화(CTI·SIP)와 채널 SDK만 지금도 직접 붙는다.

![브라우저가 붙는 서버 · Vue 대 React]({{ "/docs/products/flow-cs-app/vue-react-connections.png" | relative_url }})

| 범주 | 한 줄로 |
| --- | --- |
| 로그인·세션 | 토큰을 `localStorage`에서 메모리 + HttpOnly 쿠키로, 401이면 자동 갱신 |
| 실시간 연결 | 웹소켓은 우리 백엔드 하나, Advisor는 Redis를 거쳐 중계, 구독은 내 내선만 |
| 암호화·비밀값 | 키는 저장소 밖 파일, 자격·라이선스는 쓸 때 받고 버림, CSP 헤더 |
| 권한 | 화면 숨김 + 서버 쓰기 검사 |
| 외부 연동 | 소프트폰·채널은 직접, 나머지는 백엔드 경유. 실기 검증은 남음 |
| 화면·빌드 | 공통 부품 조립, `mvn package`가 React를 빌드, 내부망 빌드 묶음 |

## 로그인·인증·세션

토큰이 브라우저 저장소에서 사라졌다. Vue는 액세스·리프레시 토큰을 `localStorage`에 두었고, React는 액세스 토큰을 메모리에만, 리프레시 토큰을 스크립트가 못 읽는 쿠키에 둔다.

| 항목 | Vue | React |
| --- | --- | --- |
| 액세스 토큰 | `localStorage`(`accessToken`) | 탭 메모리(`tokenStore`). 새로고침하면 사라지고 쿠키로 다시 받는다 |
| 리프레시 토큰 | `localStorage`(`refreshToken`, 만료 시각도 함께) | HttpOnly·`SameSite=Strict` 쿠키 `flow_refresh_token`(경로 `/rest/api/v1/auth`). 프런트 코드는 값을 보지 못한다 |
| 쿠키 오용 막기 | 해당 없음 | 모든 요청에 `X-Flow-Auth-Mode: cookie`. 서버는 이 헤더가 있을 때만 쿠키를 쓴다(다른 사이트가 유도한 요청은 못 붙인다) |
| 토큰 주인 확인 | 없음 | 토큰을 저장하기 전에 `login-info`로 주인을 읽는다. 다른 사용자 토큰이면 저장하지 않고 세션을 끝낸다 |
| 여러 탭 | 탭마다 저장소를 함께 읽음 | 탭끼리 토큰을 주고받지 않는다. `BroadcastChannel`로 "로그인했다·로그아웃했다"만 알린다. 한 브라우저에 한 사용자 |
| 401 복구 | 자동 갱신 없음. 401이면 "토큰이 만료되었습니다" 알림 뒤 로그인 화면으로 보낸다(갱신 API 함수는 있지만 부르는 곳이 없다) | `apiClient`가 한 번만 갱신하고 실패한 요청들을 다시 보낸다. 화면은 토큰을 직접 만지지 않는다 |
| 서버 장애 | "서버와 연결할 수 없습니다" 알림만 띄운다 | 로그아웃이 아니다. 앱 시작 때 장애면 "서버에 연결할 수 없음, 다시 시도" 화면을 보이고, 세션은 서버가 거절할 때만 끝낸다 |
| 비밀번호 | 클라이언트 SHA-256 → 서버 인코더 | 같다. 신규 사용자는 서버가 12자 임시 비밀번호를 만들어 한 번만 돌려주고 첫 로그인에 변경을 강제한다 |

- 로그인 화면 모양은 Vue 그대로 옮겼다(2단 카드·브랜드 바탕).
- 근거: `rule/rules/react-security.md` "토큰", 개발 가이드 "인증과 세션".

## 실시간 연결(웹소켓)

브라우저의 서버 쪽 실시간 연결이 우리 백엔드 `/ws` 하나로 모였다. Vue는 ai-advisor 서버에도 직접 웹소켓을 열었고, React는 ai-advisor 결과를 우리 백엔드가 받아 중계한다. 브라우저는 Redis에 직접 붙을 수 없으므로 Vue도 Redis를 직접 구독하지는 않았다.

| 항목 | Vue | React |
| --- | --- | --- |
| 우리 백엔드 연결 | SockJS + webstomp, `/ws` | 브라우저 기본 WebSocket + `@stomp/stompjs`, `/ws/websocket`(SockJS 라이브러리 없음) |
| Advisor 실시간 결과 | 시스템변수 `advisor-url`로 ai-advisor에 직접 연결, `/topic/assist/{세션}/{섹션}` 구독 | ai-advisor → Redis → 우리 백엔드 → STOMP `/topic/stt/assist/<섹션>/<내선>`(채팅·메일은 `/<대화 id>`) |
| Advisor 조회 API | ai-advisor 주소를 브라우저가 알아야 함 | 우리 백엔드 `/rest/api/v1/stt/realtime/...`을 거친다. 브라우저는 Advisor 주소·토큰을 모른다 |
| 구독 인가 | 주소만 알면 구독 | 서버 `StompFrameGuard`가 STT·어시스트 토픽을 로그인 사용자의 내선(채팅은 등록한 대화 id)에만 허락한다. 남의 내선 구독은 그 프레임만 버린다 |
| 누가 구독하나 | 화면(통합상담 등)이 직접 | 셸의 앱 수준 리스너만 구독하고 화면은 저장소를 읽는다. 탭을 닫아도 통화·STT·Advisor 상태가 이어진다 |
| 토큰 전달 | CONNECT 헤더 `Authorization` | 같다. 주소에는 넣지 않고, 디버그 출력을 꺼서 토큰·본문이 로그에 안 남는다. 토큰이 바뀌면 다시 연결 |

- 예외(브라우저가 바깥 서버에 직접 붙는 것): 소프트폰 CTI(ICAPI·WS)와 SIP 오디오, 채팅·메일 채널 SDK(`agent.js`)의 채널 서버 연결. "외부 시스템 연동"에 적었다.
- Redis Pub/Sub에는 인증이 없어 Redis는 내부망에 둔다.
- 근거: `front/src/pages/tabs/CallConsult.vue`(`resolveAdvisorAssistWebSocketUrl`), `rule/rules/react-security.md`, `docs/shared/guide/Advisor-Assist-중계-계약.md`.

## 암호화·비밀값·브라우저 보안

비밀값이 저장소와 브라우저에서 빠졌다. 키는 저장소 밖 파일로, 비밀번호·라이선스는 필요할 때 서버에서 받아 쓰고 버리도록 바꿨다.

| 항목 | Vue 때 | React 브랜치 |
| --- | --- | --- |
| JWT 키·암호화 키 | `config/application.yml`에 들어 저장소에 올라감 | 저장소 밖 `application-secret.yml`(`FLOW_SECRET_FILE`, 앱 폴더 밖). 없거나 짧거나 옛 키면 기동을 막는다(`SecretKeyGuard`). `bin/generate-secrets.sh`로 만든다 |
| 전화번호 칸 암호화 | 있었음(매번 새 IV) | 그대로. 상담이력·고객정보 조회를 해시 정확 일치(`telHash`)로 고쳤다. 부분 번호 검색은 안 된다 |
| 비밀 시스템변수 | 화면에서 비밀번호 값만 가림. 서버는 값으로 검색할 수 있었다 | 값을 가리고 값 검색에서 뺀다(TURN 비밀번호 등). 쓰기 전용·숨김 필드는 검색·정렬 불가 |
| 소프트폰 자격 | `GET /softphone/config`가 비밀번호를 실어 주고, SIP 비밀번호가 없으면 `아이디 + DN`으로 만듦 | CTI 로그인 직전 `POST /auth/softphone-credentials`(`no-store`)로 받고 시도가 끝나면 버린다. 만들어 내지 않는다 |
| AUIGrid 라이선스 | 라이선스 파일이 저장소에 있음 | 서버 설정 `flow.ui.auigrid-license`를 로그인 뒤 받아 `window.AUIGridLicense`에만 둔다. 없으면 HTML 표로 그린다 |
| 브라우저 저장소 | 토큰 포함 | 비밀이 아닌 화면 설정만(아이디 저장, CTI 체크, 내선, 색 구성). 린트가 토큰 저장을 막는다 |
| CSP 헤더 | 없음 | 서버 설정 `flow.security.csp`로 조립해 모든 응답에 내린다. 기본은 같은 출처, 고객사별로 채널·CTI·SIP·KMS 호스트를 더한다. AUIGrid가 `eval`로 풀리는 빌드라 `'unsafe-eval'`은 설정으로 켜고 끈다 |
| HTML 본문 출력 | v-html(13곳), 정제는 부르는 곳마다 따로 | 공지·KMS·에디터 본문은 `FlowSafeHtml`(DOMPurify 허용 목록)로만. 수신 메일은 스크립트 없는 `iframe sandbox`. 마크다운은 생 HTML을 해석하지 않는다 |
| 외부 창 주소 | 화면마다 window.open 직접 호출(17곳) | `http`·`https`만 연다. 시스템변수로 만든 주소(녹취 청취)도 호스트를 다시 본다 |

- 함께 한 백엔드 작업: 기본 계정 비밀번호 변경 강제, CSP 호스트 값 검증(지시문 주입 방지).
- 근거: 커밋 `ba7cf812`·`fbd81745`(키), `e87e96ef`·`8fb09bd3`(전화번호), `655f2efc`·`2855f483`(CSP), `rule/rules/react-security.md`.

## 권한

화면에서 단추를 숨기는 것에 더해 서버가 쓰기를 검사하도록 바꿨다. Vue 때는 여러 쓰기 API가 로그인만 하면 불릴 수 있었다.

- 화면: 권한이 있을 때만 단추를 그린다(`usePermission`). 저장·삭제 훅도 같은 권한 코드로 한 번 더 막는다(2중 확인).
- 메뉴: 권한 없는 화면은 탭을 열지 않고, 열린 뒤 권한이 사라지면 "권한 없음" 판을 보인다.
- 서버 검사를 더한 쓰기: 게시판(글·카테고리·조회수·삭제·댓글), 공지, 문자템플릿, 알림·쪽지(본인만), Advisor 작업, 콜백·상담유형·스크립트(`CallbackAuthority`·`CounselTypeAuthority`·`ScriptAuthority`), 권한 부여·권한그룹 변경(자기가 가진 것만, 레벨 검사).
- 근거: `rule/rules/react-page-patterns.md` "권한", 커밋 `ed95aa32`·`40e0e55f`·`ff2b990a`·`92b02570`·`8564f51f`·`49dbaded`·`c7df1808`·`f36558ab`.

## 외부 시스템 연동

연동마다 "브라우저가 직접 붙을지, 우리 백엔드를 거칠지"를 다시 정했다. 통화 미디어와 채널 SDK만 직접 붙고, 나머지는 백엔드를 거친다.

| 연동 | 연결 | Vue와 달라진 점 | 실기 검증 |
| --- | --- | --- | --- |
| 소프트폰 CTI(ICAPI·WS) | 브라우저 → CTI 서버 직접 | 어댑터 두 개(`src/softphone/icapi`·`ws`)가 같은 통화 추적기·명령 큐를 쓴다. 자격은 로그인 직전 API로만 받는다 | 가짜 어댑터로만 검증, 실기 미검증 |
| SIP 오디오 | 브라우저 → SIP·TURN 직접 | 외선 접두·내선 자릿수를 시스템변수로. 마이크 권한 거부를 등록 실패와 구분해 알린다 | 미검증 |
| 채팅·메일(FlowAgent) | 브라우저 → 채널 서버 직접(SDK) | `agent.js`를 번들에 넣지 않고 설정 API가 준 채널 서버에서만 런타임에 싣는다. 채팅 비밀번호·메일 토큰은 저장하지 않는다 | 미검증 |
| AI Advisor 실시간 | ai-advisor → Redis → 백엔드 → STOMP | Vue는 브라우저가 ai-advisor에 직접 연결. 채팅·메일 상담도 같은 중계에 합쳤다 | 미검증 |
| Advisor·KMS 검색 | 백엔드 프록시(`/rest/api/v1/kms/...`, `/rag/...`) | Vue도 기본은 프록시였다. 검색 결과·본문은 정제해 그린다 | 미검증 |

- 채널 서버·CTI·SIP·KMS 호스트는 고객사마다 CSP에 적어야 브라우저가 붙는다("암호화·비밀값·브라우저 보안"의 CSP).
- 개발용 주입구 `flowDev.call`·`stt`·`softphone`으로 가짜 통화·STT·소프트폰 이벤트를 넣어 시험한다.
- 근거: `rule/rules/react-security.md`(소프트폰 자격·외부 SDK), `CLAUDE.md`의 묶음 목록(2-2·2-2b·2-7·2-8·2-8b).

## 화면 틀·공통 부품·오류 처리

화면은 공통 부품을 조립해 만든다. 배치와 동작은 Vue와 같게 두고 겉모습과 코드는 새로 만들었다.

| 항목 | Vue | React |
| --- | --- | --- |
| 기술 | Quasar(Vue 3) | React 19 · TypeScript · Vite · Mantine 9. 화면은 Mantine을 직접 못 쓰고 `Flow*` 공통 부품만 쓴다 |
| 화면 조립 | 화면마다 직접 배치 | `ManagePage`(조회 영역 → 목록 카드 → 상세)와 공통 훅(`useListQuery`·`useMasterDetail`·`useSaveMutation` 등). 기준 화면을 베끼어 만든다 |
| 그리드 | AUIGrid 직접 | `FlowGrid` 하나. 라이선스가 있으면 AUIGrid, 없거나 엔진이 죽으면 HTML 표로 자동 전환. 무한 스크롤 + 바닥 푸터(건수·조회 시간) |
| 조회·입력 규칙 | 화면마다 다름 | 값을 넣으면 조회 조건 자동 체크, 칸 아래 인라인 검증과 첫 오류 초점, 저장 안 한 변경의 이탈 방지, 문구 통일(`MESSAGES`) |
| 오류 처리 | 상태 코드별 알림(`handleApiError`) | 서버 `FlowErrorCode`에서 만든 오류 코드 표로 분기. 훅이 `showError`로 한 번만 알린다 |
| 탭 작업공간 | 탭 화면 | 인자 있는 탭 열기, 팝업 창, 떠 있는 패널(KMS 검색). 통화·상담·STT 상태는 탭이 아니라 앱 수준 저장소에 있다 |
| 규칙 강제 | 일반 ESLint | 프로젝트 전용 린트가 생 HTML·인라인 스타일·토큰 저장·`eval`·화면의 Mantine 사용 등을 막는다. 부품 카탈로그(Storybook) |

- 옮기지 않은 것: 시험(교육), 캠페인, 구 상담 화면. 그 메뉴·버튼은 숨긴다.
- 소유자가 Vue와 다르게 정한 것: 머리 채널 툴바 압축, 통합상담 배치·상세 폼 압축, AI Advisor 레일 카드형, 메뉴 고정 제거 등. 나머지는 Vue 모양을 숫자로 맞춘다.
- 근거: `rule/rules/react-migration.md`, `rule/rules/react-page-patterns.md`, `rule/rules/react-development.md`.

## 빌드·배포·운영

`mvn package` 한 번으로 React 화면이 jar에 들어간다. 인터넷 없는 고객사 내부망에서도 빌드할 수 있게 묶음을 만든다.

| 항목 | Vue(`main`) | React(이 브랜치) |
| --- | --- | --- |
| 화면 빌드 | Maven이 `front/`(Quasar)를 빌드 | Maven이 `front-react/`를 pnpm으로 빌드. `front/`는 빌드하지 않는다 |
| 내부망 빌드 | — | `bin/make-offline-bundle.sh`로 Node·pnpm·패키지 묶음을 만들고 `-Poffline`으로 네트워크 없이 빌드 |
| 의존성 산출물 | — | 빌드 때 SBOM(`sbom/bom.json`)과 라이선스 목록을 만들고, 허용 목록 밖 라이선스면 빌드가 실패한다 |
| 비밀값 파일 | 설정 파일 안 | 앱 폴더 밖 `FLOW_SECRET_FILE`. 기동 스크립트가 없으면 멈추고, 배포의 삭제 뒤에도 남는다 |
| 고객사별 설정 | — | CSP 바깥 호스트(채널·ICAPI·SIP·CTI·KMS), WebSocket 허용 오리진, AUIGrid 라이선스를 서버 설정으로 |
| 앞단 프록시 | 같다(/ws를 썼다) | nginx가 `/ws/`의 `Upgrade`·`Connection`과 Host를 넘겨야 실시간 기능이 돈다(안 넘기면 로그인은 되고 실시간만 조용히 안 됨) |
| 시험 | — | 단위 시험 약 3,600개(Vitest), 주요 흐름 E2E(Playwright), 부품 카탈로그 |

- 고객사에 따라 Vue 판(`main`)과 React 판(이 브랜치)이 따로 나간다. 이 브랜치에서 Vue 동작은 보장하지 않는다.
- 근거: `pom.xml`(offline 프로필), `docs/shared/guide/고객사-운영-설정-가이드.md`, `rule/rules/react-migration.md`.
