# Agent Ledger CLI UX

> 상태: Draft
>
> 이 문서는 Agent Ledger의 CLI 사용 흐름과 입력 규칙을 정의한다. 현재 단계에서는 특히 AI Coding Agent가 장부 항목을 미리 기억하지 않아도 안정적으로 커밋할 수 있는 UX를 우선한다.

## 1. 기본 원칙

Agent Ledger의 CLI는 두 가지를 분리한다.

1. **고정된 사용법**: AI가 미리 알고 있어야 하는 최소한의 명령 구조
2. **변하는 장부 명세**: 프로젝트마다 달라지고 시간이 지나며 추가·수정·폐기되는 실제 장부 항목

AI가 기억해야 하는 것은 장부 항목 전체가 아니라 다음 정도여야 한다.

```text
커밋 전에 현재 요구사항을 조회한다.
커밋은 agent-ledger를 통해 수행한다.
```

장부의 구체적인 필드, 타입, 질문, 허용 값은 실행 시 Agent Ledger가 알려준다.

---

## 2. 현재 요구사항 조회

커밋 전에 현재 프로젝트가 요구하는 장부 항목을 조회할 수 있어야 한다.

기본 명령은 다음과 같이 한다.

```bash
agent-ledger commit --requirements
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

  agent-ledger commit -m "<message>" \
    --set L001=<integer> \
    --set L002=<passed|failed|not-run>
```

`--requirements`의 목적은 AI가 `agent-ledger.json`을 직접 읽거나 현재 장부 항목을 미리 기억할 필요를 없애는 것이다.

---

## 3. 장부 값 전달 문법

장부 값은 반복 가능한 `--set` 옵션으로 전달한다.

기본 형식은 다음과 같다.

```text
--set <field-id>=<value>
```

예:

```bash
agent-ledger commit \
  -m "Fix IME handling" \
  --set L001=0 \
  --set L002=passed \
  --set L003=true
```

### stable ID를 입력 키로 사용한다

장부 입력에는 `key`가 아니라 stable `id`를 사용한다.

허용:

```text
--set L001=0
```

기본 입력으로 허용하지 않음:

```text
--set state_flags_added=0
```

이유는 `key`는 사람이 읽기 좋은 이름이어서 이후 변경될 수 있지만, `id`는 과거 장부와 현재 명세를 연결하는 안정적인 식별자이기 때문이다.

사람과 AI가 ID의 의미를 이해할 수 있도록 `--requirements`, `spec list`, `spec show` 등의 출력에서는 항상 ID와 key를 함께 보여준다.

예:

```text
L001  state_flags_added
L002  typecheck
```

---

## 4. 정상적인 AI 커밋 흐름

권장 흐름은 다음과 같다.

```text
AI가 작업 완료
    ↓
agent-ledger commit --requirements
    ↓
현재 장부 필드, 질문, 타입, 호출 예시 확인
    ↓
AI가 현재 변경을 검토하고 모든 값을 결정
    ↓
agent-ledger commit -m ... --set ...
    ↓
명세 및 값 validation
    ↓
Git commit
    ↓
Git Note 기록
```

예:

```bash
agent-ledger commit --requirements
```

출력을 확인한 뒤:

```bash
agent-ledger commit \
  -m "Fix IME handling" \
  --set L001=0 \
  --set L002=passed
```

---

## 5. `--requirements`를 생략한 경우의 feedback

`--requirements`는 권장되는 사전 확인 단계지만, 이를 생략했다고 해서 사용자가 막다른 상태에 빠져서는 안 된다.

AI가 바로 다음처럼 실행할 수 있다.

```bash
agent-ledger commit -m "Fix IME handling"
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

  agent-ledger commit -m "Fix IME handling" \
    --set L001=<integer> \
    --set L002=<passed|failed|not-run>
```

즉 Agent Ledger는 두 단계의 UX를 함께 제공한다.

```text
사전 안내
agent-ledger commit --requirements
→ 정상 경로

사후 feedback
필수 항목 누락 시 commit reject
→ 누락했을 때의 복구 경로
```

---

## 6. `--help`와 `--requirements`의 역할 구분

두 명령은 목적이 다르다.

### `--help`

CLI 자체의 고정된 사용법을 설명한다.

```bash
agent-ledger commit --help
```

예:

```text
Usage:
  agent-ledger commit [options]

Options:
  -m, --message <text>
  --set <id>=<value>
  --requirements
```

### `--requirements`

현재 저장소의 동적인 장부 명세를 설명한다.

```bash
agent-ledger commit --requirements
```

즉 다음과 같이 구분한다.

```text
--help
→ Agent Ledger라는 프로그램을 어떻게 사용하는가?

--requirements
→ 이 프로젝트에서 이번 커밋 전에 무엇에 답해야 하는가?
```

---

## 7. AGENTS.md에 들어갈 최소 가이드

Agent instruction에는 동적으로 변하는 장부 필드 목록을 복제하지 않는다.

기본 가이드는 다음 수준을 목표로 한다.

```text
When creating a commit:

1. Run `agent-ledger commit --requirements` to inspect the current required ledger entries.
2. Review the current change and explicitly answer every required entry.
3. Commit with `agent-ledger commit -m "<message>" --set <id>=<value> ...`.

Do not invoke `git commit` directly.
Do not use `--no-verify` or another mechanism to bypass this process.
Zero, false, not-run, and similar values must be explicitly submitted when applicable.
```

이 구조에서 AGENTS.md는 **변하지 않는 workflow**만 정의하고, 현재 장부 명세는 Agent Ledger가 실행 시 제공한다.

---

## 8. 확정된 사항

현재까지 CLI UX에서 다음을 확정한다.

1. 커밋 전에 `agent-ledger commit --requirements`로 현재 필수 장부 항목을 조회할 수 있다.
2. `--requirements`는 각 항목의 ID, key, 타입/제약, 질문, 호출 예시를 출력한다.
3. 실제 장부 값은 반복 가능한 `--set <id>=<value>` 형식으로 전달한다.
4. 장부 값의 입력 키는 mutable한 `key`가 아니라 stable `id`를 사용한다.
5. `--requirements`를 생략해도 누락된 모든 필드를 한 번에 출력하고 재시도 방법을 제공한다.
6. `--help`는 고정 CLI 사용법, `--requirements`는 프로젝트별 동적 장부 명세를 설명한다.
7. AGENTS.md에는 개별 장부 필드를 복제하지 않는다.

---

## 9. 다음에 결정할 CLI 사항

다음 항목은 아직 확정하지 않았다.

- 커밋 메시지 입력 방식 (`-m`, 여러 `-m`, editor 지원 범위)
- Git staging 정책
- 문자열/특수문자를 포함한 `--set` 값의 quoting 규칙
- 중복된 `--set L001=...`이 들어왔을 때 처리 방식
- 모든 validation 오류를 한 번에 반환하는 구체적인 형식
- 성공 시 출력 형식
- `spec add/update/...`의 구체적인 입력 UX
