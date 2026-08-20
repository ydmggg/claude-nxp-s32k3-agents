---
name: work
description: |
  Plan → approval → implement → review → log workflow.
  Use for feature development, code modification, bug fixes, code review, and work reports.
  Invoked as /work [task description].
argument-hint: "[task description]"
user-invocable: true
---

# work — lightweight workflow manager

Goal: **correct execution over fast execution.** Never modify code immediately; plan, get approval,
then run role-separated agents (brains) for implementation and review, and keep a work log.
All user-facing output in Korean.

## Overall Flow

```
/work request
  → [1] classify
  → [2] explore & plan
  → [3] user approval                    ← 승인 없이 구현 금지
  → [4] implement        (A) dev-executor | dev-nxp-s32k-executor | wpf-executor
  → [5] review x5        (B) embedded-reviewer | wpf-reviewer  — 5인 독립 병렬
  → [6] consolidate      (C) reporter | wpf-reporter — 5인 판정 취합, 합의 여부 결정
  → [7] worklog record
  → [8] final summary → 사용자 승인       ← "올려줘 / 진행시켜"
  → [9] git             (D) git-manager — 커밋·푸시
```

### 4-Agent Pipeline

| | 역할 | 에이전트 | 비고 |
|---|---|---|---|
| **A** | 구현 | `dev-executor` / `dev-nxp-s32k-executor` / `wpf-executor` | 프로젝트 타입으로 선택 |
| **B** | 검토 | `embedded-reviewer` / `wpf-reviewer` | **5인 독립 병렬**, 전원 일치 필요 |
| **C** | 취합·보고 | `reporter` (MCU) / `wpf-reporter` (WPF) | 5인 판정 취합 (매번) + 종합 보고서 (요청 시) |
| **D** | 형상관리 | `git-manager` | 사용자 승인 후에만 동작 |

Reports: 종합 보고서는 사용자가 요청할 때만 (C가 worklog 기반으로 작성)

## Step Rules

### [1] Classify
Classify the request and tell the user:
- **Development/modification**: new feature or code change → full flow
- **Bug fix**: root-cause analysis first; report the cause, then plan
- **Review only**: no code changes; run reviewer/embedded-reviewer only
- **Report**: no code changes; reporter builds a consolidated report from docs/worklog.md
- **Question**: no code changes; investigate and answer

### [2] Explore & Plan (NO code modification)
- If this folder is a git repo with a remote: **run `git pull` first** to base the plan on the
  latest code (on conflict with local changes, stop and ask the user).
- Modify nothing else in this step. Read only.
- Find and read relevant files, then present a plan.
- The plan MUST use **checkbox items**. Vague catch-alls ("etc.", "and so on") are forbidden.

Plan format (write contents in Korean):
```markdown
## 작업 계획: [제목]

### 요구사항
- [ ] (독립적으로 확인 가능한 항목 1)
- [ ] (항목 2)

### 작업 목록
- [ ] TASK-01: (내용) — 수정 파일: (경로)
- [ ] TASK-02: (내용) — 수정 파일: (경로)
(독립 TASK는 "병렬 가능" 표시)

### 확인 방법
- [ ] (완료를 어떻게 검증할지: 빌드, 실행, 테스트)
```

### [3] User Approval
- NEVER start implementation before explicit approval of the plan.
- If the user requests changes, revise the plan and get approval again.

### [4] Implement
- Delegate each TASK to the implementation agent, **chosen by project type**:

  | Detected as | Agent |
  |---|---|
  | `*.mex` + `RTD/` exist (NXP S32K3 + S32 Config Tools) | `dev-nxp-s32k-executor` |
  | `*.ioc` + `Drivers/STM32*_HAL_Driver/` exist (STM32 CubeMX) | `dev-executor` |
  | `*.csproj` with `<UseWPF>true</UseWPF>`, or `*.xaml` present (WPF) | `wpf-executor` |
  | anything else (C# WinForms, plain C/C++) | `dev-executor` |

  Detect from the folder that the TASK's target files live in, not the repository root.
  A repo can hold both an S32K3 project and other code — pick per TASK.
  State which agent was chosen and why in the final summary.
- **Run independent TASKs (different files) in parallel.**
- TASKs touching the same file, or depending on a prior TASK's output, run sequentially.
- Check off plan checkboxes as TASKs complete.

#### 구현 중 막히면 — 즉시 사용자에게 묻는다

An implementation agent that returns `[TASK-XX 중단]` is asking a question. Handle it **now**:

1. **Ask the user immediately.** Do not hold it until the final summary, and do not
   decide it yourself. Ambiguous requirements and evidence-free values are the user's call.
2. Do **not** start any new TASK while a blocking question is open.
   TASKs already running in parallel finish on their own — let them.
3. Present it compactly: 막힌 지점 / 질문 / 선택지 / 여기까지 한 것.
   Give a recommendation when there is a reason to, but do not act on it unasked.
4. Once answered, resume that TASK from where it stopped. Do not redo finished work.
5. Record the question and the user's answer in the worklog at [7] — the decision, and why.

Judgement calls that only affect style or a local detail are yours to make; carry on and
mention them in the summary. Anything that changes **what the code does** goes to the user.

### [5] Review — 5인 독립 병렬 검토

- Pick the reviewer by project type (same detection as step [4]):

  | Detected as | Reviewer |
  |---|---|
  | MCU (C/C++, STM32 / NXP S32K3) | `embedded-reviewer` |
  | C# WPF / WinForms | `wpf-reviewer` |

  (WinForms 등 WPF 외 C# 코드도 `wpf-reviewer` 의 공통 C# 체크리스트로 검토한다.)

- **Launch 5 instances in parallel, in a single message.** All five get the *same* instruction
  and the *same* target files. Do not give them different roles — agreement between independent
  identical reviews is the point.
- **Never tell a reviewer what another reviewer found.** Independence is what makes the
  consensus meaningful.
- Each returns one verdict: `통과` / `수정 필요` / `판단 보류`.

### [6] Consolidate — `reporter` / `wpf-reporter`

Pick the consolidator by project type (same detection as step [4]) — MCU 는 `reporter`,
WPF/C# 은 `wpf-reporter`. Hand all 5 verdicts to it (consolidation role) and let it decide:

```
5인 전원 통과                 → 통과 → [7]
1인 이상 "심각" 지적 (타당)     → A 에게 수정 지시 → 지적 항목만 [5] 재검토 (최대 2회)
"권장"만 존재                 → 조건부 통과. 권장 사항은 기록만 하고 [7]
"판단 보류" 존재 / 의견 모순     → 쟁점 정리해 사용자에게 판단 요청
2회 재검토 후에도 미해결        → 사용자에게 보고하고 판단을 맡긴다
```

**다수결이 아니다.** 5명 중 1명만 찾아낸 결함이라도 근거가 타당하면 수정 대상이다.
반대로 근거 없는 지적은 취합 에이전트(`reporter` / `wpf-reporter`)가 걸러낸다.

### [7] Worklog Record (do NOT create report files!)
- **After every task, append a short record to `docs/worklog.md`.** No separate report files.
- Append in this format (in Korean) at the end of the file (never erase existing records):

```markdown
## YYYY-MM-DD HH:MM — [요청 한 줄 요약]
- 분류: 개발/수정 | 버그 수정 | 검토
- 요청: (사용자가 시킨 것, 1~2문장)
- 변경: 파일명 — 무엇을 왜 (한 줄씩. 검토만 한 경우 "변경 없음")
- 검토: 5인 검토 — 통과 N / 수정 필요 N / 판단 보류 N, 취합 판정, 조치한 지적 N건
- 미결: (남은 일. 없으면 "없음")
```

### [8] Final Summary → 사용자 승인

Show the user a final summary in chat (not a file):
- full checkbox status
- **검토 취합 결과** (5인 판정 분포, 합의 여부, 조치한 지적)
- open items

그리고 **푸시 여부를 사용자에게 묻는다.** 승인 없이 [9] 로 넘어가지 않는다.

### [9] Git — `git-manager`

**사용자가 "올려줘 / 진행시켜" 라고 명시한 뒤에만** `git-manager` 를 호출한다.

- 검토 통과만으로는 부족하다. 사용자가 직접 확인하고 승인해야 한다.
- 커밋 범위: 변경 코드 + `docs/worklog.md` + 관련 보고서
- 커밋 메시지 규약과 `--force` 금지 등 세부 규칙은 `git-manager` 가 보유한다
- 충돌·빌드 산출물 다수 포함 등 이상이 있으면 `git-manager` 가 멈추고 보고한다

## Consolidated Report (ONLY on user request)

취합 에이전트(`reporter` / `wpf-reporter`)는 매 사이클 [6] 에서 **취합** 역할로 쓰이고,
아래는 **종합 보고서** 역할이다. 호출할 때 어느 역할인지 명시한다.

When the user asks for a report ("보고서 만들어줘", "지금까지 작업 정리해줘", or /work 보고서):
1. Call `reporter` (MCU) or `wpf-reporter` (WPF) with the **report role**.
2. The agent writes a consolidated report to `docs/reports/YYYY-MM-DD-title.md` based on
   worklog records not yet included in a previous report.
3. Afterwards, mark each included worklog record with `> 보고서 반영: (report filename)`.
4. 보고서를 커밋에 포함하려면 [9] 와 동일하게 **사용자 승인 후** `git-manager` 를 호출한다.

## Agent Invocation

**Invoking `/work` IS the user's explicit request to run this 4-agent pipeline.**
Do not ask again, and do not require the user to say "에이전트로 진행해줘".
Steps [4] [5] [6] [9] delegate to their agents by default.

Exceptions — handle directly without agents, and say so briefly:
- 분류가 **질문**인 경우 (코드 변경 없음)
- 오타·상수 하나 수정처럼 검토 5인이 과한 변경
  → 이때도 자체 점검 결과는 보고한다
