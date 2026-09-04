# Agent Ledger CLI UX

> 상태: Draft
>
> 이 문서는 Agent Ledger의 CLI 사용 흐름과 입력 규칙을 정의한다. 현재 단계에서는 특히 AI Coding Agent가 장부 항목을 미리 기억하지 않아도 안정적으로 커밋할 수 있는 UX를 우선한다.

## 1. Git 확장으로 제공한다

Agent Ledger는 Git의 commit 기능을 다시 구현하지 않는다.

실행 파일은 Git 외부 명령 관례에 맞춰 `git-ledger`로 배포하고, 사용자는 다음과 같이 호출한다.

```bash
git ledger ...
```

기본 커밋 명령은 다음과 같다.

```bash
git ledger commit
```

Agent Ledger가 책임지는 것은 장부 관련 입력과 검증이다. 실제 커밋 옵션과 동작은 가능한 한 기존 `git commit`에 위임한다.

---

## 2. Agent Ledger 영역과 Git 영역을 명시적으로 분리한다

`git ledger commit`은 `--`를 경계로 두 영역을 분리한다.

```text
-- 이전
→ Agent Ledger가 해석

-- 이후
→ Agent Ledger가 해석하지 않고 `git commit`에 그대로 전달
```

예:

```bash
git ledger commit \
  --ledger-set L001=0 \
  --ledger-set L002=passed \
  -- \
  -m "Fix IME handling" \
  --signoff
```

Agent Ledger가 실제로 실행하는 Git 명령은 개념적으로 다음과 같다.

```bash
git commit -m "Fix IME handling" --signoff
```

`--` 뒤의 인자를 이해하거나 검증하거나 재구성하지 않는다. 원래 순서와 값을 그대로 `git commit`에 전달한다.

이 원칙을 사용하면 Agent Ledger가 다음과 같은 Git 옵션의 의미를 알 필요가 없다.

```text
-m
--amend
--no-edit
--author
--signoff
-S
...
```

현재 Git이 지원하거나 향후 새로 지원하는 옵션도 `--` 뒤에 있으면 Agent Ledger의 변경 없이 그대로 사용할 수 있다.

---

## 3. Agent Ledger 전용 옵션은 namespace를 사용한다

Agent Ledger가 소비하는 옵션은 Git 옵션과 충돌하지 않도록 `--ledger-*` namespace를 사용한다.

초기 commit 관련 옵션은 다음과 같다.

```text
--ledger-requirements
--ledger-set <field-id>=<value>
```

장황함을 줄이는 것보다 옵션의 소유권을 명확하게 하고 Git과의 미래 충돌 가능성을 줄이는 것을 우선한다. 주 사용자가 AI Coding Agent라는 점에서 긴 옵션명 자체는 중요한 비용으로 보지 않는다.

---

## 4. 현재 요구사항 조회

커밋 전에 현재 프로젝트가 요구하는 장부 항목을 조회할 수 있어야 한다.

```bash
git ledger commit --ledger-requirements
```

출력에는 최소한 다음 정보가 포함되어야 한다.

- stable ID
- 현재 key
- 값의 타입 및 제약조건
- AI가 검토해야 할 질문
- 실제 commit 명령에서 값을 전달하는 방법

예:

```text
Required ledger entries:

L001  state_flags_added
Type: integer >= 0
Question:
  이번 변경에서 추가한 상태 플래그 수는? 없으면 0.

L002  typecheck
Type: enum [passed, failed, not-run]
Question:
  Typecheck를 수행했는가?

Commit syntax:

  git ledger commit \
    --ledger-set L001=<integer> \
    --ledger-set L002=<passed|failed|not-run> \
    -- \
    <git commit arguments...>
```

`--ledger-requirements`의 목적은 AI가 `agent-ledger.json`을 직접 읽거나 현재 장부 항목을 미리 기억할 필요를 없애는 것이다.

---

## 5. 장부 값 전달 문법

장부 값은 반복 가능한 `--ledger-set` 옵션으로 전달한다.

기본 형식은 다음과 같다.

```text
--ledger-set <field-id>=<value>
```

예:

```bash
git ledger commit \
  --ledger-set L001=0 \
  --ledger-set L002=passed \
  --ledger-set L003=true \
  -- \
  -m "Fix IME handling"
```

### stable ID를 입력 키로 사용한다

장부 입력에는 `key`가 아니라 stable `id`를 사용한다.

허용:

```text
--ledger-set L001=0
```

기본 입력으로 허용하지 않음:

```text
--ledger-set state_flags_added=0
```

`key`는 사람이 읽기 좋은 이름이어서 이후 변경될 수 있지만, `id`는 과거 장부와 현재 명세를 연결하는 안정적인 식별자이기 때문이다.

`--ledger-requirements`, `spec list`, `spec show` 등의 출력에서는 항상 ID와 key를 함께 보여준다.

---

## 6. 정상적인 AI 커밋 흐름

권장 흐름은 다음과 같다.

```text
AI가 작업 완료
    ↓
git ledger commit --ledger-requirements
    ↓
현재 장부 필드, 질문, 타입, 호출 예시 확인
    ↓
AI가 현재 변경을 검토하고 모든 값을 결정
    ↓
git ledger commit --ledger-set ... -- <git commit arguments...>
    ↓
장부 명세 및 값 validation
    ↓
git commit <git commit arguments...>
    ↓
Git Note 기록
```

예:

```bash
git ledger commit --ledger-requirements
```

출력을 확인한 뒤:

```bash
git ledger commit \
  --ledger-set L001=0 \
  --ledger-set L002=passed \
  -- \
  -m "Fix IME handling"
```

---

## 7. `--ledger-requirements`를 생략한 경우의 feedback

`--ledger-requirements`는 권장되는 사전 확인 단계지만, 이를 생략했다고 해서 사용자가 막다른 상태에 빠져서는 안 된다.

AI가 바로 다음처럼 실행할 수 있다.

```bash
git ledger commit -- -m "Fix IME handling"
```

필수 필드가 빠져 있다면 커밋을 생성하지 않고 누락된 항목을 모두 출력한다.

예:

```text
Commit rejected.

Missing required ledger entries:

L001  state_flags_added
Type: integer >= 0
Question:
  이번 변경에서 추가한 상태 플래그 수는? 없으면 0.

L002  typecheck
Type: enum [passed, failed, not-run]
Question:
  Typecheck를 수행했는가?

Retry with:

  git ledger commit \
    --ledger-set L001=<integer> \
    --ledger-set L002=<passed|failed|not-run> \
    -- \
    -m "Fix IME handling"
```

즉 Agent Ledger는 두 단계의 UX를 함께 제공한다.

```text
사전 안내
--ledger-requirements
→ 정상 경로

사후 feedback
필수 항목 누락 시 commit reject
→ 누락했을 때의 복구 경로
```

---

## 8. `--help`와 `--ledger-requirements`의 역할 구분

두 옵션은 목적이 다르다.

### `--help`

CLI 자체의 고정된 사용법을 설명한다.

```bash
git ledger commit --help
```

예:

```text
Usage:
  git ledger commit [ledger options] -- [git commit arguments...]

Ledger options:
  --ledger-requirements
  --ledger-set <id>=<value>
```

### `--ledger-requirements`

현재 저장소의 동적인 장부 명세를 설명한다.

```bash
git ledger commit --ledger-requirements
```

```text
--help
→ Agent Ledger의 고정된 CLI 계약은 무엇인가?

--ledger-requirements
→ 이 프로젝트에서 이번 커밋 전에 무엇에 답해야 하는가?
```

---

## 9. AGENTS.md에 들어갈 최소 가이드

Agent instruction에는 동적으로 변하는 장부 필드 목록을 복제하지 않는다.

기본 가이드는 다음 수준을 목표로 한다.

```text
When creating a commit:

1. Run `git ledger commit --ledger-requirements` to inspect the current required ledger entries.
2. Review the current change and explicitly answer every required entry.
3. Commit with:

   git ledger commit \
     --ledger-set <id>=<value> ... \
     -- \
     <git commit arguments...>

Do not invoke `git commit` directly.
Do not use a mechanism to bypass the Agent Ledger commit path.
Zero, false, not-run, and similar values must be explicitly submitted when applicable.
```

AGENTS.md는 **변하지 않는 workflow**만 정의하고, 현재 장부 명세는 Agent Ledger가 실행 시 제공한다.

---

## 10. 확정된 사항

현재까지 CLI UX에서 다음을 확정한다.

1. Agent Ledger는 Git 외부 명령으로 제공하고 사용자는 `git ledger ...` 형태로 호출한다.
2. 커밋 명령은 `git ledger commit`이다.
3. Agent Ledger 전용 옵션은 `--ledger-*` namespace를 사용한다.
4. `--` 앞은 Agent Ledger 영역, `--` 뒤는 native `git commit` 영역이다.
5. `--` 뒤의 모든 인자는 Agent Ledger가 해석하지 않고 원형 그대로 `git commit`에 전달한다.
6. 커밋 전에 `--ledger-requirements`로 현재 필수 장부 항목을 조회할 수 있다.
7. 실제 장부 값은 반복 가능한 `--ledger-set <id>=<value>` 형식으로 전달한다.
8. 장부 값의 입력 키는 mutable한 `key`가 아니라 stable `id`를 사용한다.
9. `--ledger-requirements`를 생략해도 누락된 모든 필드를 한 번에 출력하고 재시도 방법을 제공한다.
10. `--help`는 고정 CLI 사용법, `--ledger-requirements`는 프로젝트별 동적 장부 명세를 설명한다.
11. 커밋 메시지, amend, signoff, author, editor 등 Git commit 자체의 UX는 Agent Ledger가 재정의하지 않는다.

---

## 11. 장부 명세 관리 입력 UX

장부 명세를 추가·수정할 때는 v1에서 **CLI flag 입력을 기본 방식**으로 사용한다.

별도 JSON 파일을 만들어 입력하는 방식은 v1에 포함하지 않는다. 실제 사용에서 긴 입력이나 자동 생성 요구가 확인되면 이후 `--from-file` 같은 방식을 추가할 수 있다.

### 항목 추가

기본 형태:

```bash
git ledger spec add \
  --key state_flags_added \
  --type integer \
  --min 0 \
  --description "실행 중 값이 변하며 동작 상태를 표현하는 상태 필드의 추가 개수" \
  --question "이번 변경에서 추가한 상태 플래그 수는? 없으면 0." \
  --audit-aggregation sum \
  --audit-threshold 30
```

원칙:

- stable ID는 CLI가 자동 생성한다.
- 새 항목은 `active` 상태로 생성한다.
- v1에서는 모든 `active` 항목이 항상 제출 필수다. `required` 속성이나 `--required`, `--no-required` 옵션을 두지 않는다.
- boolean 성격의 CLI 옵션이 필요한 경우 일반적인 CLI 관례처럼 flag의 존재 여부로 `true`를 표현한다.
- JSON 파일 기반 입력은 v1 이후 필요성이 확인되면 추가한다.

### 항목 수정

기본 형태:

```bash
git ledger spec update L001 \
  --question "이번 변경에서 새로 추가한 상태 플래그 수는? 없으면 0."
```

수정할 속성만 flag로 전달한다. 전달하지 않은 속성은 기존 값을 유지한다.

Lifecycle 변경은 일반 `update`로 처리하지 않고 전용 명령을 사용한다.

### 항목 폐기

사용 이력이 있어도 더 이상 해당 관점을 관찰하지 않기로 했다면 항목을 deprecated할 수 있다.

폐기 이유는 필수 입력이다.

```bash
git ledger spec deprecate L001 \
  --reason "상태 플래그 개수 대신 상태 전이 모델 자체를 감사하기로 했기 때문"
```

성공하면 해당 항목은:

- `status: deprecated`가 된다.
- `deprecation_reason`에 입력한 이유가 저장된다.
- 이후 새 커밋에서는 값을 요구하지 않는다.
- 과거 장부를 해석하기 위한 정의와 ID는 유지된다.

`--reason`이 없거나 빈 문자열이면 deprecate를 거절한다.

### 항목 복구

deprecated 항목은 다시 관찰할 필요가 생기면 active로 복구할 수 있다.

```bash
git ledger spec restore L001
```

복구하면:

- `status: active`가 된다.
- 새 커밋에서 다시 해당 항목의 값을 반드시 요구한다.
- 현재 명세의 `deprecation_reason`은 제거한다.
- stable ID는 그대로 유지한다.

따라서 deprecated는 영구 삭제가 아니라 **관찰을 일시적으로 중단한 lifecycle 상태**다.

### 항목 삭제

아직 실제 장부 기록에서 사용된 적이 없는 항목만 물리 삭제할 수 있다.

```bash
git ledger spec remove L001
```

사용 이력이 있으면 삭제를 거절하고 `deprecate`를 안내한다.

### 조회와 검증

```bash
git ledger spec list
git ledger spec show L001
git ledger spec validate
```

`list`와 `show`에서는 active/deprecated 상태를 명확하게 보여주고, deprecated 항목은 `deprecation_reason`도 함께 표시한다.

---

## 12. 다음에 결정할 CLI 사항

다음 항목은 아직 확정하지 않았다.

- 문자열/특수문자를 포함한 `--ledger-set` 값의 quoting 규칙
- 중복된 `--ledger-set L001=...`이 들어왔을 때 처리 방식
- 모든 validation 오류를 한 번에 반환하는 구체적인 형식
- 성공 시 출력 형식
- `spec add/update/deprecate/restore/remove/...`의 구체적인 입력 UX
