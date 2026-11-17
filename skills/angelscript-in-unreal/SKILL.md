---
name: angelscript-in-unreal
description: Write and diagnose Unreal AngelScript using the target engine's syntax, bindings, reflection, lifecycle, delegates, containers, GAS and Enhanced Input extensions. Use for .as gameplay code and script reload issues; exclude unrelated scripting languages.
---

# Unreal Engine의 AngelScript

대상 프로젝트의 지침, `.uproject`, 엔진 위치, 활성화된 플러그인부터 확인하세요. EngineAssociation이 비어 있으면 소스 엔진일 수 있습니다. 런처 설치본이나 고정된 형제 디렉터리를 가정하지 마세요.

## 실제 API 찾기

1. 프로젝트의 `Script/`와 주변의 동작하는 `.as` 구현부터 검색합니다.
2. 설치된 AngelScript 플러그인의 바인딩, mixin, 확장을 검색합니다. 대소문자나 디렉터리 구조를 가정하지 말고 플러그인 descriptor를 찾으세요. 일부 빌드에서는 선택적 GAS·Enhanced Input 통합이 별도 플러그인으로 제공됩니다.
3. 대상 엔진과 플러그인의 Blueprint-callable C++ 선언을 확인합니다. Blueprint 노출 여부는 좋은 출발점이지만, 바인딩 제외, mixin, 오버로드, 숨겨진 world-context 인자, 엔진 버전에 따라 AngelScript에서 호출 가능한 시그니처가 달라질 수 있습니다.
4. 로컬 소스로 해결되지 않는 실제 공백이 있을 때만 버전에 맞는 공식 문서를 참고합니다. 패키지에 없는 로컬 문서나 예제 폴더를 지어내지 마세요.

넓은 검색에서는 generated/cache 트리를 제외합니다. 특정 generated 바인딩이나 stub 파일은 그것이 해당 정보의 원본일 때만 검색하세요.

## 작업별 참조 문서

작업에 해당하는 문서를 읽고, 예제를 적용하기 전에 대상 엔진에 설치된 바인딩을 확인하세요.

| 작업 | 참조 문서 |
| --- | --- |
| 객체/값 문법, 숫자 폭, 구조체 복사와 컨테이너 | [문법과 값](references/syntax-values.ko-KR.md) |
| Reflection, 기본 컴포넌트, 생성자/기본값과 서브시스템 | [Reflection과 생명주기](references/reflection-lifecycle.ko-KR.md) |
| 단일/멀티캐스트 콜백과 타이머 소유권 | [Delegate·event·타이머](references/delegates-events-timers.ko-KR.md) |
| 없는 API/시그니처, 스크립트 이름과 soft/full reload | [API 진단과 리로드](references/api-diagnostics-reload.ko-KR.md) |
| 선택적 AngelScript GAS 확장 | [GAS](references/gas.ko-KR.md) |
| 선택적 Enhanced Input 확장 | [Enhanced Input](references/enhanced-input.ko-KR.md) |
| 복제 프로퍼티, RepNotify, RPC와 권한 검사 | [네트워킹과 복제](references/networking-replication.ko-KR.md) |
| UMG/CommonUI 위젯, BindWidget, 위젯 생성·표시 | [UMG와 CommonUI](references/ui-umg-commonui.ko-KR.md) |
| `#if EDITOR`, 에디터 전용 코드, 메뉴 확장, CallInEditor | [에디터 스크립팅](references/editor-scripting.ko-KR.md) |
| C++ 노출 규칙, 스크립트 이름, mixin, world context, 인터페이스 | [C++ 바인딩과 mixin](references/cpp-binding-mixins.ko-KR.md) |

## 변환과 생명주기

- UObject 참조는 보통 `UObject Value`로 쓰고, 멤버 접근은 `.`, 이름은 `n"Name"`을 사용합니다.
- Delegate는 프로젝트가 지원하는 delegate API와 함수 이름 리터럴로 reflected 콜백에 바인딩합니다. C++ 멤버 함수 포인터를 그대로 옮기지 마세요.
- C++ 함수에 `meta=(ScriptName="...")`가 있으면 AngelScript에서는 그 이름으로 호출합니다. `K2_` 접두사를 떼어 이름을 추측하기 전에 확인하세요(예: `AController::K2_GetPawn` → `GetControlledPawn`).
- cast, 참조, 컨테이너, 수학 문법은 주변 코드를 따릅니다. 모든 C++ 템플릿이나 순수 C++ 함수가 바인딩되어 있다고 가정하지 마세요.
- 네이티브 서브시스템을 상속하기 전에 설치된 스크립트 전용 서브시스템 베이스를 확인합니다. 예를 들어 빌드에 따라 `UScriptGameInstanceSubsystem`과 `Subsystem::GetGameInstanceSubsystem()`이 제공될 수 있습니다.
- 프로젝트의 reflection, 수명, 컴포넌트 규칙을 따릅니다. 콜백 함수와 노출 멤버에는 실제 바인딩이 요구하는 표시를 붙이세요. 스크립트 컴파일러가 부작용 경고를 내면 증감 식을 별도 문장으로 분리합니다.
- `.as` 파일은 직접 편집하고, `.uasset`·`.umap` 파일은 지원되는 에디터 도구를 사용합니다. 가능하면 기존 에디터와 hot reload를 유지하세요.

네이티브 게임 모듈 없이 스크립트가 상태를 소유하는 최소 독립 Actor 예시입니다.

```angelscript
class ASampleCounterActor : AActor
{
	UPROPERTY(DefaultComponent, RootComponent)
	USceneComponent SceneRoot;

	UPROPERTY(EditAnywhere)
	int Counter = 0;

	UFUNCTION(BlueprintCallable)
	int AddToCounter(int Amount)
	{
		Counter += Amount;
		return Counter;
	}
}
```

예시를 응용하기 전에 대상 빌드의 매크로와 타입을 확인하세요.

## 변경 검증

AngelScript만 바꾸는 작업에는 현재 호환되는 에디터 바이너리를 사용합니다. 스크립트 편집만으로는 네이티브 모듈이나 엔진을 다시 빌드할 이유가 되지 않습니다. 바이너리를 불러올 수 없으면 빌드로 확대하기 전에 구체적인 원인을 보고하세요.

컴파일러/hot reload 진단을 읽은 뒤 바뀐 동작을 적절한 수준에서 실행해 봅니다. 컴파일된 소스, 라이브 속성 readback, 저장 후 재로드한 에셋, 게임플레이 동작은 서로 다른 증거입니다. 타임아웃이 나면 변경을 재시도하기 전에 상태를 다시 조회하세요.

작성된 Unit/Integration/Automation 테스트는 프로젝트의 runner와 실행 정책을 따릅니다. 프로젝트가 명시적 테스트 트리거를 요구하더라도 일반 컴파일과 직접 동작 확인까지 금지되는 것은 아닙니다. 그 정책을 우회하려고 등록된 테스트를 다른 진입점으로 실행하지 마세요.

Hazelight `AngelscriptTest` commandlet은 보통 컴파일과 함께 unit test suite도 실행합니다. 컴파일 전용 확인으로 쓰거나 임의의 테스트별 CLI 필터가 동작한다고 가정하지 마세요. 설치된 commandlet의 지원 옵션과 exit code를 확인합니다. World/PIE/latent 시나리오는 unit test 결과만으로는 부족하며 integration 또는 게임플레이 검증이 필요합니다.

별도 검증 프로세스를 띄울 때는 지원되는 범위에서 프로젝트별 설정 격리, 별개 또는 임시 디버거 포트, 원치 않는 원격 서버 비활성화, bind cache 쓰기 방지를 유지합니다. 직접 시작한 프로세스는 기록하고, 관련 없는 에디터를 건드리지 않고 정리하세요.

변경한 파일, 관찰한 컴파일/런타임 결과, 실제로 실행한 테스트, 아직 검증하지 못한 항목을 보고하세요.
