---
name: orchestrating-subagent-development
description: Use when 코딩 요청이 기능 개발, 버그 수정, 리팩터링 또는 여러 컴포넌트에 영향을 주는 변경이며 작업 범위·불확실성·위험도 때문에 Codex subagent 사용 여부를 판단해야 할 때.
---

# 서브에이전트 개발 오케스트레이션

## 목적

개발 요청을 받을 때 subagent를 사용할 가치가 있는지 자동으로 판단한다.
사용자가 매번 "subagent를 사용해", 특정 agent 이름을 지정하거나 agent 실행 순서를 직접 지시할 필요가 없어야 한다.

작고 명확한 변경은 메인 Codex agent가 직접 처리한다.
규모가 있거나 불확실성이 큰 작업에서는 메인 agent가 coordinator 역할을 맡고, 필요에 따라 탐색·설계·구현·테스트·리뷰를 subagent에게 자동으로 분배한다.

## 작업 규모 판정

코드를 수정하기 전에 요청을 `SMALL` 또는 `ORCHESTRATED`로 분류한다.

다음 조건을 **모두** 만족할 때만 `SMALL`로 분류한다.

- 요구된 동작과 수정할 위치가 이미 명확하다.
- 변경 대상이 서로 밀접한 1~2개 파일 정도로 예상된다.
- 원인 분석이 필요하지 않다.
- architecture, public API contract, database schema, authentication, authorization, security, concurrency, payment 또는 destructive data 관련 변경이 아니다.
- 영향 범위가 좁고 검증 방법이 단순하다.
- 메인 agent가 필요한 컨텍스트를 무리 없이 직접 처리할 수 있다.

위 조건 중 하나라도 확실하지 않다면 `ORCHESTRATED`를 우선한다.

다음 중 하나라도 해당하면 즉시 `ORCHESTRATED`로 분류한다.

- 버그의 원인이 불명확하거나 간헐적으로 발생한다.
- 변경 범위가 3개 이상의 파일, 여러 module 또는 frontend/backend/database layer에 걸친다.
- database schema, external/public API, authentication flow, security boundary, concurrency behavior 또는 payment flow가 변경된다.
- 넓은 범위의 refactoring 또는 migration 작업이다.
- 익숙하지 않은 코드 영역이라 영향도 조사가 필요하다.
- 서로 독립적으로 나눌 수 있는 작업 영역이 2개 이상이라 isolated context가 유리하다.

단순히 subagent 기능이 있다는 이유만으로 agent를 호출하지 않는다.
오타 수정, 단순 rename, 명확한 한 파일 수정, 작은 style 변경, 반복적인 기계적 수정은 일반적으로 `SMALL`로 처리한다.

## SMALL 처리 흐름

메인 agent가 직접 처리한다.

- 필요한 범위만 확인한다.
- 최소한의 변경만 수행한다.
- 관련 test/check를 실행한다.
- 실제 diff를 확인한다.
- 변경 내용과 검증 결과를 사용자에게 보고한다.

작은 작업을 굳이 subagent에게 위임하지 않는다.

## ORCHESTRATED 처리 흐름

사용자에게 subagent를 직접 실행하라고 요청하지 않는다.
사용자의 판단이 반드시 필요한 상황이 아니라면 즉시 자동 orchestration을 시작한다.

### 1. Explore

구현 위치, dependency, data flow 또는 영향 범위가 이미 명확하지 않을 때 `explorer`를 사용한다.

Explorer는 기본적으로 read-only다. 코드를 수정하지 않고 다음을 조사한다.

- 관련 파일과 symbol
- call path와 data flow
- 연관된 test
- dependency
- 변경 시 영향 범위와 제약사항

서로 독립적인 영역이라면 여러 read-only explorer를 병렬 실행할 수 있다.

권장 모델: `gpt-5.6-terra`, reasoning `low`.

### 2. 필요한 경우에만 Design

다음과 같이 의미 있는 설계 판단이 필요한 경우에만 `architect`를 사용한다.

- module 간 contract 결정
- schema/API 설계 변경
- migration 전략
- architecture 변경
- 위험도가 높은 구현 계획

일반적인 기능 구현이나 명확한 수정에는 architect를 생략한다.

권장 모델: `gpt-5.6-sol`, reasoning `high`.

### 3. Implement

구현은 가능한 한 구체적인 역할의 agent에게 위임한다.
예: `backend`, `frontend` 또는 일반 implementation worker.

Implementer에게는 다음 정보만 명확하게 제공한다.

- 소유 파일과 수정이 허용된 범위
- 담당 범위
- 구체적인 목표
- 관련 파일과 interface
- 반드시 지켜야 할 제약사항
- 실행할 검증 명령과 성공 기준

현재 작업과 관계없는 긴 대화 기록이나 불필요한 컨텍스트를 전달하지 않는다.

같은 파일을 수정하거나 shared mutable state에 의존하는 implementation agent를 동시에 실행하지 않는다.
명확하게 독립적인 workstream일 때만 병렬 구현을 허용한다.
Coordinator만 agent 분배, 병렬 실행 결정, 파일 소유권 충돌 해결을 수행한다.
소유 파일이 겹치거나 의존 순서가 확정되지 않으면 구현을 병렬화하지 않는다.

권장 모델: `gpt-5.6-terra`, reasoning `medium`.

작업 내용이 완전히 명확하고 반복적인 기계적 수정이라면 `gpt-5.6-luna`와 reasoning `low` 또는 `medium`을 사용할 수 있다.

### 4. Test

구현 후 단순하지 않은 변경은 `tester`를 사용해 검증한다.

Tester의 책임:

- 필요한 test 추가 또는 수정
- targeted test 실행
- 실패 원인 분석
- 요청된 동작과 실제 구현이 일치하는지 확인

Tester가 파일을 수정할 때도 Coordinator가 소유 파일과 범위를 배정한다.

권장 모델: `gpt-5.6-luna`, reasoning `medium`.

테스트 실패 원인이 단순하지 않고 실제 debugging이나 module 간 분석이 필요하면 `gpt-5.6-terra`로 올린다.

### 5. Review

구현을 수행하지 않은 별도의 reviewer가 실제 diff를 검토하도록 한다.

다음을 중점적으로 확인한다.

- 요구사항 누락
- bug 또는 regression 가능성
- 잘못된 assumption
- edge case
- security 문제
- test 누락
- 불필요한 복잡성
- 다른 module에 미치는 부작용

변경 범위가 크거나 위험도가 높으면 `gpt-5.6-sol`, reasoning `high`를 사용한다.
일반적인 저위험 diff는 `gpt-5.6-terra`, reasoning `medium`으로 충분하다.

### 6. Fix Loop

Test 또는 review에서 수정 가능한 문제가 발견되면 사용자에게 조정 역할을 요구하지 않는다.

하나의 finding 묶음에 대해서 자동 수정 루프는 **최대 두 번**만 수행한다.

1. 발견된 문제를 implementer에게 전달한다.
2. 수정한다.
3. 관련 test를 다시 실행한다.
4. 수정된 범위를 다시 review한다.

두 번째 루프 뒤에도 문제가 남으면 더 이상 자동으로 수정하지 않는다. Coordinator는 다음을 포함한 중단 보고를 작성하고 사용자 결정을 요청한다.

- 남은 문제와 추정 원인
- 시도한 변경과 담당 agent
- 실행한 test/review 명령과 결과
- 추가 진행에 필요한 선택 또는 context

첫 번째 루프에서 해결되지 않았을 때는 작업 범위를 더 작게 나누거나 한 단계 높은 모델로 escalation할 수 있다.

다음과 같은 경우에만 사용자에게 질문한다.

- 요구사항끼리 충돌한다.
- destructive한 선택에 승인이 필요하다.
- credential 또는 필수 context가 없다.
- 사용자만 결정할 수 있는 business decision이 필요하다.
- 실제로 작업 진행이 막혀 있다.

### 7. Final Verification

최종 결과에 대한 책임은 항상 메인 agent에게 있다.
Coordinator만 최종 검증과 사용자 보고를 수행한다.
완료했다고 보고하기 전에 다음을 확인한다.

- 통합된 최종 diff를 직접 확인한다.
- 가능한 관련 test/build/lint 명령을 실행한다.
- 여러 subagent의 변경이 서로 충돌하지 않는지 확인한다.
- 사용자가 요청한 요구사항이 빠지지 않았는지 확인한다.
- 실제로 변경한 내용과 검증 결과를 최종 보고한다.

## Handoff 결과 형식

Explorer, architect, tester, reviewer와 implementer는 결과를 다음 형식으로 Coordinator에게 전달한다.

1. **요약과 결론**: 확인하거나 변경한 내용과 결론
2. **근거**: 관련 파일·symbol 또는 실행한 명령과 결과
3. **범위**: 조사·변경·검증한 파일 목록
4. **위험 또는 미해결 항목**: 불확실한 가정, 실패, 후속 위험
5. **다음 단계 권고**: 진행·수정·재검증·중단 중 권장 행동

읽기 전용 역할은 파일을 수정, stage, commit하지 않는다.
보고에 필요한 정보가 부족하면 Coordinator는 작업을 재할당하기 전에 누락 정보를 보완한다.

## Agent 선택 규칙

프로젝트에 이미 정의된 named custom agent와 그 agent의 역할별 instruction을 우선 사용한다.

해당 역할의 custom agent가 없다면 일반 subagent에게 같은 책임을 명확하게 부여하고 적절한 model tier를 직접 지정한다.

역할을 안정적으로 수행할 수 있는 범위에서 가장 가벼운 모델을 사용한다.
다만 지나치게 약한 모델을 사용해 여러 번 재시도하는 것보다, 처음부터 충분한 모델로 적은 turn 안에 끝내는 것을 우선한다.

기본 모델 선택 기준:

| 역할 | 기본 모델 | Reasoning |
|---|---|---|
| Explorer | `gpt-5.6-terra` | `low` |
| Architect | `gpt-5.6-sol` | `high` |
| Backend / Frontend / Implementer | `gpt-5.6-terra` | `medium` |
| 단순 기계적 구현 | `gpt-5.6-luna` | `low` 또는 `medium` |
| Tester | `gpt-5.6-luna` | `medium` |
| 일반 Reviewer | `gpt-5.6-terra` | `medium` |
| 중요/고위험 Reviewer | `gpt-5.6-sol` | `high` |

custom agent 설정에서 별도 모델이 명시되어 있고 해당 역할에 적절하다면 그 설정을 우선할 수 있다.

## 병렬 실행 규칙

독립적인 **investigation은 적극적으로 병렬화**하고, **implementation은 보수적으로 병렬화**한다.

병렬 실행이 안전한 예:

- backend와 frontend를 각각 조사하는 read-only explorer
- 서로 다른 module의 독립적인 test failure 조사
- 서로 다른 파일 영역을 읽기만 하는 영향도 분석

병렬 실행이 위험한 예:

- 두 implementer가 같은 Service를 동시에 수정
- schema 변경이 끝나기 전에 해당 schema에 의존하는 코드를 동시에 구현
- 여러 agent가 shared configuration을 동시에 수정
- 서로 순서 의존성이 있는 작업을 동시에 실행

의존성이 있으면 순차적으로 실행한다.

## 예시

| 사용자 요청 | 분류 | 처리 방식 |
|---|---|---|
| "DTO 필드 이름을 바꾸고 mapper도 같이 수정해줘." | SMALL | 메인 agent가 직접 수정하고 검증한다. |
| "로그인이 가끔 실패하는데 원인을 모르겠어. 찾아서 고쳐줘." | ORCHESTRATED | Explorer/debug 조사 → implementer → tester → reviewer |
| "Controller, Service, DB, React UI까지 수정해서 기능 추가해줘." | ORCHESTRATED | 영향도 조사 → 필요 시 architect → backend/frontend 구현 → test → review |
| "이 버튼 문구만 바꿔줘." | SMALL | subagent 없이 직접 처리한다. |
| "동작은 그대로 유지하면서 주문 pipeline을 리팩터링해줘." | ORCHESTRATED | dependency 조사 → 필요 시 architect → implementer → regression test → reviewer |

## 피해야 할 동작

Orchestration이 필요한 상황에서 사용자에게 다음과 같이 말하지 않는다.

- "backend agent를 실행해 주세요."
- "이제 reviewer를 호출해 주세요."
- "tester agent를 먼저 실행할까요?"
- "subagent를 사용하시겠습니까?"

이 Skill의 목적은 사용자가 평소처럼 개발 요청만 해도 subagent 사용 여부와 실행 순서를 Codex가 자동으로 판단하도록 만드는 것이다.
