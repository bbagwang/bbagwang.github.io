---
title: "Game Server Development #6 : Spin Lock"
summary: "Spin Lock"
categories: ["Korean Post", "Programming"]
tags: ["C++", "Server", "Thread", "Lock"]
date: 2023-09-08
draft: false
showauthor: false
authors:
  - bbagwang
series: ["Game Server Development"]
series_order: 6
---

## Material
**[[C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈] Part4: 게임 서버](https://inf.run/8Chk)**

## Introduction

이전 글에서 락의 구현 방식에 대해 설명하며, 스핀 락에 대한 설명도 진행 했었다.

{{< article link="/posts/game-server-development/5-lock-implementation-theory/" >}}

`Spinlock` 은 이름에서 알 수 있듯이, 자원에 접근할 수 있을 때까지 계속해서 `Spin(Loop)` 하는 락이다.

`Spinlock` 은 굉장히 빠르게 작동하므로, 락을 획득하고 해제하는 작업이 매우 빠르게 이루어질 것이라고 예상되는 상황에서 효율적이다.

반대로 락을 획득하는 데 오랜 시간이 걸리는 경우에는 CPU 자원을 불필요하게 낭비하게 된다.

락을 획득할 때 까지 존버 하는 메타라고 보면 된다.

## Scenario

아래의 코드를 보자.

```cpp
#include <iostream>
#include <thread>

int sum = 0;

void Add()
{
	for (int i = 0; i < 100'0000; ++i)
	{
		sum++;
	}
}

void Sub()
{
	for (int i = 0; i < 100'0000; ++i)
	{
		sum--;
	}
}

int main()
{
	std::thread t1(Add);
	std::thread t2(Sub);

	t1.join();
	t2.join();

	std::cout << sum << std::endl;

	return 0;
}
```

위 코드는 결과적으로 `0` 을 출력하지 못한다.

`sum` 이라는 공유 자원에 대한 동기화를 맞춰주지 않았기 때문이다.

근본적인 내용은 아래 `Atomic` 글에서 설명했었으니, 기억이 안난다면 한 번 다시 보면 좋을 것이다.

{{< article link="/posts/game-server-development/2-atomic/" >}}

그럼 이제 위 코드의 결과로 `0` 이 나올 수 있도록, 스핀락을 직접 구현하고 사용하여 동기화를 맞춰보자.

## Spin Lock Implementation for Dummies

스핀락은 다시 쉽게 설명하면 뺑뺑이를 돌면서 계속해서 락을 획득하려고 시도하는 방식이다.

아무런 배경 지식 없이, 스핀락에 대해 아는 것은 위에 내용이 전부라고 생각해보자.

그 상태에서는 일단 아래와 같이 스핀락을 구현할 수 있을 것이다.

```cpp
class SpinLock
{
public:
	void lock()
	{
		while (_locked)
		{

		}

		_locked = true;
	}

	void unlock()
	{
		_locked = false;
	}

private:
	bool _locked = false;
};
```

lock 함수에서는 `_locked` 가 `false` 일 때까지 계속해서 뺑뺑이를 돌면서 기다린다.

`_locked` 가 `false` 가 되면, `_locked` 를 `true` 로 바꿔주며, 락을 획득한다.

unlock 함수에서는 `_locked` 를 `false` 로 바꿔주며, 락을 해제한다.

이렇게 구현하면, 위의 코드에서 `sum` 에 대한 동기화를 맞춰줄 수 있을까?

코드로 확인해보자.
  
```cpp
#include <iostream>
#include <thread>
#include <mutex>

class SpinLock
{
public:
	void lock()
	{
		while (_locked)
		{

		}

		_locked = true;
	}

	void unlock()
	{
		_locked = false;
	}

private:
	bool _locked = false;
};

int sum = 0;
SpinLock spinLock;

void Add()
{
	for (int i = 0; i < 100'0000; ++i)
	{
		std::lock_guard<SpinLock> guard(spinLock);
		sum++;
	}
}

void Sub()
{
	for (int i = 0; i < 100'0000; ++i)
	{
		std::lock_guard<SpinLock> guard(spinLock);
		sum--;
	}
}

int main()
{
	std::thread t1(Add);
	std::thread t2(Sub);

	t1.join();
	t2.join();

	std::cout << sum << std::endl;

	return 0;
}
```

위 코드의 실행 결과는 `1843` 이었다. (매번 실행할 때마다 결과가 다르다.)

결국 현재의 SpinLock 구현은 `sum` 에 대한 동기화를 맞춰주지 못했다는 것을 알 수 있다.

일단은 위 코드는 여러가지 문제가 있는데 그 중 하나는 `컴파일러 최적화` 이다.

### C++ Volatile

C++ 에서는 `volatile` 이라는 키워드가 있다.

컴파일러에게 `volatile` 로 선언된 이 변수는 최적화 하지 말라고 알려주는 역할을 한다.

{{< alert >}}
**주의!** C++ 에서의 `volatile` 은 C#, Java 와 같은 언어에서의 `volatile` 과 다르다.
([관련 문서](https://stackoverflow.com/questions/2484980/why-is-volatile-not-considered-useful-in-multithreaded-c-or-c-programming))
{{< /alert >}}

예를 들면 아래와 같은 간단한 코드를 보자.

```cpp
int main()
{
	int num = 0;
  num = 1;
  num = 2;
  num = 3;
  num = 4;
	
	return 0;
}
```

이 코드는 결론적으로 `num` 의 값이 `4` 가 될 것이다.

하지만, 중간에 1, 2, 3 을 거치는 과정이 존재한다.

이 부분들은 컴파일러에 의해 최적화 과정에서 제거될 수 있다.

디스어셈블리를 확인해보면 다음과 같다.

```cpp
int main()
{
00007FF727F31000  sub         rsp,28h  
	int num = 0;
	num = 1;
	num = 2;
	num = 3;
	num = 4;

	std::cout << num << std::endl;
00007FF727F31004  mov         rcx,qword ptr [__imp_std::cout (07FF727F320A8h)]  
00007FF727F3100B  mov         edx,4  
...
}
```

디스어셈블리로 확인해보니, `num` 에 곧바로 `4` 를 대입 해버린다.

```cpp
00007FF727F31004  mov         rcx,qword ptr [__imp_std::cout (07FF727F320A8h)]  
00007FF727F3100B  mov         edx,4  
```

이처럼 컴파일러는 최적화 과정에서, 필요 없다고 판단되는 `num` 에 대한 중간 과정을 제거할 수 있다.

하지만, 어떠한 이유로 이러한 최적화를 원치 않는 경우도 있을 것이다.

그때 `volatile` 키워드를 사용할 수 있다.

`num` 에 `volatile` 을 붙여 다시 확인해보자.

```cpp
int main()
{
	volatile int num = 0;
  num = 1;
  num = 2;
  num = 3;
  num = 4;
	
	return 0;
}
```

위 코드의 디스어셈블리 결과는 다음과 같다.

```cpp
int main()
{
00007FF633B81000  sub         rsp,28h  
	volatile int num = 0;
00007FF633B81004  mov         dword ptr [rsp+30h],0  
	num = 1;
00007FF633B8100C  mov         dword ptr [num],1  
	num = 2;
00007FF633B81014  mov         dword ptr [num],2  
	num = 3;
00007FF633B8101C  mov         dword ptr [num],3  
	num = 4;
00007FF633B81024  mov         dword ptr [num],4  

	std::cout << num << std::endl;
...
}
```

`num` 에 `volatile` 을 붙여주니, 컴파일러가 최적화를 하지 않고, 사실상 쓸대없는 중간 과정을 그대로 실행하게 되었다.

비슷한 맥락으로 `SpinLock` 클래스에서 잠김 여부를 확인하기 위해 사용한 `_locked` 변수 또한, 여러 스레드에서 확인 및 수정이 일어나는 상황이 있을 것이다.

하지만 `volatile` 키워드가 없다면, 최적화를 통해 매번 수행되어야 하는 상태 체크가 무시될 수도 있다.

그렇기 때문에, `_locked` 변수에 `volatile` 을 붙여주어 컴파일러가 최적화를 하지 않도록 해주어야 한다.

하지만, 지금 상황에서는 그것 만이 문제는 아니기 때문에, 일단 교양 차원에서 여기까지만 알아두도록 하자.

### Problems

현재의 `SpinLock` 은 여러 문제가 있지만, 가장 중요한 문제로 원자성이 보장되지 않는다는 점이 있다.

`_locked` 변수에 대한 접근은 원자적으로 이루어져야 한다.

2개 이상의 스레드가 `_locked` 를 `false` 로 보고, 동시에 `_locked` 를 `true` 로 바꾸려고 시도한다면, 두 스레드 모두 `_locked` 를 `true` 로 바꾸게 될 것이다.

이렇게 되면, 락을 제대로 획득한 것이 아닌 상태에서 락을 획득한 것으로 인식하게 되어, 문제가 발생할 수 있다.

예를 들어보자면 화장실에 두명이 달려가서 동시에 문을 잠궈버려 두 명이 큰일을 보는...?! 상황과 같다.

이러한 불상사는 일어나면 안된다. 이제 문제를 해결해보자.

## Compare And Swap

`Compare And Swap (CAS)` 연산은 원래의 값과 비교하여 값이 같으면 새 값을 저장한다.

동시성 환경에서 변경이 필요한 값의 `Memory`

최종적으로 변화하길 원하는 값인 `Desired`

변화를 만들기 위한 조건으로 기대하고 있는 값인 `Expected`

위 3 가지 요소들을 가지고 비교와 변경을 **한 번에** 진행한다.

또한, 원래의 값(`Memory`) 이 다른 스레드에 의해 도중에 변경되면 연산은 실패하고 재시도 된다.

`CAS` 를 C++ 에서 사용할 때는 다음과 같은 함수가 사용된다.

```cpp
template< class T >
bool atomic_compare_exchange_strong(
std::atomic<T>* obj,
typename std::atomic<T>::value_type* expected,
typename std::atomic<T>::value_type desired ) noexcept;

std::atomic_compare_exchange_strong(Memory, Expected, Desired); //weak 도 있다.
```

위의 `CAS` 를 의사 코드로 표현해보면 다음과 같다.

```cpp
if (Memory == Expected)
{
  Expected = Memory;
	Memory = Desired;
  return true;
}
else
{
	Expected = Memory;
  return false;
}
```

## Spin Lock Implementation

이제 `SpinLock` 을 `CAS` 를 이용하여 구현해보자.

```cpp
#include <atomic>

class SpinLock
{
public:
	void lock()
	{
    bool expected = false;
    bool desired = true;

		// CAS (Compare-And-Swap)
    while (_locked.compare_exchange_strong(expected, desired) == false)
		{
      expected = false;
		}
	}

	void unlock()
	{
		_locked.store(false);
	}

private:
	std::atomic<bool> _locked = false;
};
```

가장 먼저 `_locked` 를 `std::atomic<bool>` 로 선언해주었다.

lock 함수에서 `_locked` 가 `false` 가 되기를 **예상하면서(Expected)** `while` 문을 돌며 계속 확인한다. (Spin 한다.)

예상대로 `_locked` 가 `false` 가 되면, **원하던 대로(Desired)** `_locked` 를 `true` 로 바꿔주며, 락을 획득한다.

하지만, `_locked` 가 `true` 인 상태라면, CAS 의 행동 방식대로 `expected` 에 현재의 `_locked` 값인 `true` 가 저장된다.

그 상태로 다음 체크에 들어가게 되면, `expected` 는 `true` 가 되어있기 때문에, `while` 문을 빠져나오게 된다.

이러한 문제를 막기 위해 시도를 실패했을 때, `expected` 를 다시 `false` 로 바꿔준다.

이처럼 `CAS` 를 이용하여 원자적으로 `_locked` 를 변경해 동시에 여러 스레드가 `_locked` 를 `true` 로 바꾸는 문제를 해결할 수 있게 되었다.

아래의 코드로 확인해볼 수 있다.

```cpp
#include <iostream>
#include <thread>
#include <mutex>
#include <atomic>

class SpinLock
{
public:
	void lock()
	{
    bool expected = false;
    bool desired = true;

		// CAS (Compare-And-Swap)
    while (_locked.compare_exchange_strong(expected, desired) == false)
		{
      expected = false;
		}
	}

	void unlock()
	{
		_locked.store(false);
	}

private:
	std::atomic<bool> _locked = false;
};

int sum = 0;
SpinLock spinLock;

void Add()
{
	for (int i = 0; i < 100'0000; ++i)
	{
		std::lock_guard<SpinLock> guard(spinLock);
		sum++;
	}
}

void Sub()
{
	for (int i = 0; i < 100'0000; ++i)
	{
		std::lock_guard<SpinLock> guard(spinLock);
		sum--;
	}
}

int main()
{
	std::thread t1(Add);
	std::thread t2(Sub);

	t1.join();
	t2.join();

	std::cout << sum << std::endl;

	return 0;
}
```

실행 결과는 몇번을 실행하던지 `0` 이 나왔다.

동기화가 잘 이루어졌음을 확인할 수 있다.
