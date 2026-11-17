# UE AngelScript 스킬 검증

[English](VALIDATION.md)

1. [README](../README.ko-KR.md)의 개발 명령을 실행합니다. 패키지 테스트는 ZIP을 두 번 생성해 해시를 비교하고 스킬 세 개와 압축 해제된 매니페스트를 검사합니다.
2. 에디터 MCP 연결이 없는 새 Codex 환경에 ZIP을 설치하고 세 스킬이 표시되는지 확인합니다. UE MCP 플러그인을 추가한 환경에서는 서로 다른 여섯 스킬을 확인합니다.
3. AngelScript 프로젝트에서 로컬 바인딩을 찾아 작은 게임플레이 변경을 컴파일하고 동작을 확인합니다. 월드·PIE 테스트에는 프로젝트의 테스트 실행기를 사용합니다. EmmsUI를 사용하는 프로젝트에서는 렌더링된 에디터에서 컨트롤을 조작해 확인합니다.
4. 엔진 빌드, 설치 버전, 실제 수행한 검사를 기록합니다. 통합판 v0.1.0의 컴파일·핫 리로드·UI 관측은 [당시 검증 기록](https://github.com/coquid/unreal-engine-skills-for-codex-plugin/tree/9937d3deeb10e362a7443d849048a9bcd4da7605/docs/SANDBOX_VALIDATION.ko-KR.md)에 있습니다.

## Claude Code에서 확인

[설치 안내](../README.ko-KR.md)에 따라 설치한 뒤 새 세션을 엽니다. `/ue-angelscript-skills:`를 입력해 `angelscript-in-unreal`, `as-integration-tests`, `emmsui`가 표시되는지 확인합니다. 첫 번째 스킬로 로컬 바인딩 하나를 찾아 한국어 참조 문서를 읽을 수 있는지 확인합니다.

## 분리판 검증 결과

2026-11-17 Windows의 격리된 Codex 환경에 독립판 v0.1.0 ZIP을 설치했습니다. MCP 엔드포인트 설정 없이 설치된 스킬 세 개를 조회했고, UE MCP 플러그인을 함께 설치한 환경에서는 서로 다른 스킬 여섯 개를 확인했습니다. 이번 검증은 설치와 스킬 탐색을 대상으로 했습니다. 게임플레이와 UI 검증은 위의 통합판 당시 기록에 있습니다.
