---
layout: post
title: Linear Space와 Gamma Space
subtitle: Gamma Correction에 대한 정확한 이해를 해보자
tags: [Computer Graphicss]
author: Im Yongsik
comments: True
use_math: True
---

### 0. 서론

회사에서 업무로 엔진에 다양한 포스트 프로세싱들을 구현하게 되었는데, 지금까지 포스트 프로세싱은 커녕 감마 보정에 대한 이해마저 제대로 하고 있지 못했다. 따라서 업무를 하며 알게된 개념들을 하나하나 정리해보고자 한다.

## 1. Gamma Correction

**Gamma Encoding(감마 부호화)**이란 임의의 값(γ, Gamma)을 이용해 색을 보정한 것을 말한다. 이와 반대로 Gamma Encoding된 색상을 원래 정확한 색상으로 돌려놓는 것을 **Gamma Decoding(감마 복호화)**이라 한다.

일반적으로 디지털 이미지들은 모두 저장 장치에 저장될 때 Gamma Encoding되어 저장된다. 

왜냐하면 모든 디지털 이미지들은 결국 모니터와 같은 장치를 통해 사람의 눈에 보여지게 되는데, 이때 모니터는 최대한 사람의 눈에 익숙한 색상을 보여주기 위해 Gamma Decoding을 수행한 화면을 뿌려 주기 때문이다. **베버의 법칙(Weber's law)**에 의하면 사람의 눈은 어두움에 훨씬 민감하기 때문에, 색의 강도를 선형적으로 늘릴 경우 이를 비선형적으로 인식하게 된다. 즉, 다양한 이미지 처리 프로그램들은 결국 모니터에 해당 이미지가 보여질 때를 감안해서 의도적으로 Encoding을 통해 더 강조된 색상을 저장하게 하는 것이다.

<p align="center">
    <img src= "{{site.baseurl}}/assets/img/posts/2023-05-07/Linear-Space-And-Gamma-Correction/img00.png">
    <br>
    unity 공식 홈페이지에서 참조
</p>

이러한 Gamma Encoding/Decoding 작업을 수행하는 것을 **Gamma Correction(감마 보정)**이라 하며, Gamma Correction을 통해 변한 색상들이 놓이게 된 색공간을 **Gamma Space**라 한다.

## 2. Linear Space

문제는 지금부터 발생한다. 게임에서 어떠한 이미지를 텍스처로 읽어들여 사용한다고 가정하자. 이 이미지는 분명 앞서 말한 원리에 의해 Gamma Encoding되어 있을 것이고, 그 이미지가 나타내야 할 정확한 색상값을 가지고 있지 않을 것이다. 그러나 텍스처는 쉐이더에서 여러 수학적 처리를 통해 색상이 변경될 수 있는데, 잘못된 값을 이용해 값 계산을 하게 되면 원하는 결과를 얻지 못할 수 있다. 따라서 이러한 Encoding된 이미지를 실제 선형 색상으로 변경해 주어야 하는데, 이를 위해서 이미지를 처리하기 전 임시로 Gamma Decoding을 하고 계산을 수행한 후, 그 결과를 다시 Encoding하는 방법을 이용한다. Encoding되어 있던 색상을 강제로 Decoding하여 이미지가 원래의 정확한 선형 색상에 놓였을 때, 이를 우리는 이미지가 **Linear Space**에 존재한다고 말할 수 있다. 즉, 요약하면 다음과 같다.

* 이미지들은 기본적으로 Gamma Encoding되어 저장됨. 
* 그 이유는 모니터가 사람의 시각적 특성을 고려하여 이미지들을 Gamma Decoding하여 보여주기 때문.
* 그러나 이미지의 색상을 조작하거나 계산을 해야 하는 경우(대표적인 예가 쉐이더) 감마 보정된 색상은 정확한 색상이 아님(비선형 색상임).
* 따라서 색상 계산 전 이를 선형 색상으로 변경하는 작업을 수행하게 되고, 선형 색상이 놓이는 Linear Space는 인간의 눈으로 보기엔 뭔가 부자연스럽지만 수학적으로 정확한 계산이 가능해 짐.

## 3. 실제 구현

일반적으로 감마 보정은 감마값을 어떻게 설정하느냐에 따라 그 결과가 바뀐다. 다만 대표적으로 쓰이는 감마값은 2.2인데, sRGB가 바로 γ=2.2를 통해 부호화된 표준 색 공간이다.

sRGB상에서 Gamma Encoding은 다음과 같이 수행한다.

```glsl
pow(color, 1.0f / 2.2f);
```

이를 통해 색상이 다음과 같이 변화한다.

<p align="center">
    <img src= "{{site.baseurl}}/assets/img/posts/2023-05-07/Linear-Space-And-Gamma-Correction/img01.png">
    <br>
</p>

그래프를 보면, 어두운 색상의 할당 대역폭이 줄어든 모습을 볼 수 있다.

반대로 Gamma Decoding은 다음과 같이 수행한다.

```glsl
pow(color, 2.2f);
```

<p align="center">
    <img src= "{{site.baseurl}}/assets/img/posts/2023-05-07/Linear-Space-And-Gamma-Correction/img02.png">
    <br>
</p>

Gamma Encoding과 반대로 어두운 색상의 할당 대역폭이 늘어났다.

따라서 우리는 감마 값 혹은 감마 값의 역수를 색상에 지수승 함으로써 다음과 같이 감마 보정을 자유롭게 수행할 수 있다.

<p align="center">
    <img src= "{{site.baseurl}}/assets/img/posts/2023-05-07/Linear-Space-And-Gamma-Correction/img03.png">
    <br>
    참고. 0.45는 1.0 / 2.2의 근사값이다.
</p>
