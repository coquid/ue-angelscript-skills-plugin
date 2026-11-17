# 에디터 전용 스크립팅

에디터 툴, 메뉴 확장, `CallInEditor` 버튼, 에디터 전용 멤버를 작성하거나, 쿡·커맨드렛이 에디터 전용 코드 오류를 보고할 때 읽으세요.

## 전처리기 플래그

| 플래그 | 참인 경우 |
| --- | --- |
| `EDITOR` | 커맨드렛이 아닌 에디터 프로세스(또는 강제 지정)이고, 쿡 시뮬레이션 중이 아님 |
| `EDITORONLY_DATA` | 에디터 전용 데이터가 컴파일되어 있고 게임·데디케이티드 서버 프로세스가 아님(또는 `EDITOR`가 참) |
| `COOK_COMMANDLET` | 쿡 커맨드렛 실행 중 |
| `RELEASE` | Shipping 또는 Test 빌드 |
| `TEST` | Shipping이 아님 |
| `WITH_SERVER_CODE` | 서버 코드가 컴파일됨 |

프로젝트는 `PreprocessorFlags` 설정으로 플래그를 추가할 수 있습니다. `#if`, `#elif`, `#else`, `#endif`에는 플래그 하나만, 필요하면 부정(`#if !EDITOR`)과
함께 씁니다. `&&`, `||`, 알 수 없는 플래그는 `Invalid preprocessor condition`이므로 블록을 중첩하세요. 내장 플래그에 `#ifdef`/`#ifndef`를 쓰면
오류입니다.

`EDITOR`는 쿡과 `AngelscriptTest`를 포함한 **커맨드렛에서 거짓**입니다. 그래서 에디터 전용 코드 실수가 커맨드렛에서는 다르게 통과하거나 실패할 수
있습니다. 커맨드렛에서 에디터 코드를 컴파일하려면 `-as-force-preprocess-editor-code`, 에디터에서 쿡 경로를 확인하려면 `-as-simulate-cooked`를
쓰세요.

`UPROPERTY`/`UFUNCTION` 선언은 `EDITOR`, `EDITORONLY_DATA` 또는 프로젝트가 추가한 플래그 안에만 둘 수 있으며, 그 안의 멤버는 에디터 전용이
됩니다. `TEST`/`RELEASE` 조건은 함수 본문 안에 두세요. `EditorOnly` 프로퍼티 지정자는 없습니다.

## 에디터 전용 코드 판별

에디터 전용 C++ 타입(에디터 패키지, `/Source/Editor/`, `/Plugins/Editor/`, AngelScript 에디터 모듈, 설정에 등록된 패키지)과 그 멤버는
`#if EDITOR` 블록 밖에서 쓸 수 없습니다: `Cannot use editor-only type/function/property X outside of an EDITOR block`. 이 검사에는
`EDITORONLY_DATA`가 아니라 `#if EDITOR`만 인정됩니다. 에디터 전용 부모 함수의 override도 에디터 전용이어야 합니다.

에디터 스크립트가 꺼져 있으면 파일 탐색이 어느 깊이에서든 이름이 정확히 `Editor`, `Examples`, `Dev`, `Developers`인 폴더를 건너뜁니다. 모듈
경로에 `Editor` 세그먼트가 있으면 에디터 전용 모듈입니다. 파일을 옮겨도 쿡이 깨지지 않도록 그런 파일 안에도 `#if EDITOR`를 유지하세요.

## 메뉴 확장

아래 에디터 기반 클래스의 스크립트 하위 클래스는 모두 자동 등록되며, 반환값이 없는 `CallInEditor` 함수가 메뉴 항목이 됩니다.

| 기반 클래스 | 기본 위치 | 선택 관련 헬퍼 |
| --- | --- | --- |
| `UScriptEditorMenuExtension` | `ExtensionMenu`/`ExtensionPoint` 프로퍼티(기본은 Tools 메뉴) | `ShouldExtend()`, `GetToolMenuContext(Class)` |
| `UScriptActorMenuExtension` | 레벨 뷰포트 컨텍스트 메뉴의 `ActorPreview` 섹션 | `SupportedClasses`, `SupportsActor(AActor)`, `GetSelectedActor(s)()` |
| `UScriptAssetMenuExtension` | 콘텐츠 브라우저 에셋 컨텍스트 메뉴 | `SupportedClasses`, `SupportsAsset(FAssetData)`, `GetSelectedAsset(s)()` |

- 액터 메뉴 함수의 첫 매개변수가 액터 타입이면 선택된 액터 중 맞는 것마다 한 번씩 실행되고, 맞는 액터가 없으면 항목이 숨겨집니다. 에셋 메뉴
  함수는 첫 매개변수로 `FAssetData` 또는 지원 에셋 오브젝트를 받습니다. 그 밖의 매개변수는 입력 대화상자를 띄웁니다.
- 항목 meta: `Category`(`|`로 하위 메뉴), `EditorIcon`, `ToolbarLabel`, `SortOrder`, `ActionCanExecute`/`ActionIsVisible`/`ActionIsChecked`
  (bool 함수 이름).
- 클릭할 때마다 새 확장 인스턴스가 만들어지고(상태 유지 안 됨) scoped transaction 안에서 실행됩니다. 변경 전에 `Modify()`를 호출해야 undo에
  기록됩니다.

`UScriptEditorSubsystem`은 `ShouldCreateSubsystem`, `Initialize`, `Deinitialize`, `Tick` override(스크립트 이름에서 `BP_` 제거)와 생성된
`Get()`을 제공합니다. `EditorPrompt::`는 구조체·함수 입력 대화상자를 제공합니다.

## CallInEditor와 에디터 작업

액터·컴포넌트의 `UFUNCTION(CallInEditor)`는 매개변수와 반환값이 없을 때만 Details 패널 버튼으로 보입니다. 다른 함수는 매개변수 없는 `void`
함수로 감싸세요. 이 지정자만으로는 쿡 빌드에서 함수가 제거되지 않습니다.

자주 쓰는 에디터 작업: `Modify()`, `MarkPackageDirty()`, `System::BeginTransaction`/`TransactObject`/`EndTransaction`/`CancelTransaction`,
`GetActorLabel()`/`SetActorLabel()`. `UEditorActorSubsystem`(선택), `UEditorAssetSubsystem`(저장) 같은 에디터 서브시스템은 바인딩 규칙상
`::Get()`으로 접근합니다. 쓰기 전에 대상 빌드에서 정확한 함수를 확인하세요.

독립적으로 완결된 예제입니다. `Editor` 폴더나 다른 에디터 전용 모듈에 두세요.

```angelscript
#if EDITOR
class UActorReviewMenu : UScriptActorMenuExtension
{
	UFUNCTION(CallInEditor, Category = "Review Tools")
	void MarkAsReviewed(AActor Actor)
	{
		Actor.Modify();
		Actor.Tags.AddUnique(n"Reviewed");
		Actor.SetActorLabel("REVIEWED_" + Actor.GetActorLabel());
	}

	UFUNCTION(CallInEditor, Category = "Review Tools")
	void RenameActor(AActor Actor, FString NewLabel)
	{
		Actor.Modify();
		Actor.SetActorLabel(NewLabel);
	}
}
#endif
```

`MarkAsReviewed`는 선택된 액터마다 한 번씩 실행됩니다. `RenameActor`는 `NewLabel` 입력 대화상자를 띄웁니다.

## 에디터 전용 코드 실패 진단

| 증상 | 가능한 원인 | 조치 |
| --- | --- | --- |
| `Cannot use editor-only type/function/property X outside of an EDITOR block` | 런타임 코드에서 에디터 API 사용 | `#if EDITOR`로 감싸거나 에디터 전용 폴더·모듈로 이동 |
| `Cannot put a UPROPERTY or UFUNCTION inside preprocessor conditions other than EDITOR...` | `#if TEST`/`#if RELEASE` 안의 멤버 | 조건을 함수 본문 안으로 이동 |
| `Invalid preprocessor condition` | `&&`, `\|\|` 또는 알 수 없는 플래그 | `#if` 하나에 플래그 하나, 블록 중첩 |
| 에디터 코드 오류가 에디터에서만 또는 쿡에서만 나타남 | 커맨드렛에서 `EDITOR`가 거짓 | `-as-force-preprocess-editor-code`와 `-as-simulate-cooked`로 확인 |
| `CallInEditor` 함수의 Details 버튼이 없음 | 매개변수나 반환값이 있음 | 매개변수 없는 `void` 함수로 감쌈 |
| 메뉴 항목이 없거나 undo가 안 됨 | 반환값 있음, 첫 매개변수가 선택과 불일치, `Modify()` 누락 | `void` 반환, 타입이나 `SupportsActor` 수정, 먼저 `Modify()` 호출 |

## 소스와 검증 범위

2026-11-16에 UE 5.8.1, 네이티브 플러그인 BuildId `343c3a9c-ab61-4b28-b709-cd650e59f32e`의 소스로 확인했습니다. 경로는 엔진 루트 기준입니다.
예제는 에디터에서 실행하지 않았습니다.

- `Plugins/Angelscript/Source/AngelscriptCode/Private/Preprocessor/AngelscriptPreprocessor.cpp`: 플래그, 조건 문법, 멤버 제한, `CallInEditor`,
  에디터 전용 모듈.
- `Plugins/Angelscript/Source/AngelscriptCode/Private/AngelscriptManager.cpp`: `bUseEditorScripts`, 명령줄 스위치, 건너뛰는 폴더.
- `Plugins/Angelscript/Source/AngelscriptCode/Private/Binds/Bind_BlueprintType.cpp`, `Plugins/Angelscript/ThirdParty/source/as_builder.cpp`:
  에디터 전용 타입 판별과 오류.
- `Plugins/Angelscript/Source/AngelscriptEditor/Public/EditorMenuExtensions/`: `ScriptEditorMenuExtension`, `ScriptActorMenuExtension`,
  `ScriptAssetMenuExtension`, 입력 대화상자.
- `Plugins/Angelscript/Source/AngelscriptEditor/Public/BaseClasses/ScriptEditorSubsystem.h`: 에디터 서브시스템 이벤트.
- `Source/Editor/PropertyEditor/Private/PropertyCustomizationHelpers.cpp`: Details 패널 `CallInEditor` 버튼.
- `Source/Runtime/Engine/Classes/Kismet/KismetSystemLibrary.h`: 트랜잭션 함수.
