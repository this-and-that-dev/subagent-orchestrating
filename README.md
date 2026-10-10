# Subagent Orchestrating

Codex 개발 요청에서 subagent를 언제 사용하고 어떤 역할과 모델로 분배할지 정의한 이식 가능한 운영 패키지다. 작은 변경은 메인 agent가 직접 처리하고, 불확실성·위험도·영향 범위가 큰 요청만 coordinator가 탐색·설계·구현·테스트·리뷰로 나눈다.

## 구성

- `orchestrating-subagent-development/SKILL.md`: 작업 규모 판정, 분배, handoff, fix loop와 최종 검증 규칙
- `*.toml`: 역할별 모델과 책임 경계
- `docs/validation-scenarios.md`: 대표 요청의 기대 routing 결과
- `scripts/validate.sh`: 저장소 계약 검사

## 설치와 활성화

이 저장소를 clone하거나 파일을 복사하는 것만으로 Codex에 자동 활성화되지는 않는다. 사용자 또는 팀의 Codex 설정에서 skill과 custom agent를 등록해야 한다.

1. 사용 중인 Codex 버전에서 multi-agent 기능을 활성화한다.
2. `orchestrating-subagent-development` 디렉터리를 해당 환경의 skill 검색 경로에 복사한다. 일반적인 로컬 설치 예시는 `~/.codex/skills/orchestrating-subagent-development/SKILL.md`다.
3. 역할 TOML을 환경이 지원하는 custom-agent 등록 방식으로 등록한다. agent 설정 경로와 TOML schema는 Codex 버전·배포 환경에 따라 다르므로, 적용 전 현재 환경의 공식 문서를 확인한다.
4. Codex를 다시 시작하거나 설정을 새로고침한 뒤, skill과 역할이 목록에 나타나는지 확인한다.

이 프로젝트는 사용자 머신의 Codex 설정을 변경하거나 자동 설치하지 않는다.

## 역할과 모델

| 역할 | 모델 / reasoning | 책임 |
|---|---|---|
| `explorer` | terra / low | 읽기 전용 영향도 조사 |
| `architect` | sol / high | 설계·위험·작업 분해 제안 |
| `backend`, `frontend`, `implementer` | terra / medium | Coordinator가 지정한 파일 범위의 구현 |
| `tester` | luna / medium | 배정된 테스트 작성·실행·결과 보고 |
| `reviewer` | terra / medium | 일반 변경의 읽기 전용 리뷰 |
| `high-risk-reviewer` | sol / high | 보안·인증·DB·결제 등 고위험 변경 리뷰 |
| `commit` | luna / low | 요청된 파일만 stage·commit |

`gpt-6.1-sol`은 최고 성능, `gpt-6.1-sol`는 성능과 비용의 균형, `gpt-6-luna`는 고처리량 작업을 위한 모델로 사용한다. [공식 모델 가이드](https://developers.openai.com/api/docs/guides/latest-model)

## 운영 규칙

- Coordinator만 agent 분배, 병렬 실행 판단, 파일 소유권 충돌 해결, 최종 검증과 사용자 보고를 수행한다.
- 구현 지시에는 반드시 소유 파일, 허용된 범위, 검증 명령을 포함한다.
- explorer와 reviewer는 읽기 전용이다.
- 같은 finding 묶음의 fix loop는 최대 두 번이다. 이후에도 실패하면 자동 수정을 멈추고 근거와 필요한 사용자 결정을 보고한다.

## 검증

저장소의 역할 파일과 핵심 운영 규칙이 일치하는지 확인한다.

```sh
sh tests/validate-repository.sh
git diff --check
```

대표 요청의 기대 routing은 [검증 시나리오](docs/validation-scenarios.md)에서 확인한다.
