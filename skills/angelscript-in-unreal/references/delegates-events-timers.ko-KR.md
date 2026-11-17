# Delegate, event와 타이머

Reflected 콜백을 바인딩하거나 단일·다중 수신자를 선택하거나 타이머를 제어할 때 읽으세요.

## 선언에 맞는 호출 API 선택

| 선언 | 바인딩 | 호출 | 자신이 등록한 바인딩 해제 |
| --- | --- | --- | --- |
| `delegate void FSignal(int Amount);` | `BindUFunction(Object, n"Callback")` 또는 생성된 생성자 | `Execute` / `ExecuteIfBound` | `Clear()` |
| `event void FSignal(int Amount);` | `AddUFunction(Object, n"Callback")` | `Broadcast` | `Unbind(Object, n"Callback")` |

delegate는 단일 수신자, event는 다중 수신자입니다. `delegate.Broadcast()`를 `event.Broadcast()`처럼 사용할 수 없습니다. 바인딩되지 않은 `Execute`는 예외를 일으키며, `ExecuteIfBound`는 이를
방지합니다. 반환값이 있는 delegate의 미바인딩 기본 반환값을 의미 있는 앱 결과로 취급하지 말고, 필요하면 `IsBound`를 확인하세요. 수신자가 없는 event broadcast는 아무것도 호출하지 않고 반환합니다.

생성된 delegate와 네이티브 delegate의 API가 다를 수 있습니다. 다른 언어·빌드에서 `RemoveUFunction`, 인자 없는 `Unbind`, `AddDynamic`을 복사하기 전에 실제 타입을 확인하세요. 공유 event에서는 자신의 등록만
제거해야 하며, 전체를 비우면 다른 수신자가 제거될 수 있습니다.

Reflected 이름으로 바인딩하는 콜백에는 `UFUNCTION()`과 정확히 호환되는 시그니처가 필요합니다. 반환 타입, 인자 순서, 숫자 폭, const·참조 의미까지 맞아야 하며 같은 이름만으로는 충분하지 않습니다. C++ 멤버 포인터 대신
`n"Callback"`을 사용하세요. 객체가 유효하고 런타임 작업이 허용되는 시점에 바인딩하고, 소유자·등록의 수명에 맞춰 해제합니다.

## 완결된 독립 타이머·콜백 예제

StartNotifications는 실행 중인 게임플레이 월드의 Actor에서만 호출하세요. 소유 작업이 끝날 때 StopNotifications도 호출해야 하며, 프로젝트의 적절한 종료 이벤트에 연결합니다. 완결된 클래스지만 자동 생명주기 호출을 설정하지는 않습니다.
반복 시작 시에는 이전에 자신이 등록한 항목부터 제거합니다.

```angelscript
delegate void FCounterCallback(int Amount);
event void FCounterEvent(int Amount);

class ANotificationExample : AActor
{
	int Total = 0;
	FCounterCallback Callback;
	FCounterEvent Changed;
	FTimerHandle PulseTimer;

	UFUNCTION(BlueprintCallable)
	void StartNotifications()
	{
		StopNotifications();
		Callback.BindUFunction(this, n"ReceiveAmount");
		Callback.ExecuteIfBound(2);
		Changed.AddUFunction(this, n"ReceiveAmount");
		Changed.Broadcast(3);
		PulseTimer = System::SetTimer(this, n"OnPulse", 0.25f, true);
	}

	UFUNCTION(BlueprintCallable)
	void ToggleTimerPause()
	{
		if (System::IsTimerPausedHandle(PulseTimer))
		{
			System::UnPauseTimerHandle(PulseTimer);
		}
		else
		{
			System::PauseTimerHandle(PulseTimer);
		}
	}

	UFUNCTION(BlueprintCallable)
	void StopNotifications()
	{
		System::ClearAndInvalidateTimerHandle(PulseTimer);
		Callback.Clear();
		Changed.Unbind(this, n"ReceiveAmount");
	}

	UFUNCTION()
	void ReceiveAmount(int Amount)
	{
		Total += Amount;
	}

	UFUNCTION()
	void OnPulse()
	{
		Changed.Broadcast(1);
	}
}
```

즉시 호출로 2 + 3이 더해지고, 타이머가 활성화되어 있으면 이후 실행마다 1이 더해집니다. 실행 중이면 일시정지하고 일시정지 상태이면 재개해야 하며, 반대로 호출하면 안 됩니다. 나중에 취소해야 한다면 타이머 핸들을 보관하세요. 해제·무효화는 이후 예약된 호출을
막습니다. 바인딩 성공만으로 타이머 실행을 추정하지 말고, 실행 중인 월드에서 타이머가 콜백을 호출하도록 하여 결과 상태를 확인하세요.

## 콜백 실패 진단

| 증상 | 가능한 원인 | 수정 |
| --- | --- | --- |
| delegate에 Broadcast가 없음 | 단일·다중 수신자 선언을 혼동함 | Execute/ExecuteIfBound 사용 또는 다중 수신자가 목적이면 event 선언 |
| 콜백이 있는데 바인딩 실패 | Reflection 누락 또는 시그니처 불일치 | 실제 reflected 시그니처와 delegate 선언·바인딩 비교 |
| 핸들러가 여러 번 실행됨 | 생명주기마다 중복 등록 | 등록 소유권을 명확히 하고 대응하는 이전 바인딩 해제 |
| 에디터 전용 객체에서 타이머가 실행되지 않음 | 진행 중인 게임플레이 타이머 월드가 없음 | 의도한 런타임 월드에서 실행하고 일시정지·상태·핸들 확인 |
| 일시정지 전환이 효과 없음 | Pause/UnPause 분기가 반대임 | 일시정지 상태이면 재개, 아니면 일시정지 |
| `No matching signatures to 'System::SetTimerForNextTick(..., const FName)'` | `K2_SetTimer`는 `FName FunctionName`, `K2_SetTimerForNextTick`은 `FString`을 받음 | `SetTimer`에는 `n"OnPulse"`, `SetTimerForNextTick`에는 `"OnPulse"`를 넘기고 바인딩별 인자 타입 확인 |

## 소스와 검증 범위

2026-10-22에 UE 5.8.1, 네이티브 플러그인 BuildId `3c6a4263-8d5a-4b20-a676-deccedf5fce2`의 소스로 확인했습니다. 아래 경로는 선택한 엔진 루트 기준입니다. 특정 소스 스냅샷이며 모든 버전의 호환성을 보장하지 않습니다.
근거 수준은 [API 진단과 리로드](api-diagnostics-reload.ko-KR.md), 실제 실행 관측은 [Sandbox 기록](SANDBOX_VALIDATION.ko-KR.md)을 확인하세요.

- `Plugins/Angelscript/Source/AngelscriptCode/Private/Preprocessor/AngelscriptPreprocessor.cpp`: 생성되는 multicast `Broadcast`와 single-cast
  `Execute`/`BindUFunction`.
- `Plugins/Angelscript/Source/AngelscriptCode/Private/Binds/Bind_Delegates.cpp`: `Clear`, `Unbind`, reflected 바인딩 검증.
- `Source/Runtime/Engine/Classes/Kismet/KismetSystemLibrary.h`: `K2_SetTimer`, 핸들 기반 일시정지·재개·해제 함수와 스크립트 이름.
