---
layout: post
title: GoogleTest를 이용한 테스트 주도 개발
subtitle: 프로젝트에 단위 테스트 적용하기
tags: [Devlog, GoogleTest, TDD, UnitTest]
author: Im Yongsik
comments: true
use_math: true
---

## 0. 서론

게임 엔진 제작 프로젝트를 진행하면서 엔진의 볼륨이 점점 늘어나다 보니 코드를 짜고, 빌드해서 테스트하는 과정이 번거로워지기 시작했다.

뿐만 아니라 5월부터는 프로젝트를 함께 진행할 새 멤버도 추가될 예정인데, 현재 개발이 약간이나마 진행된 상태에서 새로 프로젝트에 들어올 경우 각 기능들의 명세나 설계 의도를 파악하는 데 많은 노력이 필요하다는 점도 고려하여, 이러한 오버헤드를 최소화할 방안이 필요했다.

결국 TDD방식을 통해 이러한 문제들을 해소해 보기로 했고, 단위 테스트 모듈로는 GoogleTest 모듈을 이용하기로 했다.

## 1. TDD

**TDD**란 Test Driven Development의 약자로, 테스트 주도 개발이라는 뜻의 **소프트웨어 개발 방법론**이다.

테스트 주도 개발은 개발 과정에서 기능을 개발하는 과정에서 **선 개발 후 테스트가 아닌** 구현하고자 하는 기능이 정상적으로 동작한다고 가정할 때 통과할 수 있는 **Test Case(혹은 Test Suits, 이하 TC)들을 먼저 작성하고 그 테스트를 통과할 수 있도록 기능을 구현**하는 방법이다.

TDD, 즉 유닛 테스팅을 통한 개발은 대표적으로 다음과 같은 장점이 있다.

* 보다 단순하고 명료한 구현을 유도

  TC를 통과할 수만 있도록 기능을 구현하면 되기 때문에 별도의 가이드라인 없이 기능을 구현할 때 보다 구현 형태가 단순명료해질 수 있다.

* 디버깅 시간의 단축

  기존 개발의 경우 개발 후 빌드 과정에서 에러가 발생할 경우 에러의 원인을 찾기 위해 생각보다 많은 시간을 소모하게 된다. 그러나 TC를 통해 걸러진 문제점은 그 발생 지점을 명확히 알 수 있으므로 디버깅 시간이 단축될 수 있다.
  
* TC가 개발자들 간의 기능 명세로 작용

  개인 프로젝트가 아닌 이상 여러 개발자간의 협업 과정에서 의사 소통은 필수적이며, 최대한 의사 소통을 한다 해도 기능 구현 과정에서 의도가 잘못 전달되어 잘못되거나 불필요한 구현을 할 가능성이 존재한다.

  TDD에서는 TC가 각 개발자들로 하여금 구현해야할 기능에 대한 명세의 역할을 할 수 있기 때문에 어떠한 기능/객체를 바라보는 개발자들의 시선을 동일하게 유도할 수 있다.

반면 TDD는 다음과 같은 단점을 가질 수 있다.

* TC 작성을 위한 추가 시간 소모

  TDD를 도입해서 잘 짜여진 설계를 기대할 수 있고, 디버깅 시간을 감소시킬 수도 있지만 결국 TC를 작성하는 추가 시간이 필요하다는 점은 사실이다.

* 테스트가 정확하지 못할 가능성

  기능을 올바르게 수행하는지 파악하기 위한 TC가 기능의 무결성을 검증할 수 없다면 결국 생각하지 못한 에러를 유발할 수 있다.

## 2. GoogleTest

C++에서 유닛 테스팅을 위한 모듈은 다양하게 존재하나, 진행 중인 프로젝트에는 **GoogleTest** 모듈을 이용하기로 했다.

GoogleTest는 이름에서 알 수 있듯이 Google에서 개발한 C++ Test모듈로, 문서가 잘 정돈되어 제공된다는 점과 Visual Studio 2022의 기본 모듈로 포함되어 있다는 점이 장점이다.  

GoogleTest의 구체적인 사양과 이용 방법을 알고 싶다면 구글에서 직접 제공하는 API문서인 [GoogleTest Primer](http://google.github.io/googletest/primer.html)를 참고하길 바란다.

<br>

GoogleTest 모듈을 기존 프로젝트에 추가하고자 할 경우 먼저 Solution Explorer에서 해당 프로젝트의 솔루션을 우클릭해 새 프로젝트를 추가한다.

![]({{site.baseurl}}/assets/img/posts/2022-04-18/TDD-on-project/img02.jpg)

이후 프로젝트 탬플릿을 선택할 때 검색 창에 `test`를 검색하면 여러 유닛테스트 모듈이 등장하는데, 이 중 `GoogleTest`를 선택한다.

![]({{site.baseurl}}/assets/img/posts/2022-04-18/TDD-on-project/img03.jpg)

그리고 프로젝트 이름을 정한 후 생성을 하려 하면 아래처럼 추가 설정이 나타난다.

![]({{site.baseurl}}/assets/img/posts/2022-04-18/TDD-on-project/img04.jpg)

**Consume Google Test as** 항목은 GoogleTest를 정적/동적 라이브러리의 형태 중 어떠한 형태로 사용할지를 정하는 항목이고, 오른쪽은 C++ 런타임 라이브러리들을 어떠한 방식으로 링크할지 정하는 항목이다.

오른쪽 항목은 별다른 사항이 없다면 추천하는 대로 동적 연결을 선택하고, 왼쪽 항목은 본인의 프로젝트 사양에 맞게 선택하면 된다. 이 글은 **동적 라이브러리(Dynamic Library)** 형태를 선택해 진행하였다.

또, 만약 솔루션 내에 프로젝트가 여러 개 존재할 경우 어떤 프로젝트를 참조(reference)할지 선택하는 창이 나올 수 있다. 만약 그러한 선택 창이 나오지 않았을 경우 유닛 테스트 프로젝트를 우클릭->Add를 선택 후 Reference를 직접 지정해 준다.

![]({{site.baseurl}}/assets/img/posts/2022-04-18/TDD-on-project/img11.jpg)

아무튼, 프로젝트가 생성되면 다음과 같은 항목들이 기본적으로 존재한다.

* pch(precompiled header) 

  unit test에 필요한 헤더가 포함되어 있다(gtest/gtest.h)

* test.cpp(Test 소스) 

  TC들을 작성한다. 꼭 이 파일에 작성할 필요는 없고 별도의 헤더나 소스를 추가해 작성할 수 있다.

* packages.config

  현재 생성된 GoogleTest Package에 대한 기본 속성들(version, encoding, ...)이 지정되어 있다.

![]({{site.baseurl}}/assets/img/posts/2022-04-18/TDD-on-project/img05.jpg)

test.cpp 소스 파일을 보면 위와 같이 아주 간단한 예시 TC가 작성되어 있는데, 예시와 같이 양식을 맞추어 TC를 작성한다.

이 글에서는 아래와 같은 간단한 수학 함수 중 `GetRadian` 함수를 GoogleTest을 통해 테스트해 볼 것이다.

![]({{site.baseurl}}/assets/img/posts/2022-04-18/TDD-on-project/img01.jpg)

`GetRadian`함수는 실수 형태의 각도를 인자로 받은 후 이를 라디안으로 변환해 리턴하는 함수이다.

radian과 degree간의 상관 관계는 다음과 같다

* $\pi(rad) = 180^{\circ}$
* $0(rad) = 0^{\circ}$

따라서 위 관계를 유지하는 TC를 작성하고, 이를 통과하도록 기능을 개발/조정한다.

이제 TC를 작성해 볼 텐데, TC를 분류에 따라 헤더(또는 소스) 단위로 구분하는 것이 좋지만, 간단한 예시이니 `test.cpp`에 그대로 TC를 작성할 것이다.

![]({{site.baseurl}}/assets/img/posts/2022-04-18/TDD-on-project/img08.jpg)

먼저 테스트할 기능이 포함된 소스의 헤더를 테스트 소스에 include한 후, TC를 작성한다, TEST 매크로의 첫 번재 인자는 TC의 이름, 두 번째 인자는 테스트의 이름을 작성한다.

이후 테스트를 위한 다양한 매크로들을 이용해 TC를 작성하는데, 위의 예시에선 `180.0f degree`를 함수에 전달할 때 `PI radian`이 리턴되어야 한다는 것과, `0.0f degree`를 함수에 전달할 때 `0 radian`이 리턴되어야 함을 명시하였다. 

이후 유닛 테스트 프로젝트의 main함수를 작성한 후 예시와 마찬가지로 GoogleTest를 초기화하고, 모든 테스트를 수행하도록 코드를 작성해 준 후, 빌드 및 실행을 하면 다음과 같이 테스트를 진행한다.

![]({{site.baseurl}}/assets/img/posts/2022-04-18/TDD-on-project/img09.jpg)

테스트가 정상적으로 수행될 경우 초록색 글씨로 **PASSED** 표시가 나타나며, 실패할 경우 **FAILED** 표시가 나타난다.

아래는 TC를 통과하지 못했을 때 어떻게 실패 표시가 나타나는지를 보여준다.

![]({{site.baseurl}}/assets/img/posts/2022-04-18/TDD-on-project/img10.jpg)

+추가로, 만약 자신의 프로젝트가 exe파일을 생성하지 않고 라이브러리를 생성하는 프로젝트라면 Unit Test project의 실행 파일이 위치할 곳에 해당 라이브러리를 제공해야 정상적으로 프로젝트가 실행된다. 아마 라이브러리를 이용한 프로젝트를 진행하는 사람이라면 다 알 것이라 예상한다.

## 3. 결론

예전부터 이런저런 프로젝트를 진행해 보면서 단위 테스트를 이용해 보고 싶다고 생각했는데, 이번 기회에 단위테스트를 적용해 볼 수 있었다.   



처음에는 TC 작성 자체가 시간이 많이 걸려서 비효율적인가 하는 생각도 들었지만 그 와중에 TC를 통해 실수들을 잡는 등의 경험을 하기도 했고, 무엇보다 예전에 진행하던 프로젝트에서 아주 사소한 실수 때문에 원인을 찾지 못해 며칠이고 고생했던 기억들이 남아있어 매우 만족하고 있다.
