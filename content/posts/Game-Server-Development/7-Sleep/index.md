---
title: "Game Server Development #7 : Sleep"
summary: "Sleep"
categories: ["Korean Post", "Programming"]
tags: ["C++", "Server", "Thread", "Lock"]
date: 2023-10-08
draft: false
showauthor: false
authors:
  - bbagwang
series: ["Game Server Development"]
series_order: 7
---

## Material
**[[C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈] Part4: 게임 서버](https://inf.run/8Chk)**

## Introduction

이전 글에서 `while` 문 내부에서 `CAS` 체크를 통해 `SpinLock` 을 구현하는 방법에 대해서 알아봤었다.

{{< article link="/posts/game-server-development/6-spinlock/" >}}

`while` 문을 통해 지속적으로 락을 획득하려고 시도하는 방식에는 CPU 자원을 낭비하는 문제가 있다.

이를 어느정도 막기 위해 `Sleep` 을 사용하여 현재 스레드의 동작을 잠시 멈추게 할 수 있다.

멈춘 스레드는 일정 시간이 지나면 다시 동작하게 되며, 멈춰있는 동안 CPU 를 사용하지 않기 때문에 CPU 자원을 낭비하지 않게 된다.

## Sleep

`sleep` 은 특정 쓰레드 또는 프로세스를 지정된 시간 동안 말 그대로 잠재운다. (일시적으로 중단한다)

호출되면, 해당 쓰레드는 설정된 시간 동안 일을 중단하고, 그 동안 CPU는 다른 스레드의 작업을 수행한다.

주로 정확한 시간 동안 쓰레드를 중지하기 위해 사용된다.

C++ 11 부터 `std::this_thread::sleep_for` 를 통해 스레드를 잠시 멈출 수 있다.

```cpp
#include <iostream>
#include <thread>
#include <chrono>

int main()
{
	std::cout << "Start" << std::endl;

	std::this_thread::sleep_for(std::chrono::seconds(1));

	std::cout << "End" << std::endl;

	return 0;
}
```

위 코드는 `Start` 를 출력하고, 1초 동안 멈춘 뒤 `End` 를 출력한다.

## Yield

`yield` 는 현재 실행 중인 쓰레드가 실행을 양보하고, 동일한 우선순위를 가진 다른 쓰레드에게 실행 기회를 제공한다.

호출할 경우, 현재 쓰레드는 준비 상태로 전환되고, 스케줄러는 동일한 우선순위의 다른 쓰레드를 실행한다.

만약 동일한 우선순위의 쓰레드가 없다면, `yield` 를 호출한 쓰레드는 계속해서 실행을 이어간다.

C++ 11 부터 `std::this_thread::yield` 를 통해 현재 스레드를 일단 멈추고, 다른 스레드에게 CPU 를 양보할 수 있다.

```cpp

#include <iostream>
#include <thread>
#include <chrono>

int main()
{
	std::cout << "Start" << std::endl;

	std::this_thread::yield();

	std::cout << "End" << std::endl;

	return 0;
}
```

위 코드는 `Start` 를 출력하고, 다른 스레드에게 CPU 를 양보한 뒤, 다시 실행하게 된다면 `End` 를 출력한다.

## Time Slice

`Time Slice` 는 CPU 가 스레드에게 할당하는 시간을 의미한다.

스레드는 `Time Slice` 를 모두 소진하면, 다른 스레드에게 CPU 를 양보하게 된다.

`Time Slice` 는 운영체제에 의해 관리되며, `Time Slice` 가 끝나기 전에 스레드가 끝나게 되면, 다른 스레드에게 CPU 를 양보하지 않고 계속해서 CPU 를 사용하게 된다.

## Spint Lock (Sleep)

이전에 작성했던 코드에서 `SpinLock` 을 사용할 때, `while` 문 내부에서 `Sleep` 을 사용하여 CPU 자원을 낭비하지 않도록 할 수 있다.

```cpp
#include <iostream>
#include <thread>
#include <mutex>
#include <atomic>
#include <chrono>

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

			// 1. chrono 라이브러리를 사용하여 100ms 동안 스레드를 멈춘다.
      		std::this_thread::sleep_for(std::chrono::milliseconds(100));
			
			// 2. 아래처럼 ms 를 사용하여 100ms 동안 스레드를 멈출수도 있다. 결론적으로 위와 같은 코드다.
			std::this_thread::sleep_for(100ms);

			// 3. yield 를 사용하여 현재 스레드를 일단 멈추고 다음 스케쥴을 기다릴 수 있다. (언제 다시 돌아올지 모름)
			std::this_thread::yield();
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