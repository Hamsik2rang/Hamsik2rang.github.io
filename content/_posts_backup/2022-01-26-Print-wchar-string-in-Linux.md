---
layout: post
title: Linux에서 wchar 문자열 출력하기
tags: [Cpp, WSL, Linux]
author: Im Yongsik
comments: True
---

## 0. 서론

최근 리눅스에서 소켓 프로그래밍을 진행하면서 출력과 관련해 막히는 부분이 있었습니다. wchar_t 문자 배열로 표현되는 한글 문자열의 출력과 관련된 부분이었는데, 이를 해결하게 되어 해결법을 공유하고자 합니다.

### 1. 한글이 "????..."으로 출력돼요!

다음 코드가 있다고 하자.

```c
#include <iostream>
#include <cwchar>

int main(int argc, char** argv)
{
    wchar_t test_str[](L"오늘 점심은 샌드위치")

    std::wcout << test_str;
    // or 
    // wprintf(L"%S", test_str);
    // note. 리눅스는 wchar를 위한 포매팅 인자로 '대문자 S'를 전달함.
    
    return 0;

}
```

만약 출력 결과가 다음처럼 나타난다면

```text
Output:
?? ??? ????
```

별도로 한글 출력을 위한 locale 설정이 필요하다.
wide character 출력은 리눅스 환경의 locale, 그 중 LC_CTYPE에 영향을 받으므로, 자신의 locale 상태를 *ko_KR.UTF-8*로 수정하거나([참고](https://beomi.github.io/2017/07/10/Ubuntu-Locale-to-ko_KR/)) 소스 코드 상에 아래와 같이 입력해 locale을 지정해야 한다.

```c
#include <locale.h>
...
// LC_CTYPE을 인자로 전달해도 상관 없음.
setlocale(LC_ALL, "ko_KR.UTF-8");
```

따라서 다음과 같이 코드를 실행하면 정상적으로 한글이 출력될 것이다.

```c++
#include <iostream>
#include <cwchar>
#include <locale.h>

int main(int argc, char** argv)
{
    wchar_t test_str[]("오늘 점심은 샌드위치");
    setlocale(LC_ALL, "ko_KR.UTF-8");
    
    std::wcout << test_str;
    // or 
    // wprintf(L"%S", test_str);
    // note. 리눅스는 wchar를 위한 포매팅 인자로 '대문자 S'를 전달함.
    
    return 0;
}
```

### 2. cout과 wcout(또는 printf와 wprintf)를 동시에 사용하고 싶어요!

만약 cout과 wcout처럼 byte-char와 wide-char 타입의 출력을 한 소스에서 동시에 진행하고자 하는 경우 둘 중 하나가 작동하지 않는 현상이 나타난다.

```c++
#include <cstdio>
#include <cwchar>
#include <locale.h>

int main(int argc, char** argv)
{
    // 리눅스 locale이 ko_KR.UTF-8로 설정된 경우
    // 두 번째 인자로 null을 전달하면 시스템 상의 locale과 동일하게 세팅됨.
    setlocale(LC_ALL, "");
    
    wprintf(L"안녕하세요?\n");
    printf("반가워요\n");
    wprintf(L"wide char\n");
    printf("byte char\n");
    
    return 0;
}
```

<br>

```
Output:
안녕하세요?
wide char
```



위 코드에서 첫 번째 wprintf와 printf의 호출 순서를 바꿔보면 출력이 다르게 나온다.

```c++
...
    setlocale(LC_ALL, "");

    printf("반가워요\n");
    wprintf(L"안녕하세요?\n");
    wprintf(L"wide char\n");
    printf("byte char\n");
...
```

<br>

```
Output:
반가워요
byte char
```

살펴보면, 첫 번째로 wide char에 대한 함수를 출력하면 이후로도 wide char 관련 함수만 수행하고, byte char에 대한 함수를 첫 번째로 출력하면 이후로도 byte char만 출력하는 것처럼 보인다.

이는 C 표준과 관련이 있는데, 한 `FILE`스트림은 자신의 **지향(orientation)**이 결정되면 해당 orientation으로만 기능을 수행한다.

임의의 stream의 orientation은 다음 방식으로 결정할 수 있다.

1. `fwide()` 함수 호출
2. 특정 속성을 지향하는 입/출력 함수 수행
3. `freopen()` 함수 호출

#### 2.1.`fwide()`

fwide() 함수의 시그니처는 다음과 같다.

```c
#include <stdio.h>
int fwide(FILE* stream, int mode)
```

첫 번째 인자로 스트림을, 두 번째 인자로 설정하고자 하는 스트림 지향 속성의 정수값을 전달한다.
0보다 큰 값을 전달하면 wide character로, 0보다 작은 값을 전달하면 byte character로 스트림 지향을 설정한다. 만약 0을 전달할 경우 스트림 지향을 변경하지 않는다.
또, 반환값으로는 함수 호출 이후의 스트림 지향 속성의 정수값을 반환하는데, 호출 후 스트림 지향이 wide character로 설정되었다면 0보다 큰 값을, byte character로 설정되었다면 0보다 작은 값을 반환한다.

`fwide()`함수는 매우 간편해 보이지만 한 가지 단점이 있는데, 이미 지향이 결정된 스트림의 경우 속성을 변경할 수 없다.
즉, 다음과 같은 코드가 있다면

```c++
int main()
{
    wprintf(L"wide char\n");
    
    if(fwide(stdout, -1) < 0)
        printf("byte char\n");
}
```

다음과 같이 출력된다.

```text
Output:
wide char
```

`fwide()`를 호출해도 스트림 지향이 변경되지 않은 것을 알 수 있다.

자세한 내용은 [참고](https://www.ibm.com/docs/en/zos/2.4.0?topic=functions-fwide-set-stream-orientation) 링크를 확인하면 된다.

#### 2.2. 특정 속성을 지향하는 입/출력 함수 수행

앞서 예시 코드로 살펴본 것 처럼, `fwide()`와 같은 함수를 별도로 호출하지 않아도 특정 지향의 표준 입출력 함수를 사용하면 자동으로 스트림이 해당 지향으로 설정된다.
예를 들어, 최초의 출력을 `wprintf()`로 수행하면 표준 출력은 wide char 지향이 되고, `printf()`로 수행하면 byte char 지향이 된다.

#### 2.3. `freopen()`

만약 앞서 소개한 방법들로 이미 고정된 스트림 지향을 바꾸고 싶다면 `freopen()` 함수를 이용해 **스트림을 닫고 다시 열어야 한다.**

`freopen()`함수의 시그니처는 다음과 같다.

```c++
#include <stdio.h>
FILE* freopen(const char* filename, const char* mode, FILE* stream);
```

`freopen()`함수는 현재 stream을 닫고, filename에 지정된 파일에 stream을 재지정한다.

만약 filename으로 null값을 지정하게 되면 스트림을 닫은 후에 특정 파일로 연결하지 않고 현재 연결 상태로 다시 열게 된다.
즉, 표준 스트림을 특정 파일에 연결할 게 아니라면 그냥 null값을 전달하면 된다.

```c++
#include <cstdio>
#include <locale.h>

int main()
{
    setlocale(LC_ALL, "ko_KR.UTF-8");
    wprintf(L"안녕하세요\n");
    
    freopen("", "w", stdout);
    printf("안녕히계세요\n");
    
    return 0;
}
Output:
안녕하세요
안녕히계세요
```

### 3. 출력 자체가 안돼요!

이 문제는 이 글을 쓰게 된 궁극적인 이유이기도 한데, 리눅스에서 소켓 통신을 구현하다가 출력이 제때 진행되지 않는 현상이 있었다.

```c++
#include <cstdio>
#include <cstring>
#include <cwchar>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <unistd.h>

...
wchar_t buf[128]{0};
while (true)
{
    memset(buf, sizeof(buf), 0);
    socklen_t len = recvfrom(recv_sock, buf, 127, 0, nullptr, 0);        
    if(len < 0)
        break;
            
    wprintf(L"%S", buf);
}
...
```

위 코드에서 무한 루프 내에서 소켓을 통해 한글 문자열을 받아올 때마다 `wprintf()`를 이용해 출력해야 하는데, 출력이 전혀 진행되지 않는 것이다!

동일 증상을 구현하기 위해 간단한 예시 코드를 하나 작성해 보자.

```c
#include <cstdio>
#include <cwchar>

int main()
{
    wprintf(L"Hello");
    sleep(5);
    
    return 0;
}
```

위 코드를 수행하면 5초 동안 `"Hello"`가 출력되지 않다가 5초가 지난 후 프로그램 종료 직전에 갑자기 출력되는 걸 볼 수 있다.

알아보니 Linux의 표준 출력 스트림은 Windows와 조금 다르게 동작하는데, Windows에서 `printf()`등의 출력 함수를 호출하면 인자로 전달한 출력 데이터를 스트림에 전달한 후 바로 출력 스트림에서 해당 데이터를 플러싱(flushing)한다. 즉, 전달한 값을 그때그때 바로 출력하는 것이다.

반면 Linux는 출력 함수를 호출해 출력 데이터를 전달해도 바로 출력하지 않고 일단 출력 버퍼에 저장만 해 둔다.
그럼 Linux에서는 언제 버퍼의 내용을 출력할까?
다음과 같다.

- 출력 버퍼에 개행 문자 `'\n'`가 들어갈 경우
- `fflush(FILE* stream)` 함수를 통해 강제로 버퍼를 비울 경우
- 프로그램이 종료될 경우

위 예시 코드(sleep)의 경우 개행 문자가 들어가거나 `fflush()`함수가 호출되지 않아 버퍼에 `"Hello"` 문자열을 저장하고 있다가 sleep이 끝나고 프로그램이 종료되자 버퍼를 비우며 해당 내용을 호출한 것이다.

따라서 출력을 위해선 다음과 같이 개행 문자를 추가하거나,

```c
wprintf(L"Hello\n");
```

출력 버퍼에 데이터를 전달한 후 `fflush()`함수를 호출해야 한다.

```c
wprintf(L"Hello");
fflush(stdout);
```

따라서 맨 처음 문제가 되었던 코드도 다음과 같이 수정해 해결하였다.

```c++
#include <cstdio>
#include <cstring>
#include <cwchar>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <unistd.h>

...
wchar_t buf[128]{0};
while (true)
{
    memset(buf, sizeof(buf), 0);
    socklen_t len = recvfrom(recv_sock, buf, 127, 0, nullptr, 0);        
    if(len < 0)
        break;
            
    // wprintf(L"%S\n", buf);로 fflush() 호출 대신 개행 문자를 삽입해도 된다.
    wprintf(L"%S", buf);
    fflush(stdout);
}
...
```

## 결론

꽤 오랫동안 이유를 찾지 못해서 고민했는데, 스택오버플로우와 여러 국내 블로그들을 통해 해결하게 되었습니다. 역시 프로그래밍은 앞서 걸어간 개발자들 덕분에 진전이 있는 것 같습니다. 이 글도 누군가에게 도움이 되길 바라며 참고한 링크들을 모두 아래 기록합니다.

- [우분투 locale 설정](https://beomi.github.io/2017/07/10/Ubuntu-Locale-to-ko_KR/)
- [fwide()](https://www.ibm.com/docs/ko/i/7.3?topic=functions-fwide-determine-stream-orientation)
- [freopen()](https://www.ibm.com/docs/ko/i/7.3?topic=functions-freopen-redirect-open-files)
- [wprintf() Linux Manpage](https://man7.org/linux/man-pages/man3/wprintf.3.html)
- [printf and wprintf in single c code - stackoverflow](https://stackoverflow.com/questions/8681623/printf-and-wprintf-in-single-c-code)
- [리눅스(Linux)에서의 표준 출력 - 김성엽 님 블로그](https://blog.naver.com/PostView.naver?isHttpsRedirect=true&blogId=tipsware&logNo=221055590809)
- [[Linux\] 표준 입출력](https://shumin.co.kr/linux-표준-입출력-stdin-stdout-stderr/)