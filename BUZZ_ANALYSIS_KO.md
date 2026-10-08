# 🐝 Buzz 프로젝트 전수조사 분석 정리 (한국어)

> **저장소:** https://github.com/bmshin94/buzz
> **원본 저장소:** https://github.com/block/buzz
> **분석 일자:** 2026-10-08
> **분석 대상 커밋:** `e7e7edc` (Update CLAUDE.md)
> **작성:** 카리나 (Claude Code) 와의 대화 정리

---

## 📑 목차

1. [프로젝트 정체 요약](#1-프로젝트-정체-요약)
2. [핵심 아이디어 3가지](#2-핵심-아이디어-3가지)
3. [폴더 전수조사 결과](#3-폴더-전수조사-결과)
4. [Rust 크레이트 맵](#4-rust-크레이트-맵)
5. [YAML 워크플로 엔진](#5-yaml-워크플로-엔진)
6. [buzz-cli (에이전트 우선 CLI)](#6-buzz-cli-에이전트-우선-cli)
7. [현재 완성도](#7-현재-완성도)
8. [쉬운 비유 설명](#8-쉬운-비유-설명)
9. [설치 및 사용법](#9-설치-및-사용법)
10. [플러그인? 스킬? MCP?](#10-플러그인-스킬-mcp)
11. [API 토큰 필요 여부](#11-api-토큰-필요-여부)
12. [AI 에이전트 구축에 주는 도움](#12-ai-에이전트-구축에-주는-도움)
13. [React / PHP 로 만들 수 있나](#13-react--php-로-만들-수-있나)
14. [유튜브 강의 제작 가능성](#14-유튜브-강의-제작-가능성)
15. [수익화 아이디어 11가지](#15-수익화-아이디어-11가지)
16. [추천 로드맵과 리스크](#16-추천-로드맵과-리스크)
17. [참고 링크](#17-참고-링크)

---

## 1. 프로젝트 정체 요약

**Buzz** 는 Block, Inc.(스퀘어 / 캐시앱 모회사)가 공개한 **오픈소스 "사람 + AI 에이전트 공동 작업실"** 이다.
슬랙 + 깃허브 + CI + 자동화 봇 + AI 에이전트를 **하나의 서버(릴레이)** 안에 통합한 셀프호스팅 플랫폼.

| 항목 | 값 |
|---|---|
| 라이선스 | **Apache License 2.0** (상업적 사용 가능, 소스 공개 의무 없음) |
| 규모 | 파일 5,577개 / 107MB |
| 백엔드 | Rust 크레이트 **33개**, 약 **48만 줄** |
| 클라이언트 | 데스크톱(Tauri 2 + React 19, v0.5.27), 웹(React 19, v0.1.0), 모바일(Flutter), 관리자 콘솔(React) |
| 기반 프로토콜 | **Nostr** (NIP-01 와이어 포맷) + NIP-42/98 인증 + NIP-29 그룹 + NIP-34 git |
| 인프라 | Postgres(이벤트 + FTS 검색) / Redis(pub-sub) / S3·MinIO(미디어, Blossom) |
| 활발도 | 2026-10-08 기준 당일 커밋 존재 (매우 활발) |

---

## 2. 핵심 아이디어 3가지

### ① 모든 것이 서명된 이벤트
채팅 메시지, 이모지 반응, 워크플로 실행, 코드 리뷰 승인, git 커밋 알림이 모두
**암호학적으로 서명된 동일 형식의 Nostr 이벤트**로 하나의 로그에 쌓인다.

- 이벤트 종류는 `kind` 정수로 구분 (현재 **127종**, `crates/buzz-core/src/kind.rs` 가 소스 오브 트루스)
- 새 기능 추가 = **새 kind 번호 하나 정의**. 모르는 kind 는 기존 클라이언트가 무시 → 하위 호환 유지

| 범위 | 의미 |
|---|---|
| 0–9999 | 표준 Nostr kind |
| 10000–19999 | 대체 가능 이벤트 (NIP-16) |
| 20000–29999 | 휘발성 이벤트 (저장/감사 안 함) |
| 30000–39999 | 파라미터화 대체 가능 이벤트 |
| 40000–49999 | **Buzz 커스텀 kind** |

주요 kind: `7`=반응, `9`=스트림 메시지, `22242`=AUTH, `20001`=프레즌스,
`40002/40003`=메시지 v2/수정, `40100`=캔버스, `43001`=에이전트 작업 요청,
`45001/45003`=포럼 글/댓글, `46001–46012`=워크플로 실행.

### ② 에이전트는 봇이 아니라 멤버
```
기존 슬랙 봇  : 권한 플래그로 제한된 손님
Buzz 에이전트 : 자기 키페어 + 자기 채널 멤버십 + 자기 감사로그
```
에이전트가 할 수 있는 일: 채널 생성, 저장소 열기, 패치 전송, 코드 리뷰, 워크플로 실행,
캔버스 편집, 음성 허들 참여, 다른 에이전트 오케스트레이션, 사람 호출.
**모두 서명되어 감사로그에 기록됨.**

### ③ 릴레이가 유일한 진실의 원천
```
클라이언트 (데스크톱 / 웹 / 모바일 / buzz-cli / AI 에이전트)
        │ WebSocket + REST
        ▼
   buzz-relay  (Rust / Axum)   ← 단일 진실의 원천
        │
  ┌─────┴─────┬──────────┐
Postgres     Redis      S3/MinIO
(이벤트+FTS)  (pub/sub)   (Blossom)
```
`buzz-relay` 가 모든 서브시스템을 직접 호출하지만, 서브시스템끼리는 서로 호출하지 않는다
(`buzz-workflow` 는 `buzz-pubsub` 을 모른다 등). 교차 조정은 릴레이에서만 일어난다.

---

## 3. 폴더 전수조사 결과

| 폴더 / 파일 | 정체 | 설명 |
|---|---|---|
| `crates/` | **Rust 백엔드 33 크레이트** | 심장부 (아래 섹션 4) |
| `desktop/` | 데스크톱 앱 | Tauri 2 + React 19, v0.5.27. Radix UI, Tailwind, TipTap 3, TanStack Query/Router/Virtual, dnd-kit, emoji-mart, MediaPipe |
| `web/` | 웹 클라이언트 | React 19 + Vite, v0.1.0 (초기). `nostr-tools`, `isomorphic-git`(브라우저 git) |
| `admin-web/` | 관리자 콘솔 | React, 릴레이 운영 대시보드 |
| `mobile/` | 모바일 앱 | Flutter (Dart 3.11), hooks_riverpod, web_socket_channel, nostr, mobile_scanner. 🚧 개발 중 |
| `docs/` | 설계 문서 약 40개 | 멀티테넌시, 푸시 억제, git on object storage, 에이전트 신원/발견, 원격 에이전트 등 |
| `docs/nips/` | **커스텀 프로토콜 스펙 20개** | NIP-AA, **NIP-AE**(에이전트 메모리), NIP-AM, NIP-AO, NIP-AP, NIP-AR, NIP-CW, NIP-DV, NIP-ER, NIP-FI, NIP-GS, NIP-IA, NIP-MP, **NIP-OA**(소유자 위임 인증), NIP-PL, NIP-PMA, NIP-RS, NIP-WP |
| `docs/spec/` | **정형 검증(formal spec)** | `MultiTenantRelay.tla`, `GitOnObjectStore.tla` (TLA+), `MultiTenantAuth.spthy` (Tamarin) |
| `examples/countdown-bot/` | 비-AI 봇 예제 | `!countdown 5` → `5 4 3 2 1 🚀`. 표준 WS + NIP-42 경로를 한 파일로 보여줌 |
| `examples/meadow-core/` | **에이전트 페르소나 팩** | Skip(총괄) / Lev(보안 리뷰) / Bana(아키텍처 리뷰) 3인조 |
| `.claude/`, `.agents/`, `.codex/`, `.goose/` | AI 도구용 스킬 폴더 | `sprout-cli`(= buzz-cli 스킬), `desktop-screenshot` |
| `migrations/`, `schema/` | Postgres 마이그레이션/스키마 | |
| `deploy/` | 배포 자산 | Docker Compose 프로덕션 번들 + Helm 차트 |
| `benchmarks/`, `perf/` | 성능 측정 | |
| `.github/workflows/` | CI 파이프라인 20여 개 | 릴레이/데스크톱/클라이언트/보안/릴리즈/도커/헬름 |
| `Justfile` (67KB) | 모든 개발 명령 | `just dev`, `just test`, `just ci` 등 |
| `.env.example` (19KB) | 환경변수 전체 레퍼런스 | "모든 기본값이 그대로 작동" |
| `ARCHITECTURE.md` (56KB) | 시스템 설계 전체 | kind 범위, 서브시스템 경계, 이벤트 파이프라인 11단계 |
| `AGENTS.md` (40KB) | 에이전트/기여 규칙 | `CLAUDE.md`, `GEMINI.md` 가 이 파일을 참조 |
| `VISION*.md` 9개 | 제품 비전 | Sovereign / Projects / Agent / Mesh / Mobile / Moderation / Activity / RemoteAgents |
| `CONTEXT.md` | 도메인 용어집 | 현재는 모바일 푸시 억제 용어만 정의 |

---

## 4. Rust 크레이트 맵

| 크레이트 | 줄 수 | 역할 |
|---|---|---|
| `buzz-relay` | 157,131 | 서버 본체. Axum WS(NIP-01) + REST, 채널/DM/미디어/워크플로/git API, 감사로그 |
| `buzz-db` | 76,039 | Postgres 레이어 (이벤트, 채널, 토큰, 워크플로, 감사) |
| `buzz-acp` | 60,927 | **ACP 하네스** — Goose / Codex / Claude Code 를 Buzz 에 연결 |
| `buzz-agent` | 41,812 | **자체 ACP 코딩 에이전트** (Anthropic / OpenAI 호환 / OpenRouter / Ollama) |
| `buzz-cli` | 26,382 | **에이전트 우선 CLI** — JSON in / JSON out |
| `buzz-test-client` | 19,371 | E2E 테스트 클라이언트 |
| `buzz-auth` | 12,908 | NIP-42/98 Schnorr 인증 + 레이트 리밋 |
| `buzz-sdk` | 12,303 | 타입 안전 이벤트 빌더 |
| `buzz-core` | 10,786 | I/O 없는 순수 타입, NIP-01 필터, Schnorr 검증, kind 레지스트리 |
| `buzz-media` | 9,572 | Blossom / S3 미디어 |
| `buzz-backend-kubernetes` | 6,651 | 쿠버네티스 백엔드 |
| `buzz-push-gateway` | 6,450 | 모바일 푸시 게이트웨이 |
| `buzz-dev-mcp` | 5,374 | **MCP 서버** — shell / str_replace / todo 도구 |
| `buzz-workflow` | 5,263 | YAML 자동화 엔진 |
| `buzz-persona` | 5,197 | 에이전트 페르소나 팩 (`PERSONA_PACK_SPEC.md`) |
| `buzz-voice` | 3,210 | 음성 허들 |
| `buzz-relay-mesh` | 3,153 | 릴레이 간 메시 |
| `buzz-deletion` | 2,970 | 삭제/보존 정책 |
| `buzz-pair-relay` / `buzz-pairing-cli` | 2,621 / 631 | 릴레이 페어링 |
| `git-sign-nostr` / `git-credential-nostr` | 2,511 / 625 | Nostr 서명 git / git 자격증명 헬퍼 |
| `buzz-pubsub` | 2,504 | Redis pub/sub, 프레즌스, 타이핑 |
| `buzz-mesh-smoke` | 2,337 | 메시 스모크 테스트 |
| `buzz-feature-flags` | 2,208 | 피처 플래그 |
| `buzz-admin` | 2,169 | 운영자 CLI |
| `buzz-search` | 1,942 | Postgres FTS 질의 |
| `buzz-conformance` | 1,775 | 적합성 테스트 |
| `buzz-audit` | 1,248 | 해시체인 감사로그 |
| `buzz-ws-client` | 564 | WS 클라이언트 |
| `ifc-core` | 507 | 정보 흐름 제어 (practical-information-flow 문서 참조) |
| `buzz-datastore-tracing` | 417 | 데이터스토어 트레이싱 |
| `sprig` | 54 | 소형 유틸 |

---

## 5. YAML 워크플로 엔진

```yaml
name: "Incident Triage"
trigger:
  on: message_posted
  filter: "str_contains(trigger_text, 'P1')"
steps:
  - id: notify
    action: send_message
    text: "P1 incident detected: {{trigger.text}}"
  - id: page
    if: "str_contains(trigger_text, 'production')"
    action: request_approval
    from: "{{trigger.author}}"
    message: "Page on-call?"
```

- **트리거 4종:** `message_posted`, `reaction_added`, `schedule`(크론), `webhook`
- **액션 7종:** `send_message`, `send_dm`, `set_channel_topic`, `add_reaction`,
  `call_webhook`(SSRF 방어, 리다이렉트 금지, 1 MiB 응답 상한), `request_approval`(기본 24h), `delay`(최대 300초)
- **템플릿 변수:** `{{trigger.text}}`, `{{trigger.author}}`, `{{steps.ID.output.FIELD}}` — 단일 패스 치환(재귀 아님)
- **조건식:** `evalexpr`, 점 표기 → 밑줄 변환(`trigger.text` → `trigger_text`),
  커스텀 함수 `str_contains/str_starts_with/str_ends_with/str_len`, **100ms 타임아웃**
- **동시성:** `Arc<Semaphore>` 100 퍼밋, `try_acquire()` — 초과 시 큐잉 없이 즉시 `CapacityExceeded`
- **루프 방지:** 워크플로 kind(46001–46012), `buzz:workflow` 태그 릴레이 서명 메시지, GIFT_WRAP 은 트리거 제외
- ⚠️ **미완성(WF-08):** `request_approval` 이 `Suspended` 와 토큰을 반환하지만 엔진이 토큰을 영속화/재개하지 않아 승인 게이트에 도달한 런은 실패 처리됨
- ✅ **크론 스케줄러는 완성** (60초 틱, 윈도 기반 매칭)

---

## 6. buzz-cli (에이전트 우선 CLI)

```bash
export BUZZ_PRIVATE_KEY="nsec1..."          # Nostr 개인키 = 에이전트 신원
export BUZZ_RELAY_URL="https://relay.example.com"

buzz messages send --channel <uuid> --content "Hello"
buzz messages send --channel <uuid> --content - < message.md
buzz messages thread --channel <uuid> --event <event-id>
buzz messages search --query "architecture" --author <npub> --since <ts>
buzz messages send-diff --channel <uuid> --diff - --repo <url> --commit <sha> < diff.patch
buzz channels create --name "my-channel" --type stream --visibility open
buzz reactions add --event <event-id> --emoji "👍"
buzz users set-status --text "heads down" --emoji "🚀"
buzz dms open --pubkey <hex>
buzz workflows trigger --workflow <uuid>
buzz workflows approve --token <uuid> --approved false --note "needs revision"
buzz canvas set --channel <uuid> --content "# Welcome"
buzz mem set <slug> "value"
buzz mem patch <slug> --base-hash <hex> < diff.patch       # 낙관적 동시성 제어
buzz repos create --id my-repo --clone <relay>/git/<pubkey>/my-repo
buzz repos protect set --id my-repo --ref refs/heads/main --push admin --no-force-push --no-delete
buzz gifs search --query "celebration"
buzz channels list | jq '.[].name'
```

**출력 계약 (LLM 도구화 핵심):**
- 모든 출력은 stdout 에 JSON, 에러는 stderr 에 JSON
- 종료코드: `0`=성공, `1`=사용자 오류, `2`=네트워크, `3`=인증, `4`=기타, `5`=쓰기 충돌
- 읽기 명령 → JSON 배열 (이벤트 읽기는 `{id, pubkey, kind, content, created_at, tags, sig}` 정규화 완전 서명 이벤트)
- 쓰기 명령 → `{event_id, accepted, message}` (+ 생성 ID: `channel_id`, `dm_id`, `workflow_id`)
- 에이전트 드래프트 → `{request_id, action, saved: false}` (소유자 데스크톱 검토 전까지 "생성됨" 아님)
- `buzz --help` 하나로 전체 명령 트리를 출력 (AI 가 `--help` 를 여러 번 치지 않게)
- `:shortcode:` 패턴을 스캔해 NIP-30 이모지 태그 자동 부착 (데스크톱 컴포저와 동일 동작)

---

## 7. 현재 완성도

| ✅ 작동 | 🚧 작업 중 | 💭 의견만 있고 코드 없음 |
|---|---|---|
| 릴레이, 채널, 스레드, DM, 캔버스, 미디어, 검색, 감사로그 | 모바일 클라이언트(iOS/Android, Flutter) | 릴레이 간 신뢰망 평판 |
| 데스크톱 앱 (Tauri + React) | 워크플로 승인 게이트 (인프라는 있고 글루 미완) | 푸시 알림 |
| `buzz-cli` + ACP 하네스 (Goose, Codex, Claude Code) | 허들 생명주기 이벤트 | 컬처 기능 |
| YAML 워크플로 (메시지/반응/스케줄/웹훅) | | |
| Git 이벤트(NIP-34) + Git 호스팅 백엔드 | | |

추가 유의점: 페르소나 팩의 **런타임 통합은 미구현** (데스크톱 Import 는 `.agent.json` / `.team.json` 스냅샷만 받음).
`buzz pack validate` / `buzz pack inspect` 로 검증·확인만 가능.

---

## 8. 쉬운 비유 설명

| 어려운 말 | 비유 |
|---|---|
| Nostr 이벤트 | 인감도장 찍힌 편지 한 장 |
| 개인키 (nsec) | 내 인감도장 (절대 공개 금지) |
| 공개키 (npub) | 내 신분증 번호 (남이 진위 확인용) |
| 서명 | 도장 찍힌 흔적 (위조 불가) |
| 릴레이 | 우체국 + 창고 (받고, 보관하고, 구독자에게 배달) |
| kind 번호 | 봉투 종류 (9번=채팅, 7번=좋아요, 43001번=AI 업무지시) |
| MCP | USB 규격 (도구 꽂는 구멍) |
| ACP | HDMI 규격 (AI 꽂는 구멍) |
| 스킬 | AI 가 읽는 사용설명서 |
| Buzz | 컴퓨터 본체 전체 |

**봇 vs 에이전트**
```
기존 봇    = 피자 배달원 : 벨 누르고 주고 감. 건물 출입 불가
에이전트   = 신입사원     : 사원증 있고, 들어갈 방 목록 있고, 일한 기록이 남음
```

**README 의 세 가지 이야기**
1. **장애 기억술** — 새벽 2시 "이 에러 전에도 본 적 있나?" → 에이전트가 6개월 히스토리에서 스레드·원인·수정 내역을 올리고 지난번 담당자 호출을 제안. 질문·답변·증거가 모두 채널에 남는다.
2. **브랜치가 곧 방** — 기능 브랜치를 열면 채널이 생긴다. 패치는 NIP-34 이벤트로, CI 결과가 올라오고, 에이전트가 1차 리뷰하고, 머지 결정이 증거와 같은 방에 남는다.
3. **스스로 쓰이는 릴리즈** — 태그에 워크플로가 발동 → 에이전트가 머지된 PR 을 읽고 릴리즈 노트 초안 작성 → 사람 리뷰 게시 → 👍 하나로 배포. 모든 단계가 서명되고 검색된다.

---

## 9. 설치 및 사용법

### 길 1: 패키지 앱만 사용
[릴리즈 페이지](https://github.com/block/buzz/releases/latest)에서 다운로드.

| OS | 파일 |
|---|---|
| macOS (Apple Silicon) | `Buzz_<ver>_aarch64.dmg` |
| macOS (Intel) | `Buzz_<ver>_x64.dmg` |
| Linux | `Buzz_<ver>_amd64.AppImage` / `.deb` |
| Windows | `Buzz_<ver>_x64-setup_alpha-unsigned.exe` |

- 기본 접속 대상은 `ws://localhost:3000` → **릴레이가 없으면 접속 실패**
- 다른 릴레이를 쓰려면 `BUZZ_RELAY_URL` 지정 또는 앱 안에서 릴레이 전환
- Windows 빌드는 코드 서명이 없어 SmartScreen 경고 발생 (More info → Run anyway)

### 길 2: 호스팅 릴레이
README 에 **Railway 원클릭 배포 버튼** 제공.
가이드: https://engineering.block.xyz/blog/run-your-own-buzz-relay

### 길 3: 소스 빌드 (개발자 경로)

필요: **Docker** + **Hermit** (또는 Rust 1.88+, Node 24+, pnpm 10+, `just`)

```bash
# 최초 1회
git clone https://github.com/bmshin94/buzz.git && cd buzz
. ./bin/activate-hermit     # 핀된 툴체인 활성화 (첫 사용 시 자동 다운로드)
just setup                  # .env 생성 + 툴 설치 + Docker 서비스 + 마이그레이션
just build                  # Rust 워크스페이스 빌드

# 매일
. ./bin/activate-hermit
just dev                    # 릴레이 + 데스크톱 앱 동시 실행
```

릴레이: `ws://localhost:3000`

```bash
# 터미널 분리 워크플로
just relay          # 릴레이 로그
just desktop-dev    # Vite 로그

# 자주 쓰는 명령
just check          # fmt + clippy + 데스크톱 검사
just test-unit      # 단위 테스트 (인프라 불필요)
just test           # 전체 (서비스 자동 시작)
just ci             # CI 전체
just reset          # 데이터 삭제 후 재생성 (주의)
```

- 프로덕션/VPS 는 루트 `docker-compose.yml`(개발용) 대신 **`deploy/compose/`** 번들 사용
  (Postgres + Redis + MinIO + 선택적 Caddy/TLS)
- Windows 는 에이전트 셸 도구가 bash 를 요구 → **Git for Windows** 설치 (또는 `BUZZ_SHELL` 지정)

### 에이전트 연결

```bash
export BUZZ_PRIVATE_KEY="nsec1..."
export BUZZ_RELAY_URL="ws://localhost:3000"

# A. 외부 AI 도구 연결 (ACP 하네스)
BUZZ_ACP_AGENT_COMMAND=goose        # 또는 codex-acp, claude-code
BUZZ_ACP_AGENT_ARGS=acp
BUZZ_ACP_AGENTS=4                   # 병렬 에이전트 수 (1~32)
BUZZ_ACP_SUBSCRIBE=mentions         # mentions(기본) / all / config
BUZZ_ACP_TURN_TIMEOUT=320           # 턴당 상한(초)
BUZZ_ACP_MAX_TURNS_PER_SESSION=50   # 선제적 세션 로테이션
BUZZ_ACP_HEARTBEAT_INTERVAL=60      # 유휴 세션 타임아웃 방지
buzz-acp

# B. 자체 에이전트 + MCP 도구함
BUZZ_AGENT_PROVIDER=anthropic
ANTHROPIC_API_KEY=sk-ant-...
ANTHROPIC_MODEL=claude-sonnet-4-5
./target/release/buzz-agent
```

---

## 10. 플러그인? 스킬? MCP?

**결론: 셋 다 아니고 "플랫폼/제품"이다.** 다만 그 안에 전부 들어 있다.

| 구분 | 포함 여부 | 설명 |
|---|---|---|
| 플러그인 | ⚠️ 부분 | `examples/meadow-core/.plugin/plugin.json` 은 **Buzz 페르소나 팩 매니페스트**(OPS 호환). Claude Code 플러그인이 아니며 런타임 통합 미구현 |
| 스킬 | ✅ 포함 | `.claude/skills/sprout-cli/SKILL.md`, `desktop-screenshot/SKILL.md`. Buzz 를 **쓰는** AI 를 위한 스킬 |
| MCP | ✅ 포함 | `buzz-dev-mcp` = MCP **서버**(shell, str_replace, todo). `buzz-agent` = MCP **클라이언트** |
| ACP | ✅ 핵심 | `buzz-acp` 가 Goose/Codex/Claude Code 를 연결. `buzz-agent` 는 ACP 에이전트라 Zed/JetBrains 에도 연결됨 |

참고: `.claude/skills/sprout-cli/SKILL.md` 는 환경변수 처리, 출력 계약, 에러 코드, 대화형 플로우까지
담긴 **잘 만든 스킬의 참고 샘플**로 가치가 크다.

---

## 11. API 토큰 필요 여부

### 필수 ①: Nostr 개인키 (API 토큰이 아님)
```bash
BUZZ_PRIVATE_KEY=nsec1...     # 또는 32바이트 hex
```
- 서버가 발급하는 토큰이 아니라 **내가 생성해 보유하는 키페어**
- 인증: **NIP-42**(WebSocket), **NIP-98**(HTTP) — 요청마다 Schnorr 서명
- 스킬 문서 지침: **값을 읽거나 출력하지 말 것**
- `BUZZ_ADMIN_TOKEN` 은 **폐기됨** (시작 시 경고와 함께 무시, 환경에서 제거 권장)

### 필수 ②(조건부): LLM 제공자 키 — 에이전트를 돌릴 때만

| 제공자 | 환경변수 |
|---|---|
| Anthropic | `ANTHROPIC_API_KEY`, `ANTHROPIC_MODEL` |
| OpenAI 호환 (OpenAI, vLLM, llama.cpp, Databricks, Block Gateway, Ollama) | `OPENAI_COMPAT_API_KEY`, `OPENAI_COMPAT_BASE_URL`, `OPENAI_COMPAT_MODEL` |
| OpenRouter | `OPENROUTER_API_KEY` |

```bash
# 로컬 모델(Ollama) → API 키 없이 비용 0원
BUZZ_AGENT_PROVIDER=openai
OPENAI_COMPAT_BASE_URL=http://localhost:11434/v1
OPENAI_COMPAT_MODEL=qwen2.5-coder:7b
```

### 선택: 기능별 추가 키

| 환경변수 | 용도 | 없을 때 |
|---|---|---|
| `BUZZ_KLIPY_API_KEY` | GIF 검색 | GIF 기능만 비활성 |
| `TYPESENSE_API_KEY` | 외부 검색 엔진 | 기본은 Postgres FTS |
| S3/MinIO 자격증명 | 미디어 저장 | Docker 로컬 MinIO 사용 |
| `DATABRICKS_MODEL_FILTER` | 모델 목록 가시성 필터 (추론 권한 아님) | 전체 표시 |

**정리:** 로컬에서 Buzz 만 돌리기 → Nostr 키만(비용 0). AI 까지 → LLM 키 필요하지만 Ollama 면 0원.
팀/서비스 운영 → Nostr 키 + LLM 키 + S3 + 선택 기능 키.

---

## 12. AI 에이전트 구축에 주는 도움

바로 가져다 쓸 수 있는 설계 패턴 7가지.

1. **에이전트 우선 CLI 설계** — JSON in/out, 종료코드 규격화, 전체 명령 트리 단일 출력 (섹션 6)
2. **에이전트 신원 모델** — 권한 플래그 대신 키페어 + 채널 멤버십. `NIP-OA` 로 봇 키를 영구 멤버로 만들지 않고 소유자 위임으로 접속 허용
3. **컨텍스트 관리** — 컨텍스트가 차면 세션이 자기 히스토리를 요약하고 계속. 턴 타임아웃 / 세션 로테이션 / 하트비트
4. **하드닝 원칙** (`VISION_AGENT.md`) — unsafe 0, panic 0, 프로세스 수명·출력 크기·히스토리 제한,
   모든 종료 경로에서 프로세스 그룹 kill, 파일 편집은 작업 디렉토리 기준 해석, 모든 취소 경로에서 히스토리 유효성 유지.
   *"삭제할 수 있으면 삭제하라. 남긴다면 성능·안전성·명료성으로 월세를 내야 한다."*
5. **프로토콜 분리로 조합성 확보** — 에이전트는 어떤 MCP 서버인지 모르고, MCP 서버는 어떤 에이전트인지 모른다.
   import 가 아니라 프로토콜로 조합 → 환경변수 하나로 LLM 교체, 에이전트 10개 병렬 각자 다른 MCP 설정
6. **멀티 에이전트 오케스트레이션** — 페르소나 = YAML 프론트매터(설정) + 마크다운 본문(시스템 프롬프트).
   `PERSONA_PACK_SPEC.md` 에 `triggers: {}` 가 "부재"가 아니라 "전면 덮어쓰기"라는 엣지케이스까지 문서화
7. **에이전트 장기 메모리 (NIP-AE)** — `buzz mem set/get/patch/rm`, `--base-hash` 로 충돌 감지 후 종료코드 5

**이미 만들어져 있어 다시 안 만들어도 되는 것:** 에이전트 간 메시징(채널/DM/스레드),
승인 게이트(`request_approval`), 해시체인 감사로그(`buzz-audit`), 작업 큐(`KIND_JOB_REQUEST` 43001),
에이전트 발견(`docs/owned-agent-discovery.md`), 셸/파일 도구(`buzz-dev-mcp`), git 통합(NIP-34 + `git-sign-nostr`).

**단점:** Rust 48만 줄로 처음엔 압도적(→ `buzz-agent` + `buzz-dev-mcp` 약 4.7만 줄만 먼저 읽기),
일부 기능 미완성(WF-08, 모바일, 푸시), Block 내부 의존 흔적(Databricks/Block Gateway), 페르소나 팩 런타임 미구현.

**추천 공략 루트 (약 1시간 40분):**
`VISION_AGENT.md` → `crates/buzz-agent/README.md` → `.claude/skills/sprout-cli/SKILL.md`
→ `crates/buzz-cli/README.md` → `crates/buzz-persona/PERSONA_PACK_SPEC.md`

---

## 13. React / PHP 로 만들 수 있나

### React — 가능. 이미 React 로 만들어져 있음
```
desktop/     React 19 + Tauri 2    (메인 앱)
web/         React 19 + Vite       (브라우저 버전, v0.1.0)
admin-web/   React                 (관리자 콘솔)
```
스택: Radix UI, Tailwind, TanStack Query/Router/Virtual, TipTap 3, `nostr-tools`,
`isomorphic-git` + `lightning-fs`, dnd-kit, emoji-mart, jdenticon, sonner, MediaPipe.

할 수 있는 선택지
1. `web/` 포크해서 나만의 웹 클라이언트 (v0.1.0 이라 기여 여지 큼)
2. 릴레이는 그대로 두고 React UI 전면 재디자인
3. Next.js 로 새로 작성 — `nostr-tools` 로 릴레이에 WS 연결

릴레이가 프로토콜(NIP-01 WebSocket + REST)만 노출하므로 **클라이언트는 언어/프레임워크 자유**.

### PHP — 클라이언트는 쉬움, 릴레이 재구현은 비권장

쉬운 경로(난이도 ⭐⭐)
- 워크플로 `call_webhook` 을 PHP 엔드포인트로 수신 → 라라벨/워드프레스/사내 시스템 연동
- `buzz-cli` 를 PHP 에서 셸 호출하고 JSON 파싱 (가장 빠른 프로토타입)
- PHP 로 Nostr 봇 직접 구현 (secp256k1 Schnorr + WebSocket; `swentel/nostr-php`, `ratchet/pawl`)
  — `countdown-bot` 이 이 패턴의 참고 구현

어려운 경로(난이도 ⭐⭐⭐⭐⭐, 비권장)
- `buzz-relay` + `buzz-db` 합계 약 23만 줄 재구현
- PHP 는 상주 WebSocket 에 Swoole/ReactPHP/Ratchet 필요
- Schnorr 검증이 핫패스 → 고성능 C 확장 필요, Rust 대비 성능 열세
- Redis pub/sub + 수천 WS 연결 유지

**현실적 추천 순위**
| 순위 | 구성 | 난이도 |
|---|---|---|
| 1 | 릴레이 = Rust(Docker 그대로) + UI = React/Next.js 직접 제작 | ⭐⭐ |
| 2 | 릴레이 그대로 + PHP 봇/웹훅 수신기 | ⭐⭐ |
| 3 | 릴레이 그대로 + `buzz-cli` 셸 호출 | ⭐ |
| 4 | `web/` 에 OSS 기여 | ⭐⭐⭐ |
| 5 | Rust 크레이트 수정/기능 추가 | ⭐⭐⭐⭐ |
| 비권장 | PHP 로 릴레이 재구현 | ⭐⭐⭐⭐⭐ |

---

## 14. 유튜브 강의 제작 가능성

**가능하고, 타이밍이 좋다.** 한국어 콘텐츠가 사실상 없고, Block 공식 OSS 라는 신뢰도가 있으며,
AI 에이전트 / MCP / 멀티에이전트 / 셀프호스팅 / Rust 키워드가 모두 상승세다.
Apache 2.0 이라 코드 설명·시연도 합법이고, 앱 UI 와 `docs/assets/screenshots/` 로 비주얼 확보가 쉽다.

### 커리큘럼 안

**시즌 1 — 입문 (10~15분)**
1. AI 를 직원으로 고용하는 오픈소스 (Block 의 Buzz)
2. Buzz 설치 10분 (Docker + just)
3. Nostr 5분 이해 — 블록체인 아닌데 서명되는 메시지
4. 채널/스레드/DM/캔버스 투어
5. AI 에이전트 입사시키기 (Claude Code 연결, `buzz-acp`)

**시즌 2 — 실전 (15~25분)**
6. YAML 10줄 자동화 봇
7. AI 없는 봇 해부 (`countdown-bot`)
8. AI 3인조 팀 구성 (`meadow-core`)
9. 에이전트에게 기억 주기 (NIP-AE)
10. Railway 릴레이 5분 배포

**시즌 3 — 심화 (20~40분)**
11. MCP 서버 직접 만들기 (`buzz-dev-mcp` 코드 리딩)
12. ACP vs MCP 차이
13. LLM 이 쓰기 좋은 CLI 설계법 (`buzz-cli`)
14. 15만 줄 Rust 릴레이 아키텍처 해부
15. TLA+ 로 분산시스템 설계 증명 (`docs/spec/`)
16. React 19 + Tauri 2 데스크톱 앱
17. 커스텀 React 클라이언트 처음부터 만들기

**시즌 4 — 유료 전환**
18~20. AI 에이전트 플랫폼 처음부터 만들기 (설계 패턴 응용)

### 반드시 지킬 것
- **상표:** Apache 2.0 6조는 상표/서비스마크/제품명 사용 허가를 부여하지 않는다.
  "Buzz 공식 강의", "Buzz 한국" 같은 공식 오인 명칭 금지. "Buzz 분석 강의 (비공식)" 형태로.
- **저작권 고지:** 코드 배포/포크 시 LICENSE 와 NOTICE 유지
- **소속 명시:** Block, Inc. 의 오픈소스임을 밝히기
- **미완성 솔직히:** 🚧/💭 영역은 "작업 중"으로 명시
- **키 노출 금지:** `BUZZ_PRIVATE_KEY`, `ANTHROPIC_API_KEY` 화면 노출 금지

---

## 15. 수익화 아이디어 11가지

### 라이선스 전제 (Apache 2.0)

| 가능 | 불가 |
|---|---|
| 상업적 사용 / 수정 / 배포 / 판매 | **상표 사용** (공식 오인 명칭·로고) — 6조 |
| **비공개 소스로 재배포** (소스 공개 의무 없음) | 저작권·라이선스 고지 제거 |
| 특허권 명시적 부여 | 보증 주장 |

필수 3가지: ① LICENSE 사본 포함 ② NOTICE 유지 ③ 수정한 파일에 변경 사실 표시.
→ **포크 + 리브랜딩 + 클로즈드소스 상용 판매도 합법** (GPL/AGPL 과 달라 수익화에 유리). 단 이름은 변경 필요.

### 티어 1 — 즉시 시작 (초기비용 ~0원)

| # | 아이디어 | 타겟 | 수익모델 / 가격 | 난이도 | 예상 |
|---|---|---|---|---|---|
| 1 | **유튜브 + 블로그** | AI 관심 개발자, CTO, 1인 개발자 | 애드센스 + 멤버십 + 제휴 + 협찬 | ⭐⭐ | 월 10~200만원 |
| 2 | **유료 온라인 강의** (인프런/유데미) | 현업 개발자 | 5~15만원, 번들 20만원 | ⭐⭐⭐ | 100명×8만원 = 800만원 |
| 3 | **한국어 문서화/번역** | 한국 개발자 커뮤니티 | 직접수익 없음, 유입·신뢰 자산 | ⭐ | 간접(컨설팅 연결) |
| 4 | **유료 뉴스레터/리포트** | CTO/PM/투자자 | 월 9,900~29,000원 | ⭐⭐ | 300명×1만원 = 월 300만원 |

- 아이디어 2 는 "Buzz 사용법"보다 **"Buzz 로 배우는 설계 원칙"** 이 수명이 길고 단가가 높다.
  (4부 "LLM 이 쓰기 좋은 CLI/API 설계"가 핵심 셀링 포인트)
- 아이디어 3 번역 우선순위: `README.md` → `VISION_AGENT.md` → `crates/buzz-cli/README.md` → `ARCHITECTURE.md`

### 티어 2 — 중기 (3~6개월)

| # | 아이디어 | 타겟 | 가격 | 난이도 | 예상 |
|---|---|---|---|---|---|
| 5 | **셋업/구축 컨설팅** ⭐추천 | 금융·의료·공공·국방·로펌·제조 | PoC 500~2,000만 / 구축 2,000만~1억 / 유지보수 월 100~500만 | ⭐⭐⭐ | 연 5,000만~2억 |
| 6 | **한국형 매니지드 호스팅** | 운영 인력 없는 중소기업/스타트업 | 사용자당 월 5,000~20,000원 또는 인스턴스당 월 10~50만원 | ⭐⭐⭐⭐ | 10팀×30만 = 월 300만 MRR |
| 7 | **템플릿 마켓플레이스** | Buzz 사용 팀 | 개당 1~10만원 / 번들 30만원 / 구독 월 2만원 | ⭐⭐ | 월 50~300만원 |
| 8 | **커스텀 클라이언트 개발** | 자체 브랜딩 원하는 기업 | 1,000만~5,000만원 + 유지보수 월 100만원 | ⭐⭐⭐ | 건당 수천만원 |

**아이디어 5 가 돈이 되는 이유:** 금융/의료/공공은 슬랙·노션·ChatGPT 에 데이터를 올릴 수 없다.
Buzz 는 완전 셀프호스팅 + 해시체인 감사로그 + 서명된 모든 행위(부인 방지) + 로컬 LLM(Ollama) +
TLA+ 로 검증된 멀티테넌시를 제공한다.

서비스 패키지: 진단 300만 / PoC 800만 / 구축 3,000만~1억 / 운영 월 200만 / 맞춤 에이전트 건당 500만.

**아이디어 6 경쟁우위:** 한국 리전(네이버클라우드·NHN·KT → 공공 요건), 한국어 지원·UI,
카카오워크/잔디/슬랙 마이그레이션, 전자금융감독규정·개인정보보호법 대응, 세금계산서 발행.
기술 준비물은 이미 있음 — `deploy/` Helm 차트 + 프로덕션 Compose, `buzz-backend-kubernetes`(6,651줄).
리스크는 Block 공식 호스팅 출시 → 한국 특화 + 컨설팅 결합으로 방어.

**아이디어 7 판매 품목:**
- 페르소나 팩: 한국어 코드리뷰(네이버/카카오 컨벤션), PM(지라/노션), 보안 감사(ISMS-P), 기술문서 작성, 신입 온보딩 멘토
- 워크플로 YAML 팩: 배포 승인 체인, 장애 대응(P1/P2/P3 분류 + 온콜), 일일 스탠드업 수집, 릴리즈 노트 생성, 취약점 알림 DM
- 장점: 한 번 만들면 계속 팔리고 유지비가 거의 없으며 YAML/마크다운이라 제작 난이도가 낮다

### 티어 3 — 장기 (6개월~)

| # | 아이디어 | 내용 | 난이도 |
|---|---|---|---|
| 9 | **버티컬 특화 제품** (포크 → 리브랜딩) | 의료 / 로펌 / 제조 / 금융 / 공공 / 교육 중 하나 선택. 라이선스 연 2,000만~2억 + 구축 + 유지보수 | ⭐⭐⭐⭐⭐ |
| 10 | **기업 교육 / 사내 워크숍** | 1일 300~500만 / 3일 1,000만 / 강사비 시간당 20~50만. 월 1건이면 월 300~500만 | ⭐⭐⭐ |
| 11 | **SI / 정부과제** | NIPA AI 바우처, 스마트공장, 디지털정부, 오픈소스 국산화. 과제당 5,000만~5억 | ⭐⭐⭐⭐ |

버티컬 후보 시장성: 금융(FinSecure)·의료(MedCollab) ⭐⭐⭐⭐⭐ > 로펌·제조·공공 ⭐⭐⭐⭐ > 교육 ⭐⭐⭐.
전략은 아이디어 5(컨설팅)로 도메인 지식을 쌓은 뒤 제품화.

### 우선순위 매트릭스

| 아이디어 | 난이도 | 초기비용 | 수익규모 | 속도 | 평가 |
|---|---|---|---|---|---|
| 템플릿 팩 판매 | ⭐⭐ | 0 | 중 | 빠름 | 최고 가성비 |
| 유튜브/블로그 | ⭐⭐ | 0 | 중 | 중간 | 모든 것의 입구 |
| 컨설팅 | ⭐⭐⭐ | 0 | 대 | 중간 | 최고 수익 |
| 유료 강의 | ⭐⭐⭐ | 소 | 중대 | 중간 | 자산형 |
| 기업 교육 | ⭐⭐⭐ | 0 | 중대 | 중간 | 컨설팅 시너지 |
| 커스텀 클라이언트 | ⭐⭐⭐ | 0 | 중대 | 느림 | React 강점 활용 |
| 뉴스레터 | ⭐⭐ | 0 | 소중 | 느림 | 꾸준함 필요 |
| 호스팅 SaaS | ⭐⭐⭐⭐ | 대 | 대 | 느림 | 반복수익이나 무거움 |
| 버티컬 제품 | ⭐⭐⭐⭐⭐ | 대 | 특대 | 매우 느림 | 최종 목표 |

---

## 16. 추천 로드맵과 리스크

### 로드맵

| 기간 | 할 일 | 목표 |
|---|---|---|
| 0~1개월 | 유튜브 입문 5편 + 한국어 README 번역 | 비용 0, 키워드 선점 |
| 1~3개월 | 템플릿 팩 제작·판매, 뉴스레터 시작 | 첫 수익 발생 |
| 3~6개월 | 유료 강의 출시, 컨설팅 문의 수신 | 월 수백만원 |
| 6~12개월 | 첫 기업 구축, 기업 교육 론칭, 호스팅 베타 | 월 1,000만원 목표 |
| 12개월+ | 버티컬 제품화, 정부과제/SI | 사업화 |

### 리스크 체크리스트

| 리스크 | 대응 |
|---|---|
| Block 이 공식 호스팅/유료 상품 출시 | 한국 특화 + 컨설팅/교육(사람이 하는 일)에 집중 |
| Buzz 가 빠르게 변함 (매일 커밋) | "사용법"보다 "설계 원칙" 콘텐츠로 수명 확보 |
| Buzz 가 인기를 얻지 못할 경우 | 설계 패턴 지식은 남음 → "AI 에이전트 아키텍처" 일반 주제로 전환 |
| 상표/라이선스 문제 | 공식 오인 명칭 금지, LICENSE/NOTICE 유지, "비공식" 명시 |
| 미완성 기능으로 인한 고객 불만 | 제안서에 🚧/💭 기능 명확히 분리 기재 |
| Rust 역량 부족 | 릴레이는 Docker 로 그대로 쓰고 React/CLI 레이어에 집중 |

### 최종 정리
> 콘텐츠로 입구를 만들고, 컨설팅으로 수익을 내고, 템플릿으로 자동 수익을 만들고,
> 최종적으로 버티컬 제품으로 사업화한다.
> Buzz 의 성패와 무관하게 **AI 에이전트 플랫폼 설계 지식 자체가 남는다.**

---

## 17. 참고 링크

### 저장소
- **이 저장소:** https://github.com/bmshin94/buzz
- **원본(업스트림):** https://github.com/block/buzz
- 릴리즈: https://github.com/block/buzz/releases/latest
- Block 내부 빌드(Block 직원용): https://github.com/squareup/buzz-releases/releases/latest
- 릴레이 운영 가이드: https://engineering.block.xyz/blog/run-your-own-buzz-relay

### 외부 프로토콜/도구
- Nostr: https://github.com/nostr-protocol/nips
- ACP (Agent Client Protocol): https://agentclientprotocol.com
- MCP (Model Context Protocol): https://modelcontextprotocol.io
- Hermit: https://cashapp.github.io/hermit/
- Docker: https://docs.docker.com/get-docker/
- Git for Windows: https://git-scm.com/download/win

### 저장소 내부 필수 문서
| 파일 | 내용 |
|---|---|
| `README.md` | 전체 개요, 빠른 시작, 완성도 표 |
| `ARCHITECTURE.md` (56KB) | 시스템 설계, kind 범위, 이벤트 파이프라인, 서브시스템 경계 |
| `VISION_AGENT.md` | buzz-agent / buzz-dev-mcp 설계 철학 (필독) |
| `VISION.md`, `VISION_SOVEREIGN.md`, `VISION_PROJECTS.md`, `VISION_MESH.md`, `VISION_MOBILE.md`, `VISION_MODERATION.md`, `VISION_ACTIVITY.md`, `VISION_REMOTE_AGENTS.md` | 제품 비전 |
| `AGENTS.md` (40KB) | 에이전트/기여 규칙 (`CLAUDE.md`, `GEMINI.md` 가 참조) |
| `crates/buzz-cli/README.md` | CLI 전체 명령 + 인증 + 출력 계약 |
| `crates/buzz-agent/README.md` | ACP 에이전트 구조 + 제공자 설정 |
| `crates/buzz-persona/PERSONA_PACK_SPEC.md` | 페르소나 팩 포맷 전체 스펙 |
| `.claude/skills/sprout-cli/SKILL.md` | 잘 만든 스킬 작성 참고 샘플 |
| `docs/nips/` | 커스텀 프로토콜 20종 (NIP-AE 메모리, NIP-OA 위임 인증 등) |
| `docs/spec/` | TLA+ / Tamarin 정형 명세 |
| `examples/countdown-bot/README.md` | 비-AI 봇 구현 + 인증 경로 2가지 |
| `examples/meadow-core/README.md` | 3인조 페르소나 팩 |
| `TESTING.md`, `CONTRIBUTING.md`, `RELEASING.md`, `SECURITY.md`, `GOVERNANCE.md`, `CODE_OF_CONDUCT.md` | 테스트/기여/릴리즈/보안/거버넌스 |
| `Justfile` (67KB) | 전체 개발 명령 |
| `.env.example` (19KB) | 환경변수 레퍼런스 |

---

*이 문서는 저장소 전수조사 결과를 한국어로 정리한 비공식 분석 자료입니다.*
*Buzz 는 Block, Inc. 의 Apache 2.0 오픈소스 프로젝트이며, 이 문서는 공식 문서가 아닙니다.*
