---
title: "Game Server Development #2 : Race Condtion, Atomic"
summary: "Race Condtion, Atomic"
categories: ["Korean_Post", "Programming"]
tags: ["C++", "Server", "Thread", "Atomic"]
date: 2023-08-11
draft: false
showauthor: false
authors:
  - bbagwang
series: ["Game Server Development"]
series\_order: 2
---

## Material
**[[C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈] Part4: 게임 서버](https://inf.run/8Chk)**

## Race Condition

먼저 아래의 코드를 보자.

```cpp
#include <iostream>

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
		sum++;
	}
}

int main()
{
	Add();
	Sub();

	std::cout << sum << std::endl;

	return 0;
}
```

이 코드의 결과 값은 `0` 이다.

스레드를 이용해도 동일한지 확인 해보자.

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

**이 코드의 결과 값은 `0` 이 아니다.**

이유를 알아보자

`Add` 함수에서 일어나는 `sum++` 을 디스어셈블리를 통해 본다면 다음과 같은 CPU 인스트럭션이 실행된다.

```
00007FF64BAC2685  mov         eax,dword ptr [sum (07FF64BAD0444h)]  
00007FF64BAC268B  inc         eax  
00007FF64BAC268D  mov         dword ptr [sum (07FF64BAD0444h)],eax 
```

3줄의 어셈블리 코드가 실행되는데, 해석해보면 다음과 같다.

1. `sum` 의 값을 `eax` 레지스터에 저장한다.
2. `eax` 레지스터의 값을 `1` 증가시킨다.
3. `eax` 레지스터의 값을 다시 `sum` 에 저장한다.

(`Sub` 함수에서도 값을 감소시키는 것만 다를 뿐, 동일한 과정을 거친다.)

위 내용을 C++ 코드로 표현하면 다음과 같다.

```cpp
int eax = sum;
eax = eax + 1;
sum = eax;
```

우리가 보는 한 줄의 C++ 코드는 컴파일을 거친 후, CPU 에서 실행되기 위한 3 줄의 어셈블리 코드로 변환되었다.

CPU 에서는 메모리에서 어떤 값을 레지스터에 꺼내오고, 연산을 수행하고, 다시 메모리에 값을 저장하는 과정들을 **각각 진행시키기 때문이다.**

이제 다시 본론으로 돌아와서 문제가 되는 상황을 보자.


`t1` 과 `t2` 스레드는 동시에 실행되기 때문에 전역 변수 `sum` 을 동시에 접근하게 된다. 그리고 **어떤 스레드가 먼저 실행될지, 언제 실행될지도 알 수 없다.**

1. `t1` 스레드에서 `eax` 레지스터에 `sum` 의 값을 저장하고 `eax` 의 값을 올린다. (`eax = 1`, `sum = 0`)
2. `t2` 스레드에서 `eax` 레지스터에 `sum` 의 값을 저장하고 `eax` 의 값을 내린다. (`eax = -1`, `sum = 0`)
3. `t1` 스레드에서 Context Switch 가 발생하여, `eax` 의 값을 `sum` 에 넣는 명령을 실행하기 직전인 상태로 대기한다.
4. `t2` 스레드는 `eax` 의 값을 `sum` 에 넣는 명령을 실행한다. (`eax = -1`, `sum = -1`)
5. `t1` 스레드가 다시 CPU Time 을 획득하여, `eax` 의 값을 `sum` 에 넣는다. (`eax = 1`, `sum = 1`)
6. **`sum` 의 값은 `0` 이 아닌 `1` 이 된다.**

결국 t2 의 연산이 무시되고, t1 의 연산만 반영된 것이다.

**이처럼 여러 스레드가 공유 자원을 동시에 변경할 때, 명령 순서에 따라 결과 값이 의도와 달라질 수 있는 상태를 경쟁 상태 (Race Condition) 라고 한다.**

## Synchronization

위와 같은 경쟁 상태를 해결하기 위해선, **공유 자원에 대한 접근 순서를 보장** 해야 한다.

접근 순서를 보장하는 것을 **동기화 (Synchronization)** 라고 한다.

동기화에는 Mutex, Condition Variable, Atomic Operation 등등 여러가지 기법들이 존재한다.

그 중에서도 가장 직관적인 Atomic Operation 을 이용해 지금의 경쟁 상태를 해결해보자.

## Atomic Operation

Atom(원자) 는 더 이상 쪼갤 수 없는 단위를 의미한다.

비슷한 맥락으로 Atomic Operation(원자적 연산) 은 **다른 스레드 간섭 없이, 연산이 한 번에 완료됨을 보장하는 연산** 을 의미한다.

조금 더 쉽게 말해보면 전부 실행이 되거나, 전혀 실행되지 않거나 둘 중 하나의 상태만 가능한 All or Nothing 연산이라고 할 수 있다.

즉, 연산이 완료되기 전까지는 다른 스레드가 접근할 수 없다. (연산을 쪼갤 수 없다)

위에서 문제가 되었던 `sum++` 가 한 번에 이뤄짐을 보장해 주었다면, Race Condition 은 발생하지 않았을 것이다.

그럼 아까의 문제 상황을 해결하기 위해 코드를 원자적 연산으로 바꿔보자.

C++ 에서는 표준으로 원자적 연산을 지원한다. `atomic` 헤더를 추가하고, `std::atomic` 템플릿을 이용하면 된다.

`sum` 의 타입을 `atomic<int>` 로 변경해보자.

```cpp
#include <iostream>
#include <thread>
#include <atomic>

std::atomic<int> sum = 0;

void Add()
{
	for (int i = 0; i < 100'0000; ++i)
	{
		sum.fetch_add(1);
	}
}

void Sub()
{
	for (int i = 0; i < 100'0000; ++i)
	{
		sum.fetch_add(-1);
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

**이 코드의 결과 값은 `0` 이다.**

`Add` 함수의 `fetch_add` 함수는 원자적으로 동작하기 때문에, `sum` 의 값을 읽어오고, `1` 을 더한 후, 다시 `sum` 에 저장하는 과정이 한 번에 이뤄진다.

만약, `Add` 함수의 `sum` 변수에 대한 연산이 진행중인 상태에서 `Sub` 함수가 `sum` 변수에 접근하려고 하면, `Sub` 함수는 `Add` 함수의 연산이 완료될 때까지 대기하게 된다.

이렇게 대기 시키는 것은 CPU 명령으로 구현되어 있어서, CPU 단에서 원자성을 보장해준다.

그럼 Atomic 을 모든 곳에 때려 박으면 모든 문제가 해결되는 것일까?

아쉽게도 그렇지 않다.

Atomic 은 퍼포먼스를 떨어뜨리는 요소이기 때문에, 필요한 부분에만 사용해야 한다.
