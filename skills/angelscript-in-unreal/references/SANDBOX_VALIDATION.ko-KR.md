# 통합판의 과거 검증 기록

아래 내용은 2026-10-22 통합판 v0.1.1의 AngelScript 참조 문서에 대해 관측한 결과입니다. 출처는 `as-ue-skills` 커밋 `385c97b`입니다. 이전 실행 결과는 [v0.1.0 샌드박스 기록](https://github.com/coquid/unreal-engine-skills-for-codex-plugin/blob/9937d3deeb10e362a7443d849048a9bcd4da7605/docs/SANDBOX_VALIDATION.ko-KR.md)에 있습니다. 독립 플러그인의 현재 [설치 안내](../../../README.ko-KR.md)와 [검증 절차](../../../docs/VALIDATION.ko-KR.md)는 별도로 제공합니다.

## 버전 0.1.1 — AngelScript 참조 문서 (2026-10-22)

이 절은 위의 과거 0.1.0 결과를 보존하면서 0.1.1의 관측을 추가합니다. 확인한 엔진은 UE 5.8.1, CompatibleChangelist 55116800,
네이티브 BuildId `3c6a4263-8d5a-4b20-a676-deccedf5fce2`입니다. 독립 프로젝트에 `AngelscriptGAS`와 `AngelscriptEnhancedInput`을 활성화했고
일치하는 기존 바이너리를 사용했습니다. 새 네이티브 모듈이나 네이티브 빌드는 추가하지 않았습니다.

### 소스 및 패키지 검사

기존 AngelScript 스킬에서 영문·한국어 참조 문서 6쌍으로 안내합니다. 문법, 리플렉션, delegate·타이머, 리로드, GAS, Enhanced Input의 계약에 대해
엔진 기준 상대 경로와 심볼을 기록했습니다. 독립 평가자는 정답과 Sandbox 데모를 보지 않고 오류 사례 세 가지를 받았습니다. 필요한 한국어 참조를 선택하고
설치된 엔진 바인딩과 대조하여 복사·참조·delegate 오류, Ability 수명, 입력 시그니처에 맞는 수정을 제안했습니다. 이는 소스 검토이며 실행 테스트가 아닙니다.

평가 결과를 반영해 GAS 생명주기 근거, ASC·Avatar 및 콜백 수명 조건, Enhanced Input 기본 인자 변환 경로를 보완했습니다. 외부 스킬 본문,
엔진 구현 소스, 특정 프로젝트 게임플레이 코드와 로컬 의존성은 가져오지 않았습니다.

패키지 validator와 skill-creator의 `quick_validate.py`가 통과했습니다. 이 Windows 호스트에서는 Python UTF-8 모드가 필요했으며, 앞선 cp949 실패는
호출 시 인코딩 문제였습니다. 언어별 6쌍의 AS 코드 블록 8개가 동일합니다. 상대 링크, manifest 버전, 스킬 6개와 ZIP 포함 범위를 별도로 검사했습니다.
고정 Idris 기준의 UE 스킬 3개, `as-integration-tests`, `emmsui`는 변경하지 않았습니다.

### 컴파일

참조 문서의 AS 코드 블록 8개를 외부 프로젝트의 `Script/Skill011/ReferenceExamples.as`에 그대로 넣었습니다. 최종 컴파일과 스크립트 리로드는
`02:55:41.928 UTC`에 완료됐으며 SHA256은 `7643ec53db07a5f96f76e7e9cb35ce59e0cc46b293946cae7a17be0c9ab5f7f3`입니다.
관측용 액터 세 개도 컴파일했습니다. 이 파일들은 플러그인 배포물 밖에 있습니다.

초기 실패도 로그에 보존했습니다. 입력 옵션 인자 누락·잘못된 값 접근, 이 포크가 const 참조로 취급하는 값 타입 인자 수정, 런타임 입력 컴포넌트 null 접근이
있었습니다. 최종 참조는 `FModifyContextOptions()`를 명시하고 실제 getter를 사용하며, 구조체 복사본 수정 시 로컬 복사를 만들고 소유 입력 컴포넌트를
런타임에 생성·검사합니다. 앞선 컴포넌트 null의 정확한 생명주기 원인은 확정하지 않았으며 DefaultComponent 전체에 대한 금지 규칙으로 만들지 않았습니다.

### 직접 에디터·PIE 관측

공식 native MCP는 PID·생성 시각·프로젝트 명령줄·엔진 실행 파일을 확인한 뒤 포트 `58241`로 연결했습니다. 조정자가 실행한 렌더링 에디터 PID는 `62408`입니다.
직접 작성한 관측 액터와 수동 등록한 로컬 Toolset으로 다음 동작을 확인했습니다. 해당 probe의 런타임 증거이며 문서의 모든 줄을 각각 실행했다는 뜻은 아닙니다.

| 영역 | 실제 관측 |
| --- | --- |
| 구조체·맵 값 | 로컬 복사본은 99, 맵 저장값은 10 유지, 저장 참조로 수정하면 25 |
| 초기화 순서 | 생성자에서 10 관측, 생성된 defaults에서 20 관측, 에디터 상태 30, BeginPlay에서 30 관측 후 40 |
| Delegate·event | 단일 호출 합계 2, multicast 합계 3, 해제 후 추가 Broadcast의 7은 반영되지 않음 |
| 타이머 | 일회 타이머 1회 실행, 반복 타이머 증가, 일시정지 중 84 유지, 재개 후 증가, 중지 후 147 유지 |
| GAS 신원 | ASC Owner·Avatar가 로컬 액터와 일치, 초기화 콜백 1회, 초기 Energy 100 |
| GAS 수명 | Activate 반환 후 활성 상태 유지, 명시적 End 후 비활성, 재활성화 시 동일 인스턴스 재사용 |
| GAS 속성 | Energy 100→75 변경 시 old=100·new=75를 담은 콜백 1회 |
| 입력 값 타입 | Boolean=true, Axis1D=2.5, Axis2D=(3,4), Axis3D=(5,6,7) |
| 입력 전달 | (2,3) 주입이 해당 액션 콜백에 1회 전달, 핸들 해제 후 (7,8) 주입에는 추가 콜백 없음 |
| 입력 정리 | 소유 context·component 등록 해제 및 보관한 입력 참조 정리 |

최초 Input 호출은 `DemoInput.BindAction`에서 null 접근이 났지만 리플렉션 호출 반환값은 true처럼 보였습니다. 실제 속성과 오류 로그를 확인하여 성공으로
처리하지 않았습니다. 런타임 컴포넌트 생성으로 수정한 뒤 설치·전달·해제·정리의 성공을 관측했습니다. 입력 주입은 Enhanced Input 처리 경로를 검증하며
물리 키보드·컨트롤러 입력 경로까지 증명하지 않습니다.

### 리로드 및 저장 상태

메서드 본문만 변경했을 때 `02:53:36.936 UTC`에 `Soft Reload: ASkill011LanguageProbe`가 기록됐습니다. 기존 액터의 메서드 결과는 101→202로 바뀌고
구조체 Count는 77로 유지됐습니다. 이후 반사 필드 `Revision = 2`를 추가하자 구조체와 액터가 full reload됐고 `02:55:16.671 UTC`에 완료됐습니다.
액터 경로를 다시 해석해 Count=77, Revision=2, LifecycleStage=30, 메서드 반환 202를 확인했습니다. 에디터 PID는 동일했으며 이 full script reload에는
에디터 재시작이나 네이티브 빌드가 필요하지 않았습니다. 네이티브 포인터 동일성까지 주장하지 않습니다.

소유한 `/Game/CodexSkill011/RuntimeValidation` 맵만 저장했습니다. Entry 맵을 로드한 뒤 소유 맵을 다시 열어도 기록한 필드 값이 보존됐습니다.
맵 저장·재로드와 이전 버전의 별도 프로세스 재시작 검증은 다른 증거이며, 이번에 새 프로세스 재시작을 확인했다고 주장하지 않습니다.

### 설치 스킬 발견 및 남은 경계

새 독립 Codex 채팅은 파일을 읽기 전에 설치 캐시 0.1.1의 스킬 6개를 제공받았습니다. 두 manifest와 참조 파일 12개에 접근할 수 있었고 언어별 코드 블록
8개도 동일했습니다. 문법·GAS·Enhanced Input 요청에서 필요한 한국어 참조를 선택하고 실제 엔진 소스를 확인했습니다. native MCP 프로젝트 신원 조회도
성공했습니다. 이 읽기 전용 평가는 예제를 실행하지 않았습니다. 첫 런타임 세션은 처음에 0.1.0을 받았고 후속 리로드 턴에서 0.1.1을 받았습니다.
각각의 관측을 구분합니다.

상세 증거는 독립 프로젝트의 `Saved/SkillValidation011/`에만 보관합니다. `independent-skill-evaluation.json`, `compilation-validation-final.json`,
`runtime-validation.*`, `runtime-followup.*`, `reload-validation.*`, `installed-skill-validation.*`를 참고합니다.
최초 Input 실패 보고서는 성공한 후속 결과와 별도로 보존했습니다. 로그·예제·맵·로컬 네이티브 플러그인은 ZIP에 포함하지 않습니다.

멀티플레이 권한·예측·복제, 물리 장치 입력, 다른 엔진 빌드, 모든 리로드·레이아웃 변경 조합은 미검증입니다. 이번 문서 릴리스에서 EmmsUI와 기존
Toolset·Agent Skill 데모는 변경하거나 재검증하지 않았으며 과거 증거를 위에 보존했습니다. 작성된 Unit·Integration·Automation 또는 Python unittest
테스트 모음은 호출하지 않았습니다. **Unit/Integration Test 미실행 (!test 요청 전).**

### 정리

최종 조회 후 PIE는 종료되어 있었고 InputSettings CDO 원래 클래스가 복원됐으며 dirty 패키지는 없었습니다. DefaultInput.ini를 만들거나 SaveConfig를
호출하지 않았습니다. runner는 `02:58:42 UTC`에 자신이 실행한 에디터를 정상 종료했습니다. 별도 프로세스·TCP 조회에서도 PID 62408 부재와
포트 58241의 listener 부재를 확인했으며 강제 종료하지 않았습니다. 외부 증거 폴더의 `cleanup-final.json`, `editor-state.json`에 기록했습니다.
runner의 일반 필드 `editor_behavior=not_checked`는 생명주기 도우미 자체의 범위이며, 런타임 관측은 위의 별도 보고서에 있습니다.
