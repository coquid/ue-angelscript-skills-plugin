# UE AngelScript Skills for Claude Code and Codex

[English](README.md)

Hazelight AngelScript를 사용하는 Unreal Engine 프로젝트용 스킬 세 개를 제공합니다.

| 스킬 | 용도 |
| --- | --- |
| `angelscript-in-unreal` | 로컬 바인딩 탐색, 게임플레이 구현, 스크립트 변경 검증 |
| `as-integration-tests` | 프로젝트의 테스트 환경에 맞춘 월드·PIE·latent 테스트 설계 |
| `emmsui` | 대상 프로젝트에 설치된 EmmsUI로 개발용 패널 작성 |

## Codex 설치

[릴리스](https://github.com/coquid/ue-angelscript-skills-plugin/releases)에서 ZIP을 받아 새 `ue-angelscript-distribution` 폴더에 압축을 풉니다. 상위 폴더에서 실행합니다.

```sh
codex plugin marketplace add ./ue-angelscript-distribution
codex plugin add ue-angelscript-skills-for-codex@ue-angelscript-skills-for-codex
```

새 프로젝트 채팅에서 `$angelscript-in-unreal`로 기존 바인딩을 찾아보세요. 세 스킬은 프로젝트 소스를 직접 읽습니다. 실행 중인 에디터도 조작하려면 [Unreal Engine MCP Skills](https://github.com/coquid/unreal-engine-skills-for-codex-plugin)를 함께 설치하세요.

## Claude Code 설치

GitHub 저장소를 마켓플레이스로 등록하고 플러그인을 설치합니다.

```sh
claude plugin marketplace add coquid/ue-angelscript-skills-plugin
claude plugin install ue-angelscript-skills@coquid-as-skills
```

새 Claude Code 세션에서 `/ue-angelscript-skills:angelscript-in-unreal`을 입력하세요. 두 클라이언트는 같은 `skills/` 디렉터리를 읽습니다. ZIP으로 설치하려면 첫 명령의 저장소 이름을 압축을 푼 폴더의 상대 경로(예: `./ue-angelscript-distribution`)로 바꾸세요.

GitHub에서 설치한 버전은 Claude Code의 `/plugin`에서 해당 마켓플레이스와 플러그인을 업데이트하세요.

## 통합판에서 이전

`angelscript-unreal-engine-skills-for-codex@angelscript-unreal-engine-skills-for-codex`가 설치되어 있다면 기존 플러그인을 삭제하고 프로젝트별 설정에 남은 해당 활성화 항목을 정리한 뒤 원하는 분리판을 설치하세요. 기존 에디터 MCP 연결 설정은 유지하세요. 세 AngelScript 스킬은 이전과 같은 이름으로 호출합니다.

## 개발

Python 3.11 이상 가상환경에서 실행합니다.

```sh
python -m pip install -r requirements-dev.txt
python scripts/validate_plugin.py
python -m unittest discover -s tests -v
python scripts/package_plugin.py
```

배포 ZIP은 `dist/`에 생성됩니다. [검증 절차](docs/VALIDATION.ko-KR.md) · [출처와 분리 기준](UPSTREAM.ko-KR.md) · [MIT 라이선스](LICENSE)
