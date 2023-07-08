---
layout: post
title: C++ 최적화 - part 4
subtitle: ~C++ 최적화~를 읽고 정리한 글입니다.
tags: [Cpp]
author: Im Yongsik
comments: True
use_math: True
---

# 0. 정리

* 문자열은 동적으로 할당된다. `std::string`의 경우 할당된 버퍼를 초기화하는 문자(문자열)가 들어올 경우, 할당된 버퍼보다 더 큰 버퍼를 새로 할당한 후 복사하는 작업을 수행하게 된다.

* 문자열은 "값" 이다. 다음과 같은 코드가 있을 경우

  ```c++
  std::string s1 = "Hello";
  std::string s2 = s1;
  s1[0] = "C";
  ```

  s1은 "Cello"이며 s2는 "Hello"이다.

  즉, 내부적으로

  ```c++
  std::string s2 = s1;
  ```

  연산이 수행되는 순간 `s2`의 문자열 버퍼에 `s1`의 문자열에 해당하는 "값을 복사"한다.

  이러한 값 복사를 막기 위해 참조 방식을 사용하는 Copy-on-Write 문자열을 구현할 수도 있다. 이 경우 위의 동일한 연산에서 `s1`과 `s2`는 같은 문자열 버퍼 메모리를 가리키며, 해당 메모리는 레퍼런스 카운트를 통해 참조하는 문자열들을 관리한다.

  그러나 C++11 표준에서 COW방식의 문자열은 문제가 발생할 수 있어 허용하지 않는다. 대입이나 인수 전달 등의 비용이 적다는 장점이 있으나 메모리를 공유하므로 const가 아닌 참조의 내용을 수정할 경우 할당 및 복사 연산을 위한 추가 비용이 발생하며, 여러 스레드에서 레퍼런스 카운트에 접근할 때 문제가 될 수 있기 때문이다(정확히는, 각 스레드가 메인 메모리에서 카운트의 복사본을 가져와 다른 스레드가 값을 변경하였는지 확인하는 atomic한 연산이 필요하다.)

  C++11에서 등장한 R-value참조와 `std::move`덕분에 문자열을 복사하는 비용이 많이 줄어들었다. 이 방식들을 이용하면 문자열이 값이 아닌 포인터를 복사한다.

* 위의 두 원칙을 명심하며 문자열 `s`에서 알파벳 `v`를 제거한 문자열을 획득하는 아래 코드를 최적화 해 보자.

  ```c++
  std::string s = "asdfzxcvwvvwagsgsgolnoinwfasd";
  
  remove_v(s);
  
  //...
  
  std::string remove_v(std::string s)
  {
      std::string result;
      for (int i = 0; i < s.length(); i++)
      {
          if (s[i] != 'v')
          {
              result = result + s[i];
          }
      }
      
      return result;
  }
  ```

  #### 1. 임시 문자열 제거하기

  위 코드에서 최적화 해볼 첫번째 여지는 문자열 연결 연산자이다. 

  ```c++
  result = result + s[i];
  ```

  위 연산은 `result`와 `s[i]`를 더하는 순간 그 결과를 저장할 임시 문자열을 생성한다. 이후 해당 임시 객체의 값을 result에 또 복사한다(물론 현대 컴파일러는 이 정도의 연산은 당연히 최적화를 해 준다. 그렇지만 예시이므로..).

  따라서 아래처럼 문자열 연결 연산자를 피하는 것 만으로도 성능을 향상시킬 수 있다.

  ```c++
  std::string remove_v(std::string s)
  {
      std::string result;
      for (int i = 0; i < s.length(); i++)
      {
          if (s[i] != 'v')
          {
              result += s[i];
          }
      }
      
      return result;
  }
  ```

  위 코드는 임시 문자열을 생성하는 대신 `result`의 문자열 버퍼가 비어있는 한 해당 버퍼에 문자를 추가할 뿐이므로 더 나은 성능을 보여준다.

  #### 2. 저장 공간 예약하기

  또 한번 최적화를 할 여지를 찾아 보자면, 위의 언급이 해답이 될 것이다. "`result`의 문자열 버퍼가 비어있는 한 해당 버퍼에 문자를 추가한다"는 말은, 문자열 버퍼가 꽉 차면 문자열 복사가 수행된다는 뜻이므로, 처음부터 `result`에 충분한 버퍼 크기를 부여하면 문자열 복사가 수행되지 않을 것이다. 따라서 다음과 같이 수정할 수 있다.

  ```c++
  std::string remove_v(std::string s)
  {
      std::string result;
  
      result.reserve(s.length());
  
      for (int i = 0; i < s.length(); i++)
      {
          if (s[i] != 'v')
          {
              result += s[i];
          }
      }
      
      return result;
  }
  ```

  #### 3. 인수 전달을 위한 복사 제거하기

  인수로 전달하는 문자열마저 참조자를 사용해 복사 연산을 제거할 수 있다.

  ```c++
  std::string remove_v(std::string& s)
  {
      std::string result;
      
      result.reserve(s.length());
      
      for (int i = 0; i < s.length(); i++)
      {
          if (s[i] != 'v')
          {
              result += s[i];
          }
      }
      
      return result;
  }
  ```

  그러나 이 경우 실제 성능이 예상한 만큼 증가하지 않을 수 있는데, 그 이유는 참조자의 경우 내부적으로 포인터로 구현되기 때문에 포인터 역참조 연산에 대한 추가 비용이 발생하기 때문이다. 따라서 이를 방지하기 위해 반복자를 이용한다.

  ```c++
  std::string remove_v(std::string& s)
  {
      std::string result;
      result.reserve(s.length());
      
      for (auto it = s.begin(), end = s.end(); it != end; ++it)
      {
          if (*it != 'v')
          {
              result += *it;
          }
      }
      
      return result;
  }
  ```

  반복자는 내부의 문자 버퍼를 가리키는 포인터이므로, 포인터의 역참조 비용을 절약할 수 있다. 또한 루프 안의 변수인 `end`를 캐싱의 용도로 반복문 초기화 부분에 미리 선언한 것을 확인할 수도 있다.

  #### 4. 반환값 복사 제거하기

  out parameter를 이용해 반환값 복사마저 제거해 더욱 최적화를 할 수도 있다.

  ```c++
  void remove_v(std::string& s, std::string& result)
  {
      result.clear();
      result.reserve(s.length());
      
      for (auto it = s.begin(), end = s.end(); it != end; ++it)
      {
          if (*it != 'v')
          {
              result += *it;
          }
      }
  }
  ```

* 이쯤에서 유념해야 할 점은 최적화와 사용성 사이에서 고민해야 한다는 것이다. 앞서 문자열을 최적화하는 과정에서 함수 `remove_v`의 인터페이스가 최초의 설계와는 많이 달라진 것을 확인할 수 있다. 지나친 최적화(심지어 위에 적지 않은 더 많은 방법들, 이를테면 C스타일 배열 사용 등을 적용할 수도 있다!)는 단순성과 안정성에 악영향을 미칠 수도 있다는 점을 항상 염두해야 한다.

* 또한, 앞서 최적화를 수행한 방식들 대신 **더 좋은 알고리즘을 채택하는** 방법도 존재한다. 문자 하나하나를 `result`에 추가하는 대신 `v`가 등장하기 전까지의 문자열 블록을 한 번에 추가한다고 생각해 보자. 할당 및 복사 연산 횟수가 줄어드는 효과가 있을 것이다.
* 또는 **더 좋은 라이브러리**를 사용하는 것도 방법이 될 것이다.
