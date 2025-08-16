---
title: "실시간 렌더링에서의 색상"
author: "Yongsik Im"
authorAvatarPath: "/images/profile.jpg"
date: "2025-07-12"
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

기본적으로 빛은 전자기파(Electromagnetic waves)의 형태이며, 파장(Wavelength)에 따라 다양한 종류로 구분할 수 있다. 
이 때 구분의 기준이 되는 파장의 범위는 감마선과 같은 nm(나노미터)단위 파장부터 ELF([Extremely Low Frequency](Extremely low frequency))과 같은 수만 km에 이르기까지 매우 광범위하며, 이 중에서 인간은 대략 400nm ~ 780nm 정도의 파장으로 구성된 영역의 빛만을 시각적으로 감지할 수 있고, 그렇기에 이 영역 안의 빛들을 가시광선(可視光線)이라 부른다.



### Radiometry

### Photometry

### Colorimetry

## Scene to Screen

### Tone Mapping

### Exposure

### Color Grading