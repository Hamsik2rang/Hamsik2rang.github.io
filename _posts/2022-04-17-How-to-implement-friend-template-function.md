---
layout: post
title: Template class에 friend 함수를 구현하는 방법
subtitle: 왜 내 template friend 함수는 컴파일 에러를 내뿜을까?
tags: [Cpp]
author: Im Yongsik
comments: True
use_math: True
---

## 0. 서론

개발 중인 게임 엔진에 들어갈 수학 라이브러리를 구현하다 문법적 에러로 인해 막힌 적이 있다.

탬플릿 구조체를 위한 연산자 오버로딩 함수를 friend 함수로 구현하며 생긴 오류인데, 분명히 클래스 내부에 friend함수의 선언을 작성하고 그 대상이 되는 전역 함수까지 정의를 완료했음에도 컴파일러가 함수를 찾을 수 없는(내 경우엔 ambiguous하다는 오류였다) 오류를 뱉는 것이다.

문제를 해결해 보려고 이리저리 검색하다 보니 그 해결책을 찾게 되어 간략하게 공유하려 한다.

## 1. 문제가 된 구현 형태

오류가 난 코드는 클래스의 볼륨이 꽤 커서, 최초의 동일한 에러를 재현하도록 의도된 짧은 코드를 게시한다. 다음과 같다.

{%highlight cpp%}

#include <iostream>

template <typename T>
class Test
{
private:
      T m_a;
      T m_b;
    
public:
      Test(T a, T b)
          : m_a(a), m_b(b)
      {}
    

      friend const bool operator!=(const Test<T>& a, const Test<T>& b);
};

template <typename T>
const bool operator!=(const Test<T>& lhs, const Test<T>& rhs)
{
      return lhs.a != rhs.a || lhs.b != rhs.b;
}
//...
int main()
{
      Test a(3, 4);
      Test b(4, 5);
    

      if (a != b)
      {
          std::cout<<"a and b are not equal";
      }
    
      return 0;
}

{%endhighlight%}

이렇게 코드를 작성하고 컴파일을 하면 **링크 에러**가 발생한다.

![]({{site.baseurl}}/assets/img/posts/2022-04-17/How-to-implement-friend-template-function/img01.jpg)

>**LNK1120**: 1 unresolved externals
>**LNK2019**:	unresolved external symbol "bool const __cdecl operator!=(class Test<int> const &,class Test<int> const &)" (??9@YA?B_NAEBV?$Test@H@@0@Z) referenced in function main

처음에는 '클래스 안에 선언된 friend함수에 탬플릿 지정이 되어있지 않아서 그런가' 하는 생각으로 해당 선언 코드를 다음과 같이 수정했다.

{%highlight cpp%}

template <typename T>
class Test
{
    //...
    
    template <typename T>
    friend const bool operator!=(const Test<T>& lhs, const Test<T>& rhs);
}

{%endhighlight%}

단순히 위 예시 코드만 보았을 땐 잘 돌아가지만 좋은 해법은 아니다. 왜냐하면 

실제 내 프로젝트 상의 코드에는 템플릿 메타 인자를 함께 사용하는데, 이 경우 재정의 경고가 발생하기 때문이다.

{%highlight cpp%}

#define TYPE_CONSTRAINT_FLOAT_INT_CHAR_UCHAR(T) template <typename T, typename =\
			std::enable_if<\
			std::is_same<float, T>::value || std::is_same<int, T>::value || std::is_same<char, T>::value ||\
			std::is_same<unsigned char, T>::value>>

TYPE_CONSTRAINT_FLOAT_INT_CHAR_UCHAR(T)
class Test
{
      T a;
      T b;
    

      //...
      TYPE_CONSTRAINT_FLOAT_INT_CHAR_UCHAR(T)
      friend const bool 

}
//...

{%endhighlight%}

>**C2572**: 'operator !=': redefinition of default argumeht: parameter 2

## 2. 원인

첫 번째 링크 에러의 경우 확인할 수 없는 외부 기호라는 에러가 나왔는데, 이는 **컴파일러가 friend 선언 구문을 읽을 때 해당 함수를 탬플릿 함수로 인지하지 못하기 때문에 일어나는 에러**이다.

즉, 클래스 내부의 friend선언 함수는 비 탬플릿 함수이고, 클래스 정의 밑에 선언된 `bool operator*=` 함수는 탬플릿 함수이니 서로 다른 함수로 인식한다는 의미이다.

그래서 첫 번째 해결책(friend 선언에도 탬플릿 인자를 명시)으로 효과를 볼 수 있었던 것이나, 이 경우 **재정의 에러가 또 발생**한다는 문제가 있었다.

## 3. 해결책

링크 에러와 재정의 에러를 모두 해결하는 해결책은 두 가지가 존재한다.

1. friend로 선언하고자 하는 함수를 클래스보다 **먼저** 선언(forward declaration)하고, 클래스 안의 friend 선언에 탬플릿 함수임을 나타낼 수 있는 `<>`기호를 추가한다.
2. friend함수를 클래스 안에서 아예 정의해버린다.

 ### 3.1.  전방 선언과 탬플릿 명시를 하는 방법

우선 클래스 선언 앞에 friend로 지정할 함수를 먼저 선언한다.

{%highlight cpp%}

template <typename T> class Test;	// friend함수의 인자에 Test가 필요하다면 전방 선언을 해야 한다.
template <typename T> const bool operator!=(const Test<T>& lhs, const Test<T>& rhs);

template <typename T>
class Test
{
	  //...
	  friend const bool operator!=<>(const Test<T>& lhs, const Test<T>& rhs);	// 탬플릿 함수임을 알리기 위한 <> 표시가 들어감
};

template <typename T>
const bool operator!=<>(const Test<T>& lhs, const Test<T>& rhs)
{
	  return lhs.a != rhs.a || lhs.b != rhs.b;
}

{%endhighlight%}

### 3.2. friend 함수를 클래스 안에 정의까지 해 버리는 방법

제일 간단한 방법이다.

클래스 안에 friend함수의 몸체까지 모두 정의한다.

{%highlight cpp%}

template <typename T>
class Test
{
	  //...
	  friend const bool operator!=(const Test<T>& lhs, const Test<T>& rhs)
	  {
		return lhs.a != rhs.a || lhs.b != rhs.b;
	  }
};

{%endhighlight%}

## 4. 결론

이외에도 Template을 이용하다 보면 이해가 되지 않는 오류들이 간혹 튀어나오는 경우가 있다. 이를테면 객체를 생성할 때 특정 타입으로 변환하지 못한다는 에러가 나오거나(해법: 각 타입에 대한 생성자를 explicit으로 선언) 하는 등이다.

언제 한 번 템플릿이나 템플릿 메타 프로그래밍 부분을 따로 다시 한 번 공부해야 할 듯 하다.

## 5. 참고

https://isocpp.org/wiki/faq/templates#template-friends

https://stackoverflow.com/questions/3989678/c-template-friend-operator-overloading