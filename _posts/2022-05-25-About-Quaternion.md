---
layout: post
title: 쿼터니언에 대하여
subtitle: Quaternion(사원수) 알아보기
tags: [Math]
author: Im Yongsik
comments: true
use_math: true
---

## 0. 서론

프로젝트 안에 사용할 수학 라이브러리를 작성하였다. 벡터, 행렬, 쿼터니언, 보간 등의 기능을 구현했는데, 이 중 쿼터니언에 대해 직접 작성한 코드와 함께 정리해 보자. 

참고로, 본 포스트의 벡터와 행렬은 왼손 좌표계*left-handed*와 열벡터*row-major*를 기준으로 작성되었다.

## 1. 회전을 구현하는 방법

3(혹은 그 이하)차원 공간에서 회전을 표현하는 방법은 다음 방법들이 있다.

* 동차좌표계(*homogeneous coordinate*)를 이용한 회전 행렬
* 오일러 각(*Euler angle*)
* 축-각 회전(*Angle-Axis rotation*)
* 쿼터니언(*Quaternion*)

#### 1.1. 동차좌표계를 이용한 회전 행렬

어떠한 물체를 회전시키고자 할 때 공간상의 표준기저 $i(1,0,0), j(0,1,0), k(0,0,1)$각각을 회전축으로 하여 각각의 축에 대한 회전각을 알고 있으면 회전 행렬 3개$(R_x, R_y, R_z)$를 구성해 해당 행렬들을 회전 순서에 맞게 곱하면 회전 행렬 $R$이 구성된다.

$$R = R_xR_yR_z$$

$$R_x = \begin{bmatrix}

1 & 0 & 0 & 0 \\

0 & cos(\theta) & sin(\theta) & 0 \\

0 & -sin(\theta) & cos(\theta) & 0 \\

0 & 0 & 0 & 1

\end{bmatrix} $$

$$R_y = \begin{bmatrix}

cos(\phi) & 0 & -sin(\phi) & 0 \\

0 & 1 & 0 & 0 \\

sin(\phi) & 0 & cos(\phi) & 0 \\

0 & 0 & 0 & 1

\end{bmatrix} $$

$$R_z = \begin{bmatrix}

cos(\psi) & sin(\psi) & 0 & 0 \\

-sin(\psi) & cos(\psi) & 0 & 0 \\

0 & 0 & 1 & 0 \\

0 & 0 & 0 & 1

\end{bmatrix} $$

$$R = \begin{bmatrix}

1 & 0 & 0 & 0 \\

0 & cos(\theta) & sin(\theta) & 0 \\

0 & -sin(\theta) & cos(\theta) & 0 \\

0 & 0 & 0 & 1

\end{bmatrix}

\begin{bmatrix}

cos(\phi) & 0 & -sin(\phi) & 0 \\

0 & 1 & 0 & 0 \\

sin(\phi) & 0 & cos(\phi) & 0 \\

0 & 0 & 0 & 1

\end{bmatrix}

 \begin{bmatrix}

cos(\psi) & sin(\psi) & 0 & 0 \\

-sin(\psi) & cos(\psi) & 0 & 0 \\

0 & 0 & 1 & 0 \\

0 & 0 & 0 & 1

\end{bmatrix} $$

#### 1.2. 오일러 각(Euler-angle)

회전 행렬은 가장 간단한 방법이지만 다음과 같은 단점이 있다.

* 회전마다 행렬곱을 수행해야 한다.
* 회전 행렬이 주어졌을 때 각 축별로 얼마나 회전하는지를 직관적으로 파악하기 어렵다.
* 3개 축에 대한 회전(3자유도 회전)을 정의하기 위해 총 9개의 값을 필요로 한다.

오일러 각은 회전 행렬과 달리 축별 회전을 더욱 직관적으로 표시하며, 연산량이 적다는 장점이 있다.

오일러 각을 통해 회전을 표현할 때 각 축에 대한 회전을 **Pitch, Yaw, Roll**로 표현하며, 다음과 같다.

<p align= "center">
    <img src = "{{site.baseurl}}/assets/img/posts/2022-05-25/About-Quaternion/img01.png">
</p>

주의해야 할 점은 **Pitch, Yaw, Roll이 x,y,z축과 항상 대응하는 것이 아니라는 점이다.** 좌표축을 어떻게 설정하느냐에 따라 x축 회전이 Pitch로 표현될 수도 있고, Roll로 표현될 수도 있다.

결론적으로, 오일러 각을 이용해 회전을 표현한다면 3개의 값(Pitch, Yaw, Roll)만으로 3차원 회전을 정의할 수 있게 된다.

$$ R_{euler} = (\theta_{pitch}, \theta_{yaw}, \theta_{roll}) $$

이렇게 놓고 보면 회전 행렬에 비해 오일러각은 장점만을 가진 것 같지만, 오일러 각을 이용한 회전 역시 문제가 있다.

* '회전각'만을 가지고 있는 경우 변환마다 해당 각에 대한 $sin$, $cos$값을 계산해야 한다. 

  그렇다고 각과 함께 해당 각에 대한 $sin$, $cos$값까지 함께 저장할 경우 결국 9개의 값을 가지고 있어야 하므로 회전 행렬과 다를 바가 없어진다.

* 정해진 회전 순서가 없다.

* 짐벌 락*gimbal lock*이 발생할 수 있다.

* 한 번 회전하는 순간 회전축이 바뀌어 있다. 즉, $R(30, 20, 60)$을 한 번 수행하는 것과 $R(15, 10, 30)$을 연속으로 두 번 회전하는 것은 다른 회전 결과를 가진다.

  물론 한 축에 대해서만 연속적인 회전이 일어나는 경우라면 아무 문제가 없다. 다시 말해 $R(0, 60, 0)$과 $R(0, 20, 0)$회전을 연속으로 세 번 진행하는 것은 같은 결과를 가진다.

* 회전 행렬에도 적용되는 문제이지만, 오일러 각 역시 표준기저를 기준으로 하는 회전이 아닌 **임의의 축**을 기준으로 하는 회전을 표현하는 데 무리가 있다.

* 보간*Interpolation*이 어렵다

##### 1.2.1 짐벌 락(Gimbal Lock)

짐벌 락은 세 축에 대한 회전을 한 번에 진행하지 않고 각각 나누어 진행할 때 발생할 수 있는 문제이다. 즉, 반드시 오일러 각을 통한 회전에서만 일어나는 문제는 아니다.

유튜브나 구글에 짐벌 락을 검색하면 직관적인 동영상 자료들이 많은데, 글로 간단히 설명하자면 다음과 같다.

$x$ -> $y$ ->$z$축 순으로 회전하기로 마음먹었을 때, x축을 먼저 회전한 후 y축을 **$90^{\circ}$** 회전시켜 보자.

그 순간 z축은 y축 회전 전의 x축과 동일해지고, 이 상태에서 z축을 회전하게 되면 맨 처음 있었던 x축 회전이 무의미해진다. 즉,

$$R_{euler}(\theta_{pitch}, \theta_{yaw}, \theta_{roll})$$

는 아래와 같은 회전이 된다.

$$R_{euler}(\theta_{pitch} - \theta_{roll}, \theta_{yaw}, 0)$$

x축에 대한 회전이 원하는 만큼 이루어지지 않을 뿐더러 z축에 대한 회전은 완전히 무시된다.

반대로 생각하면, 짐벌락을 방지하기 위해선 회전 순서가 어떻게 되는지와 관계 없이 두 번째로 수행되는 회전축이 $90^{circ}$, 혹은 $270^{circ}$만큼 회전하지 않으면 된다.

#### 1.3. 축-각 회전(Angle-Axis Rotation)

오일러 각에 대한 대표적인 문제점 중 하나인 짐벌락, 그리고 짐벌락을 해소하는 방법에 대해서 알아보았지만, 오일러 각이 임의의 축을 기준으로 한 회전에 적합하지 않다는 점은 우리에게 표준기저와 평행하지 않은 임의의 축을 기준으로 회전하는 방식의 필요성을 강조시킨다.

<br/>

**오일러의 회전 정리(Euler's rotation theorem)**에 따르면, 임의의 한 회전 또는 여러 회전의 조합은 결국 한 회전축과 그 회전축을 이용한 한 번의 회전으로 표현될 수 있다.

임의의 벡터 $v$가 앞서 살펴본 회전 행렬 혹은 오일러 각 회전등을 통해 여러 번의 회전을 거쳐 $v'$가 되었다면, $v \times v'$를 통해  $v$와 $v'$에 동시에 수직인 벡터 $a$을 찾을 수 있는데, 이때 $a$가 $v$를 $v'$로 회전시키는 회전축이 될 것이다. 또한 $v \cdot v'$를 통해 회전각 $\theta$에 대한 코사인 값 $cos(\theta)$를 찾을 수도 있다(물론 $arccos$을 이용해 완전한 회전각 $\theta$도 찾을 수 있다).

결국 임의의 회전축 $a$와 회전각 $\theta$를 이용해 회전을 표현할 수 있다.

$$R_{angle-axis} = (\hat{a}, \theta) = (\hat{a}_x, \hat{a}_y, \hat{a}_z, \theta)$$

축-각 회전은 회전하고자 하는 벡터를 회전축에 수직인 성분과 평행한 성분으로 분해한 후, 수직인 성분을 회전축을 기준으로 회전시킨 후 다시 결합하면 된다.

<p align = "center">
    <img src = "{{site.baseurl}}/assets/img/posts/2022-05-25/About-Quaternion/img03.png">
</p>

별도의 유도는 생략하고, 그 결과를 수식으로 표현하면 다음과 같다.

$$v' = cos(\theta) \cdot v + (1 - cos(\theta)) \cdot (v \cdot \hat{a}) \cdot \hat{a} + sin(\theta) \cdot (\hat{a} \times v)$$

축-각 회전은 단 한번의 회전을 수행하기 때문에 짐벌락으로부터 자유롭다.

## 2. 쿼터니언이란?

윌리엄 해밀턴(*William R. Hamilton*)에 의해 탄생한 쿼터니언(*Quaternion*, *사원수*)은 그 자체로는 회전을 위한 것이 아니라 실수 체계의 확장으로 복소수($a+bi$)가 존재하는 것처럼 하나의 수 체계로서 고안되었다. 그러나 이후 쿼터니언의 연산 중 일부가 회전을 표현하는 데 특화됨을 알게 되면서, 컴퓨터 그래픽스에서 회전을 표현하는 가장 일반적인 방법들 중 하나로 자리잡게 되었다. 

쿼터니언 체계는 다음과 같은 공리를 기반으로 한다.

$$ \textbf{q}  = ai + bj + ck + d \:일 \:때$$

$$ i^2 = j^2 = k^2 = -1 $$

$$ ij = k, jk = i, ki = j $$

$$ ji = -k, kj = -i, ik = -j $$

또한, 쿼터니언의 허수부를 하나의 벡터로 표현하면 다음과 같이 표현하기도 한다.

$$\textbf{q} = (\textbf{v}, w) $$

## 3. 쿼터니언 연산

쿼터니언은 다음과 같은 연산을 수행할 수 있다

$$ \textbf{p} = a_pi+b_pj+c_pk+d_p, \textbf{q} = a_qi+b_qj+c_qk+d_q\:일 \:때$$

$$ \textbf{p} + \textbf{q} = (v_p+v_q, w_p+w_q) = (a_p+a_q)i + (b_p+b_q)j + (c_p+c_q)k + d_p+d_q$$

$$ \textbf{p} - \textbf{q} = (v_p-v_q, w_p-w_q) = (a_p-a_q)i + (b_pb_q)k + (c_p-c_q)k + (d_p-d_q) $$

$$ |\textbf{p}| = \sqrt{v_p \cdot v_p + w_p^2} = \sqrt{a_p^2 + b_p^2 + c_p^2 + d_p^2} $$

$$ Conjugate(\textbf{p}) = \textbf{p*} = (-v_p , w_p) $$

이외에도 쿼터니언의 곱셈을 정의할 수 있는데, 쿼터니언의 곱셈은 특별히 유도를 해 볼 필요가 있다.

$$  \textbf{p} * \textbf{q} = (v_{px}i + v_{py}j + v_{pz}k + w_p)(v_{qx}i + v_{qy}j + v_{qz}k + w_q)$$

$$=v_{px} v_{qx} i^2+v_{px} v_{qy} ij+v_{px} v_{qz} ik+v_{px} w_q i$$

$$+v_{py} v_{qx} ji+v_{py} v_{qy} j^2+v_{py} v_{qz} jk+v_{py} w_q j$$

$$+v_{pz} v_{qx} ki+v_{pz} v_{qy} kj+v_{pz} v_{qz} k^2+v_{pz} w_q k$$

$$+w_p v_{qx} i+w_p v_{qy} j+w_p v_{qz} k+w_p w_q$$

위 식에서 쿼터니언의 공리를 이용해 한번 기호들을 정리해주면 아래와 같아진다.

$$=-v_{px} v_{qx} + v_{px} v_{qy}k - v_{px} v_{qz}j + v_{px} w_qi $$

$$-v_{py} v_{qx}k-v_{py} v_{qy}+v_{py} v_{qz} i+v_{py} w_q j$$

$$+v_{pz} v_{qx} j-v_{pz} v_{qy} i-v_{pz} v_{qz}+v_{pz} w_q k$$

$$+w_p v_{qx} i+w_p v_{qy} j+w_p v_{qz} k+w_p w_q$$

이제 이 식을 예쁘게 묶어 주면 다음과 같다.

$$=(v_{py} v_{qz}-v_{pz} v_{qy} )i+(v_{pz} v_{qx}-v_{px} v_{qz} )j+(v_{px} v_{qy}-v_{py} v_{qx} )k$$

$$ + (v_{px} i+v_{py} j+v_{pz} k) w_q+(v_{qx} i+v_{qy} j+v_{qz} k) w_p$$

$$ + w_p w_q-(v_{px} v_{qx}+v_{py} v_{qy}+v_{pz} v_{qz})$$

유심히 식을 살펴보면 벡터에서 익숙하게 해왔던 연산들로 표현되고 있음을 알 수 있다.

$$ pq=(v_p,w_p )(v_q,w_q )=(v_p×v_q+w_p v_q+w_q v_p,w_p w_q-v_p∙v_q) $$

이외에도 쿼터니언의 곱에 대한 항등원과 역원은 다음과 같다

$$ Identity = { ( \textbf{0}, 1)} $$

$$ Inverse = \frac{\textbf{p*}}{|\textbf{p}|}$$

## 4. 쿼터니언과 회전

앞서 중요하게 다룬 쿼터니언의 곱은 그 유도를 통해 벡터 연산으로 표현할 수 있음을 보았다.

더 중요한 점은, 쿼터니언의 곱셈 결과가 앞서 위에서 살펴보았던 **축-각 회전**과 완전히 동일하다는 것이다.

따라서 쿼터니언으로 회전할 벡터와 회전각을 표현하고 그러한 두 쿼터니언을 곱하면 우리는 벡터의 회전 연산을 하는 것과 동일한 결과를 얻게 된다.

쿼터니언으로 회전할 벡터를 표현하려면 다음과 같이 표현한다.

$$ \textbf{q} = (\textbf{v}, 0) $$

즉, 단순히 스칼라 부분을 0으로 표현하면 벡터가 되는 것이다.

회전각은 다음과 같이 표현한다.

$$ \textbf{r} = (sin\theta \cdot \textbf{u}, cos\theta) $$

이는 단위 쿼터니언(4차원 상에서 길이가 1인 쿼터니언)을 표현하는 방법이자 동시에 회전각과 축을 표현하는 방법이다.

위 표현에서 $\textbf{u}$는 회전축의 단위벡터를, $\theta$는 회전각의 **절반**을 표현한다.

그리고 마지막으로 중요한 점이 실제 곱셉을 하는 방식인데, 회전할 벡터는 스칼라가 0이었지만 단순히 회전 쿼터니언을 벡터에 곱하면 스칼라가 0이 아니게 된다(앞서 본 결과도 그렇고, 실제 연산을 해 보면 금방 이해할 것이다). 

따라서 회전을 위한 쿼터니언곱은 회전할 벡터의 왼쪽에 회전각의 **절반**을 표현하는 쿼터니언과, 오른쪽에 그 켤레를 함께 곱하여 수행한다.

$$ \textbf{v}' = \hat{\textbf{q}} \textbf{v} \hat{\textbf{q}}* $$

손으로 연산을 한 번 해보면 정확히 그 결과의 스칼라가 0이고, 벡터가 완벽히 회전된 상태임을 확인할 수 있다.

<br>

중요하니까 다시 한번 강조하자면, 만약 임의의 벡터$\textbf{v}$를 $\pi/4$만큼 회전하고 싶다면 회전 쿼터니언 $\textbf{q}$은 다음과 같이 표현된다.

$$\textbf{q} = (sin(\pi/8) \cdot \textbf{u}, cos(\pi/8))$$

$$ \textbf{v}' = \hat{\textbf{q}} \textbf{v} \hat{\textbf{q}}* $$

## 5. 구면 선형 보간(Slerp)

앞서 오일러 각을 통한 회전은 보간이 어렵다는 점이 있었다. 그러나 쿼터니언은 간단히 보간을 수행할 수 있는데, 보간 시의 결과가 마치 4차원 초구면(hyper-sphere)상의 단위구 표면에서 점이 동일한 각속도로 부드럽게 움직이듯 보간되기 때문에 **구면 선형 보간*Spherical Linear Interpolation*** 이라 부른다.

이 때 쿼터니언을 보간하는 것은 회전을 보간하는 것과 동일하다. 만약 회전을 일반적인 선형 보간(Lerp)을 통해 보간할 경우 각속도가 일정하지 않기 때문에 회전이 동일한 속도로 이루어지지 않는다. 그러나 구면 선형 보간을 이용할 경우 그러한 문제가 해소된다.

<p align = "center">
    <img src = "{{site.baseurl}}/assets/img/posts/2022-05-25/About-Quaternion/img04.png">
</p>

구면 선형 보간은 유도를 생략하고 아래와 같은 식으로 표현할 수 있다.

$$\textbf{q}(t) = \frac{sin\theta_{1-t}}{s} \hat{q}_0  + \frac{sin\theta_t}{s} \hat{q}_1  $$
