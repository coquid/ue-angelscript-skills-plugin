# AngelScript Gameplay Ability System

GAS 작업에만 읽으세요. `GameplayAbilities`, `AngelscriptGAS`와 필요한 의존성이 존재하고 활성화되어 있는지 확인합니다. 모든 AngelScript 엔진에 이 도우미가 포함되어 있지는 않습니다.

## Ability 사용 전에 Owner와 Avatar 초기화

ASC의 Owner는 Ability System의 논리적 상태를 소유하는 Actor이고, Avatar는 Ability를 수행하는 Actor입니다. 작은 데모에서는 같아도 되지만 PlayerState/Pawn 설계에서는 다를 수 있습니다. 부여·활성화 전에 실제
possession·avatar 생명주기에 맞춰 actor info를 초기화·갱신하세요. ASC가 null이 아니라는 이유만으로 성공을 추정하지 말고 실제 Owner와 Avatar를 확인합니다.

`InstancedPerActor`는 ASC에 부여된 spec과 함께 Ability 인스턴스를 유지하며 자동 활성화·종료를 의미하지 않습니다. PlayerState 소유 ASC는 Pawn Avatar보다 오래 유지될 수 있습니다. Avatar를 바꾼다고 새
Ability 인스턴스가 생성되거나 이전 활성화가 종료된다고 가정하지 말고 해당 생명주기를 명시적으로 처리하세요. 부여는 spec 핸들을 반환하고, 활성화는 해당 spec의 실행을 시도하며, `EndAbility()`는 활성 인스턴스를 종료합니다.
활성화 요청의 true 반환이나 로그만으로는 spec의 활성 상태와 실제 게임플레이 변화까지 확인한 것보다 근거가 약합니다.

## 완결된 로컬 authority 예제

확인한 AngelScript 래퍼를 사용하고 ASC Owner와 Avatar가 같은 독립 예제입니다. Authority 측 standalone·로컬 PIE 월드에서 실행하세요. 복제, 예측, 비용, 재사용 대기시간, 네트워크 입력, 멀티플레이 검증은 포함하지 않습니다.
실제 Ability에는 프로젝트의 commit·비용·재사용 대기시간·취소 계약을 구현해야 합니다.

BeginPlay 이후 ActivateGranted를 호출하고 나중에 활성 인스턴스에서 FinishGranted를 호출하세요. Ability의 클래스 기본 객체를 부여된 인스턴스처럼 호출하지 마세요.

```angelscript
class UExampleAttributes : UAngelscriptAttributeSet
{
	UPROPERTY()
	FAngelscriptGameplayAttributeData Energy;
}

class UExampleAbility : UAngelscriptGASAbility
{
	default InstancingPolicy = EGameplayAbilityInstancingPolicy::InstancedPerActor;
	default NetExecutionPolicy = EGameplayAbilityNetExecutionPolicy::ServerOnly;

	UFUNCTION(BlueprintOverride)
	void ActivateAbility()
	{
		AExampleGASActor Host = Cast<AExampleGASActor>(GetAvatarActorFromActorInfo());
		if (Host == nullptr)
		{
			EndAbility();
			return;
		}
		Host.ActivationCount += 1;
		Host.ActiveAbility = this;
	}

	UFUNCTION(BlueprintCallable)
	void FinishExplicitly()
	{
		EndAbility();
	}
}

class AExampleGASActor : AAngelscriptGASActor
{
	UPROPERTY()
	UExampleAbility ActiveAbility;

	FGameplayAbilitySpecHandle GrantedHandle;
	int ActivationCount = 0;
	float32 LastEnergy = 0.0f;

	UFUNCTION(BlueprintOverride)
	void BeginPlay()
	{
		AbilitySystem.InitAbilityActorInfo(this, this);
		AbilitySystem.RegisterAttributeSet(UExampleAttributes);
		AbilitySystem.SetAttributeBaseValue(UExampleAttributes, n"Energy", 100.0f);
		AbilitySystem.RegisterAttributeChangedCallback(
			UExampleAttributes, n"Energy", this, n"ReceiveEnergy");
		GrantedHandle = AbilitySystem.GiveAbility(UExampleAbility, 1, -1, this);
	}

	UFUNCTION(BlueprintCallable)
	bool ActivateGranted()
	{
		return AbilitySystem.TryActivateAbilitySpec(GrantedHandle, false);
	}

	UFUNCTION(BlueprintCallable)
	void FinishGranted()
	{
		if (IsValid(ActiveAbility))
		{
			ActiveAbility.FinishExplicitly();
			ActiveAbility = nullptr;
		}
	}

	UFUNCTION()
	void ReceiveEnergy(const FAngelscriptAttributeChangedData& Data)
	{
		LastEnergy = Data.GetNewValue();
	}
}
```

확인한 바인딩에서는 `FindAbilitySpecFromHandle(Handle, Spec)`과 `Spec.IsActive()`로 활성화·종료 상태를 읽습니다. 속성 관측은 지원되는 ASC setter로 등록된 속성을 바꾸고 콜백 데이터와 현재 값을 모두 확인하세요.
콜백 등록만으로 전달이 입증되지는 않습니다.

## Attribute와 콜백 버전

현재 콜백은 `FAngelscriptAttributeChangedData`를 받습니다. 개별 속성에는 `RegisterAttributeChangedCallback` 또는 `GetAndRegisterAttributeChangedCallback`을 사용하세요. 후자는 이
빌드에서 `float32` 출력으로 현재 값도 기록합니다. 실제 시그니처에 맞춰야 합니다. `RegisterCallbackForAttribute`와 `GetAndRegisterCallbackForAttribute`는 여기서 deprecated이므로 과거의
`FAngelscriptModifiedAttribute`·전역 이벤트 예제를 기본 패턴으로 삼지 마세요.

접근 전에 set을 등록하고 Avatar 변경 전후의 ASC 수명을 고려하세요. 현재 두 등록 래퍼는 해당 속성의 delegate가 콜백 객체에 이미 바인딩되었는지 검사합니다. 같은 객체에 다른 함수 이름을 등록해도 기존 바인딩이 교체되지 않습니다.
객체·속성별 콜백을 일정하게 유지하거나 변경 전에 지원되는 제거 경로를 확인하세요. 이 예제에서는 unregister 래퍼가 확인되지 않았습니다. 복제되는 AttributeSet은 이 포크의 `ReplicatedUsing`과 `OnRep_Attribute` 경로를
확인해야 합니다. 로컬 setter만으로 복제가 입증되지는 않습니다.

| 증상 | 확인할 원인 | 수정 |
| --- | --- | --- |
| Ability 부여는 되지만 actor info가 틀림 | ASC가 과거 Owner·Avatar로 초기화됨 | 실제 생명주기에 맞춰 재초기화하고 두 신원을 확인 |
| 활성화 후 끝나지 않음 | 부여된 인스턴스가 종료 경로를 호출하지 않음 | 해당 인스턴스를 명시적으로 종료하고 spec 상태 확인 |
| 콜백이 오지 않음 | 잘못된 set·이름, deprecated 래퍼, 시그니처 불일치 | 의도한 set과 현재 콜백 계약을 등록하고 ASC를 통해 변경 |
| 로컬 PIE와 네트워크 실행 결과가 다름 | Authority·예측·복제를 실행 검증하지 않음 | 네트워크 역할을 따로 설계·검증하고 데모를 멀티플레이 검증 완료로 표시하지 않기 |

## 소스와 검증 범위

2026-10-22에 UE 5.8.1, 네이티브 플러그인 BuildId `3c6a4263-8d5a-4b20-a676-deccedf5fce2`의 소스로 확인했습니다. 아래 경로는 선택한 엔진 루트 기준입니다. 특정 소스 스냅샷이며 모든 버전의 호환성을 보장하지 않습니다.
근거 수준은 [API 진단과 리로드](api-diagnostics-reload.ko-KR.md), 실제 실행 관측은 [Sandbox 기록](SANDBOX_VALIDATION.ko-KR.md)을 확인하세요.

- `Plugins/AngelscriptGAS/Source/Public/AngelscriptAbilitySystemComponent.h`: `BP_InitAbilityActorInfo`, `GiveAbility`, `TryActivateAbilitySpec`,
  `RegisterAttributeChangedCallback`.
- `Plugins/AngelscriptGAS/Source/Public/AngelscriptGASAbility.h`: UGameplayAbility를 상속하며 GameplayCue 도우미를 추가하는 기반 클래스.
- `Plugins/AngelscriptGAS/Source/Public/AngelscriptAttributeSet.h`: `OnRep_Attribute`.
- `Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h`: `K2_ActivateAbility`, `K2_EndAbility`,
  `K2_OnEndAbility`와 Instancing·네트워크 정책.
- `Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp`: `UGameplayAbility::ActivateAbility`의
  reflected 활성화 이벤트 호출과 `UGameplayAbility::K2_EndAbility`의 활성 인스턴스 종료 경로 전달.
- `Plugins/AngelscriptGAS/Source/Private/AngelscriptAbilitySystemComponent.cpp`: 현재 두 속성 콜백 등록 래퍼의 `AddUFunction` 전 `IsBoundToObject` 검사.
- `Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp`: `GiveAbility`가 ASC에 spec을 저장하고
  `InstancedPerActor` 인스턴스를 생성하는 경로.
