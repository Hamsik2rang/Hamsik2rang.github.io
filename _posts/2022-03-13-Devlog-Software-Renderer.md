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

**FEATURE**

* Vertex Buffer만을 이용해 드로우하는 구조에서 Index Buffer를 이용한 드로우 방식을 구현
* Input 데이터를 처리하는 별도의 객체를 설계 및 구현

## 1. ERROR

#### 1.1 임의의 물체가 카메라에 너무 가까이 있을 시 드로우하는 과정에서 버벅임이 발생한다.

우선 물체가 카메라의 위치에 가까워질수록 드로우 부하가 발생하는 부분은 발생을 확인하는 순간에도

> '별도의 Clipping이 없어 Viewport밖의 물체까지 그려내느라 많은 비용이 소모되는 것이 아닐까?'

라는 생각을 했다.

<br/>

Viewport밖의 물체를 그려내는 연산을 제외하기 위해 Rasterizer 단계에서 Clip Space의 범위($-1<=x<=1, -1<=y<=1, 0<=z<=1$)를 넘어서는 정점들을 모두 걸러내는 간단한 Culling 코드를 작성하였다.

![]({{site.baseurl}}/assets/img/posts/2022-03-13/Devlog-Software-Renderer/img01.jpg)

그 결과 상당한 효율 향상이 있었다. 이를 통해 추후 Clipping과 Culling 알고리즘들을 활용해 부하를 해결할 수 있다는 확신을 얻었다.

<br/>

그리고 이 외에도 단순히 화면 밖의 정점까지 그린다는 것만으로 이렇게 큰 부하가 걸리는 것을 통해 추가적인 개선 요소를 파악할 수 있었는데, 현재 프로젝트에서 삼각형을 그려내는 코드는 삼각형의 세 정점을 기준으로 Fragment Box를 구해낸 다음, Box의 안에 존재하는 모든 Fragment들에 삼각형의 내/외부 판별을 하기 때문이라는 것을 예측할 수 있었다.

따라서 기존의 방법 대신 Scan Conversion을 이용한 프래그먼트 처리를 구현하는 것이 좋겠다는 [계획](https://github.com/Hamsik2rang/Software-Renderer/issues/7)을 세웠다.

<br/>

#### 1.2 카메라 회전이 원하는 대로 정확히 이루어지지 않는다. 시점이 조금씩 멋대로 이동하거나 갑자기 화면이 튀는 현상이 발생한다.

<br/>

우선 가장 먼저 카메라 회전을 오일러각 회전을 통해 구현했다.  

<br/>

<center>$R = R_zR_yR_x$</center>

<br/>

이므로

<br/>  

<center>$$R = \left[
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
\right]$$</center>

이다.

마우스의 움직임이 입력될 때 마다 x/y좌표의 변화량을 누적한 후, 렌더 루프가 수행될 때 누적된 변화량만큼 pitch/yaw 회전각을 부여해 카메라의 $AT$을 $EYE$를 기준으로 회전시킨다. 이후 $Normalize(EYE - AT)$을 카메라의 $z$축 정규기저로 지정한 후 직교화 과정을 거쳐 카메라의 정규직교기저를 수정한다.

<br/>

식으로 표현하면 다음과 같다.

<center>$n = \frac{\overrightarrow{EYE}-\overrightarrow{AT}}{|\overrightarrow{EYE} - \overrightarrow{AT}|}$</center>

<center>$u = \frac{\overrightarrow{UP}\times\overrightarrow{u}}{|\overrightarrow{UP}\times\overrightarrow{u}|}$</center>

<center>$v = \overrightarrow{n}\times\overrightarrow{u}$</center>

<br/>

이후 테스트를 진행하는데, 어느 정도 회전하는 순간 카메라 시점이 갑자기 튀어버리는 현상이 발생했다. 원인을 이해를 못해 애를 많이 먹었는데, 카메라의 기저 좌표 로그를 찍어본 결과 어느 이상 카메라를 회전하면 기저 성분들이 NAN으로 바뀌어버리는 것을 확인할 수 있었다.

<br/>

확인 결과 벡터의 정규화 함수에서 다음과 같은 치명적인 실수(확인하는 순간 너무 부끄러웠다..ㅠㅠ)를 한 걸 확인해서 수정하였고, 이후 카메라 시점이 갑자기 튀어버리는 현상은 사라졌다.

{%highlight cpp%}

template <typename T>
Vector3D<T> Normalize() 
{
	x /= norm();
	y /= norm();	// 이 시점에서 x의 값이 바뀌었기 때문에 새로운 norm()을 호출하면 정규성이 보장되지 않음.
	z /= norm();
	return* this;
}

{%endhighlight%}

대신 이를 통해 배운 것들도 많았는데, 처음에 원인을 제대로 인식하지 못해 뻘짓을 하면서 웹에서 정보를 찾아다니다 보니 다양한 지식을 얻을 수 있었다.

<br/>

카메라 뿐만 아니라 반복된 행렬 연산 과정에서 부동소수 오차 누적에 의한 오류*-이를 drift라 부른다-*에 대한 지식이나, 많은 연산을 필요로 하는 대신 방향 벡터에 적은 양의 삼각함수 계산을 통해 새 기저를 구해내는 방법 등이 그것이다.

특히 두 번째, 행렬 대신 삼각함수 연산만을 통해 기저를 회전시키는 방법이 현재 방법보다 더 적은 연산량을 필요로 하는 것을 파악하고 카메라 회전 구현 방식을 해당 방식으로 수정하였다.

<br/>

현재 카메라는 문제 없이 이동/회전이 가능하며, 추후 쿼터니언을 이용해 회전하는 방법이나 유니티 엔전의 Scene view처럼 우클릭 한 상태에서만 카메라가 이동/회전하도록, 그리고 Raycast를 이용해 Mouse Picking등이 가능하도록 기능을 추가할 예정이다.

## 2. WARNING

#### 2.1 특정 헤더 파일을 여러 곳에서 선언 시 `#pragma once`전처리문을 명시하였음에도 중복 선언 이슈가 발생한다.

이 부분은 매우 간단히 해결하였는데, 해당 헤더 파일이 `*.hpp`파일이었고, 프로젝트에서 `*.hpp`는 `*.h`와 달리 헤더에 구현(함수 정의)이 존재하는 파일들이다.

문제가 생기는 `*.hpp`에서 정의된 일반 함수가 inline으로 선언되어 있지 않아 둘 이상의 다른 파일에서 동시에 헤더 선언 시 중복 정의(처음엔 이슈를 명확히 정의하지 못해 중복 '선언' 문제인 줄 알았다)가 일어나는 것이었다.

<br/>

헤더에 클래스가 정의되어 있고 클래스 내부에 멤버 함수까지 정의한 경우 자동으로 `inline`선언이 된다는 것은 알고 있었는데, 그 역으로 **헤더에 함수가 정의되는 경우 해당 함수를 반드시 `inline`으로 명시하거나 `templatize`해야 한다**는 것을 이번에야 알았다.  

## 3. FEATURE

#### 3.1 Vertex Buffer만을 이용해 드로우하는 구조에서 Index Buffer를 이용한 드로우 방식을 구현

지금까지는 일단 화면에 무언가를 띄우고, 카메라가 제대로 작동하는 데에 주력하느라 간단한 2차원 Plane이나 3차원 Cube를 띄울 수만 있으면 되었기 때문에 별도로 Index Buffer를 구현하지 않았다. 

<br/>

그러나 본 프로젝트에선 3D 모델을 위해 `wavefront obj`포맷 파일을 이용할 것이므로 Index Buffer(폴리곤 정보)를 이용한 드로우가 가능해야 했다.

다행히 이를 구현하는 것은 너무 쉬웠다. 마지막 드로우 시에 Vertex Buffer의 정점을 순서대로 3개씩 꺼내지 않고 Index Buffer의 정보를 3개씩 꺼내 해당 인덱스의 정점을 그리면 되기 때문이다.

#### 3.2. Input 데이터를 처리하는 별도의 객체를 설계 및 구현

맨 처음엔 InputManager 정적 클래스를 구현해 처리하려 했지만, 해당 클래스가 모든 데이터와 함수를 static으로 관리하는 것이 캡슐화를 깨뜨리는 것으로 판단되어 Singleton 패턴을 이용한 일반 클래스로 설계하였다.

InputManager를 구현하면서, 이와 같이 점점 많은 클래스들이 스크린 정보(스크린의 시작 좌표, 크기 등)를 필요로 하게 되어서 스크린과 관련된 정보를 관리하는 객체를 설계해야 할 것 같다고 느꼈다.

## 4. 결론

뭔가 조금씩 진행이 되는 것 같으면서도 갈 길이 멀다. 내일부턴 Buffer들을 독립 구조로 재설계한 후 텍스처링, 주사변환 등을 구현해 볼 생각이다.