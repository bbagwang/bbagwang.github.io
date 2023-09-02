---
title: "Game Server Development #2 : Race Condtion, Atomic"
summary: "Race Condtion, Atomic"
categories: ["Korean Post", "Programming"]
tags: ["C++", "Server", "Thread", "Atomic"]
date: 2023-08-11
draft: false
showauthor: false
authors:
  - bbagwang
series: ["Game Server Development"]
series_order: 2
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
		sum--;
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

## Issue of Atomic Operation

`Atomic` 은 **퍼포먼스를 떨어뜨리는 요소**이기 때문에, 필요한 부분에만 사용해야 한다.

퍼포먼스가 떨어지는 이유들은 다음과 같다.

### Memory Barrier/Fence

메모리 배리어(Memory Fence 라고도 함) 는 프로세서가 명령어를 재배치하는 것을 제한하거나 금지하여, 특정 명령들이 원하는 순서대로 실행되도록 보장하는 것을 말한다.

이는 멀티 스레딩 환경에서 매우 중요하다.

성능 향상을 위해서 최신 CPU는 자원을 최대한 활용하기 위해 명령어를 순서 없이 실행하는 경우가 많다.

하드웨어가 명령어 무결성을 적용하기 때문에 단일 실행 스레드에서는 이를 알아차리지 못한다. 그러나, 여러 스레드가 있는 환경에서는 예측할 수 없는 동작으로 이어질 수 있다.

메모리 배리어는 메모리 읽기/쓰기가 예상한 순서대로 발생한다는 것을 의미하는 명령어다.

*메모리 배리어는 하드웨어 개념이라는 점을 유의하자.*

*CPU 의 하드웨어 단에서 이뤄지는 재정렬은 컴파일러 최적화와는 다르다.*

#### Why

메모리 베리어가 필요한 이유는, 멀티 스레딩 환경에서는 여러 스레드가 동일한 메모리에 액세스하므로, 명령어의 실행 순서가 예상과 다를 경우 데이터의 일관성이 깨질 수 있다.

예를 들어, `스레드 A`와 `스레드 B`가 있을 때, A가 변수 `x`를 1 로 설정하고 B가 그 값을 읽는 상황을 생각해보자.

메모리 배리어가 없다면, B가 `x`를 읽은 후에 A가 `x`를 1 로 설정할 수 있다.

이런 상황을 피하기 위해 메모리 배리어가 사용된다.

#### How

메모리 배리어를 설정하면, 그 앞뒤의 명령어들은 배리어를 넘어서 재배치되지 않는다.

즉, 배리어 앞의 명령어들은 배리어가 실행되기 전에 완료되고, 배리어 뒤의 명령어들은 배리어가 실행된 후에만 실행된다.

예를 들어, 아래와 같은 코드를 보자.

```cpp
x = 1;  // 스레드 A
memory_barrier();
y = 2;  // 스레드 A
```

이 경우, `y = 2`는 `x = 1`이 완료된 후에만 실행된다.

이렇게 메모리 배리어를 사용하면 명령어의 실행 순서를 보장할 수 있다.

순서를 보장하지만, 최적화가 무시될 가능성이 높으므로, 그로 인한 성능 저하가 발생할 수 있다.

#### Reference

- https://code-piggy.tistory.com/171
- https://stackoverflow.com/a/286705
- https://ko.wikipedia.org/wiki/%EB%A9%94%EB%AA%A8%EB%A6%AC_%EB%B0%B0%EB%A6%AC%EC%96%B4

### Bus Lock
버스 락은 한 스레드가 데이터에 액세스하는 동안 다른 스레드가 그 데이터를 수정하지 못하게 하는 것을 말한다.

두 스레드가 동시에 같은 메모리 위치를 수정하려고 할 때, 버스 락을 통해 한 번에 한 스레드만 수정할 수 있도록 제어 한다.

### Cache Coherency

멀티 코어 시스템에서 각 코어의 캐시는 다른 코어의 캐시와 일관성을 유지해야 한다. 이를 캐시 일관성 이라고 한다.

한 스레드가 `코어 A`의 캐시에 있는 데이터를 수정하면, 이 변경 사항은 `코어 B`의 캐시에도 반영되어야 한다.

이를 위해 추가적인 통신과 동기화가 필요하다. 이 부분에서 성능 저하가 발생한다.

#### Cache Invalidation

캐시 무효화는 한 코어에서 캐시에 저장된 특정 메모리 위치를 변경하면, 다른 코어의 캐시에서 그 위치의 데이터가 무효화되는 것을 말한다.

이렇게 되면 다른 코어는 다시 메인 메모리에서 해당 데이터를 로드해야 한다.

캐시 무효화는 메모리 액세스 시간에 큰 영향을 미친다.

캐시에서 빠르게 데이터를 가져오는 대신, 더 느린 메인 메모리(`RAM`)에서 데이터를 가져와야 해서, 성능에 영향을 줄 수 있다.

#### Cache Line Ping Pong

두 개 이상의 코어가 동일한 캐시 라인의 데이터를 수정할 경우, 해당 캐시 라인은 코어 간에 빠르게 이동하게 된다.

이를 `핑퐁(Ping Pong)` 이라고 한다.

캐시 라인 핑퐁은 데이터를 캐시에 유지하기 어렵게 만들어, 빈번한 캐시 무효화와 메인 메모리 액세스를 유발한다.

이로 인해 성능이 크게 저하될 수 있다.

### Retring

일부 원자적 연산은 성공할 때까지 여러 번 시도될 수 있다.

#### Compare and Swap

`Compare and Swap (CAS)` 연산은 원래의 값과 비교하여 값이 같으면 새 값을 저장한다.

동시성 환경에서 변경이 필요한 값의 `Memory`

최종적으로 변화하길 원하는 값인 `Desired`

변화를 만들기 위한 조건으로 기대하고 있는 값인 `Expected`

위 3 가지 요소들을 가지고 비교와 변경을 진행한다.

또한, 원래의 값(`Memory`) 이 다른 스레드에 의해 도중에 변경되면 연산은 실패하고 재시도 된다.

`CAS` 를 C++ 에서 사용할 때는 다음과 같은 함수가 사용된다.

```cpp
std::atomic_compare_exchange_strong(Memory, Expected, Desired); //weak 도 있다.
```

위의 `CAS` 를 의사 코드로 표현해보면 다음과 같다.

```cpp
if (Memory == Expected)
{
	Memory = Desired;
}
else
{
	Expected = Memory;
}
```
