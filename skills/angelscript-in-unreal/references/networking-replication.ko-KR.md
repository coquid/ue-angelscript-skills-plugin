# 네트워킹과 복제

스크립트 액터·컴포넌트에 복제 상태, RepNotify 콜백, RPC, 권한(authority) 검사를 추가할 때 읽으세요.

## 먼저 빌드 종류 확인

확인한 빌드는 플러그인을 `WITH_ANGELSCRIPT_HAZE 0`으로 컴파일합니다. 아래의 표준 Unreal 지정자는 사용할 수 있고, Hazelight 내부의
`NetFunction`/`CrumbFunction` 시스템은 컴파일에서 제외됩니다. 그 이름을 쓰는 예제를 가져오기 전에 대상 엔진의
`Public/AngelscriptManager.h`를 확인하세요.

## 복제 프로퍼티

| 지정자 | 효과 |
| --- | --- |
| `UPROPERTY(Replicated)` | 서버에서 프로퍼티를 복제 |
| `UPROPERTY(ReplicatedUsing = OnRep_Name)` | `Replicated`를 포함하며, 받는 쪽에서 `OnRep_Name` 호출 |
| `ReplicationCondition = OwnerOnly` | `None`, `InitialOnly`, `OwnerOnly`, `SkipOwner`, `SimulatedOnly`, `AutonomousOnly`, `SimulatedOrPhysics`, `InitialOrOwner`, `Custom`, `ReplayOrOwner`, `ReplayOnly`, `SimulatedOnlyNoReplay`, `SimulatedOrPhysicsNoReplay`, `SkipReplay` 중 하나 |
| `NotReplicated` | 구조체 멤버 전용. 해당 멤버를 복제에서 제외 |

`GetLifetimeReplicatedProps`는 작성하지 않습니다. 스크립트 액터와 컴포넌트는 복제 스크립트 프로퍼티를 조건과 함께 자동 등록하며, 항상
`REPNOTIFY_OnChanged`를 씁니다. "항상 notify" 옵션은 없습니다. `ReplicationPushModel`은 피하세요. 이 빌드에는 push-model 프로퍼티를 dirty로
표시하는 스크립트 바인딩이 없습니다.

RepNotify는 같은 스크립트 클래스의 `UFUNCTION()`이어야 하며, 매개변수가 없거나 프로퍼티와 같은 타입의 매개변수 하나(이전 값)를 받습니다.
스크립트 대입은 Blueprint setter 노드가 아니라 직접적인 프로퍼티 쓰기이므로, 서버가 자기 변경에 대해 RepNotify를 받는다고 가정하지 마세요
(중요하면 PIE에서 확인). 서버에서도 같은 반응이 필요하면 공용 처리 코드를 명시적으로 호출하세요.

## RPC

| 지정자 | 참고 |
| --- | --- |
| `Server`, `Client`, `NetMulticast` | 네트워크 함수 종류 |
| `Unreliable` | 신뢰성 해제. **네트워크 함수는 기본이 reliable이며 `Reliable` 키워드는 없습니다** |
| `WithValidation` | Server/Client 전용. `UFUNCTION() bool Name_Validate(<같은 매개변수>)` 필요 |
| `BlueprintAuthorityOnly` | 표준 authority 전용 플래그 |

본문은 원래 함수 이름으로 작성하고 그 이름으로 호출합니다. 전처리기가 디스패치 래퍼를 만들고 본문을 구현 함수로 이름을 바꿉니다. 네트워크
함수는 `NotBlueprintCallable`을 붙이지 않으면 Blueprint에서 호출할 수 있게 됩니다. RPC는 static일 수 없고 `BlueprintEvent`/`BlueprintOverride`와
함께 쓸 수 없습니다. 확인한 빌드에서 Server 검증은 선택 사항입니다. 클라이언트→서버 호출에는 일반 Unreal 규칙대로 소유 연결이 필요합니다.

## 복제 설정과 권한

액터의 `bReplicates`와 컴포넌트의 `bReplicates`는 기본값을 편집할 수 있는 reflected 프로퍼티이므로, `default bReplicates = true;`가 C++
생성자에서 설정하는 것에 해당합니다. 대상 빌드에서 컴파일되는지 확인하세요. 런타임에는 액터에 `SetReplicates(bool)`, 컴포넌트에
`SetIsReplicated(bool)`을 씁니다. 그 밖에 스크립트에서 보이는 항목은 `SetReplicateMovement`, `SetNetUpdateFrequency`, `SetNetDormancy`,
`FlushNetDormancy`, `ForceNetUpdate`, `bAlwaysRelevant`, `bOnlyRelevantToOwner`, `NetPriority`입니다. 네이티브 `NetUpdateFrequency`는
deprecated이므로 setter를 쓰세요.

권한·역할 검사: `HasAuthority()`, `GetLocalRole()`/`GetRemoteRole()`(`ENetRole`), `Pawn.IsLocallyControlled()`,
`Controller.IsLocalController()`, `World.GetNetMode()`(`ENetMode`), `System::IsServer()`/`IsDedicatedServer()`/`IsStandalone()`.
결과를 결정하는 로직(데미지, 난수, 검증, 스폰)은 authority에 두고 결과를 복제하세요.

독립적으로 완결된 액터 예제입니다. `default bReplicates` 줄은 프로퍼티 플래그를 근거로 한 것이며 대상 빌드에서 컴파일 확인이 필요합니다.

```angelscript
class ANetCounterActor : AActor
{
	default bReplicates = true;

	UPROPERTY(ReplicatedUsing = OnRep_Count)
	int Count = 0;

	UFUNCTION()
	void OnRep_Count(int OldCount)
	{
		ReactToCount(OldCount);
	}

	UFUNCTION(Server, WithValidation)
	void ServerAdd(int Amount)
	{
		int OldCount = Count;
		Count += Amount;
		ReactToCount(OldCount);
		MulticastPulse(Count);
	}

	UFUNCTION()
	bool ServerAdd_Validate(int Amount)
	{
		return Amount > 0 && Amount <= 10;
	}

	UFUNCTION(NetMulticast, Unreliable)
	void MulticastPulse(int NewCount)
	{
	}

	void ReactToCount(int OldCount)
	{
	}
}
```

`ReactToCount`는 서버에서는 명시적으로, 클라이언트에서는 RepNotify를 통해 호출됩니다. `MulticastPulse`는 연출용이고 unreliable이므로 부하가 있을
때 누락될 수 있는 대상입니다.

## 복제 실패 진단

| 증상 | 가능한 원인 | 조치 |
| --- | --- | --- |
| `Unknown function specifier Reliable` | C++에서 `Reliable`을 복사함 | 제거. 기본이 reliable이며 해제는 `Unreliable` |
| `cannot both be a blueprint event and have network specifiers` | RPC와 `BlueprintEvent`/`BlueprintOverride`를 함께 씀 | RPC와 override 가능한 이벤트를 분리 |
| RepNotify `could not be found within the script class` | `UFUNCTION()` 누락 또는 이름 오타 | `UFUNCTION()`을 붙이고 이름을 정확히 맞춤 |
| RepNotify `more than 1 argument` 또는 인자 타입 오류 | 시그니처 오류 | 매개변수 없음, 또는 프로퍼티 타입 매개변수 하나 |
| `no _Validate function provided` | `WithValidation`에 맞는 검증 함수 없음 | 같은 매개변수의 `UFUNCTION() bool Name_Validate(...)` 추가 |
| 오류 없이 복제되지 않음 | 소유자 복제 꺼짐, 조건 오류, `ReplicationPushModel` | 액터·컴포넌트 복제를 켜고 조건 확인, push model 제거 |

복제는 standalone이 아니라 client/server PIE에서 테스트하세요. authority 분기와 RepNotify 경로가 달라집니다.

## 소스와 검증 범위

2026-11-16에 UE 5.8.1, 네이티브 플러그인 BuildId `343c3a9c-ab61-4b28-b709-cd650e59f32e`의 소스로 확인했습니다(이 폴더의 다른 주제와 다른
빌드). 경로는 엔진 루트 기준입니다. 특정 소스 스냅샷이며 모든 버전의 호환성을 보장하지 않습니다. 여기 예제는 실행하지 않았습니다.

- `Plugins/Angelscript/Source/AngelscriptCode/Public/AngelscriptManager.h`: `WITH_ANGELSCRIPT_HAZE`, `AS_ENFORCE_SERVER_RPC_VALIDATION`.
- `Plugins/Angelscript/Source/AngelscriptCode/Private/Preprocessor/AngelscriptPreprocessor.cpp`: 프로퍼티 복제 지정자와 조건, 함수 네트워크
  지정자, RPC 래퍼 생성, 관련 오류.
- `Plugins/Angelscript/Source/AngelscriptCode/Private/ClassGenerator/AngelscriptClassGenerator.cpp`: 네트워크 플래그, 기본 신뢰성, `_Validate` 검사.
- `Plugins/Angelscript/Source/AngelscriptCode/Private/ClassGenerator/ASClass.cpp`: 스크립트 lifetime 복제 목록.
- `Plugins/Angelscript/Source/AngelscriptCode/Private/AngelscriptManager.cpp`: `VerifyRepFunc` RepNotify 규칙.
- `Source/Runtime/Engine/Private/ActorReplication.cpp`, `Source/Runtime/Engine/Private/Components/ActorComponent.cpp`: 스크립트 복제 목록 연결.
