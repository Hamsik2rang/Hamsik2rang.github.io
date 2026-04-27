---
title: "Unreal Engine 5 분석 - Proxy"
author: "Yongsik Im"
authorAvatarPath: "/images/profile.jpg"
date: "2026-04-27"
summary: "언리얼 엔진 5.7 소스 코드 분석"
description: ""
toc: true
readTime: true
autonumber: true
math: true
tags: ["Graphics", "GPU", "AI"]
showTags: true
hideBackToTop: false
draft: true
---
## Proxy?
![image_0](/post_images/unreal-analysis-proxy/001.png)
Proxy는 Game Thread의 객체 상태를 Render Thread가 안전하게 소비할 수 있도록 옮겨 놓은 렌더링 전용 객체이다. 언리얼에서 흔히 Proxy라고 하면 `FPrimitiveSceneProxy`, `FLightSceneProxy`처럼 스레드 경계를 넘기 위해 만든 각종 scene proxy들을 묶어서 부르는 경우가 많다.  
이 글에서는 그 중에서도 `UPrimitiveComponent`가 생성하는 `FPrimitiveSceneProxy`를 중심으로 살펴본다.

렌더링에 필요한 데이터들을 담는 객체이기 때문에, 모든 컴포넌트가 동일한 형태의 Proxy를 가지는 것은 아니다. 대표적인 예시는 다음과 같다.

|Game Thread 객체|Render Thread Proxy 클래스|설명|
|---|---|---|
|`UStaticMeshComponent`|`FStaticMeshSceneProxy`|정적 메쉬의 Vertex/Index Buffer, 머티리얼 렌더링 상태, LOD 전환 및 배칭 처리 등을 담당한다.|
|`USkeletalMeshComponent`|`FSkeletalMeshSceneProxy`|Bone Transform 결과를 렌더링 경로에 전달하고, 스키닝된 메쉬의 동적 데이터 갱신을 담당한다.|
|`UInstancedStaticMeshComponent`|`FInstancedStaticMeshSceneProxy`|인스턴스별 Transform 및 인스턴싱 렌더링 데이터를 관리한다.|
|`ULightComponent`|`FLightSceneProxy`|광원의 종류, 색상, 감쇠, 섀도우 관련 파라미터 등 라이팅에 필요한 상태를 관리한다.|

여기서 한 가지 주의할 점이 있다. `ULightComponent`는 `UPrimitiveComponent`의 자식이 아니라 `USceneComponent` 계열이다. 따라서 "모든 Proxy는 `UPrimitiveComponent`로부터 생성된다" 라고 이해하면 정확하지 않다. `Primitive Scene Proxy`와 `Light Scene Proxy`는 비슷한 목적을 가지지만 서로 다른 계열의 Proxy이다.

## 왜 Proxy가 필요한가?
언리얼은 기본적으로 Game Thread와 Render Thread를 분리해서 동작한다. Game Thread에서는 액터와 컴포넌트의 생명주기, 트랜스폼, 게임 로직이 계속 변하고, Render Thread는 그 결과를 바탕으로 한 템포 늦은 안정적인 스냅샷을 소비한다.

만약 Render Thread가 매 프레임 직접 `UObject`를 뒤지며 렌더링에 필요한 정보를 읽는 구조라면 다음과 같은 문제가 생긴다.

1. Game Thread가 객체를 수정하는 도중 Render Thread가 같은 데이터를 읽게 될 수 있다.
2. 렌더링에 필요하지 않은 `UObject`의 무거운 상태를 그대로 노출하게 된다.
3. 렌더러 입장에서는 "그리기 위한 데이터"와 "게임플레이를 위한 데이터"가 강하게 결합된다.

따라서 언리얼은 렌더링에 필요한 정보만 별도의 Proxy에 모아 두고, Render Thread는 이 Proxy를 기준으로 씬을 관리한다. 즉, Proxy는 단순한 복사본이라기보다 스레드 경계를 넘기 위한 렌더링 전용 인터페이스에 가깝다.

## Proxy 생성 흐름
Actor가 스폰된 뒤 `UPrimitiveComponent`의 Proxy가 Scene에 붙기까지의 큰 흐름은 다음과 같다.

```cpp
UWorld::SpawnActor()
↓
AActor::PostSpawnInitialize()
↓
AActor::RegisterAllComponents()
↓
AActor::IncrementalRegisterComponents(0)
↓
UActorComponent::RegisterComponentWithWorld()
↓
UActorComponent::ExecuteRegisterEvents()
↓
UPrimitiveComponent::CreateRenderState_Concurrent()
↓
FScene::AddPrimitive()
↓
FScene::BatchAddPrimitivesInternal()
```

![image_1](/post_images/unreal-analysis-proxy/003.png)
![image_2](/post_images/unreal-analysis-proxy/004.png)

실제 코드 기준으로 보면 `AActor::PostSpawnInitialize()` 내부에서 조건이 맞으면 `RegisterAllComponents()`가 호출된다. 이후 `RegisterAllComponents()`는 `PreRegisterAllComponents()`를 수행한 뒤 `IncrementalRegisterComponents(0)`를 호출한다. 즉, 초안의 흐름 자체는 큰 틀에서 맞지만, 중간의 `if (...) then ...` 형태보다는 위와 같이 함수 호출 흐름으로 정리하는 편이 더 정확하다.

이후 각 컴포넌트는 `RegisterComponentWithWorld()`를 거치며, `ExecuteRegisterEvents()` 내부에서 `ShouldCreateRenderState()`가 참이면 `CreateRenderState_Concurrent()`가 호출된다.

```cpp
if(FApp::CanEverRender() && !bRenderStateCreated && WorldPrivate->Scene && ShouldCreateRenderState())
{
    CreateRenderState_Concurrent(Context);
}
```

즉, Proxy 생성은 단순히 컴포넌트가 존재한다고 해서 일어나는 것이 아니라, 월드에 등록되면서 렌더 상태를 만들 수 있는 조건이 충족될 때 시작된다.

## `CreateRenderState_Concurrent()`에서 일어나는 일
`UPrimitiveComponent::CreateRenderState_Concurrent()`는 bounds를 갱신한 뒤, 해당 컴포넌트가 Scene에 추가될 수 있으면 `FScene::AddPrimitive()`를 호출한다.

```cpp
if (ShouldComponentAddToScene())
{
    GetWorld()->Scene->AddPrimitive(this);
}
```

![image_3](/post_images/unreal-analysis-proxy/005.png)

여기서 중요한 점은, 이 함수가 곧바로 "Render Thread에서 모든 준비를 끝낸다" 는 뜻은 아니라는 것이다. 이 시점은 어디까지나 Game Thread에서 Scene 등록 절차를 시작하는 단계이다.

`FScene::AddPrimitive()`는 다시 `FScene::BatchAddPrimitivesInternal()`로 이어지며, 여기서 본격적으로 `CreateSceneProxy()`가 호출된다.

## `FScene::BatchAddPrimitivesInternal()`과 `SceneData`
`FScene::BatchAddPrimitivesInternal()`에서는 각 primitive에 대해 `FPrimitiveSceneInfoData& SceneData = Primitive->GetSceneData();` 를 가져온다.

![image_4](/post_images/unreal-analysis-proxy/006.png)

이 `SceneData`를 단순히 "렌더러가 컴포넌트에게 피드백할 정보"만 담는 구조체라고 보기는 어렵다. 실제로는 primitive의 scene 등록 과정에서 Game Thread와 Render Thread가 함께 참조해야 하는 핵심 상태를 묶어 둔 저장소에 가깝다. 예를 들면 다음과 같은 것들이 들어간다.

1. 현재 `SceneProxy` 포인터
2. primitive id
3. attachment 관련 상태
4. owner의 last render time에 대한 포인터
5. always visible 여부 등 scene 관련 메타데이터

UE 5.7 코드에는 `UPrimitiveComponent` 내부의 기존 `SceneProxy` 포인터에 대해 다음과 같은 주석이 있다.

```cpp
/** The old primitive's scene info ptr, now superceded by the ptr in SceneData, but it's still here due to pervasive usage. */
FPrimitiveSceneProxy* SceneProxy;
```

즉, 예전부터 널리 쓰이던 `UPrimitiveComponent::SceneProxy` 포인터는 여전히 남아 있지만, 현재 기준으로는 `SceneData` 쪽 포인터가 더 중심적인 저장 위치라고 볼 수 있다. 초안에서 말한 "`SceneData`의 `SceneProxy`를 사용하는 것이 권장된다" 는 방향은 대체로 맞다. 다만 이것을 단순한 스타일 가이드 수준으로 적기보다는, 엔진 내부 데이터 소유권이 `SceneData` 쪽으로 이동하는 과정이라고 이해하는 편이 더 정확하다.

또한 `BatchAddPrimitivesInternal()` 내부에는 다음과 같은 체크가 있다.

```cpp
PrimitiveSceneProxy = Primitive->GetPrimitiveComponentInterface()->CreateSceneProxy();
check(SceneData.SceneProxy == PrimitiveSceneProxy);
```

이 코드는 `CreateSceneProxy()`가 호출되는 과정에서 `SceneData.SceneProxy`도 함께 올바르게 세팅되어야 함을 보여준다.

## Proxy가 만들어진 직후 실제로 생성되는 것들
`CreateSceneProxy()`가 반환되면, 그 다음 단계로 `FPrimitiveSceneInfo`가 생성된다.

```cpp
FPrimitiveSceneInfo* PrimitiveSceneInfo = new FPrimitiveSceneInfo(Primitive, this);
PrimitiveSceneProxy->PrimitiveSceneInfo = PrimitiveSceneInfo;
```

즉, Scene 등록에서 중요한 것은 Proxy 하나만 생기는 것이 아니다.

1. `FPrimitiveSceneProxy`: 렌더링 동작과 드로우 관련 데이터를 대표하는 객체
2. `FPrimitiveSceneInfo`: Renderer가 primitive를 scene 내부에서 추적하기 위한 관리 객체
3. `FPrimitiveSceneInfoData`: 컴포넌트, 프록시, 씬 사이에서 공유되는 상태 저장소

이 세 층을 구분해서 보면 코드가 훨씬 잘 읽힌다. 흔히 Proxy만 눈에 들어오지만, 실제 renderer의 scene 관리에서는 `FPrimitiveSceneInfo` 역시 매우 중요한 축이다.

그 뒤 `BatchAddPrimitivesInternal()`은 즉시 Render Thread에서 작업을 마무리하지 않고, 렌더 명령을 enqueue한다.

```cpp
ENQUEUE_RENDER_COMMAND(AddPrimitiveCommand)(...)
```

이 명령 안에서 Render Thread는 대략 다음 순서로 작업한다.

1. `PrimitiveSceneProxy->SetTransform(...)`
2. `PrimitiveSceneProxy->CreateRenderThreadResources(...)`
3. `AddPrimitiveSceneInfo_RenderThread(...)`

즉, Game Thread에서 Proxy 객체 자체를 만들고 scene 등록 명령을 준비한 다음, 실제 렌더 스레드 리소스 생성과 scene 편입 마무리는 Render Thread가 수행한다. 이 구분은 생각보다 중요하다. "Proxy 생성" 이라는 표현 하나로 뭉뚱그리면, 객체 생성과 렌더 리소스 생성, scene 편입 시점을 서로 헷갈리기 쉽기 때문이다.

## 왜 `FPrimitiveSceneInfoData`가 추가로 필요한가?
언뜻 보면 `UPrimitiveComponent`에도 `SceneProxy`가 있고, `FPrimitiveSceneInfo`에도 `Proxy`가 있는데 왜 굳이 `SceneData`까지 필요한지 의문이 들 수 있다.

이 구조는 대략 다음 문제를 풀기 위한 것으로 볼 수 있다.

1. 기존 `UPrimitiveComponent` 중심 API를 완전히 없애지 않고 유지해야 한다.
2. 한편으로는 renderer가 scene 등록 관련 상태를 더 일관된 저장소에서 다루고 싶다.
3. primitive component가 아닌 다른 scene primitive 기술 경로와도 맞물릴 수 있어야 한다.

즉 `SceneData`는 과거 구조를 완전히 깨지 않으면서도, scene 등록에 필요한 공유 상태를 한 군데로 모으기 위한 절충 지점처럼 보인다. UE 5.7에서 `FPrimitiveSceneDesc` 경로와 함께 읽어 보면 이러한 방향성이 더 분명하게 보인다.

## Proxy는 생성만 되는 것이 아니다
지금까지 본 내용은 어디까지나 Proxy의 **초기 생성** 흐름이다. 하지만 실제 엔진에서 Proxy는 한 번 만들어지고 끝나는 객체가 아니다. 컴포넌트의 상태 변화에 따라 **업데이트**, **재생성**, **제거**까지 포함한 생명주기를 가진다.

즉 `UPrimitiveComponent`의 렌더 상태는 대략 다음 네 가지 흐름으로 나누어 볼 수 있다.

1. 초기 생성: 컴포넌트가 register될 때 scene에 처음 추가된다.
2. 업데이트: 기존 Proxy를 유지한 채 transform 혹은 동적 데이터만 갱신한다.
3. 재생성: 기존 Proxy를 내리고 새 Proxy를 다시 만든다.
4. 제거: scene에서 primitive를 제거하고 render state를 파기한다.

## 업데이트 흐름
가장 흔한 업데이트는 transform 변경이다. 이 경우 항상 Proxy를 새로 만들지는 않는다. `UPrimitiveComponent::SendRenderTransform_Concurrent()`는 bounds를 갱신한 뒤 `Scene->UpdatePrimitiveTransform(this)`를 호출한다.

```cpp
UPrimitiveComponent::SendRenderTransform_Concurrent()
↓
FScene::UpdatePrimitiveTransform()
↓
ENQUEUE_RENDER_COMMAND(UpdateTransformCommand)
↓
FScene::UpdatePrimitiveTransform_RenderThread()
```

이 경로의 핵심은 **기존 Proxy를 유지한 채 Render Thread에 transform 업데이트만 전달한다**는 점이다. 즉, 단순한 위치/회전/스케일 변화는 보통 Proxy 재생성이 아니라 transform 업데이트로 처리된다.

물론 예외는 있다. `FScene::UpdatePrimitiveTransformInternal()`을 보면 `ShouldRecreateProxyOnUpdateTransform()`가 참인 경우 `RemovePrimitive()`와 `AddPrimitive()`를 다시 수행한다. 즉, 어떤 primitive는 transform 변경조차 단순 업데이트로 처리할 수 없고, 구조상 Proxy를 다시 만들어야 할 수도 있다.

또한 transform 말고도 동적 렌더 데이터 갱신이라는 별도 축이 있다. `UActorComponent::DoDeferredRenderUpdates_Concurrent()`는 dirty flag를 보고 `SendRenderTransform_Concurrent()`, `SendRenderDynamicData_Concurrent()` 등을 호출한다. 즉, 렌더 상태 갱신은 "매번 재생성"이 아니라 dirty flag 기반으로 세분화되어 있다.

## 재생성 흐름
Proxy 재생성은 기존 render state를 내린 뒤 다시 생성하는 흐름이다. 가장 대표적인 진입점은 `MarkRenderStateDirty()`이고, 실제 처리는 `DoDeferredRenderUpdates_Concurrent()` 안에서 `RecreateRenderState_Concurrent()`로 이어진다.

```cpp
MarkRenderStateDirty()
↓
UActorComponent::DoDeferredRenderUpdates_Concurrent()
↓
UActorComponent::RecreateRenderState_Concurrent()
↓
DestroyRenderState_Concurrent()
↓
CreateRenderState_Concurrent()
```

이 경로는 transform만 바꾸는 수준으로는 처리할 수 없는 변화, 즉 Proxy가 들고 있는 렌더링 표현 자체를 새로 구성해야 하는 경우에 사용된다. 예를 들어 머티리얼, 메쉬 표현, 렌더링 관련 플래그 변화처럼 "기존 Proxy를 유지한 채 일부 값만 수정하는 것"으로는 부족한 경우가 여기에 해당한다.

중요한 점은 재생성 역시 완전히 별개의 시스템이 아니라, 결국 앞서 본 생성/제거 경로를 조합한 것이라는 점이다. 즉 재생성은 개념적으로 보면 **제거 + 다시 생성**이다.

## 제거 흐름
컴포넌트가 unregister되거나 render state가 파괴되어야 할 때는 `DestroyRenderState_Concurrent()`가 호출되고, `UPrimitiveComponent`는 여기서 `Scene->RemovePrimitive(this)`를 수행한다.

```cpp
UPrimitiveComponent::DestroyRenderState_Concurrent()
↓
FScene::RemovePrimitive()
↓
Render Thread에서 scene 제거 처리
```

또한 `UPrimitiveComponent::OnUnregister()`에서는 `World->Scene->ReleasePrimitive(this)`를 호출해 scene과의 연결을 정리한다. 즉 unregister는 단순히 UObject 생명주기 차원의 해제가 아니라, 렌더러 입장에서도 해당 primitive를 scene에서 분리하는 작업을 동반한다.

이렇게 보면 remove 경로는 생성 경로의 반대 방향이라고 이해할 수 있다. 생성 시에는 `AddPrimitive()`를 통해 scene에 편입되고, 제거 시에는 `RemovePrimitive()`를 통해 scene에서 빠진다.

## 라이프사이클 관점에서 다시 보기
결국 `UPrimitiveComponent`의 Proxy 생명주기를 가장 간단히 정리하면 다음과 같다.

```cpp
[초기 생성]
RegisterComponentWithWorld()
→ CreateRenderState_Concurrent()
→ AddPrimitive()
→ CreateSceneProxy()

[업데이트]
SendRenderTransform_Concurrent()
→ UpdatePrimitiveTransform()

[재생성]
MarkRenderStateDirty()
→ RecreateRenderState_Concurrent()
→ DestroyRenderState_Concurrent()
→ CreateRenderState_Concurrent()

[제거]
DestroyRenderState_Concurrent()
→ RemovePrimitive()
```

즉 Proxy를 이해할 때 중요한 것은 "언제 처음 생성되는가"만이 아니다. 실제 엔진에서는 컴포넌트 변화의 종류에 따라 다음처럼 서로 다른 비용의 경로가 선택된다.

1. 단순 transform 변화면 update
2. 표현 자체가 바뀌면 recreate
3. scene에서 빠지면 remove
4. 처음 등록되면 create

이 구분을 머릿속에 넣고 코드를 보면, 왜 어떤 변경은 가볍고 어떤 변경은 비싼지, 그리고 어떤 변경이 Render Thread에 어느 수준까지 영향을 주는지 훨씬 잘 보이게 된다.

## 정리
정리하면, `FPrimitiveSceneProxy`는 단순히 "컴포넌트의 렌더 스레드 복사본" 이라고만 설명하기에는 조금 부족하다. 이 표현은 직관적으로는 맞지만, 실제 엔진 코드 기준으로 보면 그보다 약간 더 구조적인 역할을 맡고 있다.

1. `UPrimitiveComponent`가 월드에 등록되면 렌더 상태 생성 절차가 시작된다.
2. 그 과정에서 `CreateSceneProxy()`가 호출되어 `FPrimitiveSceneProxy`가 생성된다.
3. 이어서 `FPrimitiveSceneInfo`가 생성되고, 관련 공유 상태는 `FPrimitiveSceneInfoData`에 연결된다.
4. 최종적인 render resource 생성과 scene 편입은 Render Thread에 enqueue된 명령에서 마무리된다.

결국 Proxy는 Game Thread의 객체를 Render Thread가 직접 들여다보지 않도록 분리해 주는 핵심 경계 객체이며, `FPrimitiveSceneInfo`, `FPrimitiveSceneInfoData`와 함께 scene 시스템의 한 축을 이룬다.

이 흐름을 더 이어서 본다면 다음 주제들도 자연스럽게 연결된다.

1. `MarkRenderStateDirty()`가 호출되면 Proxy는 어떻게 재생성되는가?
2. Transform 변경은 왜 Proxy 재생성 없이 `UpdatePrimitiveTransform()`으로 처리되는가?
3. Static mesh와 skeletal mesh proxy는 어떤 데이터를 서로 다르게 소유하는가?
4. Render Thread에 enqueue된 명령이 실제로 어떤 프레임 지연을 만드는가?

이 글은 Proxy가 왜 존재하는지, 그리고 언제 scene에 편입되는지를 보는 정도에서 마무리했다. 이후에는 `MarkRenderStateDirty()`, `RecreateRenderState_Concurrent()`, `UpdatePrimitiveTransform()`까지 따라가 보면 Proxy의 생명주기를 더 입체적으로 이해할 수 있을 것이다.

