# Enhanced Input

Enhanced Input 바인딩과 로컬 플레이어 MappingContext를 다룰 때 읽으세요. `EnhancedInput`과 `AngelscriptEnhancedInput`을 모두 활성화해야 합니다. 기존 `UInputComponent.BindAxis` 예제는 이
확장의 대체물이 아닙니다.

## 전제 조건과 실제 시그니처

MappingContext는 대상 로컬 플레이어의 `UEnhancedInputLocalPlayerSubsystem`에 속합니다. InputAction과 바인딩만 만들었다고 컴포넌트가 해당 플레이어의 입력 스택에 들어가거나 `UEnhancedPlayerInput`이
활성화되는 것은 아닙니다. 설정을 바꾸기 전에 실제 로컬 플레이어·컨트롤러와 현재 입력 설정을 확인하세요. 예제를 위해 프로젝트·전역 설정을 알리지 않고 변경하지 마세요.

확인한 BindAction 오버로드는 `(Action, TriggerEvent, FEnhancedInputActionHandlerDynamicSignature)`를 받습니다. Reflected 핸들러는 값으로 전달되는 `FInputActionValue`,
`float32` 시간 값 두 개, `const UInputAction`을 받습니다. C++ 멤버 함수 포인터나 인자 하나인 콜백은 이 오버로드가 아닙니다. `GetHandle()`을 저장하고 나중에 `RemoveBindingByHandle`로 자신의 바인딩을
제거하세요.

확인한 AS 바인딩에서는 `AddMappingContext(Context, Priority, FModifyContextOptions())`와 `RemoveMappingContext(Context, FModifyContextOptions())`의 옵션 인자를 생략할 수
없습니다. C++ 기본 인자가 AS 인자 생략 가능성의 근거는 아닙니다. 여기서 `UInputAction.ValueType`은 런타임 스크립트 대입에 읽기 전용이므로 에셋 또는 스크립트 하위 클래스의 `default` 값으로 설정하세요.

이 예제는 자신이 소유할 입력 컴포넌트를 런타임에 생성·검증한 뒤 바인딩하고, Context를 등록하여 컴포넌트를 push합니다. 정리할 때는 자신의 등록만 제거하고 생성한 컴포넌트만 파괴하며, 설정 도중의 실패도 정리합니다. 컴포넌트 선언만으로 실제 PIE 참조의
유효성이 입증되지는 않습니다. 검증 중 관측한 null 참조가 정확히 어느 생명주기 단계에서 발생했는지는 아직 확정하지 않았습니다.

## 완결된 독립 로컬 플레이어 예제

Enhanced Player Input이 이미 설정된 로컬 PIE·게임플레이 Actor에서만 사용하세요. 예제는 임시 Action·Context 객체를 만들고 시연을 위해 자신의 컴포넌트를 직접 push합니다. 실제 Pawn은 기존 프로젝트 입력 설정이 있다면 그것을
사용하세요. F9 한 키를 Axis2D action에 연결하는 작은 probe이며 완전한 이동 키 구성은 아닙니다. 자신이 소유한 Context와 바인딩만 제거하세요.

```angelscript
class UExampleAxisAction : UInputAction
{
	default ValueType = EInputActionValueType::Axis2D;
	default bConsumeInput = false;
}

class AInputBindingExample : APawn
{
	UPROPERTY(Transient)
	UEnhancedInputComponent DemoInput;

	UPROPERTY()
	UInputAction Action;
	UPROPERTY()
	UInputMappingContext Context;
	UPROPERTY()
	UEnhancedInputLocalPlayerSubsystem InputSubsystem;
	UPROPERTY()
	APlayerController LocalController;

	uint32 BindingHandle = 0;
	bool bInstalled = false;
	bool bPlayStarted = false;
	FVector2D LastValue;

	UFUNCTION(BlueprintOverride)
	void BeginPlay()
	{
		bPlayStarted = true;
	}

	UFUNCTION(BlueprintCallable)
	bool InstallInput(APlayerController TargetController)
	{
		if (!bPlayStarted || bInstalled || !IsValid(TargetController))
		{
			return false;
		}
		InputSubsystem = UEnhancedInputLocalPlayerSubsystem::Get(TargetController);
		if (InputSubsystem == nullptr || InputSubsystem.GetPlayerInput() == nullptr)
		{
			return false;
		}
		DemoInput = UEnhancedInputComponent::Create(this);
		Action = NewObject(this, UExampleAxisAction);
		Context = NewObject(this, UInputMappingContext);
		if (!IsValid(DemoInput) || !IsValid(Action) || !IsValid(Context))
		{
			RemoveInput();
			return false;
		}
		Context.MapKey(Action, FKey(n"F9"));

		BindingHandle = DemoInput.BindAction(Action, ETriggerEvent::Triggered,
			FEnhancedInputActionHandlerDynamicSignature(this, n"ReceiveAction")).GetHandle();
		LocalController = TargetController;
		InputSubsystem.AddMappingContext(Context, 0, FModifyContextOptions());
		LocalController.PushInputComponent(DemoInput);
		bInstalled = true;
		return true;
	}

	UFUNCTION()
	void ReceiveAction(FInputActionValue Value, float32 ElapsedTime,
		float32 TriggeredTime, const UInputAction SourceAction)
	{
		if (SourceAction == Action)
		{
			LastValue = Value.GetAxis2D();
		}
	}

	UFUNCTION(BlueprintCallable)
	void RemoveInput()
	{
		if (bInstalled)
		{
			if (IsValid(DemoInput))
			{
				DemoInput.RemoveBindingByHandle(BindingHandle);
			}
			if (IsValid(InputSubsystem) && IsValid(Context))
			{
				InputSubsystem.RemoveMappingContext(Context, FModifyContextOptions());
			}
			if (IsValid(LocalController) && IsValid(DemoInput))
			{
				LocalController.PopInputComponent(DemoInput);
			}
		}
		if (IsValid(DemoInput))
		{
			DemoInput.DestroyComponent();
		}
		DemoInput = nullptr;
		Action = nullptr;
		Context = nullptr;
		LocalController = nullptr;
		InputSubsystem = nullptr;
		bInstalled = false;
	}

	UFUNCTION(BlueprintOverride)
	void EndPlay(EEndPlayReason EndPlayReason)
	{
		RemoveInput();
		bPlayStarted = false;
	}
}

void ReadInputValues()
{
	FInputActionValue BooleanValue(EInputActionValueType::Boolean, FVector(1.0, 0.0, 0.0));
	FInputActionValue Axis1DValue(float32(2.5f));
	FInputActionValue Axis2DValue(FVector2D(3.0, 4.0));
	FInputActionValue Axis3DValue(FVector(5.0, 6.0, 7.0));
	bool bPressed = BooleanValue.Get();
	float32 Scalar = Axis1DValue.GetAxis1D();
	FVector2D Plane = Axis2DValue.GetAxis2D();
	FVector Space = Axis3DValue.GetAxis3D();
	Print(f"{bPressed}, {Scalar}, {Plane}, {Space}");
}
```

네 접근자는 각각 Boolean, Axis1D, Axis2D, Axis3D에 대응합니다. 여기서 Boolean 접근자는 추측한 `GetBool()`이 아니라 `Get()`이며, C++ `Get<T>()`도 노출된 AS 문법이 아닙니다. 값을 읽기 전에 Action에
설정된 값 타입과 맞추세요.

## 전달과 해제 검증

MappingContext 등록과 실제 입력 컴포넌트 스택을 확인하고, 입력을 전달하여 콜백의 원본 Action과 값을 확인합니다. 테스트용 `InjectInputForAction`은 PIE에서 Enhanced Input 처리·콜백 경로를 확인할 수 있지만 물리
키보드·장치 입력 경로까지 입증하지는 않습니다. 대상 빌드에서 modifier·trigger 인자를 정확히 확인하세요. 바인딩을 제거한 뒤 입력을 다시 전달하여 콜백 횟수가 더 늘지 않는지 확인합니다. 이어서 자신의 Context를 제거하고 직접 push했던 컴포넌트만
pop합니다.

| 증상 | 확인할 원인 | 수정 |
| --- | --- | --- |
| ValueType에 대입할 수 없음 | 런타임 속성이 읽기 전용임 | Action 생성 전에 에셋·스크립트 클래스 기본값으로 설정 |
| Add/RemoveMappingContext와 일치하는 호출 없음 | 옵션 인자 생략 | 이 바인딩에서는 FModifyContextOptions를 명시 |
| BindAction 성공 후 입력이 오지 않음 | 로컬 플레이어 Context 없음, 잘못된 PlayerInput, 컴포넌트가 스택에 없음 | 실제 객체를 확인하고 등록만으로 전달을 추정하지 않기 |
| Delegate 시그니처 거부 | 값·참조 타입 오류 또는 시간·원본 인자 누락 | 정확한 네 인자의 dynamic 시그니처 사용 |
| 종료 후에도 핸들러 실행 | 핸들 유실 또는 중복 등록 | 자신의 핸들을 제거하고 이후 콜백 전달 확인 |

## 소스와 검증 범위

2026-10-22에 UE 5.8.1, 네이티브 플러그인 BuildId `3c6a4263-8d5a-4b20-a676-deccedf5fce2`의 소스로 확인했습니다. 아래 경로는 선택한 엔진 루트 기준입니다. 특정 소스 스냅샷이며 모든 버전의 호환성을 보장하지 않습니다.
근거 수준은 [API 진단과 리로드](api-diagnostics-reload.ko-KR.md), 실제 실행 관측은 [Sandbox 기록](SANDBOX_VALIDATION.ko-KR.md)을 확인하세요.

- `Plugins/AngelscriptEnhancedInput/Source/Private/Binds/Bind_UEnhancedInputComponent.cpp`: dynamic `BindAction`, `RemoveBindingByHandle`.
- `Plugins/AngelscriptEnhancedInput/Source/Private/Binds/Bind_FInputActionValue.cpp`: 생성자, `Get`, `GetAxis1D`, `GetAxis2D`, `GetAxis3D`.
- `Plugins/EnhancedInput/Source/EnhancedInput/Public/EnhancedInputComponent.h`: `FEnhancedInputActionHandlerDynamicSignature`.
- `Plugins/EnhancedInput/Source/EnhancedInput/Public/EnhancedInputSubsystemInterface.h`: MappingContext·입력 주입 계약.
- `Plugins/EnhancedInput/Source/EnhancedInput/Public/InputAction.h`: `ValueType` 노출.

- `Plugins/Angelscript/Source/AngelscriptCode/Private/Binds/Bind_UActorComponent.cpp`: 컴포넌트 `Create`, 등록, `DestroyComponent`.
- `Plugins/AngelscriptEnhancedInput/Source/Public/Mixin/EnhancedInputSubsystemsMixinLibrary.h`: 로컬 플레이어 Context·입력 주입 래퍼.
- `Plugins/Angelscript/Source/AngelscriptCode/Private/Binds/Helper_FunctionSignature.h`: `CPP_Default_*` 메타데이터 읽기.
- `Plugins/Angelscript/Source/AngelscriptCode/Private/AngelscriptType.cpp`: 호출 기본값 구성 시 `DefaultValue_UnrealToAngelscript` 변환. C++ 기본값 메타데이터의 존재만으로 AS
  기본값이 보장되지는 않습니다.
