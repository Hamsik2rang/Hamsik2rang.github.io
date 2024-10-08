---
layout: post
title: 2023년 하반기 계획
subtitle: 상반기에 한 일 회고 및 하반기 계획
tags: [Blog]
author: Im Yongsik
comments: True
use_math: True
---

# 0. 서론

벌써 한 해의 절반이 지나갔다. 이번 상반기는 다양한 일들이 있어서 정말 빠르게 시간이 지나갔다는 느낌을 많이 받았다(그리고 나이는 1살 어려졌다 ㅎ). 7월의 첫날을 기점으로 상반기에 한 일들을 회고해 보고 하반기에 할 일들에 대한 대략적인 계획을 잡아보려 한다.

# 1. 상반기에 한 일들

* 크래프톤 정글 1기 참여(수료 아님!)

  작년 10월부터 크래프톤 정글에 참여하면서 다양한 경험을 했다. 뛰어난 동료들과 함께 프로젝트를 진행하는 경험과 전산학 지식에 대한 학습에 매진할 수 있었다. 무엇보다 크래프톤의 장병규 의장님이나 김정한 원장님을 필두로 한 정글의 훌륭한 코치님들로부터 프로그래밍에 임하는 자세나 프로그래머로서의 나의 앞길을 바라보는 새로운 시각을 얻을 수 있었고, 이를 바탕으로...

* 컴투스 입사

  컴투스에 엔진 프로그래머로 입사했다. 휴학생 신분이다 보니 원래 같았으면 이력서 자체를 쓸 생각을 안 가졌겠지만 앞서 말했듯이 정글에서 의장님과 코치님들로부터 많은 의견들을 듣고 그걸 바탕으로 나를 돌아보는 시간을 많이 가지면서 (겁이 없어짐과 동시에)새로운 도전을 해보고 싶다는 생각을 가지게 되었고, 해내었다. 맡는 업무마다 처음 해보는 일들이고 종종 해내지 못할까봐 두려움에 휩싸이기도 하지만, 단순히 학교를 다녔다면 배우지 못할 많은 지식과 경험을 쌓아가는 중이다. 


# 2. 하반기에 할 일들

## 학습

현재 나 스스로 가장 약하다고 생각하는 점들을 꼽자면 다음과 같다.

* PBR & 렌더 테크닉(그래픽스 지식)

  입사 시점에 내 그래픽스 지식은 부끄럽게도 Blinn-Phong에 머물러 있었는데, 회사에서 업무를 진행하면서 PBR이나 PostEffect와 같은 각종 렌더 테크닉에 대한 지식을 머리에 마구잡이로 쑤셔넣었다. 하반기에는 조금 시간을 들여서 무질서하게 흩어진 지식들을 차곡차곡 정리해 볼 생각이다.

* 수학(특히 선형대수)

  앞서 서술한 내용과 동일한 맥락에서, RnD를 위해 각종 렌더 방정식이나 논문들을 보면서 내가 생각보다 더 수학에 약하다는 걸 깨닫게 되었다. 따라서 선형대수나 미적분에 대한 공부가 필요하다.

* 언리얼 엔진 사용법

  대형 상용 엔진들을 RnD하다 보면 지나칠 수 없는 게 언리얼 엔진인데, 부끄럽게도 나는 언리얼 엔진을 제대로 사용해 본 경험이 없다. 적어도 내가 언리얼의 어떤 기능을 파악해 보고 싶을 때 그 기능을 테스트해 보고 코드를 뜯어볼 수 있는 수준의 언리얼 사용법을 갖춰야 한다.

이를 바탕으로 아래 학습들을 진행할 예정이다.

#### 인강

회사에서 제공해주는(정확히는 내가 요청한) 강의들과 홍정모 교수님 그래픽스 강의를 수강할 예정이다.

* 선형대수학개론(인프런)
* 이득우의 언리얼 프로그래밍(인프런) - part 1, 2
* 그래픽스(Honglab) - Part 1,2,3,4

#### 도서

모던 C++에 대한 이해를 높일 필요가 있다(특히 Template Programming). 그리고 수학과 그래픽스 학습을 진행하고, 시간이 남으면 전산학 관련 공부를 더 진행할 예정이다.

* C++ 최적화(커트 건서로스)
* A tour of C++(비야네 스트롭스트룹)
* 모던 C++ 챌린지(마리우스 반실라)
* 디지털 라이팅&렌더링(제레미 번)
* OpenGL ES를 활용한 컴퓨터 그래픽스 입문(한정현)
* 수학 리부트(강중빈)
* 3D 게임 개발을 위한 수학(양영욱)
* CODE(찰스 펫졸드)
* OSTEP

## 프로젝트

하반기에 프로젝트를 진행하면서 경험할 내용은 딱 하나다. **차세대 그래픽스 API**. 따라서 프로젝트는 Vulkan위주로 진행할 예정이다.

#### 신규

MVR(가명)

* Vulkan 기반 미니 렌더링 엔진
* ImGui 혹은 C# UI를 기반으로 할 예정
* Animated FBX 모델 뷰어 수준으로 계획중.
* Vulkan 학습과 3D 그래픽스, 쉐이더 프로그래밍 학습을 위한 목적

Kepler

* 내년 중으로 3D 게임을 하나 뽑아내기 위한 총체적인 준비 작업
* Renderpass System 추가
* Vulkan 지원
* ECS 고도화(정확히는, Scene 고도화)
* Reflection 추가, 이를 기반으로 Editor 고도화

## 기타

개발자로서의 내가 아닌 나 그 자체로서의 계획을 세워보자면 아래와 같은 것들이 있을 것 같다.

* 운동하기

  회사에서도 하루종일 앉아 있기만 하고, 짱짱한 컴투스의 스낵바 덕분에 나날이 살이 찌고 있어서, 이러다간 큰일 날 것 같다... 적어도 주 3~4회는 운동을 해 보려 한다.

* 번역

  연말까지 현재 블로그에 진행중인 *A trip through the Graphics pipeline 2011*에 대한 번역을 완료해 보려 한다. 

* 주변 사람들 많이 만나기

  보고싶은 얼굴들은 많은데, 입사하고 업무에 적응해야 한다는 이유로 보지 못한 사람들이 많다. 살면서 많은 사람들에게 도움을 받았기에, 그 만큼 내가 도와줄 수 있는 사람들이 있다면 도와주고, 꼭 그런 일들이 아니더라고 만나서 시시콜콜한 얘기를 나누고 싶은 사람들이 많아서 최대한 만남을 많이 만들고 싶다(가급적 술은 적게 마시고...)
