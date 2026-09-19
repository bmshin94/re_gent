# re_gent 전수조사 & 활용 전략 정리 (한국어)

> 이 문서는 re_gent 레포지토리를 코드 레벨까지 전수조사한 결과와,
> 설치/사용법·정체성·수익화 아이디어에 대한 논의를 정리한 기록입니다.
>
> 작성일: 2026-09-19

---

## 🔗 관련 링크

| 구분 | 주소 |
|---|---|
| **이 저장소 (포크)** | https://github.com/bmshin94/re_gent |
| **업스트림 원본** | https://github.com/regent-vcs/regent |
| VSCode 확장 | https://github.com/regent-vcs/vscode-regent |
| Homebrew Tap | `brew tap regent-vcs/tap` |
| Discord 커뮤니티 | https://discord.gg/5k2Q8AmqC |
| 릴리즈 | https://github.com/regent-vcs/regent/releases |
| 기여 가이드 | [CONTRIBUTING.md](../CONTRIBUTING.md) |
| 로드맵 | [ROADMAP.md](../ROADMAP.md) |
| 기술 스펙 | [POC.md](../POC.md) |

---

## 1. re_gent란 무엇인가

**한 줄 정의: "AI 에이전트 전용 git".** Go로 작성된 CLI 도구이며 실행 명령어는 `rgt` 입니다.

Git은 *파일이 어떻게 바뀌었는지*는 기록하지만, *어떤 프롬프트가 그 코드를 만들었는지*는 기록하지 못합니다.
re_gent는 에이전트의 매 턴을 자동으로 포착해 다음 세 가지를 가능하게 합니다.

- `rgt log` — 이 세션이 실제로 무엇을 했는가
- `rgt blame` — 이 줄은 어떤 프롬프트가 만들었는가
- `rgt show` — 그 스텝의 툴 호출과 대화 전문 확인

README의 핵심 문장:
> "We gave agents write access to our codebases. We did not give ourselves git for it."

---

## 2. 핵심 데이터 모델

`internal/store/step.go` 기준 실제 구조체:

```go
type Step struct {
    Parent          Hash      // 이전 스텝 (DAG)
    SecondaryParent Hash      // 서브에이전트 머지용 (미구현)
    Tree            Hash      // 워크스페이스 스냅샷
    Transcript      Hash      // 대화 기록
    Causes          []Cause   // 이 스텝을 유발한 툴 호출들
    SessionID       string
    Origin          string    // claude_code / codex_cli / opencode / pi
    TurnID          string
    TimestampNanos  int64
    Effects         []Effect  // 되돌릴 수 없는 부작용 기록
}
```

| 객체 | 설명 | git 대응 |
|---|---|---|
| **Blob** | 원본 바이트. `blake3(content)`로 주소화, 동일 내용은 1회만 저장 | blob |
| **Tree** | `경로 → (blob_hash, blame_hash, mode)` 맵 | tree |
| **Step** | 에이전트 한 턴 = 스냅샷 + 툴 호출 + 대화 참조 | **commit** |
| **Ref** | 세션별 포인터 (유일한 가변 객체, CAS로 갱신) | branch |
| **Cause** | 툴 호출 1건 `{tool_use_id, tool_name, args_blob, result_blob}` | - |
| **BlameMap** | 줄 단위 출처 배열 (쓰기 시점 계산 → blame 조회 O(1)) | - |
| **Effect** | HTTP 호출 등 되돌릴 수 없는 부작용 로그 | - |

저장 위치:

```
.regent/
├── objects/     # 내용주소 블롭 (BLAKE3)
├── refs/        # 세션 포인터
├── index.db     # SQLite 쿼리 인덱스
└── config.toml
```

---

## 3. 폴더 구조 전수 분석

```
cmd/rgt/              실행 진입점 + 에이전트별 훅 어댑터
  main.go             Cobra 명령어 등록
  message_hook.go     Claude: 유저 프롬프트 / 어시스턴트 응답
  tool_batch_hook.go  Claude: 툴 실행 후 스냅샷 + Step 생성
  codex_hook.go       OpenAI Codex CLI 어댑터
  opencode_hook.go    OpenCode 어댑터
  pi_hook.go          Pi 어댑터

internal/
  store/              오브젝트 스토어 (blob, tree, step, refs, blame, workspace, hash)
  index/              SQLite 인덱스 (steps, step_files, sessions, messages,
                      tool_uses, session_transcript, jsonl_snapshots)
  capture/            공용 캡처 엔진 — 모든 어댑터가 여기로 수렴
  cli/                log, blame, show, sessions, status, cat, init, graph_renderer
  snapshot/           워크스페이스 스냅샷
  treediff/ diff/     트리 비교 / Myers diff
  ignore/             .regentignore, .gitignore 파싱
  jsonl/              Claude 트랜스크립트 JSONL 파싱
  conversation/       호스트 포맷 → 내부 메시지 정규화
  style/              터미널 출력 스타일

test/                 인수/통합/세션분기 테스트
examples/bad-refactor/ 실전 데모 (청구 로직 회귀 버그 추적)
demo/                 훅 설정 및 스킬 예시
.agents/skills/       log, blame, show 스킬 래퍼
.claude/skills/rewind/ rewind 스킬 (CLI는 아직 미구현)
.github/              CI, 릴리즈, 이슈 템플릿, good first issues
scripts/, .goreleaser.yaml  멀티플랫폼 릴리즈 자동화
```

---

## 4. 동작 흐름 (Claude Code 기준)

`rgt init` 이 `.claude/settings.json` 에 아래 훅을 자동 주입합니다.

```json
{"hooks": {
  "UserPromptSubmit": [{"hooks":[{"command":"rgt message-hook user"}]}],
  "PostToolBatch":    [{"hooks":[{"command":"rgt tool-batch-hook"}]}],
  "Stop":             [{"hooks":[{"command":"rgt message-hook assistant"}]}]
}}
```

```
프롬프트 입력
  → UserPromptSubmit 훅: 프롬프트를 SQLite messages에 기록
툴 실행 (Edit / Write / Bash …)
  → PostToolBatch 훅: 툴 입출력을 blob으로 저장
                      워크스페이스 스냅샷 → Tree 생성
                      BlameMap 계산
                      Step 작성 → 세션 ref를 CAS로 전진
응답 종료
  → Stop 훅: 어시스턴트 메시지 기록
```

설계 원칙: **"절대 사용자의 에이전트를 방해하지 않는다."**
`.regent/`가 없으면 훅은 조용히 종료하고, 부가 작업이 실패해도 `.regent/log/`에만 남기고 진행합니다.

---

## 5. 검증한 사실 (직접 확인)

| 항목 | 결과 |
|---|---|
| 빌드 | ✅ `go build ./cmd/rgt` 성공 (약 16MB 단일 바이너리) |
| 코드 규모 | 75개 Go 파일, 약 17,400줄 (테스트 포함) |
| 외부 네트워크 호출 | ❌ 없음 (http.Get/Post, api_key 등 grep 결과 0건) |
| CGO | 불필요 (`modernc.org/sqlite`, 순수 Go) |
| 버전 | v1.0.0 (CHANGELOG 기준 2026-05-14) |
| 구현된 명령 | init, log, status, blame, show, sessions, cat, version, completion |
| 미구현 | `rgt rewind`, `rgt fork`, `rgt reindex` (ROADMAP Phase 3) |
| 이 레포의 성격 | `regent-vcs/regent`의 **포크**. `go.mod` 모듈명은 원본 그대로 |

기술 스택: Go 1.23+, BLAKE3(`lukechampine.com/blake3`), Cobra, go-diff,
go-gitignore, modernc.org/sqlite, TOML, charmbracelet/huh(대화형 UI)

---

## 6. 설치 및 사용법

### 설치

```bash
# 1) Homebrew (가장 쉬움)
brew tap regent-vcs/tap
brew install regent

# 2) Go install (업스트림 기준)
go install github.com/regent-vcs/regent/cmd/rgt@latest

# 3) 소스 빌드 — 이 포크를 쓰려면 이 방법
git clone https://github.com/bmshin94/re_gent
cd re_gent
go build -o rgt ./cmd/rgt
sudo mv rgt /usr/local/bin/
```

> ⚠️ 이 포크는 `go.mod` 모듈 경로가 아직 `github.com/regent-vcs/regent` 이므로
> `go install`로는 포크 버전을 받을 수 없습니다. 소스 빌드를 사용하세요.

### 사용

```bash
cd my-project
rgt init            # .regent/ 생성 + 훅 자동 설정 (대화형 선택)
                    # 1) 저장소 초기화 2) 훅 설정 3) 스킬 설치

# 이후 평소처럼 에이전트로 작업하면 자동 기록됨

rgt log                      # 활동 기록
rgt log --graph              # DAG 그래프
rgt log --conversation-only  # 대화만
rgt log --files-only         # 파일 변경만
rgt log --session <id>       # 세션 필터
rgt log --json               # JSON 출력 (외부 연동용)

rgt blame src/app.py         # 파일 전체 줄별 출처
rgt blame src/app.py:42      # 특정 줄
rgt show <step-hash>         # 툴 호출 + 대화 전문
rgt sessions                 # 세션 목록
rgt status                   # 현재 상태
rgt cat <hash>               # 원시 객체 덤프
```

`rgt init --agent claude|codex|opencode|pi|both|all` 로 대상 지정 가능.

---

## 7. 플러그인? 스킬? MCP? — 정답: 훅 기반 독립 CLI

| 구분 | 해당 여부 | 설명 |
|---|---|---|
| 플러그인 | ❌ | 플러그인 패키지 형태가 아님 |
| MCP 서버 | ❌ | MCP 프로토콜 코드 없음 |
| 스킬 | 🔶 부분 | `.agents/skills/`에 log/blame/show 래퍼 존재 |
| **CLI 도구** | ✅ | **본체는 Go 단일 바이너리 `rgt`** |

**왜 MCP가 아니라 훅인가**

| | MCP | 훅 (re_gent) |
|---|---|---|
| 실행 주체 | 에이전트의 판단 | 호스트가 무조건 실행 |
| 기록 누락 | 가능 | 없음 |
| 컨텍스트 토큰 | 소모함 | 0 |
| 에이전트 인지 | 알고 있음 | 모름(투명) |

감사(audit)가 목적이므로 "에이전트가 모르게, 빠짐없이"가 올바른 선택입니다.

---

## 8. API 토큰 필요 여부 — 불필요

- 코드 전체 grep 결과 외부 HTTP 호출·API 키 사용 **0건**
- 100% 로컬 동작, 서버 전송·텔레메트리·계정 없음
- Apache 2.0 오픈소스 (상업적 이용·재배포 가능)
- 토큰이 쓰이는 곳은 메인테이너용 릴리즈 파이프라인(Homebrew tap 푸시)뿐

### ⚠️ 프라이버시 주의

`.regent/`에는 **대화 전문과 툴 입출력이 평문으로** 저장됩니다.
프롬프트에 자격증명을 붙여넣었거나 에이전트가 `.env`를 읽었다면 그 내용도 남습니다.
**`.regent/`는 절대 git에 커밋하지 마세요.** (`rgt init`이 `.regent/.gitignore`를 자동 생성)

---

## 9. 왜 GitHub에서 주목받는가 (레포 내용 기반 추론)

1. **문제 공감대** — "5분 전엔 됐는데"는 설명이 필요 없는 보편적 고통
2. **타이밍** — 에이전트 코딩은 폭증했는데 관측/감사 도구 카테고리는 비어 있었음
3. **포지셔닝** — git을 대체하지 않고 *보완*한다고 명시 → 거부감 없음
4. **개념이 3개뿐** — log / blame / show, 이미 아는 git 단어라 학습 비용 0
5. **데모 자산** — `assets/demo.gif`, `demo.mp4` 등 시각적 증거
6. **설치 1줄** — Go 단일 바이너리 + Homebrew + GoReleaser 멀티플랫폼
7. **진영 중립** — Claude Code / Codex / OpenCode 동시 지원
8. **기여자 유치 설계** — GOOD_FIRST_ISSUES, QUICK_START, 이슈 템플릿 3종,
   Phase별 이슈 번호 명시, Discord, CodeRabbit 자동 리뷰.
   실제로 외부 기여자 PR이 지속 머지되고 있음

> 참고: 이 정리에서는 실제 스타 수를 직접 확인하지 않았습니다(레포 접근 범위 제한).
> 위 항목은 저장소 구성과 문서 품질에 근거한 추론입니다.

---

## 10. 로컬 에이전트 구축에 주는 도움

**직접적 활용**
1. 에이전트 디버깅 — 왜 그 파일을 건드렸는지 `rgt show`로 추적
2. 회귀 분석 — 어떤 프롬프트가 버그를 유발했는지 역추적
3. 평가 데이터 수집 — `rgt log --json`으로 (프롬프트 → 행동 → 결과) 축적
4. 멀티 에이전트 관찰 — 세션별 ref로 작업 분리

**설계 참고 가치**
5. 훅 어댑터 패턴 — 4개 어댑터가 `internal/capture` 하나로 수렴
6. CAS ref — 동시 쓰기에도 깨지지 않는 포인터 갱신
7. "에이전트를 방해하지 않는다" 원칙 — 실패해도 조용히 로깅
8. 내용주소 저장 — 스냅샷 중복 제거로 디스크 증가 억제

**한계**
- `rewind` / `fork` 미구현 → 롤백은 직접 구현 필요
- 대형 모노레포에서 매 스텝 전체 스냅샷은 느림 (ROADMAP 인정 사항)
- `Bash`로 `sed -i` 같은 일괄 변경 시 blame이 뭉뚱그려짐

---

## 11. React / PHP로 만들 수 있는가

| 스택 | 코어 재구현 | 평가 |
|---|---|---|
| Go (현재) | ✅ | 단일 바이너리, 빠름, CGO 불필요 — 최적 |
| Node/TypeScript | ✅ 가능 | 라이브러리는 충분하나 런타임 의존 + 훅마다 부팅 오버헤드 |
| PHP | 🔶 가능하나 비권장 | SQLite PDO는 있으나 BLAKE3 성능·프로세스 부팅·배포 부담 |
| React | ❌ 불가 | UI 라이브러리라 파일시스템 접근 불가, 훅으로 사용 불가 |

### 권장: 코어를 다시 만들지 말고 그 위에 레이어를 올릴 것

```
React 대시보드 (세션 타임라인 / blame 히트맵 / 통계)
        │ REST API (JSON)
PHP(Laravel) API 서버
  - .regent/index.db 를 PDO SQLite로 직접 읽기
  - 또는 `rgt log --json` 실행 결과 파싱
        │
rgt (Go 바이너리) — 그대로 사용
  .regent/objects/ + index.db
```

근거:
1. `rgt log --json` 이 이미 존재 → `shell_exec()` 한 줄로 데이터 확보
2. `.regent/index.db`는 평범한 SQLite → Eloquent로 바로 연결 가능
3. re_gent에는 **웹 UI가 없음** → 시장 공백
4. Apache 2.0 → 상업적 이용 가능

---

## 12. 수익화 아이디어

### 전제
AI가 작성하는 코드 비중은 급증했으나 "누가/왜 썼는지" 증명 수단은 없습니다.
금융·의료·공공은 감사 의무가 있고, 사고 시 "AI가 했다"는 면책이 되지 않습니다.
**AI 코드의 출처 증명(provenance)** 수요가 커지고, re_gent는 그 원재료를 생산합니다.

| # | 아이디어 | 난이도 | 수익 규모 | React/PHP 적합 |
|---|---|---|---|---|
| 1 | 팀 대시보드 SaaS | ⭐⭐⭐ | 높음 | ✅ 최적 |
| 2 | 컴플라이언스/감사 리포트 | ⭐⭐⭐⭐ | 매우 높음 | ✅ |
| 3 | IDE 확장 Pro 티어 | ⭐⭐ | 중간 | 🔶 TS |
| 4 | 프롬프트 성과 분석 | ⭐⭐⭐⭐ | 중상 | ✅ |
| 5 | 학습/평가 데이터 툴킷 | ⭐⭐⭐⭐ | 중상 | 🔶 |
| 6 | 어댑터 기여 + 스폰서십 | ⭐⭐ | 낮음(평판 높음) | ❌ Go |
| 7 | 온프레 구축 + 교육 | ⭐⭐⭐ | 높음 | ✅ |

### 1. re_gent Cloud — 팀 대시보드 SaaS (최우선 추천)
- 기능: 팀 활동 타임라인, **AI 작성 코드 비율 히트맵**, 프롬프트 전문 검색,
  재작업률 분석, 민감 디렉토리 수정 Slack 알림, PR 리뷰 화면에 프롬프트 출처 표시
- 스택: React + TypeScript / Laravel / PostgreSQL / rgt CLI sync
- 가격: Free $0 · Team $12~19/시트/월 · Business $39/시트/월 · Enterprise 별도
- 시뮬레이션: 20인 팀 × $15 = 월 $300 → 100개 팀이면 월 $30,000
- MVP 2~3개월

### 2. AI 코드 컴플라이언스 리포트 (단가 최고)
- EU AI Act, 금융 내부통제, ISMS-P, SOC 2 등 "AI 사용 이력 관리" 요구 대응
- 라이선스 오염 검사, 시크릿 파일 접근 이력 감사, 분기 PDF 리포트
- 가격: 건당 500~5,000만원 또는 연 라이선스 2,000만원~

### 3. IDE 확장 Pro 티어 (진입 난이도 최저)
- 인라인 blame + 프롬프트 툴팁, 타임 트래블 슬라이더,
  AI/사람 코드 컬러 구분, 팀 동기화
- 가격: 개인 $5/월, 팀 $10/시트/월 · VS Code 마켓플레이스가 유통 채널

### 4. 프롬프트 성과 분석
- 프롬프트 길이 vs 재작업률, "리팩토링해줘" 류의 버그 유발률,
  가장 많이 수정된 스텝 탐지, 팀 베스트 프랙티스 자동 추출
- 킬러 기능: "이 프롬프트는 유사 사례에서 73% 확률로 회귀 버그 발생" 경고
- 가격: 아이디어 1의 애드온 $10/시트/월

### 5. 학습/평가 데이터 파이프라인
- (프롬프트 → 행동 → 결과 → 성공/실패) 데이터를 익명화·시크릿 제거 후 데이터셋화
- 가격: 툴킷 연 1,500만원~ + 구축 컨설팅
- ⚠️ 데이터 소유권·동의 설계 필수

### 6. 어댑터 기여 + 스폰서십
- ROADMAP상 Cursor / Cline / Continue 어댑터 미착수 → 기여로 메인테이너 포지션 확보
- 직접 수익은 작으나 평판 자산 가치가 큼

### 7. 온프레미스 구축 + 교육 (한국 시장 적합)
- 폐쇄망 설치 + 기존 감사 시스템 연동: 2,000~5,000만원
- 연 유지보수 20%, "AI 에이전트 거버넌스" 교육 회당 300~500만원

### 3개월 실행 플랜
- **1개월차 (검증)**: `rgt init` 후 2주 실사용 → `rgt log --json` 구조 파악 →
  PHP PDO로 `.regent/index.db` 읽는 프로토타입
- **2개월차 (MVP)**: React 대시보드(타임라인 + AI 기여도 차트),
  Laravel API(세션/스텝/blame 조회), 로컬 전용 오픈소스 공개 + 커뮤니티 홍보
- **3개월차 (수익화)**: 팀 동기화 + 인증/결제, Free/Team 티어 분리, 런칭

> 핵심 전략: 코어와 경쟁하지 말고 **"re_gent의 웹 UI"** 포지션을 선점할 것.
> 업스트림이 성장할수록 함께 성장하는 구조를 만든다.

---

## 13. 요약

- re_gent는 **에이전트 활동 전용 버전관리 시스템**이며 git을 대체하지 않고 보완한다.
- 구현은 **훅 기반 Go CLI**이고, MCP나 플러그인이 아니다.
- **완전 로컬 동작**으로 API 토큰이 필요 없지만, 대화 전문이 평문 저장되므로 `.regent/` 관리에 주의.
- 웹 UI가 없다는 점이 **React/PHP 스택으로 진입할 수 있는 가장 큰 기회**다.
- 수익화는 **팀 대시보드 SaaS → 컴플라이언스 리포트** 순으로 접근하는 것이 현실적이다.

---

_이 문서는 re_gent 저장소(https://github.com/bmshin94/re_gent) 전수조사 결과를 정리한 것입니다._
