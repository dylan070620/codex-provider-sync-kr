<div align="center">

# codex-provider-sync

### 프로바이더를 바꾼 뒤에도 Codex의 이전 세션이 다시 보이게 하기

[![CI](https://img.shields.io/badge/CI-passing-brightgreen.svg)](#)
[![Platform](https://img.shields.io/badge/platform-cross--platform-blue.svg)](#)
[![Node](https://img.shields.io/badge/node-24%2B-brightgreen.svg)](https://nodejs.org/)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

한국어 | [English](docs/README_EN.md)

</div>

## 해결하는 문제

Codex에서 `model_provider`를 바꾸면, 예전 세션이 Desktop이나 `/resume`에서 보이지 않을 수 있습니다. 보통 세션 파일이 사라진 게 아니라, rollout 파일, SQLite 스레드 테이블, 프로젝트 경로 캐시에 들어 있는 provider / 가시성 메타데이터가 서로 맞지 않아서 생기는 문제입니다.

이 도구는 다음 위치를 동기화합니다:

- `~/.codex/sessions`
- `~/.codex/archived_sessions`
- `~/.codex/state_5.sqlite`
- `.codex-global-state.json` 안의 프로젝트 루트 경로 캐시

## 빠른 사용

Windows 사용자는 Release의 `CodexProviderSync.exe`를 우선 내려받으세요:

1. `CodexProviderSync.exe`를 엽니다.
2. `Refresh`를 클릭합니다.
3. 대상 Provider를 선택합니다.
4. `Execute`를 클릭합니다.

macOS 등 다른 환경에서는 CLI를 사용합니다:

```bash
npm install -g git+<REPO_URL>
codex-provider sync
```

CLI에는 Node.js `24+`가 필요합니다. Node 20/22를 사용하면 `node:sqlite`가 없다는 오류가 날 수 있습니다.

자주 쓰는 CLI 명령은 다음과 같습니다:

```bash
codex-provider status
codex-provider sync
codex-provider sync --provider openai
codex-provider switch apigather
codex-provider restore C:\Users\you\.codex\backups_state\provider-sync\<timestamp>
codex-provider prune-backups --keep 5
```

명령 설명:

- `status`: 현재 provider, rollout, SQLite, 프로젝트 가시성 진단만 확인합니다.
- `sync`: 로그인 상태는 바꾸지 않고, 기존 세션 메타데이터만 현재 provider에 맞게 동기화합니다.
- `switch <provider-id>`: `config.toml`의 최상위 `model_provider`를 바꾼 뒤 동기화를 실행합니다.
- `restore <backup-dir>`: 백업에서 복원합니다. `--no-config`, `--no-db`, `--no-sessions`를 지원합니다.
- `prune-backups --keep <n>`: 이 도구가 만든 오래된 백업만 정리합니다.

## 범위

이 도구는 "기존 세션 가시성"과 관련된 메타데이터만 수정하며, 세션 내용은 건드리지 않습니다.

- 로그인, 인증, `auth.json`, 서드파티 계정 전환 도구는 처리하지 않습니다.
- 메시지 기록, 세션 제목, 대화 내용은 수정하지 않습니다.
- `updated_at`은 바꾸지 않으며, 기록 순서를 바꿔서 Desktop 표시를 억지로 복구하지도 않습니다.
- 옛 세션의 `encrypted_content`를 다른 provider / account로 다시 암호화하지 않습니다.
- `encrypted_content`가 들어 있는 옛 세션은 provider/account를 바꾼 뒤 보통 목록 가시성만 복구할 수 있고, 계속 대화하거나 compact를 시도하면 여전히 `invalid_encrypted_content`가 발생할 수 있습니다.

## Codex Desktop 최근 50개 제한

현재 Codex Desktop의 Recent / 프로젝트 세션 목록에는 상위 표시 제한이 있습니다. 첫 화면에서는 최근 `50`개 세션만 불러옵니다.

영향은 다음과 같습니다:

- CLI `/resume`에서는 보이는 옛 세션이 Desktop의 프로젝트 화면에서는 여전히 "대화 없음"처럼 보일 수 있습니다.
- 옛 프로젝트 세션이 전역 최근 50개 밖으로 밀리면 Desktop 첫 화면에 나오지 않을 수 있습니다.
- `codex-provider-sync status` / GUI Refresh에서는 `first page 0/50`, `ranks 64-77` 같은 진단이 표시되어, 이 문제인지 판단하는 데 도움이 됩니다.

이 도구는 `updated_at`이나 파일 시간을 바꿔서 옛 세션을 억지로 상위 50개 안에 넣지 않습니다. 이 문제는 Codex Desktop 상위에서 프로젝트 단위 페이지네이션을 적용하거나, 첫 화면 로드 수를 늘리거나, 그 수를 열어 두는 방식으로 해결해야 합니다.

## 안전 및 문제 해결

매번 `sync` / `switch` 전에 아래 위치로 백업합니다:

```text
~/.codex/backups/provider-sync/<timestamp>
```

주의할 점:

- `state_5.sqlite`가 사용 중이면 Codex / Codex App / app-server를 종료한 뒤 다시 시도하세요.
- `state_5.sqlite`가 손상되었다면, 도구가 malformed/unreadable을 알리고 동기화를 중단합니다.
- 활성 세션이 rollout 파일을 잠그고 있으면, 그 파일은 건너뛰고 다른 옛 세션을 계속 처리합니다.
- EXE를 더블클릭해도 반응이 없으면, 먼저 압축을 제대로 풀었는지 확인한 뒤 `%AppData%\codex-provider-sync\startup-error.log`를 보거나 PowerShell에서 `./CodexProviderSync.exe`를 실행해 보세요.

GUI 안내는 [README_GUI_ZH.md](docs/README_GUI_ZH.md)에서 확인할 수 있습니다. AI / Agent 안내는 [AGENTS.md](AGENTS.md)를 참고하세요.

## 개발

```bash
git clone <REPO_URL>
cd codex-provider-sync
npm test
dotnet test desktop/CodexProviderSync.Core.Tests/CodexProviderSync.Core.Tests.csproj
pwsh ./scripts/publish-gui.ps1
```

## 라이선스

MIT
