---
title: "Game Server Development #8 : Event"
summary: "Event"
categories: ["Korean Post", "Programming"]
tags: ["C++", "Server", "Thread", "Event"]
date: 2023-10-19
draft: false
showauthor: false
authors:
  - bbagwang
series: ["Game Server Development"]
series_order: 8
---

## Material
**[[C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈] Part4: 게임 서버](https://inf.run/8Chk)**

## Event

`Event` 는 운영 체제의 스레드 동기화 매커니즘 중 하나로, 특정 이벤트가 발생했는지 여부에 따라 하나 이상의 스레드가 대기하거나 실행을 계속할 수 있도록 하는 신호를 제공한다.

주로 비동기 작업에서 상태 변화나 특정 조건의 발생을 스레드에 알리는 데 사용된다.

이벤트 관리를 위해 운영체제에서 `Kernel Object` 를 생성하며, 커널 오브젝트는 다른 프로세스에서도 접근할 수 있다.

## Event State

이벤트는 `Signaled` 와 `Non-Signaled` 상태를 가진다.

`Signaled` 상태는 이벤트가 발생했음을 의미한다.

`Non-Signaled` 상태는 이벤트가 발생하지 않았음을 의미한다.

## Event Reset Type

`Manual Reset` 은 이벤트가 `Signaled` 상태가 되면, `Reset` 을 따로 호출하기 전까지 `Signaled` 상태를 유지한다.

`Auto Reset` 은 이벤트가 `Signaled` 상태가 되면, 따로 `Reset` 을 호출하지 않아도 곧바로 `Non-Signaled` 상태로 바뀐다.

## Event Functions

`Event` 사용을 위해 Windows 운영체제에서는 다음과 같은 함수를 제공한다.

### CreateEvent

`Event` 를 생성할 때, 사용하는 함수이다.

CreateEvent 함수의 원형은 다음과 같다.

```cpp

HANDLE CreateEvent(
  LPSECURITY_ATTRIBUTES lpEventAttributes,
  BOOL                  bManualReset,
  BOOL                  bInitialState,
  LPCTSTR               lpName
);

```

- `lpEventAttributes` : 이벤트의 보안 속성을 설정한다. `NULL` 을 입력하면 기본값으로 설정된다.

- `bManualReset` : `Manual Reset` 과 `Auto Reset` 을 선택할 수 있다.

- `bInitialState` : 이벤트의 초기 상태를 설정한다. `TRUE` 를 입력하면 `Signaled` 상태로, `FALSE` 를 입력하면 `Non-Signaled` 상태로 시작한다.

- `lpName` : 이벤트의 이름을 설정한다. `NULL` 을 입력하면 이름이 없는 이벤트 를 생성한다.

- `return` : 이벤트를 생성하면 이벤트의 `HANDLE` 을 반환한다. 이벤트 생성에 실패하면 `NULL` 을 반환한다.

### SetEvent

이벤트를 `Signaled` 상태로 만들 때, 사용하는 함수이다.

### ResetEvent

이벤트를 `Non-Signaled` 상태로 만들 때, 사용하는 함수이다.

### WaitForSingleObject

이벤트가 `Signaled` 상태가 될 때까지 대기하는 함수이다.

`WaitForSingleObject` 함수의 원형은 다음과 같다.

```cpp

DWORD WaitForSingleObject(
  HANDLE hHandle,
  DWORD  dwMilliseconds
);

```

- `hHandle` : `Wait` 할 이벤트의 `HANDLE` 을 입력한다.

- `dwMilliseconds` : `Wait` 할 시간을 입력한다. `INFINITE` 를 입력하면 무한정 대기한다.

- `return` : 이벤트가 `Signaled` 상태가 되면 `WAIT_OBJECT_0` 을 반환한다. `Wait` 시간이 지나면 `WAIT_TIMEOUT` 을 반환한다.

### WaitForMultipleObjects

여러 개의 이벤트가 `Signaled` 상태가 될 때까지 대기하는 함수이다.

`WaitForMultipleObjects` 함수의 원형은 다음과 같다.

```cpp

DWORD WaitForMultipleObjects(
  DWORD        nCount,
  const HANDLE *lpHandles,
  BOOL         bWaitAll,
  DWORD        dwMilliseconds
);

```

- `nCount` : `Wait` 할 이벤트의 개수를 입력한다.

- `lpHandles` : `Wait` 할 이벤트의 `HANDLE` 을 입력한다.

- `bWaitAll` : `Wait` 할 이벤트가 모두 `Signaled` 상태가 될 때까지 대기할지 여부를 입력한다. `TRUE` 를 입력하면 모두 `Signaled` 상태가 될 때까지 대기하고, `FALSE` 를 입력하면 하나라도 `Signaled` 상태가 되면 대기를 종료한다.

- `dwMilliseconds` : `Wait` 할 시간을 입력한다. `INFINITE` 를 입력하면 무한정 대기한다.

- `return` : `Wait` 할 이벤트가 `Signaled` 상태가 되면 `WAIT_OBJECT_0` 을 반환한다. `Wait` 시간이 지나면 `WAIT_TIMEOUT` 을 반환한다.

## Behavior of WaitForSingleObject Based on Reset Type

`WaitForSingleObject` 는 `SetEvent`` 가 호출되는 순간 대기중인 모든 상태를 깨울까?

코드를 통해 테스트 해보자.

```cpp
#include <iostream>
#include <thread>
#include <chrono>
#include <Windows.h>

HANDLE g_handle;

void Signal()
{
	::SetEvent(g_handle);

	std::cout << "Signaled!" << std::endl;
}

void Receiver1()
{
	::WaitForSingleObject(g_handle, INFINITE);
	std::cout << "Receive #1" << std::endl;
}

void Receiver2()
{
	::WaitForSingleObject(g_handle, INFINITE);

	std::cout << "Receive #2" << std::endl;
}

int main()
{
    //Manual Reset 을 FALSE 로 설정하여, Auto Reset 인 경우.
	g_handle = ::CreateEvent(NULL, FALSE, FALSE, NULL);

	std::thread t1(Signal);
	std::thread t2(Receiver1);
	std::thread t3(Receiver2);

	t1.join();
	t2.join();
	t3.join();

	::CloseHandle(g_handle);

	return 0;
}
```

위 코드의 결과는 다음과 같다.

```
Signaled!
Receive #2
```

여기서 더 이상 실행되지 않는다.

이유는 `Receiver1` 의 `WaitForSingleObject` 가 통과되지 못하였기 때문이다.

그러므로, `t2` 스레드가 대기 상태로 들어가며, `t2.join` 또한 통과되지 못하기 때문이다.

`WaitForSingleObject` 는 `Auto Reset` 하게 된다면, `Signaled` 이후 즉시 `Non-Signaled` 상태로 변경되기 때문에 다른 대기중인 스레드를 깨워주지 않는다.

그렇다면 즉시 `Non-Signaled` 상태로 가지 않도록 `Manual Reset` 으로 하게 된다면 모두 울릴까?

테스트 해보자.

```cpp

#include <iostream>
#include <thread>
#include <chrono>
#include <Windows.h>

HANDLE g_handle;

void Signal()
{
	::SetEvent(g_handle);

	std::cout << "Signaled!" << std::endl;
}

void Receiver1()
{
	::WaitForSingleObject(g_handle, INFINITE);
	std::cout << "Receive #1" << std::endl;
}

void Receiver2()
{
	::WaitForSingleObject(g_handle, INFINITE);

	std::cout << "Receive #2" << std::endl;
}

int main()
{
	g_handle = ::CreateEvent(NULL, TRUE, FALSE, NULL);

	std::thread t1(Signal);
	std::thread t2(Receiver1);
	std::thread t3(Receiver2);

	t1.join();
	t2.join();
	t3.join();

	::CloseHandle(g_handle);

	return 0;
}

```

`bManualReset` 을 `TRUE` 로 설정했다.

결과는 다음과 같다.

```

Signaled!
Receive #1
Receive #2

```

모두 호출되었다.

다만, 이 경우 `ResetEvent` 를 호출하지 않았으므로, 여전히 `g_handle` 에 대한 `Event` 는 `Signaled` 상태로 유지된다.

## Problem Example

아래 코드는 `Producer` 스레드가 `Consumer` 스레드에게 데이터를 전달하는 코드이다.

```cpp
#include <iostream>
#include <thread>
#include <mutex>
#include <queue>

std::mutex m;
std::queue<int> q;

void Producer()
{
    while(true)
    {
        std::unique_lock<std::mutex> lock(m);
        q.push(100);
    }

    std::this_thread::sleep_for(100ms);
}

void Consumer()
{
    while(true)
    {
        std::unique_lock<std::mutex> lock(m);
        if(!q.empty())
        {
            int data = q.front();
            q.pop();

            //Lock 을 잡고, 콘솔 출력을 하는 것은 좋은 습관은 아님.
            std::cout<<data<<std::endl;
        }
    }
}

int main()
{
   std::thread t1(Producer);
   std::thread t2(Consumer);

   t1.join();
   t2.join();

   return 0;
}

```

만약 `Producer` 스레드에 데이터가 들어가는 시간이 100ms 가 아니라, 1 시간 이라면 어떨까?

`Consumer` 스레드는 `Producer` 스레드가 데이터를 전달할 때까지 `while` 문을 돌며 계속 대기해야 한다.

이처럼 오래 걸릴 것이 예상되는 작업에 `SpinLock` 을 사용하는 것은 `CPU` 자원을 낭비하게 된다.

## Solution by using Event

`Event` 를 사용하면 이러한 문제를 다음과 같이 해결할 수 있다.

```cpp
#include <iostream>
#include <thread>
#include <mutex>
#include <queue>
#include <Windows.h>

std::mutex m;
std::queue<int> q;
HANDLE handle;

void Producer()
{
    while(true)
    {
        {
            std::unique_lock<std::mutex> lock(m);
            q.push(100);
        }

        ::SetEvent(handle);

        std::this_thread::sleep_for(100ms);
    }
}

void Consumer()
{
    while(true)
    {
        //Signaled 상태가 될 때까지 무한정 대기한다.
        ::WaitForSingleObject(handle, INFINITE);

        //만약 Event 가 Manual Reset 으로 생성되었다면, Non-Signaled 상태로 바꿔준다.
        //::ResetEvent(handle);

        std::unique_lock<std::mutex> lock(m);
        if(!q.empty())
        {
            int data = q.front();
            q.pop();

            //Lock 을 잡고, 콘솔 출력을 하는 것은 좋은 습관은 아님.
            std::cout<<data<<std::endl;
        }
    }
}

int main()
{
    //Event 를 생성한다.
    //bManualReset 을 FALSE 로 설정했다. Auto Reset 기능을 사용한다.
    //Initial State 를 FALSE 로 설정했다. Non-Signaled 상태로 시작한다.
    handle = ::CreateEvent(NULL, FALSE, FALSE, NULL);

   std::thread t1(Producer);
   std::thread t2(Consumer);

   t1.join();
   t2.join();

    //Event 를 사용하기 위해 발급한 Handle 을 닫는다.
    ::CloseHandle(handle);

   return 0;
}

```

이처럼 `Event` 를 사용하면 `SpinLock` 을 사용하는 것보다 효율적으로 `CPU` 를 사용할 수 있다.

## Conclusion

이벤트는 특정 이벤트가 발생했음을 알리는 신호를 운영체제가 제공하는 스레드 동기화 매커니즘이다.

이벤트는 `Signaled` 와 `Non-Signaled` 상태를 가지며, `Manual Reset` 과 `Auto Reset` 으로 나뉜다.

오래 걸릴 것이 예상되는 작업에는 `SpinLock` 을 사용하는 것보다 `Event` 를 사용하는 것이 효율적으로 `CPU` 를 사용할 수 있게 해준다.

이벤트를 사용할 때는 `Wait` 할 때마다 이벤트를 `Reset` 해주어야 한다는 점을 유의해야 한다.

`Event` 를 사용할 때는 `SpinLock` 을 사용하는 것보다 `Context Switch` 가 발생할 가능성이 높으므로 주의해야 한다.

유저 모드에서 `Kernel Object` 에 접근하는 것은 `Context Switch` 를 발생시키기 때문이다.
