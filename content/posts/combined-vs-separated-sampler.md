---
title: "Vulkan에서 Combined Image Sampler 타입을 지원하는 이유"
author: "Yongsik Im"
authorAvatarPath: "/images/profile.jpg"
date: "2026-02-14"
summary: " "
description: ""
toc: true
readTime: true
autonumber: true
math: true
tags: ["Graphics"]
showTags: false
hideBackToTop: false
---
## 서론
[개인 프로젝트](https://www.github.com/hamsik2rang/HSMR)로 Vulkan, Metal 등의 그래픽스 API를 전부 지원하는 크로스 플랫폼 어플리케이션을 개발하던 도중, 그 전까지 크게 생각하지 않았는데 갑자기 의문점이 하나 생겼다. '왜 Vulkan만 API레벨에서 Combined Image Sampler를 지원하는가?' 였다.  
물론 OpenGL 역시 해당 타입을 지원하지만, 2026년 현재 시점에서 구형 기기를 반드시 지원해야 하는 것이 아니라면 OpenGL을 메인 API로 사용하는 렌더링 어플리케이션을 개발할 이유가 없기 때문에, 사실상 '모던 API'중에서는 Vulkan만이 지원하고 있다고 보는 것이 타당하다.

## 가설
자세히 찾아보기 전에 언뜻 생각하기로는 다음과 같은 이유 중 하나일 것이라고 생각했다.
* Android/Linux 플랫폼의 하드웨어 중 텍스쳐와 샘플러 디스크립터를 통합적으로 관리하는 것만을 지원하거나, 그러한 방식으로 사용할 때 성능 상의 이점이 큰 디바이스들이 존재하기 때문에 지원한다.
* Vulkan은 Metal, DirectX 12와는 달리 모든 디바이스에 대한 '표준'을 설계 철학으로 내세우기 때문에, Vulkan 이전의 표준 API였던 OpenGL(과 GLSL)의 하위 호환성을 위해 지원한다.

## Vulkan의 공식 정보들
일단 Vulkan의 텍스쳐/샘플러 바인딩 모델을 먼저 살펴보자면, 다음과 같은 두 가지 바인딩 모델을 제공한다:

| 모델 | Descriptor Type(s) | 개수 (N텍스처, M샘플러) |
|---|---|---|
| **Combined** | `COMBINED_IMAGE_SAMPLER` | N * M 개 |
| **Separated** | `SAMPLED_IMAGE` + `SAMPLER` | N + M 개 |

텍스쳐 샘플링과 관련된 [Vulkan Spec](https://docs.vulkan.org/spec/latest/chapters/descriptorsets.html#descriptorsets-combinedimagesampler)과 [Vulkan 공식 Tutorial](https://docs.vulkan.org/tutorial/latest/06_Texture_mapping/02_Combined_image_sampler.html)에서는 다음과 같이 명시되어 있다.

> On some implementations, it may be more efficient to sample from an image using a combination of sampler and sampled image that are stored together in the descriptor set in a combined descriptor.

> ...While we’ll be using a combined image sampler in this tutorial, Vulkan also supports separate descriptors for samplers (VK_DESCRIPTOR_TYPE_SAMPLER) and sampled images (VK_DESCRIPTOR_TYPE_SAMPLED_IMAGE). Using separate descriptors allows you to reuse the same sampler with multiple images or access the same image with different sampling parameters. This can be more efficient in scenarios where you have many textures that use identical sampling configurations. However, the combined image sampler is often more convenient and **can offer better performance on some hardware due to optimized cache usage**.

즉, 일반적으로 샘플링 방식을 재사용하거나, 단일 이미지를 여러 생플링 방식으로 샘플링해야 하는 경우라면 `Seperated` 방식을 이용하는 것이 좋으나  `Combined` 방식은 (관리할 핸들이 통합되므로) 더 편리할 수 있고, 특정한 하드웨어(드라이버 구현)에서는 해당 하드웨어에 최적화된 캐시 사용을 통해 더 나은 성능을 보일 수 있다는 것이다.

## 하드웨어 성능 차이?
그렇다면 어떤 하드웨어에서 Combined Image Sampler가 더 나은 성능의 동작을 보일 수 있을까.
대부분의 벤더(Nvidia, AMD, ARM, Qualcomm 등)들의 자료들을 찾아 봤지만 이에 대한 직접적인 언급이 없었기에 추측에 의존할 수 밖에 없는데, [MESA Driver](https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/amd/vulkan/radv_descriptor_set.c)의 구현을 보면 다음과 같이 Combined Image Sampler에 대한 디스크립터 업데이트를 처리하는 로직을 볼 수 있었다.

```c
static void radv_update_descriptor_sets_impl(...) {
//...
case VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER: {
    if (binding_layout->has_ycbcr_sampler) {
        radv_write_image_descriptor_ycbcr_impl(device, cmd_buffer, ptr, buffer_list, writeset->pImageInfo + j);
    } else {
        radv_write_combined_image_sampler_descriptor(device, cmd_buffer, ptr, buffer_list,
                                                    writeset->descriptorType, writeset->pImageInfo + j,
                                                    !binding_layout->immutable_samplers_offset);
    }

    if (copy_immutable_samplers) {
        const uint32_t sampler_offset = radv_get_combined_image_sampler_offset(pdev);
        const unsigned idx = writeset->dstArrayElement + j;

        memcpy((char *)ptr + sampler_offset, samplers + 4 * idx, RADV_SAMPLER_DESC_SIZE);
    }
    break;
}
```
Combined Image Sampler 디스크립터에 Immutable Sampler를 적용할 때, 해당 디스크립터의 base address에서 offset만큼 떨어진 곳에 샘플러 디스크립터를 그대로 복사한다. 이는 Combined Image Sampler가 단순히 Sampled Image와 Sampler 디스크립터를 메모리 공간 상에 연속으로 배치하는 형태일 뿐임을 암시한다.

일반적으로 GPU의 캐시 라인 크기는 64바이트이고, (AMD RDAN 아키텍쳐 기준) 텍스쳐 디스크립터가 64바이트, 샘플러 디스크립터가 32바이트이기 때문에 Combined Image Sampler를 이용할 경우 한 번의 읽기 연산만으로 두 디스크립터가 모두 캐시에 올라오게 된다(정확히는 128 Bytes aligned Pre-fetch). Seperated 핸들을 사용하게 된다면 텍스쳐 디스크립터를 읽어오더라도 샘플러 디스크립터가 프리페치될 것이라는 보장이 없기 때문에 캐시 미스가 발생할 가능성이 있다.
![img-0](/post_images/combined-vs-separated-sampler/0.png)
*Mobile GPU approaches to power efficiency - Qualcomm, 2019*  

캐시 미스의 발생은 디스크립터 참조를 위해 VRAM에 접근해야 한다는 것을 의미하는데, 이는 특히 모바일에서 크게 문제가 될 수 있다. 모바일 AP는 데스크톱용 GPU에 비해 대역폭이 현저히 낮기 때문에, VRAM 참조에 의한 병목 현상이 크게 나타날 수 있기 때문이다.

그 외에도 API 사양 측면에서, 디바이스마다 차이가 있지만 일반적으로 파이프라인에 세팅할 수 있는 디스크립터 세트의 개수(슬롯)에는 제한이 있는데, Combined Image Sampler의 경우 Texture와 Sampler를 각각의 디스크립터로 사용하는 것에 비해 차지하는 슬롯의 수가 적으므로(2개 -> 1개), 셰이더 프로그램이 더 많은 디스크립터 세트를 사용할 수 있다는 이점도 분명히 있다. [Nvidia의 아티클](https://developer.nvidia.com/blog/advanced-api-performance-descriptors/)을 보면, 다음과 같이 Combined Image Sampler 사용을 권장하고, 동시에 디스크립터 세트 개수를 최소화하라는 권고 사항들이 적혀 있다.

* **Try to keep the number of descriptor sets in pipeline layouts as low as possible.**
* **Prefer using combined image and sampler descriptors.**  
## 현실
상술한 내용들을 기반해 결론을 내리면, 모바일 디바이스와 같이 대역폭이 낮은 경우를 포함한 대부분의 경우에서 Combined Image Sampler사용을 적극적으로 권장해야 한다. 그런데 Vulkan 공식 문서나 튜토리얼을 포함해 실제 벤더들의 가이드 문서들을 뒤져 봐도, 관련한(Combined 디스크립터를 권장하는) 내용을 찾아보기 어렵다. 그 이유는 무엇일까?  

이는 상술한 근거들이 실제로는 최신 하드웨어에는 적용되지 않거나, 다른 관점으로 해석되어야 하기 때문이다.
가장 우선적으로, Combined Descriptor는 디스크립터 세트 조합이 많아질 수록 재사용성에서 열위를 보인다. 최근의 렌더링 환경에서는 PSO **폭발**이 발생하게 되는데 이 때 Combined 디스크립터를 사용하게 될 경우 대응 가능한 조합 수가 훨씬 줄어들게 된다. 맨 위에 제시했던 테이블과 같이, Combined Image Sampler만으로 모든 텍스쳐 연산 디스크립터를 구성할 경우 **N * M** 개의 디스크립터 세트가 필요하지만, Seperate Image + Sampler로 구성할 경우 **N + M**개의 디스크립터 세트만 가지면 충분하다.
디스크립터 세트의 최소화는 파이프라인 레이아웃 설정 시점에 호출하는 셋업 콜의 부하를 잠재적으로 낮출 수도 있을 것이므로, 이것이 캐시 미스의 단점을 상쇄할 수 있을 것이다.

또, 최신 하드웨어들은 디스크립터 세트 관리를 **더 효율적으로** 할 수 있는 Bindless, 즉 디스크립터 인덱싱을 기본적으로 지원하기 때문에(Vulkan 1.3 Core), 이를 지원하는 하드웨어를 타겟으로 한다면 **Bindless 시스템을 지원하**는 것이 더욱 권장되는 방법이다. 이는 앞서 참고한 Nvidia의 아티클에서도 명시되어 있다.
![img-1](/post_images/combined-vs-separated-sampler/1.png)

## 결론 

결국, Android 디바이스에서 **OpenGL ES을 함께 지원해야 하는 크로스 플랫폼 어플리케이션의 경우라면 디바이스 커버리지에 따라 Combined Image Sampler를 지원**하거나, 지원해야만 하는 상황이 있을 수 있지만, Vulkan/Metal 과 같은 **모던 API들로 모바일 디바이스를 모두 지원할 수 있다면 굳이 Combined Image Sampler를 사용할 필요가 없어 보인다**. 오히려 **Combined타입의 지원을 위한 추상화 코드가 렌더링 백엔드(와 쉐이더 트랜스컴파일 워크플로)의 복잡도를 증가**시키는 결과로 이어지게 될 것이며, 상대적으로 더 최신 디바이스들(혹은 Desktop Only)만을 지원하고자 한다면 **Bindless 렌더링 시스템을 구축**하는 것을 고려하는 게 바람직해 보인다.

## 부록
### 셰이더 지원 현황
#### GLSL

```glsl
// === Combined Image Sampler ===
layout(set=0, binding=0) uniform sampler2D albedoTex;

void main() {
    vec4 color = texture(albedoTex, uv);  // sampler2D 자체가 texture+sampler
}
```

```glsl
// === Separated Image + Sampler ===
layout(set=0, binding=0) uniform texture2D albedoTex;   // image만
layout(set=0, binding=1) uniform sampler   linearSampler; // sampler만

void main() {
    // 런타임에 조합하여 샘플링
    vec4 color = texture(sampler2D(albedoTex, linearSampler), uv);
}
```

| 타입 | GLSL 키워드 | Vulkan Descriptor |
|---|---|---|
| Combined | `sampler2D` | `COMBINED_IMAGE_SAMPLER` |
| Image만 | `texture2D` | `SAMPLED_IMAGE` |
| Sampler만 | `sampler` | `SAMPLER` |

#### HLSL

HLSL에는 `sampler2D` 같은 combined 타입이 네이티브로 없다. `Texture2D` + `SamplerState`가 기본이며, Vulkan에서 combined으로 매핑하려면 별도 어트리뷰트가 필요하다.

```hlsl
// === Separated (기본, DX12/Vulkan 공통) ===
Texture2D    albedoTex    : register(t0);
SamplerState linearSampler : register(s0);

float4 main(float2 uv) : SV_Target {
    return albedoTex.Sample(linearSampler, uv);
}
```

```hlsl
// === Combined (Vulkan 전용 — [[vk::combinedImageSampler]] 어트리뷰트) ===
// 같은 binding(0)에 texture와 sampler를 묶음
[[vk::binding(0, 0, combinedImageSampler)]]
Texture2D    albedoTex;

[[vk::binding(0, 0, combinedImageSampler)]]
SamplerState linearSampler;

float4 main(float2 uv) : SV_Target {
    return albedoTex.Sample(linearSampler, uv);  // 문법은 동일
}
```

| 모델 | HLSL 선언 | 비고 |
|---|---|---|
| Separated | `Texture2D` + `SamplerState` (각각 다른 register) | DX12/Vulkan 공통 |
| Combined | `Texture2D` + `SamplerState` + `[[vk::combinedImageSampler]]` | Vulkan 전용 어트리뷰트 |

#### Slang

```slang
// === Combined (Sampler2D 타입) ===
Sampler2D albedoSampler;

[shader("fragment")]
float4 FragmentMain(float2 uv : TEXCOORD0) : SV_Target {
    return albedoSampler.Sample(uv);  // texture+sampler 내장
}
```

```slang
// === Separated (Texture2D + SamplerState) ===
Texture2D    albedoTexture;
SamplerState linearSampler;

[shader("fragment")]
float4 FragmentMain(float2 uv : TEXCOORD0) : SV_Target {
    return albedoTexture.Sample(linearSampler, uv);
}
```

#### Slang 크로스 컴파일 동작

| Slang 선언 | → SPIR-V (Vulkan) | → MSL (Metal) |
|---|---|---|
| `Sampler2D` | `OpTypeSampledImage` (Combined) | `texture2d` + `sampler` (분리) |
| `Texture2D` + `SamplerState` | `OpTypeImage` + `OpTypeSampler` (Separated) | `texture2d` + `sampler` (분리) |

Metal은 어떤 경우든 항상 분리된다. 따라서 **Slang에서 `Sampler2D`를 사용하면 Vulkan에서는 Combined, Metal에서는 자동 분리**가 된다.

#### Slang 리플렉션에서의 차이

| Slang 선언 | 리플렉션 카테고리 | 추출 필요 항목 |
|---|---|---|
| `Sampler2D` | `DescriptorTableSlot` | Vulkan: texture만 / Metal: texture + sampler 별도 추출 |
| `Texture2D` | `ShaderResource` | texture |
| `SamplerState` | `SamplerState` | sampler |

이것이 이 프로젝트에서 sampler 바인딩 누락 버그의 원인이었다. `Sampler2D`의 `DescriptorTableSlot` 케이스에서 Metal 타겟일 때 sampler 바인딩을 별도로 추출해야 한다.

### API별 지원 현황

| API | Combined 지원 | 이유 |
|---|---|---|
| **OpenGL/ES** | 유일한 방식 (원래 설계), Core 4.2/ES 3.1부터 분리 지원 | 텍스처 = 샘플링 파라미터 내장 객체 |
| **Vulkan** | 지원 (선택) | OpenGL/ES 셰이더 호환성, 마이그레이션 비용 절감 |
| **DirectX 12** | 미지원 | 레거시 단절, 분리 모델로 통일 |
| **Metal** | 미지원 | 처음부터 분리 설계 |



--- 
# 참고자료
* [AMD - "RDNA3" Instruction Set Architecture: Reference Guide](https://docs.amd.com/v/u/en-US/rdna3-shader-instruction-set-architecture-feb-2023_0) - Chapter 10.1 Image Instructions
* [Nvidia - Tips and Tricks: Vulkan Dos and Don'ts](https://developer.nvidia.com/blog/vulkan-dos-donts/)
* [Nvidia - Advanced API Performance: Descriptors](https://developer.nvidia.com/blog/advanced-api-performance-descriptors/)
* [AMD - Vulkan Fast Paths, GDC 2016](https://gpuopen.com/download/VulkanFastPaths.pdf)
* [AMD(GPUOpen) - RDNA Performance Guide](https://gpuopen.com/learn/rdna-performance-guide/)
* [ARM - Application best practice for Vulkan](https://developer.arm.com/community/arm-community-blogs/b/mobile-graphics-and-gaming-blog/posts/mali-bifrost-usage-recommendations-for-texture-and-sampler-descriptors)
* [Arm Mali Best Practices](https://armkeil.blob.core.windows.net/developer/Arm%20Developer%20Community/PDF/Arm%20Mali%20GPU%20Best%20Practices.pdf)
* [Qualcomm - Mobile GPU approaches to power efficiency](https://www.highperformancegraphics.org/wp-content/uploads/2019/hot3d/mobile_gpu_power_and_performance.pdf)