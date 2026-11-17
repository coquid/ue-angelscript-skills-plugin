# UMG와 CommonUI 위젯

스크립트 `UUserWidget`/CommonUI 클래스를 작성하거나, 디자이너 위젯을 바인딩하거나, 위젯을 생성·표시할 때 읽으세요. 프로젝트에 자체 위젯 기반
클래스와 UI 레이어 API가 있으면 그것을 먼저 따르세요.

## 스크립트 위젯 클래스와 이벤트

위젯 이벤트는 스크립트에서 보이는 이름으로 `UFUNCTION(BlueprintOverride)`를 붙여 override합니다. 확인한 빌드에서 컴파일되는 형태는
`void PreConstruct(bool IsDesignTime)`, `void Construct()`, `void Destruct()`, `void Tick(FGeometry MyGeometry, float InDeltaTime)`,
`void OnPaint(FPaintContext& Context) const`, `FEventReply OnPreviewKeyDown(FGeometry MyGeometry, FKeyEvent InKeyEvent)` 등입니다. 이벤트 이름이
없거나 틀리면 컴파일 오류입니다. 스크립트 부모 클래스에 구현한 Tick은 Widget Blueprint 자식에서도 실행됩니다.

스크립트 이름에서는 `K2_`, `BP_`, `AS_` 접두어가 빠지고, 이벤트는 `Receive`/`Received_`도 빠집니다. 접두어를 뺀 이름을 이미 다른
Blueprint-callable 함수가 쓰고 있으면 접두어가 유지됩니다. C++ 이름을 쓰면 컴파일러가 `Use name X instead`라고 알려 줍니다. CommonUI 예:

| C++ 이벤트 | 스크립트 override |
| --- | --- |
| `BP_OnActivated` / `BP_OnDeactivated` | `void OnActivated()` / `void OnDeactivated()` |
| `BP_OnHandleBackAction` | `bool OnHandleBackAction()` |
| `BP_GetDesiredFocusTarget` | `UWidget BP_GetDesiredFocusTarget() const` (`GetDesiredFocusTarget`이 이미 있어 접두어 유지) |

스크립트 클래스는 `UCommonButtonBase`, `UCommonActivatableWidget` 같은 CommonUI 클래스를 직접 상속할 수 있습니다. 부모가 스크립트 클래스이면
`Super::Construct()` 같은 호출도 동작합니다.

## 디자이너 바인딩

`BindWidget`, `BindWidgetOptional`, `BindWidgetAnim`은 `UPROPERTY` 지정자로 쓸 수 있으며, 프로퍼티를 transient·읽기 전용·편집 불가로 만듭니다.
`BindWidgetAnimOptional` 지정자는 없으므로 `UPROPERTY(Transient, meta = (BindWidgetAnimOptional))`을 쓰세요.

런타임 바인딩은 이름 기준입니다. 각 디자이너 위젯이 같은 이름의 오브젝트 프로퍼티에 할당됩니다. 스크립트 클래스를 Widget Blueprint의 부모
클래스로 쓰고, 단독으로 인스턴스화하면 안 되면 `UCLASS(Abstract)`를 붙이세요. 필수 바인딩이 디자이너 트리에 없거나 디자이너 위젯 타입이 맞지
않으면 Widget Blueprint 컴파일 오류이고, optional 바인딩은 정보 메시지만 나옵니다.

Widget Blueprint가 아니라 스크립트 클래스 자체로 만든 위젯에는 디자이너 트리가 없어서 바인딩 프로퍼티가 null로 남습니다. 순수 스크립트 위젯은
`ConstructWidget(UClass, FName)`과 `SetRootWidget(UWidget)`으로 직접 트리를 만듭니다.

## 생성, 표시, 포커스

- 생성: `WidgetBlueprint::CreateWidget(WidgetClass, OwningPlayer)`(world context 자동 주입). `UUserWidget`을 반환하므로 구체 타입으로 cast.
- 표시·숨김: `AddToViewport(ZOrder)`, `AddToPlayerScreen`, `RemoveFromParent()`, `SetVisibility(ESlateVisibility::Collapsed)`.
- 소유자: `GetOwningPlayer()`, `SetOwningPlayer`.
- 포커스·입력: `default bIsFocusable = true;`와 `SetFocus()`, `Widget::SetInputMode_GameAndUIEx(...)` 및 다른 `SetInputMode_*` 함수.
- CommonUI 스택: `UCommonActivatableWidgetContainerBase.BP_AddWidget`은 스크립트에서 `AddWidget(TSubclassOf<UCommonActivatableWidget>)`로
  보입니다. 프로젝트에 UI 레이어 push API가 있으면 그것을 우선하세요.

## ListView/TileView 엔트리

엔트리 클래스는 `IUserObjectListEntry`를 구현해야 합니다. 확인한 플러그인에는 스크립트 클래스가 `UInterface`를 구현하는 코드 경로가 없습니다
(소스에서 추론, 컴파일 테스트는 안 함). Widget Blueprint 자식(Class Settings)이나 스크립트가 상속하는 얇은 C++ 기반 클래스에서 인터페이스를
구현하고, `OnListItemObjectSet`이 엔트리에 도달하는지 확인하세요. 직접 풀링하는 컨테이너(`CreateWidget` + `AddChild` + 명시적 데이터 바인딩)를
쓰면 인터페이스가 필요 없습니다.

독립적으로 완결된 예제입니다. 이 부모 클래스로 `WBP_Counter`를 만들고 디자이너 위젯 이름을 `CountText`(TextBlock), `IncrementButton`(Button)으로
지정한 뒤, `WidgetClass`에는 스크립트 클래스가 아니라 `WBP_Counter`를 할당하세요.

```angelscript
UCLASS(Abstract)
class UCounterWidget : UUserWidget
{
	UPROPERTY(BindWidget)
	UTextBlock CountText;

	UPROPERTY(BindWidget)
	UButton IncrementButton;

	private int Count = 0;

	UFUNCTION(BlueprintOverride)
	void Construct()
	{
		IncrementButton.OnClicked.AddUFunction(this, n"HandleIncrementClicked");
		Refresh();
	}

	UFUNCTION(BlueprintOverride)
	void Destruct()
	{
		IncrementButton.OnClicked.UnbindObject(this);
	}

	UFUNCTION()
	private void HandleIncrementClicked()
	{
		Count++;
		Refresh();
	}

	private void Refresh()
	{
		CountText.SetText(FText::FromString(f"Count: {Count}"));
	}
}

class UCounterHUDComponent : UActorComponent
{
	UPROPERTY(EditAnywhere)
	TSubclassOf<UCounterWidget> WidgetClass;

	UPROPERTY(Transient)
	private UCounterWidget Widget;

	UFUNCTION(BlueprintOverride)
	void BeginPlay()
	{
		APlayerController PC = Gameplay::GetPlayerController(0);
		if (!IsValid(PC) || !IsValid(WidgetClass.Get()))
			return;

		Widget = Cast<UCounterWidget>(WidgetBlueprint::CreateWidget(WidgetClass, PC));
		if (IsValid(Widget))
			Widget.AddToViewport();
	}

	UFUNCTION(BlueprintOverride)
	void EndPlay(EEndPlayReason EndPlayReason)
	{
		if (IsValid(Widget))
			Widget.RemoveFromParent();
	}
}
```

`UButton.OnClicked`는 multicast delegate입니다. `AddUFunction`으로 바인딩하고 `UnbindObject`로 해제하며, 핸들러는 delegate 시그니처와 정확히
같은 `UFUNCTION()`이어야 합니다.

## 위젯 실패 진단

| 증상 | 가능한 원인 | 조치 |
| --- | --- | --- |
| `A required widget binding "X" ... was not found` | 디자이너 위젯 이름이 프로퍼티와 다름 | 이름을 정확히 맞추거나 `BindWidgetOptional` 사용 |
| `The widget ... is of type ..., but the bind widget property is of type ...` | 프로퍼티 타입이 디자이너 위젯의 상위 타입이 아님 | 프로퍼티 타입 또는 디자이너 위젯 변경 |
| 런타임에 바인딩 프로퍼티가 null | Widget Blueprint 대신 스크립트 클래스로 생성 | Widget Blueprint 클래스를 넘기거나 `ConstructWidget`/`SetRootWidget`으로 트리 구성 |
| `Could not find function with this name. Is it declared UFUNCTION()?` | delegate 핸들러가 reflect되지 않음 | `UFUNCTION()` 추가 |
| `Specified function is not compatible with delegate function` | 핸들러 시그니처 불일치(C++ `float` 매개변수는 스크립트에서 `float32`) | delegate 시그니처를 정확히 복사 |
| `Use name X instead to override C++ event ...` | 접두어가 붙은 C++ 이벤트 이름 사용 | 접두어를 뺀 스크립트 이름 사용(충돌 시에만 접두어 유지) |

레이아웃과 상호작용은 렌더링되는 에디터나 PIE에서 확인하세요. NullRHI로는 UI를 검증할 수 없습니다.

## 소스와 검증 범위

2026-11-16에 UE 5.8.1, 네이티브 플러그인 BuildId `343c3a9c-ab61-4b28-b709-cd650e59f32e`의 소스로 확인했습니다. 경로는 엔진 루트 기준입니다.
개별 구문은 컴파일되는 프로젝트 코드에 있으나 이 예제 자체는 컴파일하지 않았습니다.

- `Plugins/Angelscript/Source/AngelscriptCode/Private/Preprocessor/AngelscriptPreprocessor.cpp`: `BindWidget*` 지정자.
- `Plugins/Angelscript/Source/AngelscriptCode/Private/Binds/Helper_FunctionSignature.h`: 스크립트 이름 접두어 제거.
- `Plugins/Angelscript/Source/AngelscriptCode/Private/ClassGenerator/AngelscriptClassGenerator.cpp`: override 이름 검사와 오류.
- `Plugins/Angelscript/Source/AngelscriptCode/Private/Binds/Bind_UUserWidget.cpp`: `ConstructWidget`, `SetRootWidget`.
- `Plugins/Angelscript/Source/AngelscriptCode/Public/FunctionLibraries/WidgetBlueprintStatics.h`: `WidgetBlueprint::CreateWidget`.
- `Plugins/Angelscript/Source/AngelscriptCode/Private/Binds/Bind_Delegates.cpp`: multicast `AddUFunction`/`UnbindObject`, 핸들러 검증.
- `Source/Runtime/UMG/Public/Blueprint/UserWidget.h`, `Source/Runtime/UMG/Private/WidgetBlueprintGeneratedClass.cpp`: 이벤트, 이름 기준 바인딩.
- `Source/Editor/UMGEditor/Private/WidgetBlueprintCompiler.cpp`: 바인딩 컴파일 오류.
- `Plugins/Runtime/CommonUI/Source/CommonUI/Public/CommonActivatableWidget.h`, `Widgets/CommonActivatableWidgetContainer.h`: CommonUI 이벤트, 스택 push.
- `Source/Runtime/UMG/Public/Components/ListViewBase.h`, `Blueprint/IUserObjectListEntry.h`: 엔트리 인터페이스 요구사항.
