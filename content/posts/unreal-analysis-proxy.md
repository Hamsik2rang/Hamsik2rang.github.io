---
title: "Unreal Engine 5 분석 - Proxy"
author: "Yongsik Im"
authorAvatarPath: "/images/profile.jpg"
date: "2026-04-26"
summary: ""
description: ""
toc: true
readTime: true
autonumber: true
math: true
tags: ["Graphics", "GPU", "AI"]
showTags: true
hideBackToTop: false
---
## Proxy?
![image_0](/post_images/unreal-analysis-proxy/001.png)
Proxy는 Game Thread의 `UObject`에 대응되는 Render Thread의 mirroring 객체이다.  
렌더링에 필요한 데이터들을 담는 객체이기 때문에 당연하게도 모든 컴포넌트가 Proxy 객체를 정의하고 있지는 않으며, 다음과 같은 객체들이 Proxy 데이터를 가지는 대표적인 객체들의 예이다.
|Component|렌더 스레드 Proxy클래스|설명|
|---|---|---|
|UStaticMeshComponent | FStaticMeshSceneProxy | 정적 메쉬의 Vertex/Index Buffer 및 머티리얼 렌더링 상태 관리, LOD전환 및 배칭 처리 등|
|USkeletalMeshComponent | FSkeletalMeshSceneProxy | 스켈레탈 애니메이션의 Bone Transform 데이터를 렌더 스레드로 전송, Skinned Mesh의 동적 Vertex 업데이트 등|
|UInstancedStaticMeshComponent | FInstancedStaticMeshSceneProxy | 인스턴싱 처리, 인스턴스별 Transform Buffer 관리|
|ULightComponent | FLightSceneProxy | 광원의 형태(Point, Spot, Directional 등), 색상, 감쇠 등의 라이팅 및 쉐도우 맵 렌더링 파라미터 관리|

보면 대부분 `UPrimitiveComponent`의 자식 컴포넌트들인데, `ULightComponent`처럼 `USceneComponent`의 자식 컴포넌트도 Proxy를 정의하고 있다. 따라서 '특정 객체의 Derived 객체들'이 가진다는 규칙보다는 '렌더링에 사용되는 객체들'이 가진다고 이해하는 편이 낫다.

## Proxy 생성 흐름
```
UWorld::SpawnActor()
↓
/*Called after the actor is spawned in the world.  
Responsible for setting up actor for play.*/
AActor::PostSpawnInitialize()
↓
AActor::RegisterAllComponent()
↓
if (Actor::PreRegisterAllCOmponents() && HasPreRegisteredAllComponents())
than: AActor::IncrementalRegisterComponents(0)
↓
//////////////////////////////
↓
UActorComponent::RegisterComponentWithWorld()
↓
UActorComponent::ExecuteRegisterEvents()
↓
UPrimitiveComponent::CretaeRenderState_Concurrent()
↓
...

```
위는 Actor가 생성되는 시점의 코드 흐름인데, 실제로는 Actor의 최초 생성 외에도 스트리밍에 의해 레벨에 Actor가 로드되거나, 이미 존재하는 액터에 새 컴포넌트가 추가되는 경우 등의 흐름도 있다. 중요한 점은 `RegisterComponent`, 즉 컴포넌트가 등록되는 시점이 Proxy 객체의 관점에서는 생성이 시작되는 부분이라 보는 것이 좋다.  
![image_1](/post_images/unreal-analysis-proxy/003.png)
![image_2](/post_images/unreal-analysis-proxy/004.png)

```
UPrimitiveComponent::CretaeRenderState_Concurrent()
↓
FScene::AddPrimitive()
↓
FScene::BatchAddPrimitivesInternal()
↓
...

```
![image_3](/post_images/unreal-analysis-proxy/005.png)
`FScene::BatchAddPrimitivesInternal()` 에서 `SceneData`를 이용하는데, 이는 `FPrimitiveSceneInfoData`타입의 객체이다.
![image_4](/post_images/unreal-analysis-proxy/006.png)
이 객체는 렌더스레드/렌더러(혹은 Scene Primitive)가 게임스레드/Component에게 제공(피드백)해줘야 할 정보들을 담은 구조체이다.  
이 안에도 `SceneProxy`가 있고, `UPrimitiveComponent`에도 `SceneProxy`가 있는데, UE 5.7 기준 현재로서는 이 `SceneData`에 담긴 `SceneProxy`를 사용하는 것이 권장되며, 다음과 같이 `UPrimitiveComponent::FPrimitiveSceneProxy`쪽에도 주석으로 표기되어 있는 내용이다.
![image_5](/post_images/unreal-analysis-proxy/007.png)
-TBA-

