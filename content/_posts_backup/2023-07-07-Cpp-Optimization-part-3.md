---
layout: post
title: C++ 최적화 - part 3
subtitle: ~C++ 최적화~를 읽고 정리한 글입니다.
tags: [Cpp]
author: Im Yongsik
comments: True
use_math: True
---

# 0. 정리

* 성능은 반드시 측정해야 한다.

* 테스트 가능한 예측을 만들고 "적어둔다"

* 테스트 과정에서 발생하는 코드의 변경 사항을 기록한다.

* 10/90법칙(전체 코드의 10%가 전체 실행 시간의 90%를 소비한다는 법칙)을 기억하라

* 명령문이 메모리를 몇 번 읽고 쓰는지를 셈으로써 문장의 비용을 추정할 수 있다.

* Windows의 `clock()` 함수는 신뢰할 수 있는 1밀리초 틱을 제공하며, Windows 8 이상에서는 `GetSystemTimePreciseAsFileTime()`함수가 1마이크로초 미만 틱을 제공한다.


