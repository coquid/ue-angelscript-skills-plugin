# 문법, 값과 참조

C++ 예제를 옮기거나 숫자 타입을 선택하거나 구조체·컨테이너 데이터를 변경할 때 읽으세요.

## 객체, 이름과 숫자

- UObject 변수는 객체 참조입니다. `AActor Actor;`, `Actor.Method()`, `Cast<AActor>(Object)`를 사용합니다. 선언만으로 Actor가 생성되지는 않습니다. 역참조 전에 null과 유효성을 확인하고, Actor는 적절한
  월드 API로 생성하세요.
- `n"Label"`은 FName 리터럴이고 `f"Count={Count}"`는 형식 문자열입니다. C++의 `TEXT()`나 멤버 함수 포인터 문법을 기계적으로 복사하지 말고 지원되는 표기를 사용하세요.
- AS `float`를 C++ `float`와 같다고 가정하지 마세요. 확인한 `bScriptFloatIsFloat64` 설정의 기본값은 true이며, 실제 프로젝트 설정이 별칭을 결정합니다. 특히 reflected 콜백처럼 바인딩 계약에 폭이 중요하면
  `float32`/`float64`를 사용하세요. 대상 빌드에서 리터럴, 오버로드, 변환 경고를 확인합니다.

완결된 도우미 함수입니다. 기존 Actor 참조를 반환하며 Actor를 생성하지 않습니다.

```angelscript
AActor FindActor(UObject Object)
{
	AActor Actor = Cast<AActor>(Object);
	if (!IsValid(Actor))
	{
		return nullptr;
	}
	return Actor;
}
```

## 복사와 참조

구조체 대입은 값 복사본을 만듭니다. 확인한 포크에서 스크립트 함수는 & 없이 쓴 값 타입 인자를 암묵적인 const 참조로 취급하며, 기본 타입과 객체 핸들 인자 변수도 읽기 전용입니다. 읽기 전용 객체 핸들 인자라고 해서 참조 대상 객체까지 불변이 되는 것은
아닙니다. 수정이 목적이면 로컬 복사본을 명시하고, 표기만으로 C++ 인자 의미를 추정하지 마세요. `const T&in`은 입력 참조를 읽고, `T&out`은 출력을 반환하며, `T&inout`은 호출자 값을 변경할 수 있습니다. 실제 호출 시그니처에 맞추고
reflected 콜백에 참조 지정자를 임의로 붙이지 마세요.

`TMap.Find(Key, OutValue)`는 bool을 반환하고 `OutValue`로 복사하므로 그 복사본을 수정해도 맵은 바뀌지 않습니다. 이 바인딩에서 `Map[Key]`와 `FindOrAdd`는 참조를 반환합니다. 없는 키를 `[]`로 읽으면 예외가
발생하므로 `Find`, `Contains` 또는 의도적인 `FindOrAdd`를 사용하세요. 재할당·삭제는 보관한 참조를 무효화할 수 있습니다. 맵 반복에서는 키가 const이고, 변경 가능한 맵의 값은 수정할 수 있습니다.

이 차이를 보여 주는 완결된 독립 구조체와 함수입니다. 첫 출력에서 로컬 복사본은 100, 맵은 10이며 최종 맵 값은 26입니다. 이는 의도한 의미이며 이 문서의 코드를 실제 실행했다는 주장은 아닙니다.

```angelscript
struct FCounterValue
{
	int Count = 0;
}

FCounterValue MakeChangedCopy(const FCounterValue&in Input)
{
	FCounterValue LocalCopy = Input;
	LocalCopy.Count = 99;
	return LocalCopy;
}

void ChangeReference(FCounterValue&inout Value)
{
	Value.Count += 1;
}

void UpdateCounterMap()
{
	TMap<FName, FCounterValue> Values;
	FCounterValue Initial;
	Initial.Count = 10;
	Values.Add(n"Sample", Initial);

	FCounterValue Copy;
	if (Values.Find(n"Sample", Copy))
	{
		Copy = MakeChangedCopy(Copy);
		ChangeReference(Copy);
		const int StoredCount = Values[n"Sample"].Count;
		Print(f"Local={Copy.Count}, Map={StoredCount}");
	}

	// Do not structurally modify the map while this reference is in use.
	FCounterValue& Stored = Values.FindOrAdd(n"Sample");
	Stored.Count = 25;

	for (auto Entry : Values)
	{
		Entry.Value.Count += 1;
		// Entry.Key is const.
	}
}
```

## 자주 생기는 오진

| 증상 | 확인할 원인 | 수정 |
| --- | --- | --- |
| 숫자 인자인데 콜백 바인딩이 거부됨 | `float` 별칭과 delegate의 네이티브 폭이 다름 | 바인딩의 `float32` 또는 `float64`와 정확히 일치시키기 |
| out 변수 수정 후 맵 데이터가 그대로임 | `Find`가 구조체를 복사함 | 유효한 저장 참조를 수정하거나 변경한 값을 다시 쓰기 |
| 반복자 키에 대입할 수 없음 | 해시 규칙을 보존하기 위해 키가 const임 | 무효화된 반복 중이 아닌 시점에 의도적으로 삭제·재삽입하기 |
| 배열에서 가져온 선형 검색 가정 | `TMap.Find`는 키 해시 조회를 사용함 | 선형인 `TArray.FindIndex`와 구분하고 최악의 경우까지 상수 시간이라고 주장하지 않기 |
| `Initialization lists cannot be used with 'const TArray<FName>'` | 바인딩된 `TArray`에 초기화 목록 생성자가 없음 | const가 아닌 로컬 변수를 선언하고 `Add`로 채우기; 주변 코드의 배열 구성 방식 따르기 |


## 소스와 검증 범위

2026-10-22에 UE 5.8.1, 네이티브 플러그인 BuildId `3c6a4263-8d5a-4b20-a676-deccedf5fce2`의 소스로 확인했습니다. 아래 경로는 선택한 엔진 루트 기준입니다. 특정 소스 스냅샷이며 모든 버전의 호환성을 보장하지 않습니다.
근거 수준은 [API 진단과 리로드](api-diagnostics-reload.ko-KR.md), 실제 실행 관측은 [Sandbox 기록](SANDBOX_VALIDATION.ko-KR.md)을 확인하세요.

- `Plugins/Angelscript/Source/AngelscriptCode/Public/AngelscriptSettings.h`: `bScriptFloatIsFloat64`.
- `Plugins/Angelscript/Source/AngelscriptCode/Private/Binds/Bind_Primitives.cpp`: 기본 타입 등록.
- `Plugins/Angelscript/Source/AngelscriptCode/Private/Binds/Bind_TMap.cpp`: `FAngelscriptMapBinds::Find`, `TMapIterator` 등록.
- `Plugins/Angelscript/Source/AngelscriptCode/Public/AngelscriptBinds/Bind_TMap.h`: `FMapOperations::FindPairIndex`, `OpIndex_Template`.
- `Plugins/Angelscript/Source/AngelscriptCode/Private/Preprocessor/AngelscriptPreprocessor.cpp`: `GenerateFormatString`.

- `Plugins/Angelscript/ThirdParty/source/as_builder.cpp`: 스크립트 값 인자의 const 참조 변환.
- `Plugins/Angelscript/ThirdParty/source/as_compiler.cpp`: 인자의 읽기 전용 처리.
