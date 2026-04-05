---
title: "No Graphics API"
author: "Yongsik Im"
authorAvatarPath: "/images/profile.jpg"
date: "2025-12-25"
summary: "[Sebastian Aaltonen - No Graphics API](https://www.sebastianaaltonen.com/blog/no-graphics-api)를 번역한 글입니다"
description: ""
toc: true
readTime: true
autonumber: true
math: true
tags: ["Graphics", "GPU"]
showTags: true
hideBackToTop: false
draft: false
---
* 이 포스트는 [해당 글](https://www.sebastianaaltonen.com/blog/no-graphics-api)을 번역해 옮겼습니다. 번역 과정에서 일부 의역을 포함했습니다.
* 번역 과정에서 'Modern'을 문맥에 따라 '최신', '현대', '모던'(그래픽스 API를 지칭할 때 한정) 으로 혼용합니다.
# No Graphics API
## 소개
그래픽스 API, 셰이더 프레임워크 그리고 드라이버들의 복잡성이 지난 수십년간 빠르게 증가했습니다. 파이프라인 상태 객체(Pipeline State Object, PSO) 폭발은 더 이상 손쓸 수 없을 지경입니다. 어쩌다 100GB에 달하는 로컬 셰이더 파이프라인 캐시와 이들을 호스팅하는 거대한 클라우드 서버가 탄생하게 된 걸까요? 이젠 어떻게 우리가 GPU와 상호작용하기 위한 추상화와 API 표면을 줄일 수 있을지에 대한 방법을 논의하기 시작해야 할 때입니다.

## 업계에서의 저수준 그래픽스 API들의 변화
십년 전, 새로운 저수준 PC 그래픽스 API의 소개와 함께 실시간 컴퓨터 그래픽스 분야에 중대한 변화가 일어났습니다. AMD는 2013년 Xbox One과 Playstation 4의 부품 계약을 모두 따냈습니다. 그들의 새로운 GCN(Graphics Core Next) 아키텍쳐는 사실상 AAA(**역주: Triple-A 라고 부르며, 블록버스터급 규모를 의미합니다*) 게임 개발을 위한 주력 플랫폼이 되었습니다. 해당 시점의 PC 그래픽스 API들이었던 DirectX 11과 OpenGL 4.5는 무거운 드라이버 오버헤드가 있었으며 싱글 스레드 렌더링을 위해 설계되어 있었습니다. AAA 게임 개발자들은 더 높은 성능의 API를 요구했습니다. DICE는 아예 AMD GCN에 특화된 PC 그래픽스 API인 Mantle 제작에 함께 참여하게 되었으며, 이에 응답하듯 Microsoft, Khronos, 그리고 Apple은 그들만의 저수준 API를 개발하기 시작했고, 그 결과 각각 DirectX 12, Vulkan, Metal이 탄생하게 되었습니다.  

이러한 새로운 저수준 API들에 대한 초기 반응은 엇갈렸습니다. 합성 벤치마크들과 데모들은 이들이 이전 API들에 비해 상당한 성능 향상을 이루어냈음을 보여주었지만, Unreal Engine, Unity와 같은 주요한 게임 엔진들에서는 성능 향상이 보이지 않았습니다. 제가 Ubisoft에서 일할 때, 저희 팀은 기존에 개발되어 있던 DirectX 11 기반 렌더러를 DirectX 12로 포팅할 때 종종 성능 저하가 발생하는 것을 발견했습니다. 이는 무언가 잘못된 것이었습니다.

기존에 존재하던 고수준 API들(**역주: DirectX 11, OpenGL 등*)은 최소한의 영구(Persistent) 상태만을 제공하며, 세부적인 상태 설정기와 개별적인 데이터 입력은 드로우 콜 호출 직전에 셰이더에 바인딩됩니다. 새로운 저수준 API 들은 셰이더 파이프라인 상태와 바인딩들을 영구 객체로 미리 묶음으로써 드로우 콜의 비용을 더 낮추는 것을 목표로 합니다. 이전까지의 GPU 아키텍쳐들은 매우 이질적(Heterogeneous)이었습니다. 데이터 리매핑, 유효성 검증(validation) 및 사전 업로드를 수행하는 것이 큰 도움이 되었습니다. 그러나 기존 게임 엔진들의 렌더링 하드웨어 아키텍쳐(RHI, Rendering Hardware Architecture)는 세밀한 즉각적(immediate) 모드 렌더링을 위해 설계된 반면, 새로운 저수준 API는 데이터를 영구 객체로 묶어야 했습니다.  

이러한 비호환성을 해결하기 위해 새로운 저수준 그래픽스 리매핑 레이어가 RHI 아래에 생겨났습니다. 이 레이어는 이전에는 OpenGL과 DirectX 11 그래픽스 드라이버가 처리했던 복잡성을 담당하며 리소스를 추적하고 세분화된 동적 사용자 영역과 영구적인 저수준 GPU 상태 간의 매핑을 관리합니다. 이로 인해 그래픽스 프로그래머는 두 가지 구분된 역할로 전문화되기 시작했습니다. 새로운 저수준 '드라이버 레이어'와 RHI 계층에 집중하는 **저수준(Low-level) 그래픽스 프로그래머**들과, 그들이 구현한 RHI계층 위에서 시각적 알고리즘(비주얼 프로그래밍)에 집중하는 **고수준(High-level) 그래픽스 프로그래머**들로 말이죠. 물론 비주얼 프로그래밍 또한 물리 기반 라이팅 모델들, 컴퓨트 셰이더, 이후에는 레이 트레이싱이 등장하며 더 복잡해졌습니다.

## 모던 API?
DirectX 12, Vulkan, 그리고 Metal은 종종 **'모던 API(Modern API)'** 로 불려집니다. 이 API들은 이제 등장한 지 10년도 더 넘었습니다. 그들이 설계될 당시 지원하고자 했던 GPU들은 지금으로부터 13년 전의 제품이며, 이는 GPU의 역사에서는 놀라울 만큼 긴 시간입니다. 오래 전 GPU 아키텍쳐들은 오늘날 널리 사용하는 연산(Compute) 집약적인 워크로드보다 전통적인 정점 및 픽셀 셰이더 작업들에 최적화되어 있습니다. 그들은 제조사(Vendor) 별로 구분된 바인딩 모델들과 데이터 경로를 가지고 있습니다. 하드웨어 차이는 동일한 API 속에서 래핑되어야 했습니다. 이로 인해 사전에 생성된 영구 객체는 매핑, 업로드, 유효성 검증 및 바인딩 비용을 줄이는 데 매우 중요했습니다.  

반면, 콘솔 API들과 Mantle은 그 당시로써는 선구적인 시각으로 설계된 AMD의 GCN 아키텍쳐만을 위해 독점적으로 디자인되었습니다. GCN은 복합적인 읽기/쓰기 캐시 계층과 텍스쳐/버퍼 디스크립터를 저장하는 스칼라 레지스터를 자랑하며 사실상 모든 것을 메모리처럼 취급했습니다(**역주: 포인터 연산과 같은 방식으로 모든 자원에 자유롭게 접근 가능하다는 것을 의미합니다*). 데이터를 리매핑하는 데 어떠한 복잡한 API도 필요하지 않았고, (드로우 콜 이전에 필요한)사전 작업의 필요량이 상당하게 줄어들었습니다. 콘솔 API들과 Mantle은 단 하나의 최신 GPU 아키텍쳐만을 위해 설계했기 때문에 더 적은 API 복잡도을 가졌습니다.  

10년이 지났고, GPU들은 상당한 진화를 거쳤습니다. 모든 최신 GPU 아키텍쳐들은 이제 일관성 있는 최종 레벨 캐시를 갖춘 완전한 캐시 계층 구조를 특징으로 합니다. PCIe ReBAR(**역주, AMD에선 동일한 기술을 SAM-Smart Access Memory-이라 부릅니다*)나 UMA를 이용해 CPU는 GPU 메모리에 직접적으로 쓰기 동작이 가능하며, 64비트 GPU 포인터가 셰이더에서 직접적으로 지원됩니다. 텍스쳐 샘플러들은 바인딩이 필요가 없으므로(Bindless) CPU 드라이버가 디스크립터 바인딩을 구성할 필요가 없으며, 텍스쳐 디스크립터는 GPU 메모리(디스크립터 힙 으로 불립니다) 안에 배열 형태로 곧바로 저장될 수 있습니다. 만약 우리가 오늘날의 최신 GPU들을 위한 API를 설계한다면, 앞선 '모던 API'들의 특징인 영구적인 '유지 모드' 객체 대부분은 필요하지 않을 것입니다. DirectX 12.0, Metal 1, Vulkan 1.0이 감수해야 했던 타협점들은 더 이상 필요하지 않습니다. API를 극적으로 단순화할 수 있습니다.

지난 10년은 '모던 API'들의 약점이 드러난 시간이었습니다. PSO 순열 폭발은 우리가 해결해야 할 가장 큰 문제입니다. 제조사들(Valve, Nvidia 등)은 서로 다른 각각의 아키텍쳐/드라이버 조합을 위한 테라바이트 단위의 PSO를 저장하기 위한 거대 규모의 클라우드 서버를 가지고 있으며, 유저들의 로컬 PSO 캐시 사이즈는 100GB를 초과하기도 합니다. 게이머들이 게임의 로딩 시간이 너무 오래 걸리고 끊김(스터터링)이 심하다고 불평하는 것이 전혀 놀랍지 않습니다.

## GPU와 그래픽스 API의 역사
그래픽스 API의 표면을 벗겨내는 것에 대해 이야가히기 전에, 그래픽스 API들이 왜 이러한 방식으로 설계되었는지에 대한 역사적인 이해가 필요합니다. OpenGL은 일부러 느리게 만들어진 것이 아니며, Vulkan도 이유 없이 복잡하게 만든 것이 아닙니다(**역주: Vulkan은 RGB 삼각형 하나를 그리는 단순한 튜토리얼에도 1000줄이 넘는 코드가 필요한 것으로 악명높습니다*).  10-20년 전 GPU 하드웨어들은 극도로 다양했으며 빠르게 진화했습니다. 이러한 다양한 하드웨어 조합을 위한 크로스 플랫폼 API를 설계하기 위해선 타협이 필요했습니다.

![image_0](/post_images/no-graphics-api-translation/0.jpg)  
*3dFX Voodoo 2 12MB (1998): 개별 프로세서와 이를 메모리 칩(각 프로세서당 1MB 칩 4개)과 연결하는 트레이스(trace)가 명확하게 보입니다. 이미지 © TechPowerUp.*


고전(Classic)부터 시작해봅시다. 3dFX Voodoo 2 12MB (*1998*)은 세 개의 칩 설계를 가지고 있었는데, 이는 4MB 프레임버퍼 메모리와 연결된 하나의 단일 래스터라이저 칩과 각각 자신만의 4MB 텍스쳐 메모리와 연결된 두 개의 텍스쳐 샘플링 칩입니다. 기하 파이프라인과 프로그래밍 가능한 셰이더는 존재하지 않았습니다. CPU는 사전 변환된 삼각형 정점들을 래스터라이저에 보냈습니다. 래스터라이저는 전달받은 정점의 색상과 두 텍스쳐 샘플러를 어떻게 결합될지를 컨트롤하기 위해 구성 가능한 블렌딩 방정식을 가졌습니다. 두 텍스쳐 샘플러들은 서로간의 메모리 혹은 프레임버퍼의 값을 읽을 수 없었습니다. 그러므로 멀티 렌더 패스 역시 지원되지 않았죠. 하드웨어가 윈도우 합성을 지원하지 않았기 때문에 전용 2D 비디오 카드를 연결하기 위한 루프백 케이블이 있었습니다. 3D 렌더링은 오직 전체 화면 모드에서만 정상적으로 수행 가능했습니다. 3D 그래픽카드는 오늘날의 GPU 및 대규모 프로그래밍 가능한 SIMD 배열들과는 공통점을 찾아보기 힘든 매우 특수한 하드웨어였습니다. 이 시대의 하드웨어는 DirectX(*1995*)와 OpenGL(*1992*)의 설계애 막대한 영향을 미쳤습니다. 하위 호환성을 위해 API는 적극적인 변화 대신 점진적인 개선이 이루어졌고, 30년 전의 이러한 API설계 방식은 오늘날 우리가 소프트웨어를 작성하는 방식에 여전히 영향을 미치고 있습니다. 

Nvidia의 Geforce 256은 GPU라는 용어를 만들어냈습니다. 해당 제품은 래스터라이저 외에도 최초로 기하 프로세서를 가졌습니다. 기하 프로세서, 래스터라이저 그리고 텍스쳐 샘플링 유닛은 모두 동일한 다이(die)에 통합되었고 메모리를 공유했습니다. 이에 발맞춰 DirectX 7은 두 가지 새로운 컨셉을 소개했습니다. 바로 렌더 타겟 텍스쳐(Render Target Textures)와 유니폼 상수(Uniform Constants)입니다. 멀티 패스 렌더링은 텍스쳐 샘플러들이 래스터라이저의 출력을 읽을 수 있음을 의미했으며, 이로 인해 3dFX Voodoo2의 별도 메모리 설계가 무용지물이 되었습니다.

기하 프로세서 API는 변환 행렬들(`float4x4`), 빛의 위치나 색상을 위한 유니폼 데이터 입력을 특징으로 합니다. 이에 대한 GPU의 구현 방식은 제조사들마다 다양했으나, 많은 제조사가 기하 엔진 내부에 작은 상수 메모리 블록을 내장하는 방식을 택했습니다. 물론 이것이 유일한 방법은 아니었습니다. OpenGL API에선 각 셰이더가 자신만의 전용 유니폼 데이터를 가질 수 있습니다. 이러한 설계는 드라이버가 상수를 셰이더 연산 스트림 안에 곧바로 임베드하는것이 가능하게 만들었으며, 이는 오늘날 OpenGL 4.6 및 OpenGL ES 3.2에도 여전히 남아 있는 API 특이점입니다.

그 당시 GPU들은 범용 읽기/쓰기 캐시가 없었습니다. 래스터라이저는 블렌딩과 깊이값 저장(Depth Buffering)을 위한 스크린 로컬 캐시를 가지고 있었고, 텍스쳐 샘플러는 데이터 프리페치를 위해 선형 보간된 정점 UV에 의존했습니다. DirectX 8 셰이더 모델 1.0에서 셰이더가 도입되었을 때, 픽셀 셰이더에서 텍스쳐의 UV를 계산하는 것은 지원되지 않았습니다. UV는 정점 단위로 계산되었으며, 하드웨어를 통해 보간되고 텍스쳐 샘플러로 곧장 전달되었습니다.  

DirectX 9는 셰이더 명령어 제한을 크게 증가시켰지만, 셰이더 모델 2.0은 여전히 새로운 데이터 경로를 노출시키지 않았습니다. 정점/픽셀 셰이더 모두는 여전히 1:1 입/출력 방식으로 동작했으며, 사용자는 정점 및 속성(attributes)의 변환 계산과 픽셀 색상만 정의할 수 있었습니다. 프로그래머블한(**역주. '프로그래머블-Programmable'은 코드 단위로 통제 가능함을 의미합니다*) load/store 연산도 지원되지 않았고, 정점 페치, 유니폼(상수) 메모리와 텍스쳐 샘플러라는 고정 기능(fixed-function) 입력 블록이 그대로 유지되었습니다.
정점 셰이더는 분리된 연산 단위였습니다. 인덱스 상수(`float4` 배열로 제한되었지만)와 같은 새로운 기능들을 얻었지만 여전히 텍스쳐 샘플링 지원은 미진했습니다.  

Direct9 셰이더 모델 3.0은 명령어 제한을 65536개 까지 증가시켜 인간이 더 이상 셰이더 어셈블리를 작성하거나 유지보수하기 어렵게 만들었습니다. 이로 인해 HLSL(2002)과 GLSL(2002-2004)같은 고수준 쉐이딩 언어가 등장했습니다. 이러한 언어들은 각 셰이더 계산 요소들과의 1대1 대응 변환 설계를 채택했습니다. 각 셰이더 실행(Invocation)은 단일 데이터 요소(정점, 혹은 픽셀)에 대해 연산되었습니다. 프레임워크 스타일의 셰이더 설계는 그 이후 그래픽스 API 설계에 무거운 영향을 끼쳤습니다. 이것은 그 당시의 하드웨어들 강늬 차이를 추상화하는 매우 멋진 방법이었지만, 오늘날에는 확장성 문제를 드러내고 있습니다.  

DirectX 11은 컴퓨트 셰이더, 범용 읽기-쓰기 버퍼, 그리고 Indirect Drawing의 지원에 대해 발표했고 이는 데이터 모델에 대한 중대한 변화(Shift)였습니다. GPU는 (상술한 기능들을 활용해)자체적으로 충분히 데이터를 공급받을 수 있게 되었습니다. 범용 버퍼의 포함은 셰이더 프로그램이 코드 수준에서 메모리 위치를 수정하고 접근할 수 있도록 했고, 이는 하드웨어 벤더들이 범용 캐시 계층을 구현하도록 강제해습니다. 셰이더들은 간단한 1대1 데이터 변환을 넘어서, 특수화되고 하드코딩된 데이터 경로의 종말을 알렸습니다. GPU 하드웨어는 범용 SIMD 설계를 향해 변화하기 시작했습니다. SIMD 유닛들은 이제 정점(Vertex), 픽셀(Pixel), 기하(Geometry), 헐(Hull), 도메인(Domain) 그리고 컴퓨트까지 서로 다른 모든 셰이더 타입들을 실행할 수 있게 되었습니다. 오늘날 이 프레임워크는 서로 다른 셰이더 시작 지점(Entry Point) 가집니다. 이는 많은 API 표면을 추가했고 구성을 어렵게 만들었습니다. 그 결과 GLSL과 HLSL은 여전히 활발한 라이브러리 생태계를 갖추지 못하고 있습니다.  

DirectX 11은 수많은 버퍼 타입의 지원을 추가했는데, 각각은 특정한 하드웨어 데이터 경로의 특징을 수용하도록 설계되었습니다. 타입 지정 SRV(Shader Resource View)와 UAV(Unordered Access View), 바이트 주소 SRV & UAV, 구조화된(Structured) SRV & UAV, Append와 Consume(counter를 포함해서), 상수, 정점, 그리고 인덱스 버퍼 등입니다. 텍스쳐와 마찬가지로, DirectX에서 이 버퍼들은 불퉁명한 디스크립터를 활용합니다. 디스크립터들은 사이즈, 포맷, 프로퍼티들과 GPU 메모리상에서 데이터의 주소를 인코딩한 하드웨어 종속적인(일반적으로 128-256 비트의) 데이터 블롭입니다. DirectX 11를 지원하는 GPU들은 그들의 텍스쳐 샘플러들을 버퍼 로드 연산을 위한 지렛대로 사용합니다. 이것은 샘플러가 이미 타입 변환 하드웨어와 작은 읽기 전용 데이터 캐시를 가지고 있었던 점에서 자연스러운 결과였습니다. 타입 지정 버퍼들은동일 포맷의 텍스쳐로 지원되었으며, DirectX는 동일한 SRV 추상화를 텍스쳐와 버퍼 양쪽 모두에 사용했습니다.  

불투명 버퍼 디스크립터의 사용은 버퍼 포맷이 셰이더 컴파일 시점엔 알 수 없음을 의미했습니다. 이러한 점은 텍스쳐 샘플러에 의해 관리되는 읽기 전용 버퍼들은 문제가 없었습니다. 읽기-쓰기 버퍼(DirectX의 UAV)는 초기에는 32비트와 128비트(`float4`) 유형으로 제한되었습니다. 이후 API 및 하드웨어 개정을 통해 UAV의 크기 제한은 점차 해결되었지만, 여전히 중요한 문제가 지속되었습니다. 바로 디스크립터가 간접 참조를 필요(포인터 포함)로 하고, 컴파일러 최적화는 제한적이며(데이터 유형을 런타임에만 알 수 있으므로), 포맷 변환 하드웨어가 (raw한 L1 캐시 로드 대비) 지연 시간을 발생시키고, 로드 시 확장은 레지스터를 더 오래 점유하며(사용 시 확장 대비), 디스크립터 관리는 CPU 드라이버의 복잡성을 증가시키고, 서로 다른 10개의 버퍼 타입을 지원해야 함으로 인해 API 자체도 복잡하다는 것입니다.  
(**역주. "사용/로드 시 확장"의 확장(expand)은 UAV, Texel Buffer등에 패킹된 데이터-RGBA8 혹은 R11G11B10등-를 실제 데이터로 사용하기 위해 벡터/스칼라 레지스터에 언패킹하는 것을 의미합니다*)

DirectX 11에서 구조화된 버퍼(StructuredBuffer)는 유저 정의 구조체 타입을 사용할 수 있는 유일한 버퍼 타입이었습니다. 다른 모든 버퍼 타입들은 단순한 스칼라/벡터 원소들의 균일한(Homogeneous) 배열과 동일하게 표현되었습니다. 불행하게도, 구조화된 버퍼는 다른 버퍼 타입들과 레이아웃이 호환되지 않았습니다. 사용자는 타입 지정 버퍼, 바이트 주소 버퍼, 또는 정점/인덱스 버퍼들로 구조화된 버퍼 뷰를 생성하는 것이 허용되지 않았습니다. 그 이유는 구조화된 버퍼가 내부적으로 특수한 SoAoS 스위즐 최적화를 사용했기 때문인데, 이는 오래된 vec4 아키텍쳐에서 중요했습니다. 이 하드웨어에 특화된 최적화로 인해 구조화된 버퍼의 사용성이 제한되었습니다.

DirectX 12는 모든 버퍼를 메모리에 선형적으로 만들어, 그들이 상호간에 호환될 수 있도록 만들었습니다. SM 6.2는 또한 `load<T>` 라는 문법적 설탕을 바이트 주소 버퍼를 위해 추가하여, 임의의 오프셋에서 깔끔한 구조체 로딩 구문을 사용할 수 있게 허용했습니다. 모든 오래된 버퍼 타입들은 여전히 하위 호환성을 위해 여전히 지원되며 모든 버퍼들은 여전히 불투명 디스크립터를 사용합니다. HLSL은 여전히 64비트 GPU 포인터 지원이 부족합니다. 반면에, Nvidia의 CUDA 컴퓨팅 플랫폼(*2007*)은 64비트 포인터에 ***완전히*** 의존했으나, 그 인기는 학술적인 영역에서만 존재했습니다. 그러나 오늘날 이(CUDA)는 선도적인 AI 플랫폼이며 최신 하드웨어 설계에 강력한 영향을 미치고 있습니다.

DirectX12가 출시되었을 때 16비트 레지스터와 16비트 수학 연산 지원은 체계적이지 못했습니다. 마이크로소프트는 초기에 DirectX12를 윈도우7로 백포팅(Backporting, 하위 지원 포팅)하지 않기로 하는 의문스러운 결정을 내렸습니다. Windows8을 대상으로 하는 쉐이더 바이너리들은 16비트 타입을 지원했으나, 대부분의 게이머들은 여전히 윈도우7을 사용하고 있었습니다. 개발자들은 두 세트의 쉐이더를 배포하고 싶어하지 않았습니다(**역주. 즉, 대부분의 게임 개발자들은 유저 커버리지를 위해 이 기능을 사용하지 않기로 결정했습니다*). OpenGL의 `lowp`/`mediump` 사양 또한 난장판이었습니다. 비트 깊이가 제대로 표준화되지 않았습니다. `mediump`는 모바일 게임에서 인기 있는 최적화 옵션이었지만, 대부분의 PC 드라이버들은 이를 무시했기에, 게임 개발자들의 인생을 비참하게 만들었습니다. fp16 2배 속도 지원과 함께 PS4 Pro가 출시되기 전까지 AAA게임들은 대부분 16비트 수학 연산을 무시했습니다.

AI, 레이 트레이싱, 그리고 GPU 주도 렌더링(GPU-Driven Rendering)의 부상과 함께, GPU 제조사들은 그들의 원시 데이터 로드 경로를 최적화하는 것과 더 크고 빠른 범용 캐시를 제공하는 것에 집중하기 시작했습니다. 텍스쳐 샘플러를 통한 로드 라우팅(타입 변환)은 현대 쉐이더에서 종속 로드 체인이 읿반적이게 됨에 따라 매우 큰 지연 시간(latency)을 발생시킵니다. 하드웨어는 좁은(narrow) 8비트, 16비트, 그리고 64비트 타입과 포인터에 대한 네이티브 지원을 갖추게 되었습니다.
(**역주. 종속 로드 체인-Dependent Load Chain-은 특정 텍스쳐를 샘플링하고, 그 결과로 다시 특정 텍스쳐를 샘플링하는 LUT같은 형식의 사용을 의미합니다*)

대부분의 제조사들은 그들의 고정 기능 정점 페치(Fetch) 하드웨어를 폐기하고, 대신 정점 쉐이더에 표준 원시 로드 명령어를 생성하도록 했습니다. 완전히 프로그래밍 가능한 정점 페치는 개발자가 클러스터화된 GPU 주도 렌더링(Clustured GPU-Driven Rendering)과 같은 새로운 알고리즘을 작성할 수 있게 해주었습니다. 또한 고정 기능 하드웨어를 위한 트랜지스터 예산은 이제 다른 곳에 사용할 수 있게 되었습니다. 

메쉬 셰이더는 래스터라이저 진화의 정점으로, 인덱스 중복 제거 하드웨어와 변환 후(Post-Transform) 캐시를 없앴습니다. 이러한 패러다임에서, 모든 입력은 원시 메모리로 취급되었습니다. 유저는 메쉬를 내부적으로 정점을 공유하는 독립적인 메쉬렛(Meshlet)으로 분할하는 책임을 집니다. 이러한 프로세스는 종종 오프라인에서 완료됩니다. GPU는 더 이상 각 드로우 콜 마다 병렬 인덱스 중복 제거를 수행할 필요가 없으므로, 전력과 트랜지스터를 아낄 수 있습니다. 오늘날 게임이 Nvidia의 매출의 10%만 차지하는 반면, AI는 90%를 차지하고 레이 트레이싱은 성장하고 있는 점을 고려할 때, 고정 기능 기하 하드웨어가 최소한의 기능만 남게 되고 드라이버들이 자동으로 정점 쉐이더를 메쉬 쉐이더로 변환하게 되는 것은 시간 문제일 가능성이 높습니다. 

모바일 GPU는 타일 기반 렌더러입니다. 타일러(Tilers)은 개별적인 삼각형들을 작은 타일(일반적으로 16x16 에서 64x64 크기 사이의 픽셀들로 구성됨)들로 분류합니다. 메쉬 셰이더는 이 용도로 사용하기엔 너무 조잡(coarse)합니다(**역주. 오버스펙이라는 의미입니다*). 메쉬렛을 작은 타일들로 분류하는 것은 심각한 지오메트리 오버셰이딩을 야기할 가능성이 높습니다. 깔끔한 수렴 경로가 없습니다. 우리는 여전히 정점 쉐이더 경로를 지원해야 합니다.

10년 전 DirectX 12.0, Vulkan 1.0, 그리고 Metal 1.0이 등장했을 때, 당시 존재하는 GPU 하드웨어들은 바인드리스 리소스를 폭넓게 지원하지 않았었습니다. API들은 하드웨에들 간의 차이를 추상화하기 위해 복잡한 바인딩 모델을 채택했습니다. DirectX 12는 스테이지당 128개의 리소스를 인덱싱하는 것을 허용했으며, Vulkan과 Metal은 초기에는 디스크립터 인덱싱을 아예 지원하지 않았습니다. 개발자들은 전통적인 바인딩 변경에 따른 오버헤드를 감소시키기 위해 기존의 우회 방법들을 사용해야만 했습니다. 지난 10년간 GPU 하드웨어는 크게 발전했으며 범용 바인드리스 SIMD 설계로 수렴했습니다.

이제 현대의 바인드리스 하드웨어 전용으로 설계한다면 그래픽 API와 셰이더 언어가 얼마나 더 단순해질 수 있는지 살펴봅시다.

## 최신(Modern) GPU 메모리 관리

저희의 여정을 메모리 관리에 대해 이야기하는것으로 시작합시다. 구형 그래픽스 API들은 GPU 메모리 관리를 완전히 추상화해습니다. 오래된 GPU들은 분리된 메모리나 다양한 캐시 일관성 문제를 가지고 있었기에 추상화들은 필수적이었습니다. DirectX 12와 Vulkan이 등장한 10여년 전에, GPU 하드웨어들은 배치 힙(Placement Heaps)을 노출해도 될 만큼 충분히 성숙했습니다. 콘솔들은 이미 몇 세대 동안 메모리를 노출해 왔으며, 개발자들은 비슷한 유연성을 PC와 모바일에서도 요구했습니다. 애플은 Vulkan과 DirectX 12보다 4년 늦은 시점에 Metal2에서 배치 힙을 도입했습니다.

모던 API에서는 사용자가 어떤 종류의 메모리를 GPU드라이버가 제공해야 할 지에 대해 알아내기 위해 힙 타입을 열거할 것을 요구했습니다. 큰 청크 안에서 메모리를 사전 할당하고 이를 유저 영역 얼로케이터로 세부 할당하는 방식이 좋은 용법이었습니다. 그러나 Vulkan에는 먼저 사용자의 텍스쳐/버퍼 오브젝트를 만들어야만 한다는 설계적 결함이 있습니다. 그런 다음에야 사용자는 새로운 리소스에 어떤 힙 타입이 적합한지 확인할 수 있습니다. 이는 유저가 지연 할당 패턴에 의존하게 되는데, 이는 런타임에 성능 저하나 메모리 스파이크를 유발할 수 있었습니다. 또한 이는 GPU 메모리 할당을 크로스 플랫폼 라이브러리로 감싸기도 어렵게 만듭니다. 예를 들어 AMD VMA는 메모리를 할당하는 것 외에도 Vulkan에 특화된 버퍼/텍스처 객체를 생성합니다. 우리는 이러한 관심사를 완전히 분리하고자 합니다.

오늘날 CPU는 GPU 메모리의 내부를 완전히 볼 수 있습니다. 내장 GPU(Integrated GPU)는 UMA를 가지며, 최신 외장 GPU는 PCIe Resizable BAR를 가지고 있습니다. 모든 GPU 힙은 매핑될 수 있습니다. Vulkan 힙 API는 CPU 매핑이 가능한 GPU 힙을 자연스럽게 지원합니다. DirectX 12는 2023년부터 지원을 시작했습니다(`HEAP_TYPE_GPU_UPLOAD`).

CUDA는 GPU 메모리 할당을 위한 간단한 설계를 가집니다. GPU malloc API는 할당할 크기를 입력으로 받고, 호출 결과로 매핑된 CPU 포인터를 반환합니다. GPU free API는 메모리를 직접 해제합니다. CUDA는 CPU 매핑 가능한 GPU메모리를 지원하지 않습니다. GPU는 CPU 메모리를 PCIe 버스를 통해 읽습니다. CUDA는 또한 GPU 메모리 할당을 지원하지만, CPU에서 바로 해당 메모리에 쓰기 연산을 할 수 없습니다.

우리는 CUDA malloc 설계에 CPU 매핑 가능한 GPU 메모리(UMA/ReBAR)를 결합합니다. 이 데이터는 CPU가 쓰기에 빠르고 GPU가 읽기에 빠르면서도, 깔끔하고 쉬운 사용성을 유지한다는 두 장점을 모두 취합니다.

```c
// Allocate GPU memory for array of 1024 uint32
uint32* numbers = gpuMalloc(1024 * sizeof(uint32));

// Directly initialize (CPU mapped GPU pointer)
for (int i = 0; i < 1024; i++) numbers[i] = random();

gpuFree(numbers);

```

기본 gpuMalloc 정렬은 16바이트(vec4 정렬)입니다. 만약 당신이 더 넓은 정렬을 필요로 한다면 `gpuMalloc(size, alignment)` 오버로드를 사용하세요. 저의 예시 코드는 `gpuMalloc(elements * sizeof(T), alignof(T))`를 수행하는 `gpuMalloc<T>` 래퍼를 사용합니다.

그리기 인수(draw argument), 유니폼(uniform), 그리고 디스크립터와 같은 작은 데이터는 GPU에 바로 쓰는 것이 최적입니다. 큰 크기의 지속성 데이터의 경우, 우리는 여전히 복사 명령을 사용해야 합니다. GPU는 캐시 지역성 향상을 위해 텍스쳐를 [Morton-order](https://en.wikipedia.org/wiki/Z-order_curve)와 비슷한 스위즐된 레이아웃으로 저장하길 원합니다. DirectX 11.3과 12는 표준화된 스위즐 레이아웃을 시도했지만, 모든 GPU 제조사가 이를 수용하도록 하진 못했습니다. 텍스쳐 스위즐링을 수행하는 일반적인 방법은 드라이버가 제공하는 복사 명령를 사용하는 것입니다. 복사 명령은 선형 텍스쳐 데이터를 CPU에 매핑된 "업로드" 힙으로부터 읽고 이를 전용 GPU 힙에 스위즐된 레이아웃으로 써넣습니다. 모든 최신 GPU는 또한 무손실 델타 색상 압축(DCC, Delta Color Compression)를 지원합니다. 최신 GPU의 복사 엔진은 DCC 압축 및 압축 해제가 가능합니다. DCC와 Morton 스위즐은 우리가 텍스쳐를 전용 GPU힙으로 복사해야 하는 주요한 동기입니다. 최근 GPU에는 버퍼 데이터를 위한 범용 무손실 메모리 압축 또한 추가되었습니다. 만약 메모리 힙이 CPU에 매핑되어 있다면, GPU는 CPU가 어떻게 읽기 및 쓰기를 수행하는지 알 수 없으므로 제조사별로 특화된 무손실 압축을 활성화할 수 없습니다. 따라서 데이터를 압축하려면 반드시 복사 명령을 사용해야 합니다.

GPU malloc 함수에 전용 GPU 메모리 지원을 추가하기 위해 우리는 메모리 타입 매개변수가 필요합니다. 표준 메모리 타입은 CPU 매핑 가능한 GPU 메모리(쓰기 결합 CPU 접근)가 되어야 합니다. GPU가 읽기에 빠르며, CPU는 이를 마치 CPU 메모리 포인터인 것 처럼 직접 쓸 수 있습니다. GPU 전용 메모리는 텍스쳐와 거대한 GPU 전용 버퍼에 사용됩니다. CPU는 직접적으로 이러한(GPU 전용) 포인터에 쓰기 연산을 할 수 없습니다. 사용자는 먼저 CPU에 매핑된 GPU 메모리에 데이터를 쓰고 난 뒤 데이터를 최적의 압축 포맷으로 변환하는 복사 명령을 요청합니다.
최신 텍스쳐 샘플러들과 디스플레이 엔진은 압축된 GPU데이터를 곧바로 읽을 수 있으므로, 부가적인 데이터 레이아웃 변환은 필요하지 않습니다(후술된 Modern barriers 챕터를 확인하세요). 업로드된 데이터는 즉시 사용할 수 있습니다.

GPU 포인터는 CPU 매핑 가상 주소와 GPU 가상 주소 두 가지 유형이 있습니다. GPU는 오직 GPU 주소를 역참조하는 것만이 가능합니다. GPU 자료 구조의 모든 포인터들은 GPU 주소를 가져야만 합니다. CPU에 매핑된 주소는 오직 CPU 쓰기에만 사용됩니다. CUDA는 CPU 매핑된 주소를 GPU 주소로 변환하는 API를 가지고 있습니다. Metal 4 버퍼 오브젝트는 `.contents`(CPU에 매핑된 주소)와 `.gpuAddress`(GPU 주소)의 두 가지 getter 함수를 가집니다. gpuMalloc API가 (Metal과 같이)관리형 객체 핸들을 반환하지 않고 포인터를 반환하기 때문에, 우리는 CUDA의 접근 방식(`gpuHostToDevicePointer`)를 택했습니다. 이 API 호출은 공짜가 아닙니다. 드라이버는 이를 해시맵을 통해 구현할 가능성이 높습니다(만약 기본 주소 외의 주소 변환이 필요할 경우, 트리가 필요합니다). 가급적 할당 당 한번만 주소 변환을 호출하고 사용자 영역 구조체(void *cpu, void *gpu)에 캐싱하는 것이 좋습니다. 이것이 제 사용자 영역 GPUBumpAllocator가 사용하는 접근 방식입니다(전체 구현은 부록 참조).

```c
// Load a mesh using a 3rd party library
auto mesh = createMesh("mesh.obj");
auto upload = uploadBumpAllocator.allocate(mesh.byteSize); // Custom bump allocator (wraps a gpuMalloc ptr)
mesh.load(upload.cpu);

// Allocate GPU-only memory and copy into it
void* meshGpu = gpuMalloc(mesh.byteSize, MEMORY_GPU);
gpuMemCpy(commandBuffer, meshGpu, upload.gpu);

```

Vulkan은 최근 [`VK_EXT_host_image_copy`](https://docs.vulkan.org/refpages/latest/refpages/source/VK_EXT_host_image_copy.html)라는 새로운 확장 기능(Extension)을 출시했습니다. 드라이버는 하드웨어 특화 테스쳐 스위즐을 수행하는 CPU에서 GPU로의 직접 이미지 복사 연산을 구현합니다. 이 확장 기능은 현재 UMA 아키텍쳐에서만 이용 가능하지만, PCIe ReBAR에서도 이용하지 못할 (기술적인)이유는 없습니다. 불행하게도 이 API는 DCC를 지원하지 않습니다. 아마 CPU에서 DCC 압축을 수행하는 것이 매우 많은 비용이 들기 때문일 것입니다. 이 확장 기능은 주로 DCC를 요구하지 않는 블록 압축 텍스쳐들에 유용합니다. 결론적으로 이는 GPU 전용 메모리로의 하드웨어 복사 명령을 완전히 대체할 수 없습니다.  

읽어오기(Readback) 목적을 위해 세 번째 메모리 타입인 CPU 캐시(CPU-cached) 또한 필요합니다. 이 메모리 타입은 CPU와의 캐시 일관성 때문에 GPU가 쓰기 연산을 하기엔 속도가 더 느립니다. 게임은 읽어오기 연산을 드물게 사용합니다. 일반적인 사용 사례로는 스크린샷과 가상 텍스처 읽기가 있습니다. AI 훈련 및 추론과 같은 GPGPU 알고리즘은 CPU와 GPU 간의 효율적인 통신에 의존합니다.

CUDA malloc과 CPU 매핑된 GPU 메로리를 결합하면 우리는 최소한으로 API 표면을 노출하는 유연하고 빠른 GPU 메모리 할당 시스템을 가지게 됩니다. 이는 최소화된 모던 그래픽스 API를 위한 훌륭한 출발점입니다.

## 최신(Modern) 데이터
CUDA, Metal, 그리고 OpenCL은 64비트 포인터 의미론을 갖춘 C/C++ 셰이더 언어를 활용합니다. 이러한 언어는 적절하게 정렬된 GPU 메모리 위치에서 구조체를 로드하고 저장하는 것을 지원합니다. 컴파일러는 와이드 로드(병합), 레지스터 매핑, 비트 추출 등 백그라운드 최적화를 처리합니다. 많은 현대 GPU는 레지스터의 8비트/16비트 부분을 추출하기 위한 무료 명령어 한정자(Modifier)를 제공하여, 컴파일러가 8비트 및 16비트 값을 단일 레지스터에 패킹할 수 있게 합니다. 이로 인해 셰이더 코드가 깔끔하고 효율적으로 유지됩니다.

32비트 값 8개로 이루어진 구조체를 로드하면, 컴파일러는 대개 128비트 폭의 로드 명령어 두 개(각각 레지스터 4개를 채움)를 생성하여 로드 명령어 수를 4분의 1로 줄입니다. 넓은 폭의 로드는 특히 구조체에 좁은 8비트 및 16비트 필드가 포함된 경우 훨씬 빠릅니다. GPU는 ALU 밀도가 높고 큰 레지스터 파일을 갖추고 있지만, CPU에 비해 메모리 경로는 상대적으로 느립니다. CPU는 종종 사이클마다 하나의 로드를 수행하는 로드 포트 두 개를 갖추고 있는 경우가 많습니다. 현대적인 GPU에서는 4사이클마다 하나의 SIMD 로드를 달성할 수 있습니다. 셰이더에서 넓은 폭의 로드와 언팩(Unpack)을 조합하는 것이 데이터를 처리하는 가장 효율적인 방법인 경우가 많습니다.

컴팩트한 8~16비트 데이터는 전통적으로 DirectX 게임에서 텍셀 버퍼(Buffer<T>)에 저장되었습니다. 최신 GPU는 컴퓨트 워크로드에 최적화되어 있습니다. 오늘날 원시 버퍼 로드 명령어는 텍셀 버퍼보다 최대 2배 더 높은 처리량과 최대 3배 더 낮은 대기 시간을 제공합니다. 텍셀 버퍼는 더 이상 최신 GPU에서 최적의 선택이 아닙니다. 텍셀 버퍼는 구조화된 데이터를 지원하지 않으므로, 사용자는 여러 텍셀 버퍼에 데이터를 SoA 레이아웃으로 분리해야 합니다. 각 텍셀 버퍼는 고유한 디스크립터를 가지며, 데이터에 접근하기 전에 이를 로드해야 합니다. 단일 64비트 원시 포인터를 사용하는 것에 비해 리소스(SGPR-*Scalar General Purpose Registers*, 디스크립터 캐시 슬롯)를 소비하고 초기 대기 시간을 추가합니다. 또한 SoA 데이터 레이아웃은 비선형 인덱스 조회(예: 머티리얼, 텍스처, 삼각형, 인스턴스, 본 ID)에서 캐시 미스를 상당히 더 많이 유발합니다. 텍셀 버퍼는 정규화된([0,1] 및 [-1,1]) 타입을 부동 소수점 레지스터로 변환하는 기능을 무료로 제공합니다. ALU 비용은 들지 않는 것이 사실이지만, 와이드 로드 지원(로드 결합)을 잃게 되며 명령어는 느린 텍스처 샘플러 하드웨어 경로를 통과하게 됩니다. 좁은 텍셀 버퍼 로드는 또한 레지스터 부풀림을 유발합니다. `RGBA8_UNORM`을 `vec4`로 로드하면 즉시 4개의 벡터 레지스터를 할당합니다. 샘플러 하드웨어는 이 레지스터들에 값을 최종적으로 기록합니다. 컴파일러는 로드 명령어를 셰이더 시작 부분으로 이동시켜 로드→사용 거리를 최대화하려고 합니다. 이는 ALU를 통해 로드 대기 시간을 숨기고 여러 로드를 겹치게 만듭니다. 대신 와이드 원시 로드를 사용하면 `uint8x4` 데이터는 단 하나의 32비트 레지스터만 차지합니다. 사용 시점에 8비트 채널을 언팩합니다. 레지스터 수명은 훨씬 짧아집니다. 현대 GPU는 언팩 없이 레지스터의 16비트 하위/상위 절반에 직접 접근할 수 있으며, 일부는 8비트 접근도 가능합니다(AMD SDWA 수정자). 팩(Pack) 처리된 2배 속도 연산은 2x16비트 변환 명령어를 더 빠르게 만듭니다. 일부 GPU 아키텍처(Nvidia, AMD)는 VRAM에서 그룹공유 메모리(groupshared memory)로 64비트 포인터 원시 로드를 직접 수행할 수 있어, 대기 시간 숨김에 필요한 레지스터 낭비를 더욱 줄여줍니다. 64비트 포인터를 사용함으로써 게임 엔진은 AI 하드웨어 최적화의 혜택을 받습니다.

포인터 기반 시스템은 메모리 정렬을 명시적으로 만듭니다. DirectX나 Vulkan에서 버퍼 객체를 할당할 때는 API에 정렬 방식을 쿼리해야 합니다. 버퍼 바인딩 오프셋도 올바르게 정렬되어야 합니다. Vulkan은 바인딩 오프셋 정렬을 쿼리하는 API를 제공하고, DirectX는 고정된 정렬 규칙을 가지고 있습니다. 정렬 규칙을 계약(Contract)함으로써 저수준 셰이더 컴파일러는 최적의 코드(예: 정렬된 4x32바이트 너비 로드 등)를 생성할 수 있습니다. DirectX의 `ByteAddressBuffer` 추상화에는 설계상 결함이 있습니다: load2, load3, load4 명령어는 4바이트 정렬만 요구합니다. 새로운 SM 6.2 `load<T>` 역시 요소 단위 정렬(`half4` = 2, `float4` = 4)만 요구합니다. 일부 GPU 벤더(Nvidia 등)는 ByteAddressBuffer.load4를 네 개의 개별 로드 명령어로 분해해야 합니다. 버퍼 추상화가 항상 사용자를 나쁜 코드 생성으로부터 보호해 주지는 못합니다. 오히려 생성된 나쁜 코드를 수정하기 어렵게 만듭니다. C/C++ 기반 언어(CUDA, Metal)는 alignas 속성을 사용해 사용자가 구조체 정렬을 명시적으로 선언할 수 있게 합니다. 우리는 모든 예제 코드의 루트 구조체에 `alignas(16)`를 사용합니다.

기본적으로 GPU의 쓰기 작업은 같은 스레드 그룹(= 컴퓨트 유닛) 내의 스레드에만 보입니다. 이로 인해 비일관적(non-coherent) L1 캐시 설계가 가능합니다. 일반적으로 배리어(barrier)를 통해 가시성을 제공합니다. 사용자가 단일 디스패치 내 그룹 간의 메모리 가시성이 필요한 경우 버퍼 바인딩에 `[globallycoherent]` 속성을 지정합니다. 셰이더 컴파일러는 해당 버퍼에 대한 접근에 일관성(coherent) 있는 로드/저장 명령을 생성합니다. 우리는 버퍼 객체 대신 64비트 포인터를 사용하므로, 명시적인 일관적 로드/저장 명령을 제공합니다. 문법은 원자적 로드/저장과 유사합니다. 마찬가지로 캐시 계층 전체를 우회하는 비일시적(non-temporal) 로드/저장 명령도 제공할 수 있습니다.

Vulkan은 64비트 포인터를 (2019년) [VK_KHR_buffer_device_address](https://docs.vulkan.org/samples/latest/samples/extensions/buffer_device_address/README.html) 확장을 통해 지원합니다. 버퍼 디바이스 주소 확장은 모든 GPU 벤더(모바일 포함)에서 폭넓게 지원되지만, Vulkan 1.4 코어의 일부는 아닙니다. BDA(Buffer Device Address)의 주요 문제는 GLSL 및 HLSL 셰이더 언어에서 포인터 지원이 부족하다는 점입니다. 사용자는 대신 원시 64비트 정수를 사용해야 합니다. 64비트 정수는 구조체로 캐스팅할 수 있습니다. 구조체는 사용자 정의 BDA 구문으로 정의됩니다. 사용자가 컴파일러가 인덱스 주소 지정 연산을 생성하도록 하려면 배열 인덱싱에 배열이 포함된 추가 BDA 구조체 타입을 선언해야 합니다. 현재 디버깅 지원은 제한적입니다. 사용성은 매우 중요하며, HLSL과 GLSL이 포인터를 기본적으로 지원할 때까지 BDA는 틈새 시장에 머물 것입니다. 이는 기본 포인터 지원이 언어의 핵심 요소이며 디버깅이 완벽하게 작동하는 CUDA, OpenCL 및 Metal과는 현격한 대조를 이룹니다.

DirectX 12은 셰이더에서 포인터를 지원하지 않습니다. 그 결과, HLSL은 함수 매개변수로 배열을 전달하는 것을 허용하지 않습니다. UBO/SSBO 내부에 머티리얼 배열을 두는 것과 같은 간단한 작업도 매크로를 사용해 우회해야 합니다. 그룹 공유 메모리 배열을 함수 간에 전달할 수 없기 때문에 리덕션(누적 합, 정렬 등)을 위한 재사용 가능한 함수를 만드는 것은 불가능합니다. 물론 각 유틸리티 헤더나 라이브러리마다 별도의 전역 배열을 선언할 수는 있지만, 컴파일러는 이들 각각에 대해 그룹 공유 메모리를 별도로 할당하므로 점유율이 감소합니다. 그룹 공유 메모리를 별칭으로 사용하는 쉬운 방법이 없습니다. GLSL도 동일한 문제를 안고 있습니다. CUDA와 Metal MSL과 같이 포인터 기반 언어는 배열에 있어 이러한 문제가 없습니다. CUDA는 방대한 제3자 라이브러리 생태계를 보유하고 있으며, 이러한 생태계는 Nvidia를 지구상에서 가장 가치 있는 기업으로 만들었습니다. 그래픽 셰이딩 언어도 현대적인 표준을 맞추기 위해 발전해야 합니다. 우리에게도 라이브러리 생태계가 필요합니다.

제 예제에서는 CUDA와 Metal MSL과 유사한 C/C++ 스타일의 셰이딩 언어를 사용하겠으며, 그래픽에 특화된 부분에는 HLSL 스타일의 시스템 값(SV) 시맨틱을 일부 혼합하여 사용할 것입니다.

## Root arguments 루트 인수(Root arguments)

운영 체제의 스레딩 API는 일반적으로 스레드 함수에 64비트 void 포인터를 하나만 제공합니다. 운영 체제는 사용자의 데이터 입력 레이아웃에 관여하지 않습니다. GPU 커널 데이터 입력에도 동일한 개념을 적용해 보겠습니다. 셰이더 커널은 하나의 64비트 포인터를 받으며, 이를 커널 함수 시그니처에 따라 원하는 구조체로 캐스팅합니다. 덕분에 개발자는 CPU와 GPU 양쪽에서 동일한 공유 C/C++ 헤더를 사용할 수 있습니다.

```c++
// Common header...
struct alignas(16) Data
{
    // Uniform data
    float16x4 color; // 16-bit float vector
    uint16x2 offset; // 16-bit integer vector
    const uint8* lut; // pointer to 8-bit data array

    // Pointers to in/out data arrays
    const uint32* input;
    uint32* output;
};

// CPU code...
gpuSetPipeline(commandBuffer, computePipeline);

auto data = myBumpAllocator.allocate<Data>(); // Custom bump allocator (wraps gpuMalloc ptr, see appendix)
data.cpu->color = {1.0f, 0.0f, 0.0f, 1.0f};
data.cpu->offset = {16, 0};
data.cpu->lut = luts.gpu + 64; // GPU pointers support pointer math (no need for offset API)
data.cpu->input = input.gpu;
data.cpu->output = output.gpu;

gpuDispatch(commandBuffer, data.gpu, uvec3(128, 1, 1));

// GPU kernel...
[groupsize = (64, 1, 1)]
void main(uint32x3 threadId : SV_ThreadID, const Data* data)
{
    uint32 value = data->input[threadId.x]; 
    // TODO: Code using color, offset, lut, etc...
    data->output[threadId.x] = value;
}
```

예제 코드에서는 GPU 인자를 할당하기 위해 간단한 선형 범프 할당자(myBumpAllocator)를 사용합니다(구현 방법은 부록 참조). 이 함수는 구조체 `struct {void* cpu, void *gpu}`를 반환합니다. CPU 포인터는 영구 매핑된 GPU 메모리에 직접 쓰는 데 사용되며, GPU 포인터는 GPU 데이터 구조체에 저장하거나 디스패치 명령 인자로 전달할 수 있습니다.

대부분의 GPU는 웨이브(또는 워프-*Warp*)를 실행하기 직전에 루트 유니폼(64비트 포인터 포함)을 상수 또는 스칼라 레지스터로 미리 로드합니다. 이 최적화는 여전히 유효합니다. 드로우/디스패치 명령은 기본 데이터 포인터를 전달하며, 모든 입력 유니폼(다른 데이터에 대한 포인터 포함)은 기본 포인터로부터 작은 고정 오프셋에서 찾을 수 있습니다. 셰이더는 미리 컴파일되며 PSO 생성 중에 장치별 마이크로코드로 추가 최적화되므로, 드라이버는 레지스터 사전 로드 및 유사한 루트 데이터 최적화를 설정할 충분한 기회를 갖습니다. 일부 아키텍처에서는 루트 데이터 크기가 제한되어 있으므로 사용자는 가장 중요한 데이터를 루트 구조체의 시작 부분에 배치해야 합니다. 우리의 루트 구조체는 엄격한 크기 제한이 없습니다. 셰이더 컴파일러는 나머지 필드에 대해 표준(스칼라/유니폼) 메모리 로드를 생성합니다. 셰이더에 제공되는 루트 데이터 포인터는 `const`입니다. 즉 셰이더는 루트 입력 데이터를 수정할 수 없습니다. 이는 명령 프로세서가 새로운 웨이브에 데이터를 미리 로드하는 데 여전히 사용될 수 있기 때문입니다. 출력은 비 `const` 포인터를 통해 수행됩니다(위 예제의 Data::output 참조). 루트 데이터를 `const`로 강제함으로써, 우리는 GPU 드라이버가 특수 유니폼 데이터 경로 최적화를 수행할 수 있도록 허용합니다.

특별한 유니폼 버퍼 타입이 필요할까요? 최신 셰이더 컴파일러는 자동적인 균일성(uniformity) 분석을 수행합니다. 명령어의 모든 입력이 균일(uniform)하면 출력도 균일합니다. 균일성은 셰이더 전체로 전파됩니다. 모든 최신 아키텍처는 스칼라 레지스터/로드 또는 이와 유사한 구성(Intel의 SIMD1 등)을 갖추고 있습니다. 균일성 분석은 벡터 로드를 스칼라 로드로 변환하는 데 사용되며, 이로써 레지스터를 절약하고 지연 시간을 줄입니다. 균일성 분석은 버퍼 타입(`UBO` vs `SSBO`)에 관계없이 동작합니다. 리소스는 읽기 전용이어야 합니다(그래서 GLSL에서는 항상 `SSBO`에 `readonly` 속성을 지정하고, DirectX 12에서는 `UAV`보다는 `SRV`를 선호하는 것입니다). 또한 컴파일러가 해당 포인터가 별칭(alias)이 아님을 증명할 수 있어야 합니다. C/C++의 const 키워드는 이 포인터를 통해 데이터를 수정할 수 없다는 뜻이지, 다른 읽기-쓰기 포인터가 동일한 메모리 영역을 별칭으로 사용하지 않음을 보장하지는 않습니다. C99는 이를 위해 restrict 키워드를 추가했으며, CUDA 커널에서도 이를 자주 사용합니다. Metal의 루트 포인터는 기본적으로 별칭이 없는(restrict) 상태이며, Vulkan과 DirectX 12의 버퍼 오브젝트도 마찬가지입니다. 우리도 컴파일러가 최적화를 더 자유롭게 수행할 수 있도록 동일한 규칙을 따라야 합니다.

(**역주, 균일성(Uniformity)이란 워프 내의 모든 스레드가 같은 값을 참조/사용함이 보장되었음을 의미합니다. 마찬가지로 GLSL에서 `uniform` 키워드를 사용하는 리소스 역시 쉐이더를 실행하는 임의의 스레드에 대해 해당 리소스가 항상 같은 값임이 보장된다는 걸 의미합니다*)

셰이더 컴파일러가 컴파일 타임에 주소(포인터)의 균일성을 항상 증명할 수 있는 것은 아닙니다. 최신 GPU는 동적인 균일 주소 로드를 기회에 따라 최적화합니다. 메모리 컨트롤러가 벡터 로드 명령어의 모든 레인(Lane, SIMD에서의 각 데이터를 담는 경로)이 균일한 주소를 가지고 있음을 감지하면 SIMD 와이드 대신 단일 레인 로드를 수행합니다. 결과는 모든 레인으로 복제됩니다. 이 최적화는 투명하게 이루어지며, 셰이더 코드 생성이나 레지스터 할당에 영향을 주지 않습니다. 특히 새로운 빠른 원시 로드 경로와 결합하면, 동적으로 균일한 데이터로 인한 성능 저하는 과거에 비해 훨씬 작습니다.

일부 GPU 벤더(ARM Mali 및 Qualcomm Adreno)는 균일성 분석을 한 단계 더 심화시킵니다. 셰이더 컴파일러는 균일 로드(uniform load)와 균일 연산을 추출합니다. 셰이더 실행 전에 스칼라 사전 연산(Preamble)이 실행되며, 드로우/디스패치 전체에 대해 균일 메모리 로드와 연산이 한 번만 수행되고 그 결과는 특수 하드웨어 상수 레지스터(root constants가 사용하는 동일한 레지스터)에 저장됩니다.

위의 모든 최적화를 결합하면 고전적인 16KB/64KB 균일/상수 버퍼 추상화보다 균일 데이터를 처리하는 더 나은 방법을 제공합니다. 많은 GPU가 여전히 root constants, 시스템 값, 프리앰블(위의 단락 참조)을 위해 특수한 균일 레지스터를 가지고 있습니다.

## Texture bindings 텍스처 바인딩

이상적으로는 텍스처 디스크립터가 GPU 메모리의 다른 데이터와 동일하게 동작함으로써, 다른 데이터와 함께 구조체(struct)에서 자유롭게 혼합될 수 있어야 합니다. 하지만 이러한 수준의 유연성을 모든 최신 GPU가 지원하지는 않습니다. 다행히 지난 10년 동안 바인드리스(bindless) 텍스처 샘플러 설계는 256비트 원시 디스크립터와 인덱싱된 디스크립터 힙이라는 두 가지 주요 방식으로 통합되었습니다.

AMD의 원시 디스크립터 방식은 GPU 메모리에서 256비트 디스크립터를 직접 컴퓨트 유닛의 스칼라 레지스터로 로드합니다. 8개의 연속된 32비트 스칼라 레지스터에 단일 디스크립터가 포함됩니다. SIMD 텍스처 샘플 명령어 실행 중, 셰이더 코어는 256비트 텍스처 디스크립터와 레인별 UV 좌표를 샘플러 유닛으로 전송합니다. 이를 통해 샘플러는 어떤 간접 참조도 없이 텍셀을 주소 지정하고 로드하는 데 필요한 모든 데이터를 얻게 됩니다. 단점은 256비트 디스크립터가 많은 레지스터 공간을 차지하며, 각 샘플 명령어마다 샘플러로 다시 전송해야 한다는 점입니다.

인덱스 기반 디스크립터 힙 방식은 32비트 인덱스를 사용합니다(구형 Intel 내장 GPU에서는 20비트). 32비트 인덱스는 구조체에 저장하기 쉽고 표준 SIMD 레지스터에 로드하기 좋으며 전달하기에도 효율적입니다. SIMD 샘플 명령어를 수행하는 동안 쉐이더 코어는 텍스처 인덱스와 레인별 UV를 샘플러 유닛으로 보냅니다. 샘플러는 `힙 베이스 주소 + 텍스처 인덱스 * 스트라이드(현대 GPU에서는 256비트)` 연산을 통해 얻어낸 디스크립터 힙 주소에서 디스크립터를 가져옵니다. 텍스처 힙 베이스 주소는 드라이버에 의해 추상화되거나(Vulkan 및 Metal), 사용자가 제공할 수 있습니다(DirectX 12의 SetDescriptorHeaps). 텍스처 힙 베이스 주소를 변경하면 내부 파이프라인 배리어가 발생할 수 있습니다(구형 하드웨어의 경우). 현대 GPU에서는 텍스처 힙의 64비트 베이스 주소가 종종 각 샘플 명령어 데이터의 일부로 포함되어 있어, 여러 힙에서 원활하게 샘플링이 가능합니다(레인별로 64비트 베이스 + 32비트 오프셋). 샘플러 유닛은 첫 번째 접근 후 간접 읽기를 피하기 위해 아주 작은 내부 디스크립터 캐시를 가지고 있습니다. 이러한 디스크립터 캐시는 디스크립터 힙이 수정될 때마다 무효화해야 합니다.

몇 년 전만 해도 AMD의 스칼라 레지스터 기반 텍스처 디스크립터가 장기적으로 승리할 공식처럼 보였습니다. 스칼라 레지스터는 디스크립터 힙보다 유연하여 디스크립터를 GPU 데이터 구조 내에 직접 포함할 수 있게 합니다. 하지만 단점도 있습니다. 레이 트레이싱과 지연 텍스처링(Nanite) 같은 현대 GPU 워크로드는 균일하지 않은(non-uniform) 텍스처 인덱스에 의존합니다. 텍스처 힙 인덱스가 SIMD 웨이브(워프)에서 균일하지 않은 경우가 많습니다. 32비트 힙 인덱스는 4바이트에 불과하므로 레인(lane)별로 전송할 수 있습니다. 반면 256비트 디스크립터는 32바이트입니다. 레인마다 완전한 256비트 디스크립터를 가져와서 전송하는 것은 현실적으로 불가능합니다. 현대의 Nvidia, Apple, Qualcomm GPU는 샘플 명령어에서 레인별 디스크립터 인덱스 모드를 지원하여 균일하지 않은 경우의 효율성을 높입니다. 샘플러 유닛은 필요하다면 내부 루프를 수행합니다. 샘플러 유닛의 입력/출력은 힙 인덱스의 일관성 여부와 상관없이 한 번만 전송됩니다. AMD의 스칼라 레지스터 기반 디스크립터 아키텍처는 셰이더 컴파일러가 텍스처 샘플 명령어 주변에 스칼라화(scalarization) 루프를 생성하도록 요구합니다. 이는 추가 ALU 사이클을 소모하고 (부분적으로 마스크된) 샘플러 데이터를 여러 번 전송하고 수신해야 합니다. 이것이 Nvidia가 AMD보다 레이 트레이싱에서 더 빠른 이유 중 하나입니다. ARM과 Intel도 32비트 힙 인덱스를 사용하지만(Nvidia, Qualcomm, Apple처럼), 최신 아키텍처에는 아직 레인별 힙 인덱스 모드가 없습니다. 비균일 인덱스(non-uniform index)의 경우 AMD와 유사한 스칼라화 루프를 생성합니다.

All of these differences can be wrapped under an unified texture descriptor heap abstraction. The de-facto texture descriptor size is 256 bits (192 bits on Apple for a separate texture descriptor, sampler is the remaining 32 bits). The texture heap can be presented as a homogeneous array of 256-bit descriptor blobs. Indexing is trivial. DirectX 12 shader model 6.6 provides a texture heap abstraction like this, but doesn’t allow direct CPU or compute shader write access to the descriptor heap memory. A set of APIs are used for creating descriptors and copying descriptors from the CPU to the GPU. The GPU is not allowed to write the descriptors. Today, we can remove this API abstraction completely by allowing direct CPU and GPU write to the descriptor heap. All we need is a simple (user-land) driver helper function for creating a 256-bit (uint64[4]) hardware specific descriptor blob. Modern GPUs have UMA or PCIe ReBAR. The CPU can directly write descriptor blobs into GPU memory. Users can also use compute shaders to copy or generate descriptors. The shader language has a descriptor creation intrinsic too. It returns a hardware specific uint64x4 descriptor blob (analogous to the CPU API). This approach cuts the API complexity drastically and is both faster and more flexible than the DirectX 12 descriptor update model. Vulkan’s VK_EXT_descriptor_buffer (https://www.khronos.org/blog/vk-ext-descriptor-buffer) extension (2022) is similar to my proposal, allowing direct CPU and GPU write. It is supported by most vendors, but unfortunately is not part of the Vulkan 1.4 core spec.
이러한 모든 차이는 통합된 텍스처 디스크립터 힙 추상화로 감쌀 수 있습니다. 사실상의 텍스처 디스크립터 크기는 256비트(Apple에서는 텍스처를 위해 디스크립터의 192비트를 사용한 후, 나머지 32비트에 샘플러 저장)입니다. 텍스처 힙은 256비트 디스크립터 블롭(Blob)의 균질한 배열로 제시될 수 있습니다. 인덱싱은 단순합니다. DirectX 12 셰이더 모델 6.6은 이와 같은 텍스처 힙 추상화를 제공하지만, 디스크립터 힙 메모리에 대한 직접적인 CPU 또는 컴퓨트 셰이더 쓰기 액세스를 허용하지 않습니다. 디스크립터를 생성하고 CPU에서 GPU로 디스크립터를 복사하기 위해 일련의 API가 사용됩니다. GPU는 디스크립터를 쓸 수 없습니다. 오늘날 우리는 디스크립터 힙에 대한 직접적인 CPU와 GPU 쓰기를 허용함으로써 이 API 추상화를 완전히 제거할 수 있습니다. 우리에게 필요한 것은 256비트(uint64[4]) 하드웨어 특정 디스크립터 블롭을 생성하는 간단한(사용자 공간) 드라이버 도우미 함수뿐입니다. 최신 GPU는 UMA 또는 PCIe ReBAR를 갖추고 있습니다. CPU는 GPU 메모리에 디스크립터 블롭을 직접 쓸 수 있습니다. 사용자는 컴퓨트 셰이더를 사용하여 디스크립터를 복사하거나 생성할 수도 있습니다. 셰이더 언어에도 디스크립터 생성 내장 함수가 있습니다. 하드웨어별 uint64x4 디스크립터 블롭(blob)을 반환합니다(CPU API와 유사). 이 접근 방식은 API 복잡성을 획기적으로 줄여주며, DirectX 12 디스크립터 업데이트 모델보다 더 빠르고 유연합니다. Vulkan의 [VK_EXT_descriptor_buffer](https://www.khronos.org/blog/vk-ext-descriptor-buffer) 확장(2022년)은 제안과 유사하며, CPU와 GPU의 직접 쓰기를 허용합니다. 대부든 벤더가 지원하지만, 안타깝게도 Vulkan 1.4 코어 사양의 일부는 아닙니다.

```c++
// App startup: Allocate a texture descriptor heap (for example 65536 descriptors)
GpuTextureDescriptor *textureHeap = gpuMalloc<GpuTextureDescriptor>(65536);

// Load an image using a 3rd party library
auto pngImage = pngLoad("cat.png");
auto uploadMemory = uploadBumpAllocator.allocate(pngImage.byteSize); // Custom bump allocator (wraps gpuMalloc ptr)
pngImage.load(uploadMemory.cpu);

// Allocate GPU memory for our texture (optimal layout with metadata)
GpuTextureDesc textureDesc { .dimensions = pngImage.dimensions, .format = FORMAT_RGBA8_UNORM, .usage = SAMPLED };
GpuTextureSizeAlign textureSizeAlign = gpuTextureSizeAlign(textureDesc);
void *texturePtr = gpuMalloc(textureSizeAlign.size, textureSizeAlign.align, MEMORY_GPU);
GpuTexture texture = gpuCreateTexture(textureDesc, texturePtr);

// Create a 256-bit texture view descriptor and store it
textureHeap[0] = gpuTextureViewDescriptor(texture, { .format = FORMAT_RGBA8_UNORM });

// Batched upload: begin
GpuCommandBuffer uploadCommandBuffer = gpuStartCommandRecording(queue);

// Copy all textures here!
gpuCopyToTexture(uploadCommandBuffer, texturePtr, uploadMemory.gpu, texture);
// TODO other textures...

// Batched upload: end
gpuBarrier(uploadCommandBuffer, STAGE_TRANSFER, STAGE_ALL, HAZARD_DESCRIPTORS);
gpuSubmit(queue, { uploadCommandBuffer });

// Later during rendering...
gpuSetActiveTextureHeapPtr(commandBuffer, gpuHostToDevicePointer(textureHeap));
```





-작성중-

------

## QnA

Q. Lane? 

A. 병렬 실행되는 워프 내의 개별 스레드



Q. A 32-bit heap index is just 4 bytes, we can send it per lane. In contrast, a 256-bit descriptor is 32 bytes. It is not feasible to fetch and send a full 256-bit descriptor per lane. >> 32비트(4바이트) 인덱스를 사용하는 이유?

A. 

* 32-bit 인덱스:

  - 작음 (4바이트)

  - 32개 레인 = 128바이트 (한 번에 전송 가능)

  - 충분한 용량(2^32개 텍스쳐)

  - non-uniform 한 인덱싱에 최적

* 256-bit 디스크립터:
  * 큼 (32바이트)
  * 32개 레인 = 1024바이트(현실적으로 불가)
  * 균일 인덱싱에서만 사용 가능(모든 레인이 서로 같은 디스크립터 세팅으로 이루어질 때)

GPU는 워프 내 레인들이 개별 스택 메모리를 가지지 않고(로컬 메모리를 가질 수 있지만 VRAM에 할당하므로 느림) 공유 레지스터 집합을 사용함. 이 공유 레지스터 파일은 총 512바이트.

만약 모든 레인에서 서로 다른 디스크립터를 사용해야 한다면? ex) 레이 트레이싱으로 32개 픽셀에서 광선을 쐈는데 모두가 다른 머티리얼의 오브젝트와 Intersect하는 경우

-> 디스크립터 인덱싱 -> 4Byte(32-bits) * 32 = 128 Bytes >> 레지스터에 전부 올릴 수 있음

-> 디스크립터 데이터 -> 32Byte(256-bits) * 32 = 1024 Bytes >> 레지스터 전체 용량 초과

따라서 디스크립터 데이터를 사용할 경우 워프 내 모든 스레드가 SIMT(Parallel)하게 동작이 불가능해져서 병목이 발생함.

