# Reflection, 컴포넌트와 생명주기

스크립트 클래스를 정의하거나 엔진 이벤트를 재정의하거나 초기화 위치를 결정할 때 읽으세요.

## 노출과 재정의는 다릅니다

`UPROPERTY()`와 `UFUNCTION()`은 엔진·에디터가 필요로 하는 reflected 멤버를 제공합니다. C++ 지정자를 모두 요구한다고 가정하지 말고 로컬 기본 동작을 확인하세요. 이 포크에서는 일반적으로 단순한 `UFUNCTION()`도
Blueprint에서 호출할 수 있습니다. 의도한 경계를 명확히 해야 할 때 `BlueprintCallable`, `BlueprintReadOnly`, `EditDefaultsOnly`, `NotBlueprintCallable`을 명시하세요.

`UFUNCTION(BlueprintOverride)`는 `BeginPlay` 같은 지원되는 reflected 엔진 이벤트를 구현하며, 임의의 네이티브 가상 함수를 재정의하는 방법이 아닙니다. `BlueprintEvent`는 Blueprint가 재정의할 수 있는
스크립트 구현을 노출합니다. 스크립트 이름 변경을 포함해 실제 부모 이벤트의 이름과 시그니처를 확인하세요. 이름만 비슷한 메서드는 호출되지 않을 수 있습니다.

## 컴포넌트 생성과 초기화 순서

`DefaultComponent`는 컴포넌트를 생성합니다. `RootComponent`는 씬 루트를, `Attach`는 씬 부착 부모를 명시합니다. 이 둘이 항상 필수인 것은 아닙니다. 확인한 빌드에서는 부착 지정을 생략한 씬 기본 컴포넌트가 루트가 없으면 루트가
되고, 있으면 기존 루트에 부착됩니다. 일반 ActorComponent에는 씬 계층이 없습니다. 의미 있는 경우에 의도한 계층을 명시하세요.

확인한 Actor 생성 경로는 네이티브 생성자, 기본 컴포넌트 생성, AS 생성자, 생성된 defaults 순서입니다. 부모 defaults가 자식 defaults보다 먼저 실행됩니다. 따라서 `default`는 AS 생성자의 다른 표기일 뿐인 것이 아닙니다. 나중에
실행되는 default가 생성자 대입을 덮어쓸 수 있습니다. 게임플레이 상호작용과 월드 의존 바인딩은 적절한 런타임 생명주기 이벤트에 두세요. ConstructionScript는 에디터에서 반복 실행될 수 있으므로 재구성을 견디도록 작성합니다.

완결된 독립 Actor입니다. 이후 에셋·인스턴스 재정의가 없다면 BeginPlay까지 의도한 Stage 순서는 1 -> 2 -> 3 -> 4입니다. 저장된 Blueprint·인스턴스 기본값이 최종 관측값을 바꿀 수 있으므로 실제 객체를 확인하세요.

```angelscript
class ADefaultsExample : AActor
{
	UPROPERTY(DefaultComponent, RootComponent)
	USceneComponent SceneRoot;

	UPROPERTY(DefaultComponent, Attach = SceneRoot)
	USceneComponent Marker;
	default Marker.RelativeLocation = FVector(0.0, 0.0, 50.0);

	UPROPERTY()
	int Stage = 1;

	ADefaultsExample()
	{
		Stage = 2;
	}

	default Stage = 3;

	UFUNCTION(BlueprintOverride)
	void BeginPlay()
	{
		Stage += 1;
	}

	UFUNCTION(BlueprintCallable)
	int GetStage() const
	{
		return Stage;
	}
}
```

## 스크립트 서브시스템

대상 빌드에 존재하는 스크립트 전용 기반 클래스를 사용하세요. 네이티브 `Initialize(FSubsystemCollectionBase&)` 시그니처를 스크립트 재정의에 그대로 복사하지 마세요. 확인한 `UScriptWorldSubsystem`은 인자 없는
`Initialize()`와 `Deinitialize()` 이벤트를 노출합니다.

이 바인딩에 맞춘 완결된 서브시스템 선언입니다. 생성 정책과 지원 월드는 여전히 대상 기반 클래스와 프로젝트에 따릅니다.

```angelscript
class UExampleWorldSubsystem : UScriptWorldSubsystem
{
	UFUNCTION(BlueprintOverride)
	void Initialize()
	{
	}

	UFUNCTION(BlueprintOverride)
	void Deinitialize()
	{
	}
}
```

## 생명주기를 바꾸기 전에 진단하기

| 증상 | 확인할 원인 | 수정 |
| --- | --- | --- |
| 재정의가 컴파일되지 않거나 호출되지 않음 | 스크립트 이벤트 이름 변환 없이 네이티브 이름·시그니처를 복사함 | 부모 reflected 이벤트와 스크립트 노출 이름 확인 |
| 생성자에서 지정한 값이 사라짐 | defaults 또는 직렬화된 에셋·인스턴스 값이 나중에 적용됨 | 생성자 작업, 기본 설정, 실행 중 인스턴스 상태 구분 |
| 컴포넌트 선언이 거부됨 | 추상 클래스·컴포넌트가 아닌 타입·잘못된 부모·루트 중복 | 실제 클래스와 계층 확인; 모든 ActorComponent에 Attach를 추가하지 않기 |
| 생성 중 타이머·delegate 작업 실패 | 런타임 전용 작업을 default·생성자에서 수행함 | 필요한 객체·월드가 존재한 뒤 바인딩하고 대응하는 종료 시점에 해제 |

한 예제만으로 인터페이스의 지원·미지원을 단정하지 마세요. `UInterface`/`TScriptInterface`는 대상 엔진의 바인딩 타입, 작성 제한, 주변의 실제 코드를 확인하세요. 컴포넌트 기반 설계는 실제 요구에 맞는 경우에 선택합니다.

## 소스와 검증 범위

2026-10-22에 UE 5.8.1, 네이티브 플러그인 BuildId `3c6a4263-8d5a-4b20-a676-deccedf5fce2`의 소스로 확인했습니다. 아래 경로는 선택한 엔진 루트 기준입니다. 특정 소스 스냅샷이며 모든 버전의 호환성을 보장하지 않습니다.
근거 수준은 [API 진단과 리로드](api-diagnostics-reload.ko-KR.md), 실제 실행 관측은 [Sandbox 기록](SANDBOX_VALIDATION.ko-KR.md)을 확인하세요.

- `Plugins/Angelscript/Source/AngelscriptCode/Private/ClassGenerator/ASClass.cpp`: `StaticActorConstructor`, `CreateDefaultComponents`,
  `ExecuteDefaultsFunctions`.
- `Plugins/Angelscript/Source/AngelscriptCode/Private/Preprocessor/AngelscriptPreprocessor.cpp`: 함수·속성 지정자 처리.
- `Plugins/Angelscript/Source/AngelscriptCode/Public/BaseClasses/ScriptWorldSubsystem.h`: `BP_Initialize`, `BP_Deinitialize`.
