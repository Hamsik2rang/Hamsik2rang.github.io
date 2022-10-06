---
layout: post
title: SIMD에 대해 알아보자
subtitle: SIMD의 개념과 간단한 용례를 알아보자.
tags: [Computer Architecture, Cpp]
author: Im Yongsik
comments: True
use_math: True
---

## 1. SIMD란?

SIMD는 Single Instruction Multiple Data의 약자로, **하나의 명령어로 여러 데이터를 처리하는 것**을 의미한다. 이는 플린의 분류(Flynn's taxonomy)상 병렬 프로세싱(단일 명령어-복수 자료)의 범주에 속한다.

SIMD연산은 특수한 명령어 집합을 통해 수행되는데, MMX(MultiMedia eXtension), SSE(Streaming SIMD Extension), AVX(Advanced Vector eXtension), AVX512등이 그것이다. 이 글에서는 SSE 명령어 집합을 기준으로 SIMD를 살펴볼 것이다.

## 2. SSE

SSE는 Streaming SIMD Extension의 약자이며, SIMD연산 명령어 세트 중 하나이다. SSE는 아래와 같은 데이터들의 동시 연산을 수행할 수 있도록 한다.

* 32bits 실수 4개
* 64bits 실수 2개
* 8bits 정수(signed/unsigned) 16개 (SSE2)
* 16bits 정수(signed/unsigned) 8개 (SSE2)
* 32bits 정수(signed/unsigned) 4개 (SSE2)
* 64bits 정수(signed/unsigned) 2개 (SSE2)

이를 위해서 SSE 명령어 세트는 128bits 크기를 가진 XMM 레지스터 8개를 이용하는데, 아래와 같은 구조이다.

<p align="center">
    <img src="{{site.baseurl}}/assets/img/posts/2022-10-06/What-is-SIMD/img01.jpg"/>
</p>

SSE 명령어를 이용하려면 먼저 XMM에 데이터 집합을 로드한 후 명령어를 수행하도록 하면 된다.

## 3. SIMD와 SISD의 차이

우리가 일반적으로 수행하는 명령어 세트는 SISD 명령어라 하는데, Single Instruction Single Data의 약자이다. 즉, 두 값의 합을 수행하려면 두 값을 각각 레지스터에 올린 후 ALU, 혹은 FPU를 이용해 덧셈을 수행하고, 그 결과를 레지스터에 올림으로써 연산이 완료되는 것이다.

<br>

그러나 SIMD 덧셈 연산을 수행하는 경우 위에 적힌 데이터 집합을 동시에 XMM레지스터에 각각 올린 후 SIMD연산을 수행해 데이터 집합 단위로 덧셈을 수행한 후 그 결과를 XMM레지스터에 올림으로써 연산을 완료하게 된다.

<br>

32bits 실수 X1, X2, X3, X4와 Y1, Y2, Y3, Y4를 서로 각각 더해주는 연산을 SISD와 SIMD로 수행할 경우 각각 아래 도식처럼 표현될 수 있다(편의상 덧셈 연산이 1Tick만에 수행된다고 가정한다). 

<p align="center">
    <img src="{{site.baseurl}}/assets/img/posts/2022-10-06/What-is-SIMD/img02.jpg"/>
</p>

보다시피 X1-Y1, X2-Y2, X3-Y3, X4-Y4 덧셈 연산을 하는 데 SISD의 경우 4Tick의 시간이 소요되지만 SIMD의 경우 1Tick의 시간이 소요됨을 알 수 있다.

그렇다면 SIMD는 어느 경우에 주로 사용될까? 동일 형태의 데이터 묶음의 연산은 벡터나 행렬의 연산이라고 생각할 수 있으므로 벡터/행렬을 주로 다루는 분야에서 성능 향상을 도모할 때 SIMD 연산을 고려해볼 수 있으며, 행렬/벡터를 주로 사용하는 게임이나 인공지능, 컴퓨터비전 등의 분야에서 사용될 수 있을 것이다.

참고로, 현대 컴파일러는 32bits float의 연산을 수행할 시 따로 SIMD연산을 명시하지 않아도 내부적으로 XMM 레지스터를 이용해 SSE연산을 수행한다(단, Single Data). 따라서 벡터 연산을 SISD형태로 구현하더라도 컴파일러가 이를 SIMD로 연산하도록 최적화할 수도 있다. 그러나 컴파일러의 최적화 여부는 프로그래머가 세부적으로 조정할 수 없기 때문에 항상 이러한 최적화를 기대할 순 없으므로 수학 함수 등과 같이 확실히 SIMD연산을 수행해야 하는 경우라면 컴파일러 최적화에 기대지 말고 직접 SIMD코드를 이용해 함수를 작성하는 것이 좋다.

## 4. SIMD를 사용하기 위한 준비

SIMD연산을 이용하는 방법은 크게 두 가지가 있다.

* 어셈블리어 코드 작성
* Complier Intrinsic 코드 작성

어셈블리어 코드는 말 그대로 어셈블리어를 이용해 SIMD연산을 명시적으로 작성하는 방법을 의미하며, Complier Intrinsic은 C/C++에서 지원하는 컴파일러 내장 함수를 이용해 SIMD연산을 수행하는 방법을 의미한다. Complier Intinsinc을 위한 함수들은 `intrin.h` 표준 라이브러리에 정의되어 있다.

이 포스트에선 두 가지 방법 모두를 이용해 Visual Studio에서의 간단한 용례를 보일 것이다.

Visual Studio의 32bit 컴파일러는 Inline-Assembly를 지원하므로 이를 이용해 작성해도 된다.

## 5. SIMD를 이용한 내적 연산

이제 실제로 SIMD연산을 수행해 보자. 본 포스트에서는 예시로 4차원 벡터의 내적을 SIMD코드로 작성해 볼 것이다.

#### 5.1. __m128

**`__m128`**은 C 내장 타입으로, XMM 레지스터에 들어갈 수 있는 데이터 크기의 공용체(union)로 구성되어 있다.

실제 `intrin.h` 헤더상의 정의를 보면 아래와 같다.

<br>

{%highlight cpp%}

typedef union __declspec(intrin_type) __declspec(align(16)) __m128 {
     float               			m128_f32[4];
     unsigned __int64       m128_u64[2];
     __int8              		m128_i8[16];
     __int16             		m128_i16[8];
     __int32             		m128_i32[4];
     __int64             		m128_i64[2];
     unsigned __int8     	m128_u8[16];
     unsigned __int16      m128_u16[8];
     unsigned __int32      m128_u32[4];
 } __m128;

{%endhighlight%}

<br>

앞서 살펴보았던 SSE가 지원하는 데이터 집합 종류와 일치하는 것을 볼 수 있다.

또, 눈여겨보아야 할 점이 **`__declspec(align(16))`**인데, XMM 레지스터에 데이터를 올리기 위해선 올릴 데이터가 16바이트 정렬이 되어 있음이 보장되어야 한다. 이를 위해 `__declspec(align(16))`을 이용해(혹은 C++의 경우 **`alignas(16)`**을 이용해)컴파일러에 해당 타입을 16바이트 정렬하도록 강제한다.

#### 5.2. Set

SIMD연산을 위해 맨 처음 해야 할 일은 `__m128`타입으로 데이터를 준비하는 것이다. 

32bits 실수 4개를 묶어 __m128타입으로 생성하고 싶다면 아래와 같이 사용한다.

{%highlight cpp%}

#include <intrin.h>

__m128 ps = _mm_set_ps(1.0f,  2.0f,  3.0f,  4.0f);

{%endhighlight%}

#### 5.3. Load

만약 이미 16바이트 정렬된 데이터 집합이 존재한다면 load 연산을 이용해 __m128 타입을 정의할 수 있다.

* Complier Intrinsic

{%highlight cpp%}

#include <intrin.h>

__declspec(align(16)) float data1[] = {1.0f,  2.0f,  3.0f,  4.0f};
__declspec(align(16)) float data2[] = {4.0f, 3.0f, 2.0f, 1.0f};

__m128 ps1 = _mm_load_ps(data1);
__m128 ps2 = _mm_load_ps(data2);

{%endhighlight%}

* Assembly

{%highlight asm%}

movaps xmm0, xmmword ptr[data1]
movaps xmm1, xmmword ptr[data2]

{%endhighlight%}

보다시피 Complier Intrinsic을 이용할 경우 `_mm_load_ps` 함수를, Assembly의 경우 `movaps` 니모닉을 이용한다.

만약 16바이트 정렬되지 않은 데이터를 이용해 load를 하는 경우 `_mm_loadu_ps`함수를, Assembly의 경우 `movups`니모닉을 이용한다. 그러나 정렬되지 않은 데이터를 load하는 경우 내부적으로 재정렬 후 load하기 때문에 `_mm_load_ps` 혹은 `movaps`에 비해 속도가 느리다.

#### 5.4. Mul

알다시피 벡터의 내적은 두 벡터의 같은 성분을 곱한 후 그 결과의 각 성분들을 더해야 한다. 

두 __m128 타입의 데이터를 곱하는 경우 아래와 같이 수행한다.

* Compiler Intrinsic

{%highlight cpp%}

__m128 mul = _mm_mul_ps(ps1, ps2);

{%endhighlight%}

* Assembly

{%highlight asm%}

mulps xmm0, xmm1

{%endhighlight%}

#### 5.5. hadd

이제 결과로 나온 벡터의 각 성분을 더해야 한다. 한 벡터의 성분을 더하기 위해선 조금 특별한 연산을 이용해야 하는데, 바로 hadd 연산이다.

* Compiler Intrinsic

{%highlight cpp%}

__m128 hadd = _mm_hadd_ps(mul, mul)

{%endhighlight%}

* Assembly

{%highlight asm%}

haddps xmm0, xmm0

{%endhighlight%}



hadd는 horizontal add의 약자로, 여덟 성분(네 성분의 데이터 집합 2개)을 서로 이웃한 두 성분끼리 더해서 새로운 4차원 벡터를 만든다.

즉, 도식으로 나타내면 아래와 같다.

<p align="center">
    <img src="{{site.baseurl}}/assets/img/posts/2022-10-06/What-is-SIMD/img03.jpg"/>
</p>

그럼 이제 이를 바탕으로 직접 어떻게 내적을 구할지 고민해 보자. 힌트를 주자면, hadd 두번으로 우리가 원하는 결과를 만들어낼 수 있다.

<p align="center">
    <img src="{{site.baseurl}}/assets/img/posts/2022-10-06/What-is-SIMD/img04.jpg"/>
</p>

충분히 고민해 보았는가?

정답은 아래와 같다.

먼저, 곱한 결과인 `mul`을 자기 자신과 hadd한다.

<p align="center">
    <img src="{{site.baseurl}}/assets/img/posts/2022-10-06/What-is-SIMD/img05.jpg"/>
</p>

이후 그 결과를 또 자기 자신과 hadd한다. 그럼 4차원 벡터의 모든 성분에 결과가 담겨 있게 된다.

<p align="center">
    <img src="{{site.baseurl}}/assets/img/posts/2022-10-06/What-is-SIMD/img06.jpg"/>
</p>

따라서 지금까지의 벡터 내적을 실제 코드로 나타내면 아래와 같다.

* Compiler Intrinsic

{%highlight cpp%}

const float Dot(const float a[4], const float x[4])
{
        // a b c d
	__m128 aps = _mm_load_ps(a);
        // x y z w
	__m128 xps = _mm_load_ps(x);

	//ax by cz dw
	__m128 mul = _mm_mul_ps(aps, xps);
	// ax+by cz+dw ax+by cz+dw
	result = _mm_hadd_ps(mul, mul);
	// ax+by+cz+dw ax+by+cz+dw ax+by+cz+dw ax+by+cz+dw
	result = _mm_hadd_ps(result, result);
	
	return result.m128_f32[0];	// 4성분 중 아무 성분이나 리턴해도 된다.
}

{%endhighlight%}

* Assembly

{%highlight asm%}

movaps xmm0, xmmword ptr [a]
movaps xmm1, xmmword ptr [x]
mulps xmm0, xmm1 ; [ax, by, cz, dw]
haddps xmm0, xmm1 ; [ax+by, cz+dw, ax+by, cz+dw]
haddps xmm0, xmm1 ; [ax+by+cz+dw, ax+by+cz+dw, ax+by+cz+dw, ax+by+cz+dw]
movaps xmmword ptr [result] xmm0

{%endhighlight%}

## 6. 추가 기능 탐구를 위한 레퍼런스

SIMD연산은 위에서 살펴본 연산 외에도 매우 많은 연산들이 존재한다. SIMD연산에 대한 추가 기능들은 다음 레퍼런스들을 이용하자(Intel 기준)

[Intel Intrinsic Guide](https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#ig_expand=4874,950,4874)

[SIMD 프로그래밍 소개#1 - 유영천 님 유튜브](https://www.youtube.com/watch?v=vMmggKKimOw&list=PL00yTT-RECdWsBjP-rQcDBelgehOyToy3&index=14&t=842s)

[KeplerMath](https://github.com/hamsik2rang/KeplerMath) - 수줍게 제가 작성한 SIMD수학 라이브러리도 올려 봅니다..ㅎ
