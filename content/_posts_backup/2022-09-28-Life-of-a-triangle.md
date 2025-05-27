---
layout: post
title: Modern GPU에서 삼각형이 그려지는 과정
subtitle: 현대적인 GPU의 논리 파이프라인을 조명해보자
tags: [Graphics]
author: Im Yongsik
comments: True
use_math: False
---

## Overview

<p align="center">
    <img src="{{site.baseurl}}/assets/img/posts/2022-09-28/Life-of-a-triangle/img01.jpg" />
</p>

## Modern GPU Architecture

과거 GPU(GeForce8XX Series Architecture 이전)는 물리적인 Graphics Pipeline을 따라 삼각형들을 처리했지만, G80이후 논리 파이프라인(Logical pipeline)으로 삼각형들을 처리하게 되었다.

물리 파이프라인의 경우 한 삼각형을 구성하는 모든 구성 데이터에 대해 한 단계가 끝나야 다음 단계를 진행했음과 달리, 논리 파이프라인은 한 삼각형의 구성 데이터도 쉐이더 코어의 진행 상황에 따라 제각기 다른 파이프라인 단계를 진행할 수 있게 되었다. 

<br>

다만 G80의 경우 쉐이더 코어가 정점/픽셀 처리를 모두 수행할 수 있게 된 건 맞지만 여전히 프리미티브/래스터라이제이션 연산은 순서에 맞게 연속적으로 이루어졌는데, Fermi Architecture 이후 완전한 병렬화를 이루게 되었다. 즉, 한 삼각형을 구성하는 정점/픽셀들 중 특정 정점은 정점 쉐이더를 수행하고 있을 때 같은 삼각형의 어떤 픽셀은 래스터 연산이 수행되고 어떤 픽셀은 픽셀 쉐이더가 수행될 수 있다는 뜻이기도 하며, 심지어는 어떤 쉐이더 코어는 해당 삼각형을 모두 처리하고 다음 삼각형에 대한 프리미티브 연산을 수행하고 있을 수도 있다는 것을 의미한다.

<p align="center">
    <img src="{{site.baseurl}}/assets/img/posts/2022-09-28/Life-of-a-triangle/img02.jpg" />
</p>
Fermi는 이와 같은 완전 병렬 연산을 위해 **Giga Thread Engine**을 하나 가지고 있는데, Giga Thread Engine은 모든 진행중인 연산을 관리하는 유닛이며, GPU는 Giga Thread Engine의 관리 하에 4개의 **GPC(Graphics Processing Cluster)**로 분할되어 있다.

<br>

각 GPC는 여러 개의 **SM(Streaming Multiprocessor)**와 하나의 **Raster Engine**을 가지며, 각 GPC간의 교차 연결(통신이나 동기화)을 위한 Crossbar가 존재한다. 이 때 크로스바는 GPC간의 연결 외에도 **ROP(Render Output Unit)**등과의 연결에도 쓰인다.

<br>

각 SM들은 수많은 쉐이더 코어들로 구성되어 있으며 이들을 이용해 쓰레드들을 처리하는데, 쓰레드는 GPU에서 이루어질 수 있는 수학적 연산들을 총칭한다. 대표적인 예가 Vertex Shader나 Pixel Shader 등이 있을 것이다.

<br>

이러한 코어들과 다른 SM 내 유닛들은 Warp Scheduler들에 의해 동작되는데, Warp는 32개의 코어가 묶인 하나의 덩어리를 의미하며 이 말은 즉 Warp마다 한 번에 32개의 쓰레드를 처리할 수 있다는 뜻이다. Warp Scheduler들은 워프들을 관리하는 것 외에도 수행할 명령들을 Dispatch Unit에 전달하기도 한다.

<br>

따라서 코드의 로직은 각각의 쉐이더 코어가 아닌 워프 스케줄러에 의해 다루어진다. 왜냐하면 디스패치 유닛이 명령어 디스패치를(명령어들을 수행) 담당하기 때문이다. 코어들은 개개의 경우 CPU코어와 비교했을 때 매우 멍청하며, 대개 빠른 "연산"만을 할 수 있다.

<br>

지금까지 설명한 이러한 유닛들이 GPU에 얼마나 많이 존재하는지는 GPU구성에 따라 달라지는데(대개 가격과 비례함), 예를 들어 GM204 그래픽카드는 4개의 SM으로 구성된 GPC가 4개 존재하는 반면, Targa X1 그래픽카드는 2개의 SM으로 구성된 GPC가 오직 1개 존재한다. 그러나 두 그래픽카드는 모두 같은 아키텍처(Maxwell Architecture)를 가진다.

<br>

또한, SM의 디자인(쉐이더 코어의 수나 유닛의 구성) 역시 세대를 거듭해 변화하며 점차 효율적으로 진화하고 있다.

## The Logical Pipeline

논리 파이프라인의 흐름을 단순하게 요약하면 다음과 같다.

먼저 어플리케이션에서 이미 드로우 콜을 한 후라 Index/VertexBuffer가 GPU DRAM에 채워져 있다고 가정하자.

<p align="center">
    <img src="{{site.baseurl}}/assets/img/posts/2022-09-28/Life-of-a-triangle/img03.jpg" />
</p>
가장 우선적으로, 드라이버가 드로우 콜의 유효성(Legal)을 검사한 후, GPU가 읽을 수 있게 인코딩된 명령들을 Pushbuffer에 담는다. 이 때 파이프라인 상의 대부분의 병목 현상이 발생하기 때문에, 숙련된 그래픽스 프로그래머는 이 과정에서 API와 GPU를 최대한 효율적으로 사용하고자 노력한다.

<br>

이후 Flush 명령이 호출되면, 드라이버는 Pushbuffer에 채워진 명령들을 수행하기 위해 GPU에 전달한다. 이 때 GPU의 Host Interface는 Front End를 통해 수행할 명령들을 선택한다.

<br>

이 때 Primitive Distributer는 인덱스 버퍼의 인덱스를 이용해 프리미티브 배치를 생성해 GPC들에게 적절히 분배한다.

<p align="center">
    <img src="{{site.baseurl}}/assets/img/posts/2022-09-28/Life-of-a-triangle/img04.jpg" />
</p>
여기까지 마치면 이제 GPC별로 병렬적인 작업이 수행되게 되는데, GPC안에서 임의의 SM 내의 Poly Morph Engine이 프리미티브 배치의 인덱스를 기준으로 정점 데이터를 Fetch하며, 이를 **Vertex Fetch**라 한다.

<br>

데이터가 fetch된 후, 32개의 스레드가 워프 내에서 스케줄되며 fetch된 정점에 대한 작업을 수행한다.

<br>

워프 단위 작업이 수행될 때, 워프 스케줄러는 마스킹 기능을 통해 특정 스레드의 작업을 멈추거나, 진행시킬 수 있다. 이러한 기능이 필요한 이유는 한 워프 내의 스레드들이 분기문을 만났을 때 특정 스레드는 true에 대한 연산을 수행하고, 특정 스레드는 false에 대한 연산을 수행해야 하는 경우가 생기는데 모든 스레드는 워프 단위로 동시에 작동해야 하므로 개개별로 다른 연산을 수행할 수 없다. 따라서 true연산을 할 때에는 false에 해당하는 스레드들을 마스킹 아웃해 잠시 멈춘 후 true연산을 수행하고, 그 다음 true 연산을 마친 스레드들을 마스킹 아웃한 뒤 false 스레드들을 깨워 false연산을 수행한다.

<br>

워프 단위 연산을 할 때 특정 연산의 경우 매우 적은 디스패치 횟수로 수행될 수 있으며, 반대로 상대적으로 많은 디스패치 횟수가 필요한 경우도 존재할 수 있다. 심지어는 Memory Fetch처럼 오랫동안 기다려야 하는 작업도 존재한다.

<br>

 이 때 워프 스케줄러는 해당 동작이 완료될 때까지 대기하는 것이 아니라 다른 워프로 스위칭을 하게 된다. 이 방식이 GPU가 Latency를 극복하는 방법이며 스위칭은 매우 적은 비용으로 빠르게 수행된다. 이는 워프 스케줄러가 모든 워프를 관리하고, 각각의 워프들이 레지스터 파일 내에서 자신에게만 할당된 레지스터를 사용하기 때문이다. 이는 다르게 말하면, 특정 쉐이더가 많은 레지스터 공간을 요구할수록 동시에 더 적은 수의 워프가 스케줄링될 수 있다는 뜻이다. 이용 가능한 워프의 수가 적으면, 스위칭 될 수 있는 워프의 수가 적다는 뜻이므로 GPU가 더 낮은 병렬 효율을 가지게 될 확률이 생긴다.

<p align="center">
    <img src="{{site.baseurl}}/assets/img/posts/2022-09-28/Life-of-a-triangle/img05.jpg" />
</p>

이러한 과정을 거쳐 워프가 프리미티브 처리를 완료했다면 그 결과에 Viewport Transform이 수행된다. 삼각형은 Clipspace volume으로부터 Clipping이 진행되며 rasterization 준비를 하게 된다. 이 때 각각의 작업들로 데이터를 전달하는 과정에서 L1&L2 캐시가 사용된다.

<p align="center">
    <img src="{{site.baseurl}}/assets/img/posts/2022-09-28/Life-of-a-triangle/img06.jpg" />
</p>

뷰포트 변환과 클리핑이 끝나면 래스터라이제이션이 시작된다. Attribute Setup을 통해 프리미티브 데이터를 픽셀 수준으로 보간하고 Raster Engine은 자신이 맡은 그리드 안에 들어온 삼각형의 일부들에 대해 프리미티브 데이터를 픽셀 데이터로 변환한다.

<p align="center">
    <img src="{{site.baseurl}}/assets/img/posts/2022-09-28/Life-of-a-triangle/img07.jpg" />
</p>

래스터라이제이션이 끝나면 픽셀 연산이 수행되는데, 프리미티브 연산과 동일하게 워프 단위로 워프 스케줄러에 의해 수행된다. 마찬가지로 32개의 픽셀이 동시에 스레드를 수행하는데, 알아둘 점은 32개의 픽셀이 정확히는 8개의 2x2 픽셀 그리드 라는 점이다. 이러한 연관 픽셀들을 한 번에 처리함으로써 텍스처 밉맵 필터링과 같은 미분 연산에 대한 이점을 얻어낼 수 있다.

<p align="center">
    <img src="{{site.baseurl}}/assets/img/posts/2022-09-28/Life-of-a-triangle/img08.jpg" />
</p>
픽셀 쉐이딩이 끝나고 나면 그 결과를 ROP Subsystem으로 전달한다. 이 때 지금까지 병렬적으로 수행했던 픽셀 연산들의 결과를 원래 API에 의해 드로우 콜 되었을때의 데이터 순서로 정렬해 전달해야 한다. ROP에서는 Depth testing이나 Blending이 수행되는데, 모든 연산들은 원자적(Atomic)으로 수행된다. 이는 한 픽셀에 삼각형 A의 색상 데이터와 삼각형 B의 깊이 값이 동시에 들어가는 것과 같은 경우를 방지하기 위함이다.

<br>

여기까지 마치면 드디어 프레임버퍼에 특정 픽셀에 대한 값이 쓰여지게 되고, 모든 픽셀들에 적절한 값이 쓰이면 한 프레임에 대한 이미지 생성이 완료된다.

참고: [NVIDIA - Life-of-a-triangle](https://developer.nvidia.com/content/life-triangle-nvidias-logical-pipeline)
