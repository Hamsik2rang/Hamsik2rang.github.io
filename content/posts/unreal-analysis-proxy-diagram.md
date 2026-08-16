---
title: "Unreal Engine 5 분석 - Proxy (다이어그램)"
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
draft: true
---
## Proxy 생성 도식

`UPrimitiveComponent`의 초기 Proxy 생성 흐름을 최대한 단순화하면 다음과 같다.

```cpp
[Game Thread]

UPrimitiveComponent Register
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
↓
CreateSceneProxy()
↓
FPrimitiveSceneProxy 생성
↓
FPrimitiveSceneInfo 생성
↓
ENQUEUE_RENDER_COMMAND(AddPrimitiveCommand)

============================================
Thread Boundary
============================================

[Render Thread]

AddPrimitiveCommand 실행
↓
FPrimitiveSceneProxy::SetTransform()
↓
FPrimitiveSceneProxy::CreateRenderThreadResources()
↓
FScene::AddPrimitiveSceneInfo_RenderThread()
↓
Scene 편입 완료
```

좀 더 짧게 요약하면 다음과 같이 볼 수 있다.

1. 컴포넌트가 register되면 render state 생성 절차가 시작된다.
2. `CreateSceneProxy()`를 통해 `FPrimitiveSceneProxy` 객체 자체는 Game Thread에서 생성된다.
3. 이후 Render Thread에 명령이 enqueue된다.
4. Render Thread는 transform 설정, render resource 생성, scene 등록 마무리를 수행한다.

즉, Proxy의 **객체 생성**과 **렌더러 편입 완료**는 같은 시점이 아니다.

- `FPrimitiveSceneProxy` 인스턴스 생성: Game Thread
- Render resource 생성: Render Thread
- Scene 편입 마무리: Render Thread

따라서 "Proxy는 어디서 생성되는가?" 라는 질문에는 보통 다음처럼 답하는 것이 가장 정확하다.

> `FPrimitiveSceneProxy` 객체 자체는 Game Thread에서 생성되며, Render Thread에서 리소스 초기화와 scene 편입이 완료된다.

## 진입 경로

이 흐름이 반드시 `SpawnActor()`로만 시작되는 것은 아니다. 핵심 조건은 **`UPrimitiveComponent`가 World에 등록되는가** 이다.

대표적인 진입 경로는 다음과 같다.

1. `SpawnActor()` 이후 `RegisterAllComponents()`가 호출되는 경우
2. 레벨 로드 후 `AddToWorld()` 과정에서 기존 액터들의 컴포넌트가 등록되는 경우
3. 스트리밍 레벨이 visible 상태가 되면서 컴포넌트가 등록되는 경우
4. 런타임에 `NewObject<>()` 등으로 생성한 컴포넌트를 수동으로 `RegisterComponentWithWorld()` 하는 경우

즉, Proxy 초기 생성의 핵심 키워드는 `SpawnActor()`가 아니라 `RegisterComponentWithWorld()`라고 보는 편이 더 정확하다.
