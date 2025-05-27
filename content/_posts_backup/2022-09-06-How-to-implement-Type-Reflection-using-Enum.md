---
layout: post
title: 열거체와 가상 테이블을 이용해 타입 리플렉션 구현하기
subtitle: 개발일지
tags: [Devlog]
author: Im Yongsik
comments: True
use_math: True
---

※ [개인 프로젝트](https://github.com/Hamsik2rang/Kepler) 도중 마주한 문제와 해결 과정을 토대로 작성했습니다.

<br>

## 0. 서론

현대를 살아가는 대부분의 프로그래머는 객체지향 패러다임에 기반한 코드를 작성한다. 이는 쉬운 유지 보수와 함께 '**상속**'이라는 강력한 무기가 프로그래머들에게 절차지향 패러다임에서라면 복잡하기 짝이 없을 구조를 이전보다 한층 편리하게 표현할 수 있도록 도와주기 때문이다.

<br>

상속을 사용하는 것은 단순히 코드의 중복을 막는 것 뿐만 아니라 **다형성(****Polymorphism)**을 이용해 여러 객체를 하나의 공동체로 바라보며 사용할 수 있도록 해 준다. 이는 다르게 말하면 객체 개개를 그만의 '타입'에 의존하지 않고 그들을 묶는 '집단'의 구성원으로 바라본다는 것이기도 하다.

## 1. 다형성의 문제점

그러나 같은 학교의 모든 학생들이 같은 시간에 같은 수업을 들으면서도 같은 모습과 행동을 갖지 않듯, 한(또는 여러) 부모 객체로부터 파생된 객체들은 각각 자신만의 행동을 정의할 수 있고, 행동할 수 있으며, 행동해야 한다.

<br>

그러나 다형성에 기반해서 객체들을 관리할 경우, 모든 객체들을 그들의 부모 객체를 기준으로 바라보며 관리하기 때문에 그들만의 행동을 수행하는 데 제약이 발생한다. Java와 같이 Down Casting 자체를 불가능하게 하는 언어들이라면 말할 것도 없고, C++와 같은 명시적인 Down-Casting이 가능한(C++의 dynamic_cast<>())경우에도 이는 유효하다.

<br>

클래스 A로부터 B, C, D가 파생되었고, 이들의 명세로부터 정의된 객체가 100개가 한 컨테이너에 존재한다고 가정해 보자. C++ 코드로 나타내면 다음과 같을 것이다.

{%highlight cpp%}

/* main.cpp */
class A
{
    //...
}

class B : public A
{
    //...
    void FuncB(){/*...*/}
}
class C : public A
{
    //....
    void FuncC(){/*...*/}
}
class D : public A
{
    //...
    void FuncD(){/*...*/}
}

int main()
{
    std::vector<A*> container // 이 안에 각각이 누구인지 모를 객체 100개가 들어 있다.    
}

{% endhighlight %}

이 때 b(), c(), d()함수는 각각 B, C, D 객체에서만 정의된 함수라고 가정하자.

container의 객체들을 각각 B, C, D중 무엇인지 구분해 FuncB(), FuncC(), FuncD()함수를 수행하도록 하려면 어떻게 해야 할까?

<br>

dynamic_cast를 이용한다 하더라도 사용자는 container안의 각 원소들이 정확히 어떤 타입인지 알 길이 없다.

그렇다면, 만약 그들에게 직접 자신의 타입을 물어볼 수 있도록 한다면 어떨까?

아마 다음과 같은 코드의 작성이 가능해질 것이다.

{%highlight cpp%}

void DoSomething(const std::vector<A*>& container)
{
	for (int i = 0; i < container.size(); i++) 
    {
        auto type = container[i].GetType();
        // A는 인터페이스이므로 타입 구분에서 배제된다고 가정
        switch (type)
        {
        case B:
            container[i].FuncB();
            break;
        case C:
            container[i].FuncC();
            break;
        case D:
            container[i].FuncD();
            break;
        }
    }
}

{%endhighlight%}

## 2. 리플렉션(Reflection)

이처럼 완성된 객체로부터 객체의 정보(타입, 함수, 변수의 선언 정보)를 받아오는 것을 **'리플렉션(Reflection)'**이라 한다.

C#과 Java등에서는 언어 자체에서 리플렉션 기능을 지원하지만, Native C/C++의 경우 별도의 리플렉션 기능이 지원되지 않는다.

<br>

일반적으로 C++프로그래머는 탬플릿 메타프로그래밍 기법을 통해 자신의 프로그램에 리플렉션 기능을 구현한다.

심지어 C++ 공식 라이브러리에서도 *std::is_abstract_v()* 와 같은 이러한 메타 함수들을 일부 제공한다.

<br>

그러나 탬플릿 메타프로그래밍은 일반적으로 가독성이 좋지 않고 코드의 크기가 매우 커지는 단점이 존재한다. 또한 모든 것이 컴파일 시점에 이루어지는 특성상 빌드 타임이 늘어나는 것도 주요한 단점이다.

<br>

이러한 단점들을 감안할 때 단순히 상속받은 객체들 간의 구분에 메타프로그래밍 기법을 도입하는 것은 간단히 결정할 수 있는 일이 아니었고, 따라서 다른 방법을 통해 리플렉션 기능을 구현할 방법을 찾아야 했다.



##  3. 구현 명세

모든 개발 과정이 그러하듯, 타입 리플렉션을 구현하기 전 어떠한 기능들이 필요할지 그 명세를 먼저 작성하였고, 다음과 같았다.

\* 같은 클래스로부터 만들어진 모든 객체가 그들이 가진 데이터에 관계없이 모두 같은 동작을 수행해야 함

\* 서로 구분하고자 하는 클래스들 모두가 같은 동작을 수행해야 함

\* 객체들은 모두 부모 클래스의 형태로 간주되어 관리되므로 부모 클래스가 그 인터페이스를 가지고 있어야 함

\* 서로의 타입을 비교할 수 있어야 함

\* 가능하면 불필요한 자원 낭비가 발생하지 않아야 함

<br>

C++을 어느정도 사용한 프로그래머라면 위 명세를 보자마자 어떠한 방식을 사용해야 할 지 감을 잡을 것이다. 맨 위 첫 번째 요구는 정적 함수를 통해 구현할 수 있으며, 다음 두 가지는 결국 부모 클래스인 A클래스에서 **(순수)가상 함수**를 정의하고 각 자식 클래스들이 해당 함수에서 자신의 타입을 구분할 정보를 제공하게 하면 된다. 그리고 마지막 두 가지는 타입을 비교할 수 있도록 값의 형태를 제공하되 이를 위해 메모리 점유가 일어나는 것을 피하는 것이 좋다는 뜻이므로 **열거체(enum) 혹은 열거 클래스(enum class)**를 사용해 해결할 수 있을 것이다.

<br>

그런데 문제가 되는 것은 **정적 클래스 함수는 virtual로 선언할 수 없다**는 것이다. 따라서 각 함수가 정적 클래스 함수를 가지게 한 후 가상 함수에서 해당 함수의 호출 결과를 리턴하도록 구현해야 한다.

## 4. 구현 결과

명세를 바탕으로 타입 리플렉션을 구현하면 다음과 같을 것이다.

{% highlight cpp %}

/* reflection.hpp */
enum eType { A = 0, B, C, D }

class A
{
    //...
    virtual eType GetType() const = 0;
}

class B : public A
{
    //...
    void FuncB();
    static eType GetStaticType() { return eType::B; }
    virtual eType GetType() const override { return GetStaticType(); }
}
class C : public A
{
    //...
    void FuncC();
    static eType GetStaticType() { return eType::C; }
    virtual eType GetType() const override { return GetStaticType(); }
}
class D : public A
{
    //...
    void FuncD();
    static eType GetStaticType() const { return eType::D; }
    virtual eType GetType() const override { return GetStaticType(); }
}

{% endhighlight %}

이제 위에서 보았던 switch문이 정상적으로 동작할 수 있음을 알 수 있다.

만약 코드의 중복이 마음에 들지 않는다면, 전처리문을 통해 정리할 수 있다. 그러나 가독성이 더 좋은지는 판단에 맡긴다.

{% highlight cpp %}

/* reflection.hpp */
enum eType { A = 0, B, C, D }

#define CLASS_TYPE(type) static eType GetStaticType() { return eType::##type; }\
                         virtual eType GetType() const override { return GetStaticType(); }

class A
{
    //...
    virtual eType GetType() const = 0;
}

class B : public A
{
    //...
    void FuncB();
    CLASS_TYPE(B)
}
class C : public A
{
    //...
    void FuncC();
    CLASS_TYPE(C)
}
class D : public A
{
    //...
    void FuncD();
    CLASS_TYPE(D)
}

{% endhighlight %}