---
title: 기술 소개·구동·배포 매뉴얼
parent: flow-cs-app
grand_parent: 제품
nav_order: 1
permalink: /products/flow-cs-app/manual/
---

{: .note }
> 원본은 제품 저장소의 `docs/shared/guide/상담AP-React판-기술-구동-배포-매뉴얼.md`다. 저장소 안 상대 링크는 경로로만 적었다.

기준 일자 2026-11-11, 브랜치 `flow-counsel-react`, 기준 커밋은 `git log -1`로 확인한다.

처음 이 저장소를 받아 든 사람을 위한 글이다. 무엇을 만드는 시스템인지, 어떤 재료로 짜여 있는지, 내 컴퓨터에서 어떻게 띄우고 고객사 서버에 어떻게 얹는지를 한 번에 따라갈 수 있게 적었다. 사실은 전부 저장소의 문서와 `pom.xml`·`bin/`·`config/`에서 가져왔고, 저장소가 답하지 않는 자리는 "확인 필요"로 남겼다.

이 저장소는 한 지붕 아래 두 집이다. 뒤채가 Spring Boot 백엔드(`src/`), 앞채가 React 프런트(`front-react/`)이고, 둘 사이의 문은 `mvn package`가 닫아 준다. 자세한 절차는 각 가이드에 있으므로 이 글은 길을 알려 주는 쪽에 선다. 절마다 끝에 어느 문서로 가면 되는지 적어 두었다.

## 1. 제품 소개

Flow 상담 AP는 컨택센터 상담사가 하루를 보내는 작업대다. 전화가 들어오면 소프트폰이 받고 통합상담 화면이 고객을 찾아 상담 기록을 연다. 실시간 자막이 흐르고, 오른쪽 레일에서 VOC와 평가를 처리한다. 머리의 지식베이스 단추는 KMS를 검색하는 떠 있는 패널을 연다. 그 뒤에 사용자·조직·코드·메뉴·권한·시스템변수·공지·게시판·문자템플릿 같은 관리 화면이 있고, 콜백 관리와 상담유형 관리가 상담의 양옆에 선다.

이 브랜치는 이름만 브랜치이고 사실상 별도 제품이다. 고객사 요청으로 프런트를 React로 납품하기 위해 갈라졌고, Vue 판은 `main`에 그대로 산다. 고객사에 따라 Vue 판 또는 React 판이 나가며, 납품 뒤에는 고객사 개발자가 외부 인터넷이 없는 내부망에서 직접 개발하고 운영한다. 그래서 코드와 문서는 사람이 읽고 따라 하기 쉬워야 한다는 요구가 설계의 첫 줄에 있다. 저장소의 `front/`(Vue)는 고치지 않고 동작을 확인하는 참고 자료로만 쓰며, 이 브랜치에서 Vue의 동작은 보장하지 않는다.

소프트폰은 iPRON CTI에 붙는다. 방식은 시스템변수 `softphone-type`으로 `ICAPI`(HTTP 롱폴 SDK) 또는 `WEBSOCKET`(WS SDK, 선택으로 SIP/WebRTC 음성)을 고른다. 두 방식 모두 React로 옮겨졌지만 가짜 어댑터와 단위 시험으로만 검증했고, iPRON 실기 환경에서는 아직 확인하지 않았다. 이 점은 이 글의 여러 곳에 되풀이해 나온다.

지금까지 만들어진 범위는 단계표로 적혀 있다. 아래가 2026-11-11 기준으로 끝난 묶음이다. 3명이 규칙 문서와 기준 화면을 함께 검토해 확정하는 일은 남았다.

| 단계 | 들어 있는 것 |
|---|---|
| 0단계(0-A~0-F) | 뼈대·디자인 토큰, 인증과 API 계층, `shared/ui` 공통 부품, 로그인과 셸, `FlowGrid`와 화면 틀, Maven 패키징·오프라인 빌드·CSP·SBOM·E2E |
| 1단계 관리 묶음 | 알림·쪽지·접속로그·사용자·사이트링크·조직·코드·메뉴·시스템변수·권한·공지사항·게시판·문자템플릿·프로필 14개 화면 |
| 2-1 | 셸 계약(통화 세션·팝업 창·인자 있는 탭·시스템변수 알림), 고객정보·상담이력 관리 |
| 2-3, 2-4, 2-4b | 통합상담 core, 오른쪽 레일(VOC·평가), 실시간 STT 대화 |
| 2-5 | 콜백 관리, 상담유형 관리 |
| 2-2, 2-2b | 소프트폰 ICAPI, WEBSOCKET + SIP(실기 미검증) |
| 2-6 | KMS 셸 검색(떠 있는 패널) |
| 빌드 기반 | npm → pnpm 12.18.0 전환 |

아직 없는 것도 분명하다. 채팅·메일 채널과 Advisor 레일(실시간 Assist·AI 요약·상담유형 추천과 KMS 추천·하이브리드·AI 검색. 2026-11-11에 백엔드 중계로 받기로 정했다), 상담현황 대시보드는 다음 묶음이다. 캠페인과 시험(교육)은 React 판으로 옮기지 않기로 했고 백엔드 코드만 남는다. 운영 프로필은 이 경로를 `disabled-api-prefixes`로 닫아 둔다. 제외 영역의 메뉴와 단추는 React 판에서 숨긴다.

단계표와 남은 일의 원본은 react-migration(`rule/rules/react-migration.md`)의 "진행 단계"와 "남은 2단계"다.

## 2. 기술 구성

### 백엔드

Java 21 위의 Spring Boot 3.4(부모 3.4.0)이고 서블릿 컨테이너는 Undertow다. 인증은 Spring Security와 JWT(jjwt)로, 액세스 토큰은 응답 본문으로, 리프레시 토큰은 HttpOnly 쿠키 `flow_refresh_token`으로 내린다. 토큰 폐기 목록과 자동 로그아웃 감지, 서버 사이 알림 중계는 Redis가 맡는다. 운영 프로필(`op`, `aws-prod`)은 `com.wecoms24.flow.redis.required: true`라 Redis 없이는 기동하지 않는다.

데이터 계층은 JPA와 MyBatis를 함께 쓴다. 주 DB는 MariaDB(`org.mariadb.jdbc.Driver`)이고 프로필의 `ddl-auto: update`가 첫 기동 때 테이블을 만든다. 기동할 때마다 `initial-data.yml`의 시스템변수·코드·역할·메뉴를 "없을 때만" 넣고, 관리자 계정(`admin`)이 없으면 만든다. Lombok은 `pom.xml`이 1.18.38로 고정해 두었다.

REST 경로는 모두 `/rest/api/v1` 아래다. `FlowAppConstants`가 접두를 들고 있고, 도메인마다 `/rest/api/v1/{domain}/{resource}` 모양으로 이어진다. 자주 보는 것만 적으면 `auth`·`user/users`·`settings/{code,menu,role-group,system/variable}`·`counsel/type/types`·`consultation/consultations`·`callback/callbacks`·`customer-info/customer-infos`·`voc`·`evaluation/...`·`content/contents`(KMS)·`stt/realtime`·`ui`(그리드 라이선스)다. 최상위 패키지는 `auth`·`board`·`campaign`·`campaignredesign`·`chat`·`chatbot`·`common`·`core`·`counsel`·`dashboard`·`evaluation`·`exam`·`integration`·`kms`·`settings`·`softphone`·`ums`·`user`·`webrtc`다.

점검 경로는 둘뿐이다. `GET /actuator/health`는 로그인 없이 `{"status":"UP"}` 같은 상태 값만 주고, `/actuator/info`는 로그인한 사용자에게 빌드 버전을 보여 준다. Swagger UI(`/swagger-ui.html`)는 로컬과 dev에서만 켜져 있다.

### 프런트

React 19.3.1과 TypeScript 6.0.3, Vite 8.3.6, Mantine 9.7.5가 바탕이다. 서버 데이터는 TanStack Query 5가 받아 캐시하고, 로그인 상태 같은 작은 전역 상태는 Zustand 5가 든다. HTTP는 axios이고 라우팅은 react-router 7이다. 실시간은 `@stomp/stompjs` 7, 리치 텍스트는 Tiptap 3, HTML 정제는 DOMPurify가 맡는다. 목록 그리드는 AUIGrid 3.0.20.2를 벤더 파일(`public/vendor/auigrid/`)로 들였고 라이선스가 없으면 HTML 표로 그린다. iPRON SDK 두 벌(`ipron.js`, `ipronWS.js`)과 JsSIP 수정판 + `sa.js`도 벤더 파일이다. 전체 목록과 라이선스는 DEPENDENCIES.md(`front-react/DEPENDENCIES.md`)에 있다.

패키지 관리자는 pnpm 12.18.0이다. `package.json`의 `packageManager`가 버전을 고정하고, Node는 24.15.0(`front-react/.nvmrc`)이다. 저장소 루트의 `.node-version`은 Vue용 20.20.2이므로 헷갈리지 않는다.

### 둘이 어떻게 붙는가

개발할 때 두 집은 각자 문을 연다. 백엔드는 대개 8090에서, Vite 개발 서버는 5173에서 뜬다. Vite가 `/rest/api/v1`과 `/ws/` 요청을 `http://127.0.0.1:8090`으로 넘기고(`FLOW_DEV_API_TARGET`으로 바꿀 수 있다), 웹소켓 연결(`/ws/websocket`)도 같이 넘긴다. 그래서 브라우저는 5173 하나만 본다.

배포할 때는 한 채가 된다. `mvn package`가 `prepare-package` 단계에서 Node 24.15.0을 `front-react/.node`에 받고, Node에 든 npm의 `exec`로 pnpm 12.18.0을 받아 `pnpm install --frozen-lockfile`과 `pnpm run build`를 돌린다. 결과 `front-react/dist`를 같은 단계의 `copy-react-dist`가 jar의 `static/`으로 복사한다. 그다음 `pnpm run sbom`과 `pnpm run licenses`가 보안 산출물을 만든다. `mvn compile`과 `mvn test`는 npm을 부르지 않으며, `-DskipFrontend=true`를 주면 화면 빌드를 통째로 건너뛴다. `front/`(Vue)는 더 이상 빌드하지 않는다.

### 디렉터리 지도

| 경로 | 무엇이 있는가 |
|---|---|
| `src/main/java/com/wecoms24/flow/` | 백엔드. 도메인 패키지와 `FlowAppConstants` |
| `src/main/resources/` | `initial-data.yml`, 메시지, 커밋된 정적 자산(favicon, `icons/`) |
| `front-react/src/app/` | 진입점·라우터·프로바이더·셸·화면 등록표(`screenRegistry.ts`)·앱 수준 리스너(`consult/`, `stt/`, `softphone/`) |
| `front-react/src/shared/` | 공통 부품(`ui/`), 화면 틀(`page/`), 그리드, API, 인증, 실시간, 통화·상담·STT 저장소, VOC·평가·KMS 공용 |
| `front-react/src/softphone/` | 소프트폰 어댑터 계약, 통화 추적기, ICAPI·WS·가짜 어댑터, SIP 계층 |
| `front-react/src/features/<이름>/` | 업무 화면 하나씩. 화면마다 `PARITY.md` |
| `front-react/e2e/` | Playwright E2E(화면마다 `<영역>.spec.ts`) |
| `front-react/catalog/` | 공통 부품 카탈로그(Storybook) |
| `front/` | Vue 참고 자료. 고치지 않는다 |
| `config/` | `application.yml`, 프로필별 `application-*.yml`, `environment-*.properties`, `logback-*.xml`, `application-secret.yml.sample` |
| `bin/` | `server-start.sh`, `server-stop.sh`, `server-restart.sh`, `generate-secrets.sh`, `make-offline-bundle.sh`, `verify-offline-build.sh` |
| `assembly-*.xml` | 프로필별 배포 zip 구성 |
| `ddl/` | 수동 적용 SQL |
| `rule/`, `docs/` | 작업 규칙과 가이드 |

### 실시간 구조

서버는 Redis Pub/Sub으로 서버 사이에 메시지를 중계하고, 각 서버가 STOMP WebSocket(`/ws`)으로 브라우저에 내린다. 중계하는 토픽 목록은 설정이 아니라 코드(`FlowAppConstants.REDIS_BROADCAST_TOPICS`)가 정한다. 공지는 `/topic/regist-notice`이고 사용자별 토픽은 `/topic/transmit-ticket`·`/topic/ticket-reservation`·`/topic/send-message`·`/topic/callback-ticket`·`/topic/request/init-password` 다섯이다. 기능 스위치는 `/topic/advisor-feature`, STT는 `/topic/stt/{message,event,result,anomaly}` 네 토픽이며 그 밖에 채팅·대시보드·캠페인 토픽이 있다.

브라우저 쪽에서는 `shared/realtime`의 `realtimeClient` 하나가 연결을 갖고, 셸(`MainLayout`의 `useShellSubscriptions`)만 구독을 건다. 탭 화면은 구독하지 않는다. 숨긴 탭은 Effect가 멈춰 메시지를 놓치기 때문이다. 받은 알림은 `notify`와 쿼리 무효화로 화면에 닿고, STT 메시지는 `sttStore`로 간다. STT 토픽은 로그인 사용자의 내선이 붙은 주소만 서버 `StompFrameGuard`가 허락한다. 끊기면 5초 뒤 다시 붙고, 서버가 ERROR 프레임으로 거절하면 60초부터 두 배씩 늘려 5분에서 멈춘다. 머리의 점이 "실시간 연결됨"인지로 상태를 본다.

### 소프트폰 구조

층은 셋이다. `src/softphone/`이 어댑터 계약과 통화 추적기 `callTracker`, ICAPI·WEBSOCKET·가짜 어댑터, SIP 오디오 계층을 갖는다. `src/app/softphone/`의 컨트롤러가 시스템변수·로그인 사용자·자격 API로 접속 설정을 만들어 어댑터를 띄우고 결과를 `callSessionStore`로 보낸다. `src/features/softphone/`은 머리 툴바와 인입 팝업, 다이얼패드, 호전환 대화상자이고 `useSoftphone()`만 쓴다.

발행 규칙의 원본은 추적기 하나다. ICAPI든 WEBSOCKET이든 CTI 이벤트를 추적기에 넣고, 추적기가 낸 `callInStarted`·`callOutRequested`·`callOutStarted`·`callEnded`·`processEnded`만 통화 세션 저장소에 간다. 통합상담의 리스너는 이 이벤트만 믿고 상담을 만들고 잇고 끝낸다. WEBSOCKET은 SDK 이벤트를 `normalizeWsEvent`로 ICAPI 모양으로 바꿔 같은 추적기에 넣으므로 상담 쪽 계약은 방식에 따라 달라지지 않는다.

SIP의 몫은 소리뿐이다. `softphone-ws-sip-enabled=true`면 [CTI 로그인] 때 마이크 권한 → SIP 등록(8초) → CTI 연결 순서로 가고, SIP 등록이 실패하면 CTI를 붙이지 않는다. 비밀번호(CTI·SIP·TURN)는 로그인 직전에 `POST /rest/api/v1/auth/softphone-credentials`로 한 번 받아 쓰고 버린다. 자세한 계약은 react-development(`rule/rules/react-development.md`)의 "실시간"·"통화 세션"·"소프트폰"이다.

## 3. 로컬 구동

### 전제 도구

Java 21과 Maven 3.9 이상, MariaDB와 Redis, Node 24.15.0과 pnpm 12.18.0이 필요하다. pnpm은 Node 24에 든 Corepack으로 `corepack enable`을 한 번 하거나 `npm i -g pnpm@12.18.0`으로 준비한다. 준비 없이 쓰려면 `corepack pnpm <명령>`으로 부른다.

### DB와 Redis

저장소에는 MariaDB와 Redis를 띄우는 compose 파일이 없다(`config/docker-compose.yml`은 Elasticsearch 용이다). 각자 띄우고 프로필 파일의 `datasource`와 `redis` 값에 맞춘다. DB에는 접속 주소에 적은 빈 스키마를 먼저 만든다. 개인 로컬 프로필은 `redis.required`가 기본 `false`라 Redis 없이도 뜨지만, 알림 중계와 자동 로그아웃 감지는 Redis가 있어야 돈다.

### 프로필과 비밀 파일

로컬 설정은 개인별 파일이고 공통 로컬 프로필은 없다. `config/application-local-<이름>.yml`은 있는 파일을 복사해 `on-profile`과 DB·Redis 접속 정보를 자기 것으로 고치고, `config/logback-local-<이름>.xml`은 `config/logback-local.xml`을 복사한다. `application.yml`의 `logging.config`가 프로필 이름으로 logback 파일을 찾으므로 이 파일이 없으면 서버가 뜨지 않는다. 포트는 대개 `8090`이며 파일에 `server.port`가 없으면 기본 `10202`다.

키 파일 `config/application-secret.yml`도 만들어야 한다. JWT 서명 키 2개와 데이터 암호화 키 3개는 저장소에 없고, 이 파일이 없으면 서버는 기동하지 않는다. 빈 DB로 시작하면 다음 한 줄이다.

```bash
bin/generate-secrets.sh --new-install
```

이미 데이터가 있는 로컬 DB를 쓰면 새 암호화 키를 만들면 안 된다. 예전에 저장된 전화번호가 옛 키로 암호화되어 있기 때문이다. 그때는 `--keep-encryption-from <예전 application.yml>`로 암호화 키를 옮기고 JWT 키만 새로 만든다. 절차는 로컬 실행 가이드(`로컬-실행-가이드.md`)의 "2. 백엔드 실행"과 운영 설정 가이드(`고객사-운영-설정-가이드.md`)의 "키와 비밀값"에 있다.

### 백엔드 실행

저장소 루트에서 실행한다.

```bash
java -version
mvn spring-boot:run -Dspring-boot.run.profiles=local-<이름>
```

첫 실행에서 `ddl-auto: update`가 테이블을 만들고, 기본 데이터와 관리자 계정 `admin`이 들어간다. 초기 비밀번호는 설정 `com.wecoms24.flow.initial-admin-password`가 있으면 그 값이고, 없으면 서버를 띄운 폴더의 `initial-admin-password.txt`에 남는다. 첫 로그인 뒤 비밀번호를 바꿔야 다른 화면을 쓸 수 있다. 확인은 `http://localhost:8090/swagger-ui.html`과 `curl http://localhost:8090/actuator/health`다.

HTTP로 쓰면서 새로고침 뒤에도 로그인을 유지하려면 자기 프로필에 `com.wecoms24.flow.auth.refresh-cookie.secure: false`를 넣는다. 기본값 `true`인 채로 HTTP 접속하면 브라우저가 리프레시 쿠키를 저장하지 않아 새로고침마다 로그아웃된다.

jar로 띄워 보려면 프로필 없는 빌드에 `-Dassembly.skipAssembly=true`를 붙인다. 이 빌드는 React 화면도 함께 빌드하므로 `pnpm run dev`나 카탈로그를 띄운 채로 돌리지 않는다.

```bash
mvn clean package -DskipTests -Dassembly.skipAssembly=true
java -jar target/counsel-flow-app.jar --spring.profiles.active=local-<이름>
```

### 프런트 실행

```bash
cd front-react
corepack pnpm install --frozen-lockfile
pnpm run dev
```

주소는 출력에 나온다(기본 `http://localhost:5173`). 처음 열면 "로그인 상태를 확인하고 있습니다"가 잠깐 보인 뒤 로그인 화면으로 가고, 백엔드가 꺼져 있으면 "서버에 연결할 수 없습니다"와 다시 시도 단추가 보인다. 백엔드 포트가 8090이 아니면 `FLOW_DEV_API_TARGET=http://127.0.0.1:<포트> pnpm run dev`로 띄운다.

로그인하면 머리(햄버거·홈·로고·업무지원·지식베이스·실시간 점·알림·사용자 칩)와 소프트폰 툴바, 왼쪽 퀵메뉴, 탭 작업공간이 보이고 홈 탭에 통합상담이 뜬다. 메뉴에서 고른 화면 중 아직 옮기지 않은 것은 "준비 중" 탭으로 열린다.

### 검증

프런트 검사는 `front-react`에서 돌린다. `pnpm run verify`가 타입 검사·ESLint·CSS 린트·Vitest·의존성 검사 다섯을 차례로 돌리고 하나라도 실패하면 멈춘다. `pnpm run build`는 타입 검사 뒤 배포용 번들(`dist/`)을 만들고, `pnpm run catalog`는 공통 부품 카탈로그를 `http://localhost:6006`에 띄운다.

```bash
pnpm run verify
pnpm run build
pnpm run catalog
```

E2E는 `verify`에 들어 있지 않다. 서버와 계정이 있어야 하기 때문이다. 백엔드(8090)와 개발 서버(5173)를 띄우고, 처음 한 번 `pnpm exec playwright install chromium`으로 브라우저를 받은 뒤 계정을 환경 변수로 주고 돌린다. 계정이 없으면 시험은 건너뛴다. 실패 기록(`test-results/`)에는 입력한 값이 그대로 남으므로 남에게 넘기지 않는다.

```bash
export FLOW_E2E_USER=<아이디>
read -rs FLOW_E2E_PASSWORD && export FLOW_E2E_PASSWORD
pnpm run e2e
pnpm run e2e:ui
pnpm run e2e:report
```

백엔드는 저장소 루트에서 `mvn test`다. 1000건이 넘어 몇 분 걸린다. 자세한 읽는 법은 React 프런트엔드 개발 가이드(`React-프런트엔드-개발-가이드.md`)의 "검사"와 "E2E 돌리기", e2e/README.md(`front-react/e2e/README.md`)에 있다.

### 소프트폰을 로컬에서 켤 때

CTI 서버 없이도 상담 화면은 쓸 수 있다. 통화가 오지 않으니 수동 신규 상담만 되고, 개발 서버에서는 브라우저 콘솔의 `window.flowDev.softphone.useFake()`나 `window.flowDev.call.inbound('01012345678')`로 통화를 흉내 낸다. 운영 번들에는 이 주입구가 없다.

실제 CTI에 붙이려면 네 가지가 맞아야 한다. 첫째, 로그인한 사용자에게 CTI 아이디·CTI 비밀번호·내선이 있어야 하고, SIP를 켠 고객사면 SIP 아이디·비밀번호도 있어야 한다. 없으면 툴바가 "CTI AgentID가 없습니다" 같은 문구로 잠긴다. 둘째, 시스템변수 "CTI 서버 정보" 묶음(`cti-server-protocol`·`cti-server-active-host`·`-port`·`cti-server-standby-*`·`cti-server-app-name`·`cti-server-tenant-name`)과 "소프트폰 설정" 묶음(`softphone-type`·`softphone-ws-ipron-server1/2`·`softphone-ws-sip-enabled`·`softphone-ws-server-address`·`-domain`·TURN 값들)이 고객사 값이어야 한다. 기본 데이터의 값은 벤더 시연 서버다. 셋째, 로컬 CSP `connect-src`에 CTI와 SIP 주소가 있어야 [CTI 로그인]이 통과한다. 넷째, SIP 음성은 마이크(`getUserMedia`)를 쓰므로 https 또는 `localhost`에서만 열린다.

마이크 권한은 따로 알린다. 2026-11-11 실기 확인에서 마이크 권한 거부가 SIP 등록 실패와 같은 문구로 뭉쳐 원인을 못 찾은 일이 있어 커밋 `58a6f608`이 둘을 갈랐다. 지금은 브라우저나 OS가 마이크를 막으면 "마이크 권한이 필요합니다. 브라우저 주소창의 마이크 권한을 허용한 뒤 다시 로그인하세요."가 뜨고, 그 밖의 실패(벤더 적재·소켓·등록·타임아웃)는 "SIP 등록에 실패했습니다. 소프트폰 설정을 확인하세요."다. 앞 문구는 설정 탓이 아니고, 뒤 문구는 설정을 본다.

## 4. 빌드와 배포

### 묶음 만들기

배포 묶음은 저장소 루트에서 환경 프로필을 붙여 만든다. 프로필(`dev`·`dev-omni`·`aws-prod`·`op`)이 묶음 설명 파일 `assembly-<id>.xml`과 넣을 설정 파일을 고르며, 프로필이 없으면 zip을 만드는 단계에서 실패한다.

```bash
mvn clean package -DskipTests -P op
```

zip을 풀면 jar 하나(`counsel-flow-app.jar`. `aws-prod`와 `dev-omni`는 `omni-flow-app.jar`)와 `bin/`(`server-start.sh`·`server-stop.sh`·`server-restart.sh`·`generate-secrets.sh`), `config/`(`application.yml`·`application-<id>.yml`·`environment.properties`·`logback-<id>.xml`·`application-secret.yml.sample`), `sbom/`이 나온다. 화면 파일은 jar 안의 `static/`에만 있고 zip 루트에 따로 풀리지 않는다. 비밀 파일은 견본만 들어간다.

| 프로필 | 용도 | 포트 |
|---|---|---|
| `local-<이름>` | 개인별 로컬 개발 | `8090` (`server.port`가 없으면 `10202`) |
| `dev` | 개발 서버 | `10202` |
| `dev-omni` | 개발 서버(omni) | `10202` |
| `aws-prod` | AWS 운영 | `10202` |
| `op` | 운영 | `8080` |

### 서버에서 띄우고 내리기

묶음을 앱 홈에 풀면 `config/environment.properties`가 기동의 열쇠다. `JAVA`(자바 경로)·`LOG_HOME`·`ARTIFACT_ID`·`PROFILE_NAME`·`JAR_FILE_NAME`·`LOG_FILE_NAME`·`JAVA_HEAP_SIZE`·`JAVA_HEAP_MAX_SIZE`·`SERVER_PORT`(헬스 확인 포트)·`FLOW_SECRET_FILE`(비밀 파일 경로)을 서버에 맞춘다. 기동 스크립트는 이 파일이 없거나 `LOG_HOME`이 비어 있으면 멈춘다.

```bash
bin/server-start.sh
bin/server-stop.sh
bin/server-restart.sh
```

`server-start.sh`는 비밀 파일이 있는지와 실행 계정이 읽을 수 있는지, jar가 있는지를 먼저 보고 JVM을 띄운다. 그 뒤 `http://localhost:<SERVER_PORT>/actuator/health`를 2초마다 불러 최대 120초(`HEALTH_TIMEOUT_SECONDS`) 기다리고, `UP`을 받았을 때만 `started successfully!!!`를 찍는다. 실패하면 `did not start!!!`와 함께 `console.log`와 애플리케이션 로그의 경로, 가장 최근 로그의 마지막 40줄을 보여 주고 종료 코드 1로 끝난다. 프로세스가 아직 떠 있으면 PID를 알려 주며, 그대로 두면 다음 `server-start.sh`가 "already running"으로 끝나므로 `server-stop.sh`로 내린 뒤 원인을 고친다. `server-stop.sh`는 jar 이름으로 프로세스를 찾아 `kill -9`한다.

### `mvn deploy`

`mvn deploy -P <프로필>`은 그 프로필의 서버에 scp로 zip을 올린 뒤 원격에서 앱 홈을 `rm -rf`로 지우고, zip을 다시 풀고, `server-stop.sh`와 `server-start.sh`를 차례로 부른다. 앱 홈 안의 모든 것이 사라진다는 뜻이다. 그래서 비밀 파일은 앱 홈 밖(`FLOW_SECRET_FILE` 경로)에 두고, 서버에서만 고친 설정은 다음 배포 때 사라지므로 저장소의 같은 파일도 함께 고친다. 접속 비밀번호는 `pom.xml`이 아니라 Maven `settings.xml`의 `<server>`(id는 프로필의 `remote.deploy.server.id`)에 둔다. 로컬 확인에 이 명령을 쓰지 않는다.

### 환경별 설정 덮어쓰기

서버는 프로필 파일과 logback 설정을 실행 디렉터리의 `config/`에서 읽는다. jar에 든 것은 공용 기본 설정 `application.yml`뿐이고 다른 환경의 프로필 파일은 없다. 그래서 고객사 값은 `config/application-<프로필>.yml`에 적고, 이미 있는 `com:` → `wecoms24:` → `flow:` 블록 안에 넣는다. `com:`을 한 번 더 쓰면 중복 키로 서버가 뜨지 않는다. 처음 설치 때 반드시 바꾸는 키는 `spring.datasource.main.url`·`username`·`password`와 `com.wecoms24.flow.redis.host`·`port`, `com.wecoms24.flow.core.file.host`·`rootPath`, `server.undertow.accesslog.dir`·`server.port`·`com.wecoms24.flow.isMasterServer`다.

키와 비밀값은 세 길 중 하나로 넣는다. 비밀 파일(`FLOW_SECRET_FILE`)과 `FLOW_*` 환경변수(`FLOW_JWT_ACCESS_KEY`·`FLOW_JWT_REFRESH_KEY`·`FLOW_PHONE_ENCRYPTION_KEY`·`FLOW_ADVISOR_ENCRYPTION_KEY`·`FLOW_TARGET_SOURCE_ENCRYPTION_KEY`·`FLOW_AUIGRID_LICENSE`), 그리고 Spring 설정 이름의 환경변수나 `-D` 시스템 속성이다. 파일과 `FLOW_*` 환경변수가 둘 다 있으면 파일 값을 쓰고, Spring 설정 이름 쪽은 파일보다 앞선다. 한 서버에서는 한 가지 방식만 쓴다. 설정 파일 위치 자체를 바꾸어야 하는 설치는 Spring 표준 `-Dspring.config.location`으로 할 수 있으나 저장소 문서가 다루지 않으므로 확인 필요다.

### 오프라인(내부망) 빌드

내부망에는 nodejs.org도 npm 레지스트리도 닿지 않는다. 그래서 인터넷이 되는 머신에서 오프라인 묶음을 만들어 반입하고 `-Poffline`으로 빌드한다. 묶음은 빌드할 커밋(같은 `pnpm-lock.yaml`)에서 만들어야 하며, lock 하나에 묶음 하나다.

```bash
# 온라인 머신, 저장소 루트
bin/make-offline-bundle.sh                          # linux-x64
bin/make-offline-bundle.sh --platform darwin-arm64  # 다른 플랫폼. 여러 번 줄 수 있다
bin/make-offline-bundle.sh --all                    # 네 플랫폼 모두
```

스크립트는 Node 압축본을 `SHASUMS256.txt`로 검증해 받고, pnpm 자신을 `offline/npm-cache`에, 패키지를 `pnpm fetch`로 `offline/pnpm-store`에 담은 뒤 묶음만으로 설치되는지 확인하고 완성 표지 `offline/pnpm-lock.sha256`을 쓴다. `offline/` 폴더를 통째로 내부망 저장소 루트 바로 아래에 옮긴다. 백엔드 의존성과 Maven 플러그인은 이 묶음에 없으므로 사내 Maven 저장소가 없으면 온라인 머신에서 같은 명령(`clean package`)을 한 번 성공시킨 `~/.m2/repository`를 함께 반입한다. `clean`을 빼면 `maven-clean-plugin`이 저장소에 없어 내부망에서 바로 멈춘다.

```bash
# 내부망 빌드 머신
bin/verify-offline-build.sh --profile op --yes      # 묶음과 lock이 맞는지 미리 확인
mvn -o -Poffline -Pop -DskipTests clean package
```

`offline` 프로필은 Node를 `file://<저장소>/offline/node/`에서 읽고, pnpm을 묶음의 npm 캐시에서 꺼내고, 설치를 `--offline --store-dir`로 돌린다. 묶음 구성 요소가 하나라도 없으면 `validate` 단계에서 "offline/ 묶음이 없습니다"로 멈춘다. 저장소 경로에 공백이 있으면 인자가 나뉘어 실패하니 공백 없는 경로에 둔다. 전체 절차와 E2E 브라우저 반입, 납품 전 점검은 React 오프라인 빌드 가이드(`React-오프라인-빌드-가이드.md`)에 있다.

### 보안 산출물

묶음의 `sbom/`에 고객사 보안 검토용 파일이 들어간다. 백엔드 SBOM은 `bom.json`·`bom.xml`(CycloneDX 1.6)이고 `cyclonedx-maven-plugin`이 Maven 온라인 모드에서만 만든다. 화면 SBOM `front-bom.json`은 `pnpm run sbom`이 운영 의존성만 담아 만들고, 라이선스 목록 `front-licenses.csv`는 `pnpm run licenses`가 만들며 허용 목록 밖 라이선스가 있으면 빌드가 멈춘다. `README.txt`에는 들어 있는 파일과 빠진 파일의 이유가 적힌다. 취약점 보고서 `audit.json`은 `-Paudit`를 더한 빌드에서만 만들어지며 npm 레지스트리에 묻기 때문에 인터넷이 필요하다.

```bash
mvn clean package -DskipTests -Pop,audit
```

오프라인 빌드는 백엔드 SBOM과 `audit.json`이 빠지고 그 사실이 `README.txt`에 적힌다. 납품에 필요하면 온라인 머신에서 만들어 따로 낸다.

### 헬스체크와 로그

`GET /actuator/health`는 상태 값만 준다. 세부 구성요소는 숨기므로 `DOWN`(HTTP 503)이면 애플리케이션 로그에서 Redis나 DB 연결 오류를 찾는다. 앞단 프록시가 외부에 노출하는 경로에 `/actuator`를 넣지 않는다.

로그는 두 곳이다. 애플리케이션 로그는 `$LOG_HOME/counsel-flow-app-날짜.번호.log`이고 logback이 50MB 단위로 나누어 14일, 합계 1GB까지 둔다. `$LOG_HOME/console.log`는 프로세스의 표준 출력으로 기동 스크립트가 이어 쓰며, 자동으로 나뉘지 않아 logrotate(`copytruncate`)로 돌린다. GC 로그는 `$LOG_HOME/gclogs`에 1MB씩 100개까지다. 모든 응답에 `X-Request-Id`가 붙고 로그 한 줄에 `[요청ID 사용자ID]`가 찍히므로, 고객이 알려 온 요청 ID로 그 요청의 로그만 `grep`할 수 있다.

## 5. 운영 설정 요약

처음 설치할 때 정하는 것들이다. 항목마다 한두 문장만 적고 자세한 절은 고객사 운영 설정 가이드(`고객사-운영-설정-가이드.md`)의 같은 이름 절로 연결한다.

세션 길이는 시스템변수 `base-expiration-time`(분, 기본 540)이 정한다. 갱신으로 늘어나지 않고 다음 로그인부터 적용되며, `refresh-expiration-time`(기본 1380)보다 작게 둔다. 절 "세션 길이 바꾸기".

HTTP로 쓰는 고객사는 `com.wecoms24.flow.auth.refresh-cookie.secure: false`를 넣는다. 그렇지 않으면 로그인은 되지만 새로고침하면 로그아웃된다. HTTP에는 구간 암호화가 없으므로 고객사와 합의해 둔다. 절 "HTTP로 쓰는 고객사".

프록시 뒤에 두면 `/ws/` 경로에 `Upgrade`·`Connection` 헤더를 넘기고 `Host`를 브라우저 값으로 유지해야 WebSocket이 붙는다. `server.forward-headers-strategy: native`는 앱 포트가 그 프록시에서만 닿고 프록시가 `X-Forwarded-For`를 덮어쓸 때만 켠다. 켤 수 없으면 `none`으로 두고 `com.wecoms24.flow.websocket.allowed-origins`에 브라우저 주소를 적는다. 절 "앞단 프록시와 접속 IP"와 "WebSocket 허용 오리진".

CSP는 모든 응답에 붙으며 기본은 같은 출처만이다. CTI(ICAPI는 `https://`, WEBSOCKET은 `wss://`)와 SIP WSS 주소를 `flow.security.csp.connect-src`에 주·대기 모두 적고, Advisor iframe은 `frame-src`에 적는다. 처음 넣을 때는 `report-only: true`로 띄워 위반만 보고 끈다. 절 "CSP"와 "소프트폰(ICAPI·WEBSOCKET)".

AUIGrid 라이선스는 고객사 접속 주소별로 발급받아 비밀 파일의 `flow.ui.auigrid-license`나 환경변수 `FLOW_AUIGRID_LICENSE`에 넣는다. 없으면 HTML 표로 그려지고 끌어 옮기기·셀 편집만 빠진다. 절 "그리드 라이선스(AUIGrid)".

비밀키는 설치마다 `bin/generate-secrets.sh`로 만든다. 새 설치는 `--new-install`, 데이터가 있는 서버는 `--keep-encryption-from`이다. 데이터가 있는 서버에서 `--new-install`을 쓰면 저장된 전화번호와 Advisor 비밀번호를 다시는 읽을 수 없다. 비밀 파일은 반드시 백업에 넣는다. 절 "키와 비밀값".

초기 관리자 비밀번호는 `com.wecoms24.flow.initial-admin-password`로 정하거나, 정하지 않으면 서버가 무작위 값을 `initial-admin-password.txt`에 남긴다. 첫 로그인 뒤 바꾸지 않으면 다른 요청이 1013(`PASSWORD_CHANGE_REQUIRED`)으로 거절된다. 예전 판의 샘플 계정(`GRADER1`, `USER1`, `USER2`)은 운영에서 삭제한다. 절 "기본 데이터와 초기 관리자".

운영 프로필은 Redis가 필수이고 `notify-keyspace-events Ex`가 켜져 있어야 자동 로그아웃 기록이 남는다. DB 연결 풀은 `op`가 20이며 동시 상담사 수의 1/3에서 1/2을 출발점으로 잡는다. 절 "Redis 필수 설정"과 "DB 연결 수".

## 6. 문제 해결

| 증상 | 원인 | 조치 |
|---|---|---|
| 서버가 뜨지 않고 "키와 비밀값 파일 … 이(가) 없어 기동하지 않습니다" | `application-secret.yml`이 없다 | 로컬은 `bin/generate-secrets.sh --new-install`(데이터가 있으면 `--keep-encryption-from`). 서버는 `FLOW_SECRET_FILE` 경로에 만들고 실행 계정이 읽을 수 있게 600으로 둔다 |
| 로컬에서 `mvn spring-boot:run`이 logback 설정을 못 찾고 멈춘다 | `config/logback-local-<이름>.xml`이 없다 | `config/logback-local.xml`을 복사해 프로필 이름으로 둔다 |
| 로그에 `Connection is not available, request timed out` | DB 연결 풀이 모자란다. `connection-timeout` 20초를 기다린 뒤 실패한다 | 그 시각에 느린 외부 호출(Advisor 등)이 있었는지 먼저 본다. 그다음 `spring.datasource.main.hikari.maximum-pool-size`를 올린다(`op` 20, `dev` 5) |
| Java 21에서 Lombok 관련 컴파일 오류 | `pom.xml`은 Lombok을 1.18.38로 고정한다. IDE나 상위 설정이 다른 판을 끌어오면 Spring Boot 3.4.0 부모의 기본 Lombok이 JDK 21에서 깨진다(개발 메모 기준, 확인 필요) | `pom.xml`의 `lombok.version`을 그대로 두고 IDE의 어노테이션 프로세서가 같은 판인지 본다 |
| `pnpm install`이 `ERR_PNPM_IGNORED_BUILDS`로 멈춘다 | 설치 스크립트(postinstall)가 있는 새 패키지가 `pnpm-workspace.yaml`의 `allowBuilds`에 없다 | `allowBuilds`에 `true`/`false`를 이유 주석과 함께 적고 다시 설치한다. 절차는 `front-react/DEPENDENCIES.md` |
| `pnpm install`이 `ERR_PNPM_UNSUPPORTED_ENGINE`으로 멈춘다 | Node 24.15.0이 아니다 | `node -v`를 확인하고 PATH의 Node를 바꾼다. 루트 `.node-version`(20.20.2)은 Vue용이다 |
| 그리드가 HTML 표로 나온다 | AUIGrid 라이선스가 없거나 그 접속 주소용이 아니다 | `GET /rest/api/v1/ui/grid-license` 응답의 `license`가 `null`인지 본다. 유효한 키를 `flow.ui.auigrid-license`에 넣고 CSP에 `'unsafe-eval'`이 있는지 본다. 엔진이 "The Date has expired"를 띄우면 기간이 지난 키다 |
| [CTI 로그인] 때 "마이크 권한이 필요합니다…" | 브라우저나 OS가 마이크를 막았다. 설정 탓이 아니다 | 주소창 왼쪽 아이콘이나 OS 마이크 권한에서 브라우저를 허용하고 다시 로그인한다. https 또는 `localhost`가 아니면 마이크가 열리지 않는다 |
| [CTI 로그인] 때 "SIP 등록에 실패했습니다. 소프트폰 설정을 확인하세요." | 벤더 적재·소켓·등록·타임아웃(8초) 중 하나다 | `softphone-ws-server-address`·`-domain`, 사용자 SIP 계정, CSP `connect-src`의 SIP WSS, 방화벽의 TURN 포트를 본다. 콘솔에 `connect-src` 위반이 있으면 CSP다 |
| 툴바가 "CTI 서버 설정이 없습니다" 또는 "서버연결 오류" | CTI 시스템변수가 비었거나 CSP `connect-src`에 CTI 주소가 없다 | "CTI 서버 정보"·"소프트폰 설정" 묶음을 고객사 값으로 채우고, `connect-src`에 주·대기 주소를 스킴·포트까지 적는다. https 앱에 `http`/`ws://` 주소면 혼합 콘텐츠로 막힌다 |
| 로그인과 화면은 되는데 알림·STT만 조용하다. `/ws/...`가 403이거나 끊긴다 | 프록시가 `Upgrade`·`Connection`을 넘기지 않거나 `Host`를 앱 주소로 바꿔 같은 오리진 판정이 틀린다 | 운영 설정 가이드 "프록시 뒤의 WebSocket"의 nginx 예시대로 고친다. `/ws/info`가 200, 이어지는 연결이 101이면 정상이다 |
| 로그인은 되는데 새로고침하면 로그아웃된다 | HTTP 접속인데 리프레시 쿠키의 `Secure`가 `true`다 | `com.wecoms24.flow.auth.refresh-cookie.secure: false` |
| `mvn package`가 zip을 만드는 단계에서 실패한다 | 프로필이 없어 `assembly-<id>.xml`을 고를 수 없다 | `-P <프로필>`을 붙이거나, jar만 필요하면 `-Dassembly.skipAssembly=true` |
| `-Poffline` 빌드가 "offline/ 묶음이 없습니다"로 멈추거나 `pnpm install --offline`이 "store에 없다"로 실패한다 | 묶음이 없거나, 묶음을 만든 뒤 `pnpm-lock.yaml`이 바뀌었다 | 빌드할 커밋에서 `bin/make-offline-bundle.sh`로 다시 만든다. `bin/verify-offline-build.sh`가 lock 해시 불일치를 먼저 잡는다 |
| 운영 서버가 Redis 이유로 기동하지 않는다 | `op`·`aws-prod`는 `redis.required: true`인데 `redis.enable`이 `true`가 아니다 | Redis를 준비하고 `com.wecoms24.flow.redis.enable`·`host`·`port`를 맞춘다 |
| 로그인 뒤 모든 요청이 1013으로 거절된다 | 비밀번호 변경이 필요한 계정이다(초기 관리자, 새 사용자, 초기화 승인) | 비밀번호 변경 화면에서 바꾼다. E2E 계정으로는 쓸 수 없다 |

## 7. 참고 문서 지도

| 하려는 일 | 문서 |
|---|---|
| 제품 방향, 이관 범위, 단계표, 실기 확인 목록 | react-migration(`rule/rules/react-migration.md`) |
| 백엔드와 DB·Redis를 로컬에 띄우기 | 로컬 실행 가이드(`로컬-실행-가이드.md`) |
| 프런트 설치·실행·검사·E2E·카탈로그, 지금 할 수 있는 일, 인증과 세션 동작 | React 프런트엔드 개발 가이드(`React-프런트엔드-개발-가이드.md`) |
| 처음 설치, 세션 길이, HTTP 쿠키, 프록시, CSP, 그리드 라이선스, 키와 비밀값, 패키징, 보안 산출물, 로그, 헬스체크 | 고객사 운영 설정 가이드(`고객사-운영-설정-가이드.md`) |
| 내부망에서 화면까지 빌드하기 | React 오프라인 빌드 가이드(`React-오프라인-빌드-가이드.md`) |
| 소프트폰 시스템변수와 사용자 값의 뜻(경로는 Vue 기준) | 소프트폰 통합 가이드(`소프트폰-통합-가이드.md`), SIP 사용 가이드(`SIP-사용-가이드.md`) |
| React 디렉터리 책임, 실시간·통화·소프트폰·STT 계약, 금지 사항 | react-development(`rule/rules/react-development.md`) |
| 화면 하나 만드는 순서와 기준 화면 | react-page-patterns(`rule/rules/react-page-patterns.md`) |
| 빌드·패키징 경계 | runtime-boundaries(`rule/rules/runtime-boundaries.md`) |
| 프런트 패키지 목록과 추가 절차 | DEPENDENCIES.md(`front-react/DEPENDENCIES.md`) |
| E2E 환경 변수와 시험별 범위 | e2e/README.md(`front-react/e2e/README.md`) |
| 프런트 명령 목록 | front-react/AGENTS.md(`front-react/AGENTS.md`) |
| 저장소 전체 입구 | README.md(`README.md`), rule/index.md(`rule/index.md`) |
