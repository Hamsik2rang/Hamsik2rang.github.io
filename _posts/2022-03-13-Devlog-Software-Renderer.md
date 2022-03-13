---
layout: post
title: 개발 일지 - 3월 2주차
subtitle: Software Renderer 개발 일지
tags: [Devlog]
author: Im Yongsik
comments: True
use_math: True
---

## 0. 서론

이번 주 초까지 가까운 물체의 렌더링 비용과 카메라 회전 개발에 애를 먹고 있었다. 정확히는 다음과 같은 이슈들이 존재했다.

**ERROR**

* 임의의 물체가 카메라에 너무 가까이 있을 시 드로우하는 과정에서 버벅임이 발생한다.
* 카메라 회전이 원하는 대로 정확히 이루어지지 않는다. 시점이 조금씩 멋대로 이동하거나 갑자기 화면이 튀는 현상이 발생한다.

<br/>

시급하진 않지만 원인을 규명해야 할 자잘한 이슈는 다음이 있었다.

**WARNING**

* 특정 헤더 파일을 여러 곳에서 선언 시 `#pragma once`전처리문을 명시하였음에도 중복 선언 이슈가 발생한다.

<br/>

그 외에도 구현해야 할 목표들은 다음이 있었다.

**GOAL**

* Vertex Buffer만을 이용해 드로우하는 구조에서 Index Buffer를 이용한 드로우 방식을 구현해야 한다.
* Input 데이터를 처리하는 별도의 객체를 설계 및 구현해야 한다.

## 1. ERROR

#### 임의의 물체가 카메라에 너무 가까이 있을 시 드로우하는 과정에서 버벅임이 발생한다.

우선 물체가 카메라의 위치에 가까워질수록 드로우 부하가 발생하는 부분은 발생을 확인하는 순간에도

> '별도의 Clipping이 없어 Viewport밖의 물체까지 그려내느라 많은 비용이 소모되는 것이 아닐까?'

라는 생각을 했다.

<br/>

Viewport밖의 물체를 그려내는 연산을 제외하기 위해 Rasterizer 단계에서 Clip Space의 범위($-1<=x<=1, -1<=y<=1, 0<=z<=1$)를 넘어서는 정점들을 모두 걸러내는 Simple한 Culling 코드를 작성하였다.

![]({{site.baseurl}}/assets/img/posts/2022-03-13/Devlog-Software-Renderer/img01.jpg)

그 결과 상당한 효율 향상이 있었다. 이를 통해 추후 Clipping과 Culling 알고리즘들을 활용해 부하를 해결할 수 있다는 확신을 얻었다.

<br/>

그리고 이 외에도 단순히 화면 밖의 정점까지 그린다는 것만으로 이렇게 큰 부하가 걸리는 것을 통해 추가적인 개선 요소를 파악할 수 있었는데, 현재 프로젝트에서 삼각형을 그려내는 코드는 삼각형의 세 정점을 기준으로 Fragment Box를 구해낸 다음, Box의 안에 존재하는 모든 Fragment들에 삼각형의 내/외부 판별을 하기 때문이라는 것을 예측할 수 있었다.

따라서 기존의 방법 대신 Scan Conversion을 이용한 프래그먼트 처리를 구현하는 것이 좋겠다는 [계획](https://github.com/Hamsik2rang/Software-Renderer/issues/7)을 세웠다.

#### 카메라 회전이 원하는 대로 정확히 이루어지지 않는다. 시점이 조금씩 멋대로 이동하거나 갑자기 화면이 튀는 현상이 발생한다.

<br/>

우선 가장 먼저 카메라 회전을 오일러각 회전을 통해 구현했다.  

<br/>

<center>$R = R_zR_yR_x$</center>

<br/>

이므로

<br/>  
$$
R = \left[
\begin{matrix}
1 & 0 & 0 & 0\\
0 & cos\theta_x & -sin\theta_x & 0\\
0 & sin\theta_x & cos\theta_x & 0\\
0 & 0 & 0 & 1
\end{matrix}
\right]
\left[
\begin{matrix}
cos\theta_y & 0 & sin\theta_y & 0\\
0 & 1 & 0 & 0\\
-sin\theta_y & 0 & cos\theta_y & 0\\
0 & 0 & 0 & 1
\end{matrix}
\right]
\left[
\begin{matrix}
1 & 0 & 0 & 0\\
0 & cos\theta_z & -sin\theta_z & 0\\
0 & sin\theta_z & cos\theta_z & 0\\
0 & 0 & 0 & 1
\end{matrix}
\right]
$$
이다.

-추후작성-