# API 진단과 리로드

함수 누락, 시그니처 불일치, 잘못된 C++ 변환, 리로드 후 반영되지 않는 변경을 다룰 때 읽으세요.

## 실제 호출 계약 찾기

프로젝트의 동작하는 AS 코드, 설치된 플러그인의 바인딩 등록·mixin, reflected C++ 선언 순서로 확인합니다. Blueprint-callable 메서드는 유용한 단서지만 AS에서도 같은 이름·시그니처라는 증거는 아닙니다. `ScriptName`, 이벤트
접두사 변경, 명시적 `.Method("...")` 등록, `ScriptMixin`의 수신 객체 삽입, 숨겨진 world-context 인자, 오버로드, 바인딩 제외를 확인하세요.

네이티브 C++ 템플릿이 자동으로 AS 템플릿이 되지는 않습니다. 예를 들어 확인한 Enhanced Input 확장은 `Get<FVector2D>()`를 그대로 옮기는 대신 `GetAxis2D()`를 노출합니다. 숨겨진 WorldContextObject 역시 보통 현재
스크립트 문맥에서 공급됩니다. 바인딩을 확인하지 않고 인자를 임의로 추가·제거하지 마세요.

| 오류·증상 | 확인할 원인 | 수정과 근거 |
| --- | --- | --- |
| 네이티브 헤더에 있는데 멤버를 찾지 못함 | 스크립트 이름·mixin·플러그인 비활성화·명시적 제외 | 실제 모듈에서 ScriptName과 .Method 검색, 플러그인 활성화 확인 |
| 일치하는 오버로드·콜백 시그니처 없음 | 숫자 폭·참조 지정자·인자 변환이 다름 | C++ 선언만 보지 말고 등록된 전체 시그니처 비교 |
| 서브시스템 Initialize 재정의 거부 | 네이티브 FSubsystemCollectionBase 인자를 복사함 | 스크립트 기반 클래스의 인자 없는 이벤트 사용; reflection 문서 참고 |
| 입력 값에 템플릿 문법을 쓸 수 없음 | C++ 템플릿에 이름이 있는 AS 래퍼가 있음 | 설치된 바인딩의 GetAxis1D/GetAxis2D/GetAxis3D 또는 Get() 사용 |
| 코드 수정 후 기존 객체 값이 과거 상태임 | 소스 리로드·객체 기본값·저장 데이터가 서로 다름 | 컴파일 결과·객체 신원·직렬화된 재정의를 각각 확인 |
| 이동 Actor가 계속 장애물을 통과함 | Sweep 비활성화 또는 루트 충돌 설정 부적합 | bSweep과 루트 충돌 확인; bTeleport는 물리 상태 처리를 제어 |

마지막 행을 설명하는 완결된 도우미입니다. Sweep은 루트 컴포넌트를 검사하며, 이 함수가 충돌을 설정하거나 특정 객체의 이동 차단을 보장하지는 않습니다. `bTeleport = false`는 이동에 따라 물리 속도를 갱신하며 sweep을 활성화하는 설정이 아닙니다.

```angelscript
bool MoveWithSweep(AActor Actor, const FVector&in Target)
{
	if (!IsValid(Actor))
	{
		return false;
	}
	FHitResult Hit;
	return Actor.SetActorLocation(Target, bSweep = true, SweepHitResult = Hit, bTeleport = false);
}
```

## Soft reload, full reload와 재시작

- 메서드 본문 변경은 보통 soft reload가 가능합니다. 실제 컴파일·리로드 진단을 읽고 변경한 메서드를 다시 실행하세요. 소스 표식 변경만으로 동작 갱신이 입증되지는 않습니다.
- 구조체 레이아웃을 포함해 reflected 속성의 레이아웃·타입을 바꾸면 full script reload가 필요할 수 있습니다. 이는 네이티브 재빌드나 무조건적인 에디터 재시작과 다릅니다.
- 확인한 에디터는 PIE 밖에서 full reload를 허용하며 PIE 중에는 필요한 변경을 대기시키거나 거부합니다. PIE를 정상 종료하고 후속 full reload 결과를 확인하세요. 리로드를 강제하려고 dirty 에셋을 버리지 마세요.
- 구체적인 엔진 진단이나 관측 상태가 요구할 때만 재시작하며, 자신이 만든 변경과 사용자 에디터 상태를 보존하세요. AS만 수정한 것 자체는 C++ 빌드의 이유가 아닙니다.
- 변경한 defaults가 기존에 직렬화된 에셋·인스턴스 값을 덮어쓰지는 않을 수 있습니다. 저장·리로드 후 보존은 컴파일·실시간 실행과 별도로 확인해야 합니다.

## 근거 수준과 재현성

**2026-10-22**에 확인한 소스 기준은 UE **5.8.1**, `CompatibleChangelist = 55116800`, 네이티브 플러그인 BuildId `3c6a4263-8d5a-4b20-a676-deccedf5fce2`입니다. 다른 빌드에서는 다시
확인하세요. 버전 표시와 이 참조 문서는 모든 환경의 지원을 약속하지 않습니다.

소스 확인, 컴파일 성공, PIE·에디터 직접 동작, 저장·리로드 상태, 작성된 테스트 실행을 구분하세요. 소스에 맞춘 예제의 의도는 기록된 런타임 통과가 아닙니다. 미실행 항목과 시작 오류를 포함한 실제 관측은 [Sandbox
기록](SANDBOX_VALIDATION.ko-KR.md)을 참고하세요. 대상 프로젝트의 테스트 실행 조건을 따르고, 테스트를 실행하는 AngelscriptTest commandlet을 컴파일 전용 우회 수단으로 사용하지 마세요.

엔진 루트 기준 근거 위치:

- `Build/Build.version`과 네이티브 모듈 `.modules` 매니페스트: 버전과 BuildId.
- `Plugins/Angelscript/Source/AngelscriptCode/Private/Binds/Helper_FunctionSignature.h`: `GetScriptNameForFunction`, `GetScriptNamespaceForClass`.
- `Plugins/Angelscript/Source/AngelscriptCode/Private/ClassGenerator/AngelscriptClassGenerator.cpp`: 속성·구조체의 `EReloadRequirement` 판단.
- `Plugins/Angelscript/Source/AngelscriptCode/Private/AngelscriptManager.cpp`: `CheckForHotReload`, `FullReloadSuggested`, `FullReloadRequired`, PIE
  처리.
- `Source/Runtime/Engine/Classes/GameFramework/Actor.h`: `K2_SetActorLocation`과 `ScriptName`/`bSweep`/`bTeleport` 계약.
- `Plugins/AngelscriptEnhancedInput/Source/Private/Binds/Bind_FInputActionValue.cpp`: 이름이 있는 값 접근자.

관련 문서: [reflection·생명주기](reflection-lifecycle.ko-KR.md), [값·참조](syntax-values.ko-KR.md), [Enhanced Input](enhanced-input.ko-KR.md).
