# 260915_cc

![Spring Boot](https://img.shields.io/badge/Spring%20Boot-4.1.1-6DB33F?logo=springboot&logoColor=white)
![Java](https://img.shields.io/badge/Java-17-007396?logo=openjdk&logoColor=white)
![Spring Security](https://img.shields.io/badge/Spring%20Security-7-6DB33F?logo=springsecurity&logoColor=white)
![Spring AI](https://img.shields.io/badge/Spring%20AI-2.0.1-6DB33F)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-pgvector-4169E1?logo=postgresql&logoColor=white)
![Redis](https://img.shields.io/badge/Redis-DC382D?logo=redis&logoColor=white)
![Claude Code](https://img.shields.io/badge/Claude%20Code-D97757?logo=claude&logoColor=white)

Google OAuth2 로그인 기반 Spring Boot 4 웹 애플리케이션. 이 저장소는 `Claude Code`로 개발을 진행하며, 협업 방식(rules), 반복 작업 지식(skills), 자동 기록(hooks)을 `.claude/` 아래에 함께 관리한다.

## 기술 스택

- **Spring Boot 4.1.1** / Java 17 toolchain
- **Spring Security 7** (Google OAuth2 로그인)
- **Spring Data JPA** + PostgreSQL(pgvector), **Redis**
- **Spring AI 2.0.1** (Google GenAI 챗/임베딩 모델)
- Thymeleaf, springdoc-openapi, Lombok

## Rules — `CLAUDE.md`

프로젝트 루트의 `CLAUDE.md`는 `Claude Code`가 이 저장소에서 항상 따르는 행동 규칙이다.

- 모든 작업을 시작하기 전에 적절한 모델(`Opus 5` / `Sonnet 5`)과 effort를 먼저 제안하고, 사용자 동의와 `"시작하자"` 명시적 입력을 받은 뒤에만 실행한다.

**의의**: 모델/effort를 매번 사람이 직접 확인하게 함으로써, 작업 규모에 안 맞는 과금이나 응답 품질 저하를 방지하고 실행 전 합의 지점을 명시적으로 만든다.

## Skills — `.claude/skills/`

Skill은 특정 작업을 수행할 때 `Claude Code`가 참고하는 재사용 가능한 절차/지식 문서다.

| Skill | 설명 |
|---|---|
| `commit-convention` | 커밋 메시지를 `<영어 category> : <한글 설명>` 형식(`feat`, `fix`, `refactor`, `docs`, `test`, `chore`, `style`)으로 강제한다. |
| `spring-boot-4` | 이 프로젝트가 쓰는 `Spring Boot 4` / `Spring Security 7` / Jackson 3 / `JSpecify` 등 최신 문법과 breaking change를 정리해, 3.x 시절 문법(`antMatchers`, `spring-boot-starter-web` 등)으로 회귀하지 않도록 가이드한다. |

**의의**: 코드베이스별 컨벤션과 프레임워크 버전 지식을 대화 맥락이 아니라 파일로 고정해, 세션이 바뀌거나 새로 합류한 협업자(사람이든 에이전트든)도 동일한 기준으로 작업하게 한다.

## Hooks — `.claude/settings.json`

Hook은 `Claude Code`의 세션 라이프사이클 이벤트에 맞춰 자동 실행되는 셸 커맨드다. 이 프로젝트는 주요 이벤트마다 프로젝트 루트의 `claude.log`에 한 줄씩 기록을 남긴다.

| 이벤트 | 기록 내용 |
|---|---|
| `SessionStart` | 세션 시작 및 source(startup/resume/clear 등) |
| `UserPromptSubmit` | 사용자가 입력한 프롬프트 앞 50자 |
| `PreToolUse` (Bash) | 실행 직전 bash 명령어 앞 50자 |
| `PostToolUse` (Write/Edit) | 생성/수정된 파일 경로 |
| `Notification` | 알림 메시지 앞 50자 |
| `PreCompact` | 컨텍스트 압축 트리거(manual/auto) |
| `Stop` | 응답 종료 시각, session id, 마지막 응답 텍스트 앞 50자 |

**의의**: 세션이 끝나면 사라지는 대화 로그와 달리, 무엇을 언제 실행했고 무엇을 응답했는지가 파일로 남아 사후 감사·디버깅·작업 이력 추적이 가능해진다.

## TIL

### 2026-09-15

- **`SecurityFilterChain` 빈이 없으면 Spring Security는 모든 요청에 인증을 요구한다.** `spring-boot-starter-security` + `spring-boot-starter-security-oauth2-client`만 의존성에 넣어두면, 명시적인 `SecurityFilterChain` 설정 없이는 `/actuator/health`, `/swagger-ui.html` 같은 공개용 엔드포인트까지 OAuth2 로그인 뒤로 막혀버린다. `authorizeHttpRequests`에서 필요한 경로만 `permitAll()`로 열어주고 나머지는 `anyRequest().authenticated()`로 두는 최소 설정이 필요하다.
- **설정만 있는 스캐폴드에서 "최소 구현"은 범위 합의가 먼저다.** JPA/Redis/OAuth2/Spring AI/pgvector처럼 서브시스템 설정은 다 돼 있는데 실제 도메인 코드(엔티티/컨트롤러)가 하나도 없는 저장소에서는, "비즈니스 기능을 만드는 것"과 "설정된 배선이 실제로 동작하는지 검증하는 것"이 전혀 다른 작업이 된다. 먼저 범위를 좁혀 합의하고 시작하는 게 헛수고를 줄인다.
- **`@SpringBootTest`에 인프라 빈을 직접 autowire해서 배선 검증 테스트를 만들 수 있다.** `DataSource`, `RedisConnectionFactory`, `ChatModel`, `EmbeddingModel`, `VectorStore`를 필드로 주입받아 `assertThat(...).isNotNull()`만 확인해도, 컨텍스트 로딩 시점에 각 서브시스템 자동 설정이 깨지지 않았는지 빠르게 검증할 수 있다. (단, 이 테스트는 실제 Postgres/Redis가 떠 있어야 통과한다 — Hibernate가 부트스트랩 시점에 dialect 확인을 위해 DB에 접속하고, pgvector 벡터스토어도 `initialize-schema: true`면 스키마를 실제로 만들려고 시도한다.)
- **`docker-compose`로 pgvector 확장이 포함된 Postgres 이미지를 바로 쓸 수 있다.** `postgres:16` 대신 `pgvector/pgvector:pg16` 이미지를 쓰면 `CREATE EXTENSION vector` 없이도 벡터 컬럼/인덱스를 바로 쓸 수 있어서, Spring AI pgvector 벡터스토어의 `initialize-schema: true` 옵션과 바로 맞물린다.
- **다른 fork를 같은 워크스페이스로 가져올 땐 `git clone`보다 `git remote add` + `fetch`가 더 안전할 때가 있다.** 이미 커밋 안 된 변경사항이 있는 디렉터리에 다른 저장소를 그대로 clone할 순 없다. 대신 원격을 추가해서 fetch하면 브랜치(`origin/main`, `fork/main` 등)를 나란히 두고 비교할 수 있고, 실수로 다른 폴더에 잘못 clone했을 때도 되돌리기 쉽다.
- **워킹트리를 다른 브랜치 내용으로 완전히 바꿔치기해야 할 때는 `git switch --detach`가 유용하다.** `git stash push -u`로 현재 변경사항(추적/미추적 파일 전부)을 먼저 안전하게 보관한 뒤 `git switch --detach <remote>/<branch>`로 이동하면, 로컬 브랜치 포인터는 그대로 둔 채 워킹트리만 원하는 커밋 상태로 바꿔볼 수 있고 `git switch -` 한 번으로 원래대로 돌아올 수 있다. 커밋이나 푸시 없이도 "이 브랜치라면 어떤 모습일지" 확인할 수 있는 방법이다.
