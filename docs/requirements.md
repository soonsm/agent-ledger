# Agent Ledger 요구사항 초안

> 상태: Draft
>
> 이 문서는 Agent Ledger의 초기 제품 요구사항을 정의한다. 구현 세부사항보다 먼저, 도구가 어떤 동작을 보장해야 하는지 명확히 하는 것을 목적으로 한다.

## 1. 제품 목표

Agent Ledger는 AI Coding Agent가 커밋을 만들 때 프로젝트가 요구하는 수행·검토 항목에 **명시적으로 답하도록 강제**하고, 그 답을 커밋과 연결된 구조화된 장부로 보존한다.

이 장부는 이후 다음 질문에 답하는 데 사용한다.

- 마지막 검토 이후 특정 종류의 변화가 얼마나 누적되었는가?
- 특정 피드백 또는 감사가 언제 마지막으로 수행되었는가?
- 지금 다시 코드베이스를 넓게 검토해야 하는 시점인가?

Agent Ledger가 강제하는 것은 AI 판단의 정확성이 아니라 **필수 판단 또는 수행 보고의 생략 방지**다.

예를 들어 `state_flags_added`가 필수 항목이라면 다음은 서로 다른 상태다.

```text
state_flags_added = 0
```

```text
state_flags_added = <missing>
```

첫 번째는 "AI가 현재 변경을 검토했고 상태 플래그 추가가 없다고 판단했다"는 명시적 assertion이다. 두 번째는 검토했는지 알 수 없으므로 커밋을 허용하지 않는다.

---

## 2. 기본 전제

### 2.1 Git 저장소를 대상으로 한다

Agent Ledger는 기존 Git workflow를 보완하는 도구다. Git 자체를 대체하지 않는다.

사용 환경에는 Git이 설치되어 있다고 가정한다.

### 2.2 별도 언어 런타임을 요구하지 않는다

최종 사용자는 Node.js, JDK, Python 등의 런타임을 별도로 설치하지 않아도 되어야 한다.

초기 구현 언어는 Go를 기본 선택으로 한다.

배포 대상은 최소 다음을 포함한다.

- Windows x86-64
- macOS Apple Silicon
- macOS x86-64
- Linux x86-64

가능하면 추가 아키텍처를 지원한다.

### 2.3 AI의 악의적인 우회 방지는 v1의 보안 목표가 아니다

초기 버전은 별도 OS 사용자, 컨테이너, privileged daemon 같은 capability isolation을 요구하지 않는다.

AI가 의도적으로 `git commit`을 직접 실행하면 Agent Ledger를 우회할 수 있다.

대신 프로젝트의 `AGENTS.md` 등 agent instruction에서 다음을 요구하는 방식을 기본으로 한다.

```text
커밋은 반드시 agent-ledger를 사용한다.
git commit을 직접 호출하지 않는다.
--no-verify 등의 방식으로 커밋 절차를 우회하지 않는다.
```

Agent Ledger 내부에서는 장부 항목 누락을 우회할 수 없어야 한다.

---

## 3. 구성 요소

초기 시스템은 다음 요소로 구성한다.

```text
AGENTS.md 또는 이에 준하는 agent instruction
        │
        │ agent-ledger 사용을 지시
        ▼
agent-ledger 실행 파일
        │
        ├─ 장부 명세 관리
        ├─ 커밋 입력 검증
        ├─ Git commit 생성
        ├─ Git Notes 기록
        └─ 누적 상태 조회 / 감사 기록
        │
        ▼
agent-ledger.json
        │
        │ 프로젝트별 장부 명세
        ▼
Git commit + Git Notes
```

### 역할 구분

- **Agent instruction**: 어떤 커밋 경로를 사용해야 하는지 정의한다.
- **`agent-ledger.json`**: 현재 프로젝트에서 무엇을 기록하고 검토해야 하는지 정의한다.
- **`agent-ledger` CLI**: 명세를 검증하고 집행하며 Git과 상호작용한다.
- **Git Notes**: 각 커밋에 실제 제출된 장부 값을 저장한다.

---

## 4. 장부 명세

### R-SPEC-001 프로젝트별 명세 파일

각 저장소는 프로젝트별 장부 명세 파일을 가질 수 있어야 한다.

기본 파일명:

```text
agent-ledger.json
```

파일은 저장소에서 버전 관리할 수 있는 일반 텍스트 JSON이어야 한다.

### R-SPEC-002 명세와 실행 엔진 분리

장부 항목을 프로그램 코드에 하드코딩하지 않아야 한다.

새 장부 항목을 추가하거나 기존 항목의 질문을 명확히 하거나, 기존 항목을 폐기하고 다른 타입·감사 기준의 새 항목을 추가할 때 `agent-ledger` 바이너리를 다시 빌드하지 않아야 한다.

`agent-ledger` 바이너리는 장부 명세가 따라야 하는 meta-schema와 validation 로직을 가진다.

### R-SPEC-003 안정적인 항목 ID

각 장부 항목은 변경되지 않는 고유 ID를 가져야 한다.

예:

```text
L001
L002
L003
```

사람이 읽는 key나 질문 문구는 바뀔 수 있지만, 이미 사용된 ID는 다른 의미로 재사용해서는 안 된다.

예:

```text
L001
과거 key: state_flags_added
현재 key: mutable_state_added
```

과거 기록과 현재 명세의 계보는 `key`가 아니라 ID로 연결한다.

### R-SPEC-004 지원 타입

v1은 최소 다음 타입을 지원한다.

- integer
- boolean
- enum
- string

타입별 validation을 지원해야 한다.

예:

- integer: 최소값, 최대값
- enum: 허용 값 목록
- string: 비어 있음 허용 여부

### R-SPEC-005 질문과 설명

각 필드는 AI가 무엇을 검토해야 하는지 이해할 수 있는 설명을 포함할 수 있어야 한다.

최소한 다음 속성을 고려한다.

```text
id
key
type
status
question
description
```

예:

```json
{
  "id": "L001",
  "key": "state_flags_added",
  "type": "integer",
  "min": 0,
  "status": "active",
  "question": "이번 변경에서 추가한 상태 플래그 수는? 없으면 0.",
  "description": "실행 중 값이 바뀌며 제어 흐름이나 동작 상태를 나타내는 상태 필드"
}
```

### R-SPEC-006 기본값 금지

필수 장부 항목에는 암묵적인 기본값을 적용해서는 안 된다.

특히 다음 변환은 금지한다.

```text
missing → 0
missing → false
missing → not-run
```

명시적 `0`, `false`, `not-run`은 유효한 값일 수 있지만 반드시 호출자가 제출해야 한다.

---

## 5. 장부 명세 관리

장부 명세의 정상적인 변경 경로는 `agent-ledger` CLI여야 한다.

사람이나 AI가 JSON을 직접 편집하는 것을 기술적으로 막을 필요는 없지만, Agent instruction과 문서에서는 CLI를 통한 변경을 기본으로 안내한다.

### R-SPEC-101 항목 추가

다음에 준하는 기능을 제공해야 한다.

```text
git ledger spec add
```

새 항목 추가 시 다음을 검증한다.

- ID 중복
- key 중복
- 지원하는 type인지
- 필수 속성 존재 여부
- type별 제약조건의 유효성

ID는 CLI가 자동 생성하는 방식을 우선 고려한다.

### R-SPEC-102 항목 수정

다음에 준하는 기능을 제공해야 한다.

```text
git ledger spec update <id>
```

v1에서는 같은 관찰 항목의 표현을 더 명확하게 만드는 metadata 수정만 허용한다.

수정 가능:

- `key`
- `question`
- `description`

수정 불가:

- `type`
- `min` / `max`
- `values`
- `audit.aggregation`
- `audit.threshold`
- `status`
- `deprecation_reason`

수정 불가 속성을 바꿔야 한다면 기존 항목을 `deprecate`하고 새 항목을 `add`한다.

판단 기준은 다음과 같다.

> 같은 것을 더 명확하게 설명하는 변경은 같은 ID를 유지한다. 무엇을 측정하거나 어떻게 판단·누적하는지가 바뀌면 새 ID를 사용한다.

### R-SPEC-103 항목 폐기

이미 사용된 항목이라도 더 이상 해당 관점을 관찰하지 않기로 결정했다면 폐기할 수 있어야 한다.

```text
git ledger spec deprecate <id> --reason "<reason>"
```

폐기 이유는 필수다.

폐기된 항목은:

- 신규 커밋에서는 값을 요구하지 않는다.
- 과거 Git Notes를 해석할 때는 정의가 유지된다.
- ID를 다른 의미로 재사용하지 않는다.
- `deprecation_reason`에 관찰을 중단한 이유를 저장한다.

### R-SPEC-104 항목 복구

폐기한 항목을 다시 관찰해야 할 필요가 생기면 같은 ID로 다시 활성화할 수 있어야 한다.

```text
git ledger spec restore <id>
```

복구되면:

- `status`는 `active`가 된다.
- 이후 새 커밋에서 해당 항목의 값을 다시 반드시 요구한다.
- 현재 명세의 `deprecation_reason`은 제거한다.
- stable ID는 유지한다.

### R-SPEC-105 항목 삭제

아직 어떤 장부 기록에서도 사용되지 않은 항목은 물리적으로 삭제할 수 있어야 한다.

```text
git ledger spec remove <id>
```

이미 사용 이력이 있는 경우 삭제를 거절하고 `deprecate` 사용을 안내해야 한다.

예:

```text
L001 has already been recorded in 37 commits.
The field cannot be removed.
Use `git ledger spec deprecate L001` instead.
```

### R-SPEC-106 명세 조회

최소 다음 조회 기능을 제공한다.

```text
git ledger spec list
git ledger spec show <id>
```

### R-SPEC-107 명세 검증

다음 명령을 제공한다.

```text
git ledger spec validate
```

최소 검증 대상:

- JSON syntax
- 지원하는 schema version
- 필수 속성 존재 여부
- ID 중복
- key 중복
- field type
- enum values
- integer range
- status
- audit 설정

명세가 유효하지 않은 상태에서는 `git ledger commit`이 실행되어서는 안 된다.

### R-SPEC-108 안전한 파일 갱신

CLI를 통한 명세 변경은 validation을 통과한 경우에만 원본 파일에 반영해야 한다.

권장 절차:

```text
현재 명세 읽기
→ 메모리에서 변경
→ 전체 validation
→ 임시 파일 작성
→ atomic replace
```

변경된 명세가 유효하지 않으면 기존 파일은 그대로 유지한다.

---

## 6. 커밋 workflow

### R-COMMIT-001 전용 커밋 명령

AI가 사용할 기본 커밋 명령을 제공한다.

```text
git ledger commit
```

CLI 옵션이나 입력 방식은 구현 과정에서 결정할 수 있지만, AI Coding Agent가 비대화된 명령문을 작성하지 않아도 사용할 수 있어야 한다.

### R-COMMIT-002 기존 Git staging workflow와 호환

v1에서는 일반적인 Git index를 사용한다.

즉 사용자는 기존과 동일하게 필요한 파일을 stage할 수 있고, `git ledger commit`은 현재 staged changes를 대상으로 커밋한다.

Agent Ledger가 사용자 의도 없이 자동으로 모든 working-tree 변경을 stage해서는 안 된다.

### R-COMMIT-003 active 필드 전부 요구

현재 명세에서 `active` 상태인 모든 장부 항목은 커밋 전에 모두 명시적으로 제출되어야 한다.

하나라도 누락되면 실제 Git commit을 생성하지 않는다.

### R-COMMIT-004 누락 feedback

필드가 누락된 경우 단순히 validation error만 출력하지 않고 AI가 다시 판단할 수 있는 정보를 제공해야 한다.

예:

```text
Commit rejected.

Missing required ledger entries:

L001 state_flags_added
  이번 변경에서 추가한 상태 플래그 수는? 없으면 0.

L003 duplicate_logic_added
  이번 변경에서 기존 로직의 새 사본을 추가했는가? 개수를 적는다. 없으면 0.
```

이 feedback을 통해 AI가 전체 장부 명세를 사전에 기억하지 않아도 되게 한다.

### R-COMMIT-005 값 validation

모든 필드가 존재하더라도 타입 또는 제약조건을 만족하지 않으면 커밋을 거절한다.

예:

```text
state_flags_added = -1
```

은 `min = 0`일 경우 실패한다.

### R-COMMIT-006 판단의 의미적 정확성은 검증하지 않음

Agent Ledger는 다음과 같은 semantic truth를 검증하려 하지 않는다.

```text
실제로 상태 플래그가 2개 추가되었는가?
실제로 중복 로직이 만들어졌는가?
테스트 계층 검토가 충분했는가?
```

AI가 제출한 값은 assertion으로 기록한다.

### R-COMMIT-007 커밋 메시지와 장부 분리

장부 전체를 commit message에 삽입하지 않는다.

일반 Git commit message는 사람이 읽는 기존 역할을 유지한다.

구조화된 장부는 Git Notes를 기본 저장소로 사용한다.

### R-COMMIT-008 커밋 성공 후 note 기록

정상 흐름은 다음과 같다.

```text
장부 명세 validation
→ 제출 필드 validation
→ git commit
→ 생성된 commit SHA 획득
→ 해당 commit SHA에 Git Note 기록
```

Git Note에는 최소 다음 정보가 포함되어야 한다.

```json
{
  "schema": 1,
  "source": "agent-ledger",
  "entries": {
    "L001": 0,
    "L002": "passed"
  }
}
```

실제 형식은 versioned schema로 정의한다.

### R-COMMIT-009 note 기록 실패를 성공으로 숨기지 않음

Git commit은 생성되었지만 Git Note 기록이 실패한 경우 이를 정상 성공으로 보고해서는 안 된다.

사용자 또는 AI가 복구할 수 있도록 생성된 commit SHA와 실패 원인을 명확히 출력해야 한다.

향후 필요하면 기존 커밋에 유효한 ledger entry를 복구해서 연결하는 명령을 제공할 수 있다.

---

## 7. 장부 저장 및 Git Notes

### R-NOTE-001 commit SHA를 기본 연결 키로 사용

별도의 ledger ID를 commit message에 넣는 것을 필수로 하지 않는다.

Git commit SHA 자체를 장부 기록의 기본 대상 식별자로 사용한다.

### R-NOTE-002 구조화된 데이터

Git Note는 사람이 읽을 수 있으면서 기계적으로 안정적으로 파싱할 수 있는 구조화된 형식이어야 한다.

JSON을 기본 포맷으로 한다.

### R-NOTE-003 schema version

모든 Note에는 schema version을 포함해야 한다.

과거 schema를 읽을 수 있는 호환 정책을 정의할 수 있어야 한다.

### R-NOTE-004 전용 notes ref

Agent Ledger의 기록은 다른 Git Notes 사용 사례와 충돌하지 않도록 전용 notes ref를 사용하는 것을 기본으로 한다.

예:

```text
refs/notes/agent-ledger
```

정확한 ref 이름은 구현 단계에서 확정한다.

### R-NOTE-005 원격 동기화 안내

Git Notes는 일반 branch와 동일하게 기본 push/fetch 대상이 아닐 수 있다.

Agent Ledger는 사용자가 원격 저장소에서도 장부를 유지하려면 필요한 push/fetch 설정을 문서화해야 한다.

향후 CLI가 notes 동기화를 보조하는 기능을 제공할 수 있다.

---

## 8. 사람의 일반 커밋

### R-HUMAN-001 일반 git commit 허용

v1에서는 사람에게 `git ledger commit` 사용을 강제하지 않는다.

사람은 필요하면 기존 `git commit`을 사용할 수 있다.

### R-HUMAN-002 provenance 해석

Agent Ledger note가 존재하면 다음 사실만 보장한다.

> 이 커밋은 Agent Ledger가 관리하는 경로를 통해 장부 기록이 생성되었다.

Note가 없는 커밋을 반드시 "사람이 만든 커밋"이라고 해석해서는 안 된다.

정확한 분류는 다음과 같다.

```text
note 있음 → managed commit
note 없음 → unmanaged commit
```

Git author/email 등의 identity를 AI/사람 구분을 위한 핵심 보안 메커니즘으로 사용하지 않는다.

---

## 9. 수행 보고와 판단 보고

장부 항목은 성격상 두 종류가 있을 수 있다.

### 수행 assertion

AI가 특정 명령이나 작업을 수행했다고 보고하는 값.

예:

```text
typecheck = passed
lint = passed
test = not-run
```

### 판단 assertion

AI가 코드를 읽고 의미적으로 판단한 값.

예:

```text
state_flags_added = 0
duplicate_logic_added = 1
test_layer_reviewed = true
```

### R-ENTRY-001 v1에서는 둘 다 명시적 입력 가능

v1에서는 수행 assertion과 판단 assertion 모두 AI가 명시적으로 제출하는 모델을 허용한다.

`typecheck`가 실제로 실행되었는지를 추적하는 별도 instrumentation은 v1 필수 범위가 아니다.

### R-ENTRY-002 향후 기계적 관측으로 확장 가능

향후 typecheck, lint, test처럼 기계적으로 관측 가능한 항목은 Agent Ledger가 직접 실행하거나 verification stamp 등을 통해 자동 기록할 수 있어야 한다.

이 확장이 기존 판단형 장부 모델을 깨뜨려서는 안 된다.

---

## 10. 누적 상태와 감사

Agent Ledger의 장부는 단순 보관이 아니라 **다음 피드백 시점을 계산하는 입력**으로 사용할 수 있어야 한다.

### R-AUDIT-001 항목별 누적 기준

장부 항목은 선택적으로 감사 기준을 가질 수 있어야 한다.

예:

```text
L001 state_flags_added
aggregation = sum
threshold = 30
```

마지막 관련 감사 이후 `L001` 값의 합이 30 이상이면 감사를 수행할 시점으로 판단할 수 있다.

### R-AUDIT-002 감사 종류별 독립적인 기준점

서로 다른 감사가 동일한 기준점을 공유할 필요는 없다.

예:

```text
상태 플래그 감사
→ state_flags_added 누적 합 기준

공개 표면 감사
→ public_surface_added 누적 합 기준
```

한 감사를 수행했다고 다른 감사의 누적 상태가 자동으로 초기화되어서는 안 된다.

### R-AUDIT-003 감사 수행 기록

누적 감사 수행 자체를 Git 이력과 연결된 형태로 기록할 수 있어야 한다.

감사 기록은 최소 다음을 표현할 수 있어야 한다.

- 어떤 감사인지
- 언제 수행했는지
- 어느 커밋까지를 대상으로 했는지
- 이후 누적 계산의 기준점

정확한 저장 모델은 구현 단계에서 결정한다.

### R-AUDIT-004 상태 조회

다음에 준하는 명령을 제공해야 한다.

```text
agent-ledger status
```

예:

```text
State flags
  accumulated: 27
  threshold:   30
  remaining:    3

Duplicate logic
  accumulated: 8
  threshold:   20
  remaining:   12
```

threshold에 도달하거나 초과한 항목은 명확하게 표시해야 한다.

### R-AUDIT-005 감사 후 재누적

감사가 정상적으로 기록되면 해당 감사의 기준점 이후부터 새로 누적할 수 있어야 한다.

장부의 과거 값을 삭제하거나 변경해서 카운터를 0으로 만드는 방식은 사용하지 않는다.

---

## 11. CLI UX

### R-CLI-001 AI 친화적인 오류

오류 메시지는 사람뿐 아니라 Coding Agent가 다음 행동을 결정할 수 있을 만큼 구체적이어야 한다.

나쁜 예:

```text
validation failed
```

좋은 예:

```text
Commit rejected.
Missing required field L001 (state_flags_added).
Review the current staged change and provide an integer >= 0.
If no state flag was added, explicitly provide 0.
```

### R-CLI-002 안정적인 exit code

성공과 실패를 shell에서 확실히 판별할 수 있도록 exit code를 일관되게 사용한다.

필요하면 향후 오류 종류별 exit code를 정의한다.

### R-CLI-003 non-interactive 사용 우선

AI Coding Agent가 안정적으로 호출할 수 있도록 non-interactive invocation을 우선 지원한다.

사람을 위한 interactive prompt는 추가할 수 있지만 자동화 가능한 입력 방식을 반드시 제공한다.

### R-CLI-004 자기 설명 가능한 명령

다음과 같은 도움말을 제공한다.

```text
agent-ledger --help
git ledger commit --help
git ledger spec --help
```

AI가 명령 사용법을 추가 문서 없이도 어느 정도 복구할 수 있어야 한다.

---

## 12. AGENTS.md 통합 가이드

Agent Ledger는 저장소에 추가할 수 있는 최소 agent instruction 예시를 제공해야 한다.

예:

```text
## Commits

When creating a commit, always use `git ledger commit`.
Do not invoke `git commit` directly and do not use `--no-verify`
or another mechanism to bypass this process.

Agent Ledger may require ledger entries before committing.
Review the current change and explicitly provide every required value,
including zero, false, or not-run values. Never omit a field because
it appears irrelevant. Follow the field description returned by the CLI.

Use `git ledger spec ...` commands when changing the ledger specification.
Do not edit the ledger specification directly unless recovery requires it.
```

장부 필드 목록 자체를 AGENTS.md에 복제하지 않는다.

---

## 13. 비기능 요구사항

### R-NFR-001 단일 실행 파일

일반 사용자가 언어 런타임이나 package manager를 설치하지 않아도 사용할 수 있는 단일 실행 파일로 배포한다.

### R-NFR-002 빠른 시작

`git ledger commit` 자체의 startup overhead는 사람이 체감할 정도로 커서는 안 된다.

AI가 제출한 semantic assertion을 CLI가 다시 추론하지 않으므로 기본 validation은 로컬에서 빠르게 완료되어야 한다.

### R-NFR-003 외부 서비스 의존 없음

기본 commit/ledger workflow는 SaaS, API key, 네트워크 연결을 요구해서는 안 된다.

AI 모델 호출은 Agent Ledger 자체의 필수 기능이 아니다. 판단은 Agent Ledger를 호출하는 Coding Agent가 수행한다.

### R-NFR-004 프로젝트 이식성

Git 저장소와 `agent-ledger.json`, Git Notes만으로 프로젝트의 장부 상태를 다른 컴퓨터에서도 재구성할 수 있는 방향을 지향한다.

### R-NFR-005 명세 변경에 대한 하위 호환성

명세의 key나 설명이 변경되어도 stable ID를 통해 과거 장부를 조회할 수 있어야 한다.

---

## 14. v1 필수 범위

v1은 피드백 루프의 가장 작은 유효한 형태를 검증하는 데 집중한다.

### 필수

1. Go 기반 단일 실행 파일
2. `agent-ledger.json` 명세 읽기
3. 명세 validation
4. `spec add`
5. `spec update`
6. `spec deprecate`
7. `spec restore`
8. `spec remove`
9. `spec list`
10. `spec show`
11. `git ledger commit`
12. 모든 active 필드의 명시적 제출 강제
13. 누락 시 commit 거절 및 구체적인 feedback
14. field type/constraint validation
15. Git commit 생성
16. 전용 Git Notes ref에 구조화된 장부 기록
17. AGENTS.md 통합 가이드

### v1에서 구현 여부를 설계 단계에서 추가 결정할 기능

- `status`와 threshold 계산을 v1에 포함할지 여부
- 감사 수행 event의 구체적인 저장 모델
- 기존 커밋에 note를 복구하는 repair 명령
- Git Notes push/fetch 보조 명령

목적상 누적 감사와 threshold는 최종 제품의 핵심 기능이지만, 첫 구현은 **"커밋마다 필수 assertion을 빠짐없이 남길 수 있는가"**를 먼저 검증한 뒤 확장할 수 있다.

---

## 15. 비목표

초기 버전에서 다음은 목표가 아니다.

- AI가 제출한 판단 값의 의미적 정확성 검증
- 상태 플래그나 중복 로직 등을 자동 판별하는 정적 분석기
- 악의적인 AI의 모든 Git 우회 경로 차단
- OS/container 수준의 권한 격리
- 사람의 일반 `git commit` 차단
- AI와 사람을 Git author/email로 완벽하게 식별
- Git 자체 대체
- AI 세션 transcript 전체 저장
- 자동 코드 리뷰 서비스 제공
- Agent Ledger 자체가 LLM을 호출해 판단 수행

---

## 16. 핵심 성공 조건

첫 번째 성공 조건은 다음과 같다.

> 장부 항목이 늘어나거나 변경되어도 AI는 항목 목록을 미리 기억할 필요가 없으며, 커밋하려는 순간 Agent Ledger의 feedback을 통해 모든 필수 항목에 명시적으로 답하게 된다.

두 번째 성공 조건은 다음과 같다.

> 장부 항목을 추가·수정·폐기해도 Agent Ledger 바이너리를 수정하거나 다시 빌드할 필요가 없다.

장기적인 성공 조건은 다음과 같다.

> 사람이 모든 개별 코드 변경을 직접 읽지 않더라도, 프로젝트가 중요하다고 정의한 변화가 누적되는 것을 기억하고 적절한 시점에 AI가 코드베이스를 다시 넓게 검토하는 피드백 루프를 유지할 수 있다.

---

## 17. 아직 확정하지 않은 설계 사항

다음은 요구사항을 만족시키는 구현 선택지이며, 구현 전에 별도 설계 결정을 내린다.

- `agent-ledger.json`의 정확한 JSON schema
- field ID 생성 규칙
- `git ledger commit`에 장부 값을 전달하는 구체적인 CLI UX
- Git Notes ref의 정확한 이름
- audit event의 데이터 모델
- threshold의 aggregation 종류 (`sum`, `count` 등)
- schema migration 정책
- Git Notes 원격 동기화 UX
- commit 성공 후 note 실패 시 repair workflow

이 항목들은 제품 목적이나 핵심 요구사항을 변경하지 않는 범위에서 결정한다.
