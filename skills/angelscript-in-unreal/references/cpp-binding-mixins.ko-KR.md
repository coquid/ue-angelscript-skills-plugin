# C++를 스크립트에 노출하기: 바인딩, 이름, mixin

스크립트에서 쓸 C++를 작성할 때, C++ 함수나 프로퍼티가 스크립트에 보이지 않을 때, 스크립트 이름이 예상과 다를 때 읽으세요.

## 자동 바인딩 대상

| C++ 항목 | 바인딩되는 경우 | 제외·차단 요인 |
| --- | --- | --- |
| UClass | 제외되지 않으면 항상 | `NotInAngelscript`, `BlueprintType` 없는 `NotBlueprintType` |
| UStruct | `BlueprintType`, Blueprint에서 보이거나 편집·할당 가능한 프로퍼티, 또는 `meta = (ForceAngelscriptBind)` | `NoAutoAngelscriptBind`, `NotInAngelscript` |
| UEnum | 제외되지 않으면 항상 | `NotBlueprintType`, `NotInAngelscript` |
| UFUNCTION | `BlueprintCallable`, `BlueprintPure`, `ScriptCallable` | `NotInAngelscript`, `UsableInAngelscript` 없는 `BlueprintInternalUseOnly`, `CustomThunk`, UINTERFACE에 선언, 스크립트 타입이 없는 매개변수·반환 타입 |
| UPROPERTY | `BlueprintReadOnly`/`ReadWrite`, `BlueprintAssignable`, `Edit*`, `ScriptReadOnly`/`ScriptReadWrite`/`ScriptEditOnly` | `NotInAngelscript`, 정적 배열 |

호출 지정자가 없는 C++ `UFUNCTION()`은 바인딩되지 않습니다. Blueprint에는 노출하지 않고 스크립트에만 노출하려면 `ScriptCallable`을 쓰세요. 이때
C++ 기본 인자도 스크립트 기본값으로 유지됩니다. `BlueprintEvent`와 네트워크 함수는 override 가능한 이벤트로 바인딩됩니다. 지원되지 않는 매개변수
타입이 있으면 오류 없이 함수가 조용히 빠집니다.

편집 전용 접근(`BlueprintReadWrite` 없는 `Edit*`)은 `default` 문, ConstructionScript, 에디터·초기화 컨텍스트에서만 쓸 수 있습니다.
`BlueprintProtected`는 스크립트에서 protected가 됩니다. deprecated callable은 deprecation 메시지와 함께 바인딩되지만, deprecated BlueprintEvent는
바인딩되지 않습니다.

## 스크립트 이름

- 함수: `meta = (ScriptName = "Name")`이 우선합니다. 없으면 `K2_`, `BP_`, `AS_`가 빠지고(이벤트는 `Receive`/`Received_`도) 뺀 이름이 그 클래스의
  다른 Blueprint 노출 함수와 겹치면 원래 이름이 유지됩니다.
- static 함수 namespace: 클래스 `ScriptName` 또는 타입 이름에서 시작해 가장 긴 접두어(`UKismet`, `UBlueprint`)와 접미어(`Statics`, `Library`,
  `BlueprintLibrary`, `BlueprintFunctionLibrary`, `FunctionLibrary`)를 뺍니다. 타입 이름에서 접미어만 뺐다면 앞의 `U`도 뺍니다.
  `UKismetSystemLibrary` → `System::`, `UExampleLibrary` → `Example::`, `UNavigationSystemV1` → `UNavigationSystemV1::`.
  이 목록은 프로젝트 설정이므로 대상 설정을 확인하세요.
- `meta = (ScriptGlobalScope)`는 static 함수를 전역 범위에 둡니다.

## Mixin

첫 매개변수가 오브젝트 포인터나 참조인 static 함수는, 함수에 `meta = (ScriptMethod)`가 있거나 소유 클래스에
`UCLASS(meta = (ScriptMixin = "AActor FVector"))`(공백으로 구분한 스크립트 타입 이름)가 있고 첫 매개변수 타입이 목록에 있으면 그 타입의
메서드가 됩니다. 첫 매개변수는 스크립트 시그니처에서 사라지고, `const` 첫 매개변수는 const 메서드가 됩니다. 값으로 받는 구조체 첫 매개변수는
해당하지 않습니다. `ScriptMixinConstructor`/`ScriptMixinImplicitConstructor`는 생성자를 만들며 Blueprint-callable이 아니라 ScriptCallable이어야
합니다.

## World context

`meta = (WorldContext = "WorldContextObject")`로 지정한 매개변수는 스크립트에서 숨겨지고 현재 world context로 채워집니다. 명시적으로 넘기면
`No matching signatures`로 실패합니다. meta 없이 이름만 `WorldContextObject`인 매개변수는 일반 인자입니다.

Unreal이 스크립트를 호출할 때 context는 멤버 함수면 `this`, world-context 인자가 있는 static 함수면 그 인자, 그 밖에는 null입니다. context는
CDO/archetype이 아니고 world가 있을 때만 쓸 수 있습니다. 그런 context 없이 world-context 함수를 호출하면 에디터에서
`Calling a function that requires WorldContext, but the current object is not in a world.` 예외가 납니다. `OptionalWorldContext` 또는
`CallableWithoutWorldContext`는 이 검사를 건너뜁니다. world 밖의 오브젝트(데이터 에셋, 태스크 오브젝트, CDO)에서는 scope를 여세요:
`FAngelscriptGameThreadScopeWorldContext Ctx(ActorInWorld);`. 생성자와 `default` 문에서는 world-context 호출을 피하세요.

## 인터페이스

확인한 플러그인의 클래스 생성기와 타입 시스템에는 인터페이스 지원이 없습니다. 스크립트 클래스는 C++ UInterface를 구현할 수 없고,
`TScriptInterface` 매개변수·프로퍼티에는 스크립트 타입이 없어 해당 멤버가 빠지며, UINTERFACE에 선언된 callable 함수는 바인딩되지 않습니다.
공용 API는 C++ 기반 클래스, 컴포넌트, static 라이브러리에 두세요.

## 수동 바인딩

reflection으로 표현할 수 없는 함수(와일드카드 `?&out` 매개변수, UHT가 모르는 타입, CustomThunk)는 `AngelscriptCode`에 의존하는 모듈에서 수동
바인딩을 작성합니다: `EOrder`를 지정한 `FAngelscriptBinds::FBind`, `ExistingClass`/`ReferenceClass`/`ValueClass`, `.Method(...)`,
`BindGlobalFunction`, `FNamespace`, 그리고 translation unit이 링크되도록 `AS_FORCE_LINK`. 작성 전에 `Public/AngelscriptBinds.h`와 기존 바인딩
파일을 읽으세요.

## 변경 반영

에디터에서는 시작할 때 한 번 live reflection으로 바인딩을 만듭니다. UFUNCTION, UPROPERTY, USTRUCT, UENUM이나 바인딩 meta를 추가·변경하면 C++를
다시 빌드하고 에디터를 재시작하세요. 스크립트 hot reload로는 반영되지 않고, 플러그인에는 바인딩용 Live Coding 훅이 없습니다.
`ConfigRestartRequired` 설정도 재시작이 필요합니다. 에디터가 아닌 빌드는 `Script/Binds.Cache`를 읽으며, 캐시가 없거나 오래되면 fatal입니다.
에디터 커맨드렛 실행(쿡 포함)은 `-as-skip-write-bind-db`를 주지 않으면 이 캐시를 다시 씁니다.

완결된 최소 예제입니다(`MYGAME_API`는 모듈의 API 매크로로 바꾸세요).

```cpp
UCLASS(meta = (ScriptMixin = "AActor"))
class MYGAME_API UExampleLibrary : public UBlueprintFunctionLibrary
{
	GENERATED_BODY()
public:
	// Namespace function: Example::SquareValue
	UFUNCTION(BlueprintPure, Category = "Example")
	static float SquareValue(float Value);

	// Mixin: const method on AActor
	UFUNCTION(BlueprintPure, Category = "Example")
	static float GetFlatDistanceTo(const AActor* Self, const AActor* Other);

	// Hidden world context; first parameter is not a mixin type, so it stays namespaced
	UFUNCTION(BlueprintCallable, Category = "Example", meta = (WorldContext = "WorldContextObject"))
	static int32 CountActorsWithTag(const UObject* WorldContextObject, FName Tag);
};
```

```angelscript
float Squared = Example::SquareValue(3.0);
float Distance = SomeActor.GetFlatDistanceTo(OtherActor);
int Count = Example::CountActorsWithTag(n"Enemy");

// From an object that is not in a world:
FAngelscriptGameThreadScopeWorldContext Ctx(ActorInWorld);
int ScopedCount = Example::CountActorsWithTag(n"Enemy");
```

## 누락·이름 문제 진단

| 증상 | 가능한 원인 | 조치 |
| --- | --- | --- |
| 새 C++ 멤버가 스크립트에서 안 보임 | C++ 변경 후 에디터를 재시작하지 않음 | 빌드 후 에디터 재시작 |
| 일반 `UFUNCTION()`이 안 보임 | 호출 지정자 없음 | `ScriptCallable`(또는 `BlueprintCallable`) 추가 |
| BlueprintCallable 함수가 조용히 빠짐 | 지원되지 않는 매개변수 타입, CustomThunk, `BlueprintInternalUseOnly`, `NotInAngelscript`, UINTERFACE | 시그니처 변경, `UsableInAngelscript` 추가, 또는 수동 바인딩 |
| world-context 인자를 넘기면 `No matching signatures` | `WorldContext` 매개변수가 숨겨짐 | 인자 제거 |
| `...requires WorldContext, but the current object is not in a world` | CDO, 에셋, 태스크 오브젝트에서 호출 | world 안의 액터로 `FAngelscriptGameThreadScopeWorldContext` 열기 |
| namespace가 예상과 다르거나 `K2_` 이름이 남음 | 접두어·접미어 규칙 또는 이름 충돌 | 클래스·함수 `ScriptName` 지정 |
| 패키지 빌드에서 `Binds.Cache` 관련 fatal | 오래된 바인딩 캐시 | 에디터 커맨드렛 실행이나 쿡으로 다시 생성 |

## 소스와 검증 범위

2026-11-16에 UE 5.8.1, 네이티브 플러그인 BuildId `343c3a9c-ab61-4b28-b709-cd650e59f32e`의 소스로 확인했습니다. 경로는 엔진 루트 기준입니다.
C++ 예제는 확인한 규칙을 따르지만 컴파일하지 않았습니다.

- `Plugins/Angelscript/Source/AngelscriptCode/Private/Binds/Bind_BlueprintType.cpp`: 클래스·함수 바인딩 선택.
- `Plugins/Angelscript/Source/AngelscriptCode/Private/Binds/Bind_BlueprintCallable.cpp`: 함수 제외 조건.
- `Plugins/Angelscript/Source/AngelscriptCode/Private/Binds/Bind_UStruct.cpp`, `Bind_UEnum.cpp`: 구조체·열거형 바인딩 규칙.
- `Plugins/Angelscript/Source/AngelscriptCode/Private/Binds/Helper_FunctionSignature.h`: 스크립트 이름, namespace, mixin, world context.
- `Plugins/Angelscript/Source/AngelscriptCode/Private/Binds/Helper_PropertyBind.h`: 프로퍼티 접근 규칙.
- `Plugins/Angelscript/Source/AngelscriptCode/Public/AngelscriptSettings.h`: namespace 접두어·접미어 목록, world-context 오류 설정.
- `Plugins/Angelscript/Source/AngelscriptCode/Public/AngelscriptManager.h`, `Private/AngelscriptManager.cpp`: world-context scope, `AS_USE_BIND_DB`,
  시작 시 바인딩, 바인딩 캐시 쓰기.
- `Plugins/Angelscript/Source/AngelscriptCode/Private/AngelscriptType.cpp`: 지원되지 않는 `TScriptInterface` 타입.
- `Plugins/Angelscript/Source/AngelscriptCode/Public/AngelscriptBinds.h`: 수동 바인딩 API.
- `Source/Programs/Shared/EpicGames.UHT/Specifiers/UhtFunctionSpecifiers.cs`, `UhtPropertyMemberSpecifiers.cs`: `ScriptCallable`과 스크립트 프로퍼티 지정자.
