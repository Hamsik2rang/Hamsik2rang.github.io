---
title: "실시간 렌더링에서의 색상"
author: "Yongsik Im"
authorAvatarPath: "/images/profile.jpg"
date: "2025-08-16"
summary: "빛과 색의 기본적인 지식들"
description: ""
toc: true
readTime: true
autonumber: true
math: true
tags: ["Graphics", "GPU", "AI"]
showTags: false
hideBackToTop: false
# draft: true
---
## Light Quantities
PBR(Physically-Based Rendering)등의 렌더링 기법을 이용해 현실적인 장면을 렌더링하려면 현실에 적용되는 물리적인 법칙들을 최대한 비슷하게 묘사해야 한다. 그 중에는 당연히 Shading을 위한 빛의 법칙들도 포함된다.  
빛은 입자(Particle)와 파동(Wave)의 성질을 모두 가지고 있는데, 그래픽스에서는 빛을 조명과 재질의 상호작용으로서 사용하기 때문에 각 성질은 아래의 용도로 사용될 수 있을 것이다.
* 입자: 굴절(Refresciton), 충돌(Collision), 반사(Reflection), 산란(Scattering), 차폐(Occlusion), 흡수(Absorption)와 음영 등
* 파동: 색상(Color), 회절(Diffraction), 간섭(Interference) 등

완벽하진 않지만 간단하게 요약하면, 파동적 성질은 픽셀이 "어떤" 구성의 값을, 입자적 성질은 "어느 정도로" 가질지 결졍한다고 여길 수 있다.
본 포스트에선 색에 대한 이야기를 주로 할 것이기 때문에, 당연하게도 파동으로서의 빛을 주로 살펴볼 것이다.  

기본적으로 빛은 전자기파(Electromagnetic waves)이며, 파장(Wavelength)에 따라 다양한 종류로 구분할 수 있다. 
이 때 구분의 기준이 되는 파장의 범위는 감마선과 같은 nm(나노미터)단위 파장부터 ELF([Extremely Low Frequency](https://en.wikipedia.org/wiki/Extremely_low_frequency))과 같은 수만 km에 이르기까지 매우 광범위하며, 이 중에서 인간은 대략 400nm ~ 780nm 정도의 파장으로 구성된 영역의 빛만을 시각적으로 감지할 수 있고, 그렇기에 이 영역 안의 빛들을 가시광선(可視光線)이라 부른다.

렌더링 장면을 사용자에게 보여주는 것은 디스플레이 장치의 각 픽셀들이 어떤 색상의 빛을 어느 강도로 표현할지 나타내는 것이므로, 이를 위해  

1. 빛의 양을 측정/표현하는 방법
2. 연속 데이터인 빛의 파장을 이산 데이터인 디지털 정보로 변환하는 방법

에 대해 알아야 하며, 첫 번째인 빛의 양을 측정하는 것 부터 시작하고자 한다.

### Radiometry
빛의 양은 '실제 물리적인 양'과 '인간이 지각하는 양'이라는 두 기준으로 측정할 수 있을 것이다.
물리적인 양을 측정하는 단위인 Radiometry는 한국어로 '방사 측정', 혹은 '방사량' 이라고 표현할 수 있다.
방사량을 측정하는 단위에 대해 먼저 제시한 후 각 단위의 의미를 살펴보자.
| Name |  Symbol | Units | Description |
|:-----|:-------:|:-----:|-------------|
|방사속(*radiant flux*)|$$\Phi$$|W (watt)|단위 시간당 방사되는 빛의 양|
|방사 조도(*irradiance*)|E|$$W/m^{2}$$|단위 면적당 방사속(단위 면적에 들어오는 빛의 양)|
|방사 강도(*radiant intensity*)|I|$$W/sr$$|단위 입체각(Steradian)당 방사속(단위 입체각으로 들어오는 빛의 양)|
|방사 휘도(*radiance*)|L|$$W/(m^{2}sr)$$|단위 면적 및 단위 입체각당 방사속|

보통 방사속, 방사 조도 및 방사 강도에 대해서는 직관적으로 쉽게 이해할 수 있다.
그런데 방사 휘도(radiance)는 직관적으로 이해하기 어려워하는 경우가 많은데, 조금 더 쉽게(?) 표현해 보자면 단위 면적 및 단위 입체각당 방사속은 결국 수식으로 아래처럼 표현될 수 있으며,
{{<rawhtml>}}
$$\frac{d\Phi}{dAd\omega}, (A=Area)$$
{{</rawhtml>}}

일반적으로 물체에 있어서 '면적'은 표면(Surface)을 의미할 것이므로, 치환해 표현하자면   

'**표면에 입사되는 수많은 광선들 중 단위 입체각으로 입사되는 광선들의 방사속**'  

을 의미하게 된다.

여기서 알 수 있는 점은 아래의 두가지가 있다.

* radiance를 $\pi$만큼 적분하면 irradiance를 구할 수 있다.
* 입체각 $\omega$를 아주 작은 단위(미소 입체각)로 표현해 '단일 광선'의 방사속으로 근사(Approximation)할 수 있다.

이러한 아이디어는 Image-based Lighting이나 Illumination 계산에 사용된다.

### Photometry
빛의 양을 물리적으로 얼마나 정확하게 측정하는지보다도 결국 렌더링된 화면은 인간이 보게 되므로, 인지적 측면에서의 빛의 양을 측정할 수 있어야 한다.
Photometry는 '광도 측정', 혹은 '광량', 으로 표현할 수 있다.
사실 광량은 인간의 눈을 기준으로 한다는 것을 제외하면 방사량과  동일한 기준으로 측정되며, 다음과 같은 단위를 사용한다
| Name |  Symbol | Units | Description |
|:-----|:-------:|:-----:|-------------|
|광도(*luminous flux*)|lm|$lumen$|단위 시간당 광량|
|방사 조도(*illuminance*)|lx|$lux$|단위 면적당 광량(단위 면적에 들어오는 빛의 양)|
|방사 강도(*luminous intensity*)|cd|$candela$|단위 입체각(Steradian)당 광량(단위 입체각으로 들어오는 빛의 양)|
|방사 휘도(*luminance*)|nit|$$cd/m^{2}$$|단위 면적당 방사 강도|


### Colorimetry

## Scene to Screen

### Tone Mapping

### Exposure

### Color Grading