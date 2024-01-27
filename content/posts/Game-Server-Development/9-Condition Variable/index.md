---
title: "Game Server Development #9 : Condition Variable"
summary: "Condition Variable"
categories: ["Korean Post", "Programming"]
tags: ["C++", "Server", "Thread", "Condition Variable"]
date: 2023-11-10
draft: false
showauthor: false
authors:
  - bbagwang
series: ["Game Server Development"]
series_order: 9
---

## Material
**[[C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈] Part4: 게임 서버](https://inf.run/8Chk)**

## Problem of Event

{{< article link="/posts/game-server-development/8-event/" >}}

위 글에서 Windows 플렛폼에서 어떻게 `Event` 가 동작하는지에 대해서 간단히 알아봤었다.

하지만, 이벤트에는 몇 가지 문제점이 있다.

### Non Standard

이벤트는 `Windows` 플렛폼에서만 사용하는 비표준 내용이다.

`Linux` 같은 다른 플렛폼에서는 `Semaphore` 같은 것을 사용해야 한다.

이러한 표준적이지 않은 기능을 사용하면, 플렛폼에 종속적인 코드가 되어버린다.

### Overkilling by using Kernel Objects

이벤트는 `Kernel` 에서 동작한다.

이벤트를 사용하는 것은 잦은 `Context Switch` 를 발생시킨다.

`Kernel Object` 를 조작하는 명령을 수행하기 위해서는, `User Mode` 에서 `Kernel Mode` 로 전환해야하기 때문이다.

`CV` 의 경우 `User Mode` 에서 동작하기 때문에, `Context Switch` 가 발생할 가능성이 적다.

이러한 이벤트의 문제 상황을 해결하기 위해 C++11 부터 표준화된 `Condition Variable` 을 사용할 수 있다.

## Condition Variable

`Condition Variable`(이하 `CV`) 은 `Event` 와 비슷한 기능을 제공한다.

`CV` 는 `Wait` 을 하다가 `Notify` 가 울리면, 신호를 수신한 `CV` 는 `Wait` 을 벗어나기 위한 `Condition` (조건) 을 확인한다.

조건은 2가지 이다.

1. `Lock` 을 획득했는가?
1. `Condition` 이 만족되었는가? (Predicate 를 사용하는 경우에만 체크한다)

### If Condition is Satisfied

`Lock` 을 획득했고, `Condition` 이 만족되었다면 대기 상태를 벗어나서 다음 줄로 진행한다.

### If Condition is NOT Satisfied

`Lock` 을 획득하지 못한 경우, 다시 대기 상태로 돌아간다.

`Lock` 을 획득했으나, `Condtion` 이 만족되지 않았다면, `Lock` 을 풀고 대기 상태로 돌아간다.

`CV` 는 `Unique Lock` 을 사용한다.

### Why Using Unique Lock?

{{< article link="/posts/game-server-development/3-lock/" >}}

위 글에서 알아봤듯, `Unique Lock` 을 사용하면, 명시적인 `lock` 함수 호출 이전까지, 락을 획득하지 않은 상태로 대기할 수 있다.

또한, Scope 가 끝나기 전이라도 `unlock` 함수를 호출해 락을 해제할 수 있다.

조건을 만족하지 못할 경우 락을 풀어주고 다시 대기해야 하기 때문에, `Unique Lock` 을 사용한다고 볼 수 있다.

## Spruious Wakeup

이전 예제를 다시 생각해보자.

`Consumer` 가 이벤트를 받아 `Wait` 에서 벗어났지만, `Producer` 가 찰나의 순간에 먼저 다시 락을 잡고, 데이터를 넣었을 수도 있다.

만약 이러한 상태가 계속 지속된다면, `Consumer` 는 락을 획득하지 못해, `Queue` 에 접근할 수 없게 된다.

또한, `Queue` 에 데이터가 지속적으로 누적되어 메모리가 부족해질 수도 있다.

**이러한 문제의 이유는 `Event` 와 `Lock` 이 개별적으로 동작하기 때문이다.**

**조건이 만족되지 않은 상태에서 대기중인 스레드가 일어나 버리는 것을 `Spuirous Wakeup (가짜 기상)` 이라고 한다.**

`Event` 나 `Notify` 등을 받거나 운영체제의 스케쥴러가 깨워서 다음 작업을 진행하려 스레드가 일어났지만, 원하던 조건이 만족되지 않은 상태를 의미한다.

이러한 가짜 기상은 2 가지 정도의 문제로 바라볼 수 있다.

### Incorrect State Progression

실제로 처리해야 할 데이터나 상태가 준비되지 않았음에도 불구하고 작업을 진행하려 시도할 수 있다.

이러한 동작은 데이터 손상이나 논리적 오류를 초래할 수도 있다.

### Wasting System Resources

스레드가 필요 없이 깨어나 작업을 시도하면, 시스템 리소스가 낭비될 수 있다.

그러므로 최대한 `Spuirous Wakeup` 을 방지할 수 있는 로직을 작성해야 한다.

대부분은 `Predicate` 와 같은 조건을 통과하는지 체크하는 것으로 해결할 수 있다.

## Condition Variable Functions

`CV` 를 사용하기 위해서는 `mutex` 헤더를 추가해야 한다.

만약 좀 더 일반화된 상황에서 직접 `Lock` 을 구현해 사용하는 경우라면 `std::condition_variable_any` 를 사용하기 위해 `condition_variable` 헤더를 추가해 사용하면 된다.

`CV` 가 표준으로 제공하는 함수들은 다음과 같다.

## Waiting

대기를 위해 사용되는 함수 시리즈이다.

### wait
`wait` 은 그냥 대기하는 버전과 `Predicate` 를 사용하는 버전이 있다.

```cpp

void wait( std::unique_lock<std::mutex>& lock );

template< class Predicate >
void wait( std::unique_lock<std::mutex>& lock, Predicate pred );

```

### wait_for

`CV` 가 깨어나거나, 지정된 `시간 (Timeout Duration)` 이 지나면 현재 스레드를 대기시킨다.

### wait_until

`CV` 가 깨어나거나, 지정된 `시점 (Time Point)` 에 도달할 때까지 현재 스레드를 대기시킨다.

## Notifing

대기 상태를 깨우는(Wake Up) 방법에 대한 함수 시리즈이다.

### notify_one

`CV` 가 대기 중인 스레드 중 하나를 깨운다.

### notify_all

`CV` 가 대기 중인 모든 스레드를 깨운다.

## Condition Variable Example

`CV` 를 사용하여 위에서 작성했던 `Event` 의 예제 코드를 다시 작성해보자.

```cpp

#include <iostream>
#include <thread>
#include <mutex>
#include <queue>

std::mutex m;
std::queue<int> q;
std::condition_variable cv;

void Producer()
{
    while(true)
    {
        {
            std::unique_lock<std::mutex> lock(m);
            q.push(100);
        }

        cv.notify_one();
    }
}

void Consumer()
{
    while(true)
    {
        std::unique_lock<std::mutex> lock(m);

        //Condition Variable 이 조건을 만족할 때 까지 대기한다.
        //1. lock 을 획득하려고 시도한다.
        //2. q 가 비어있지 않은지 체크한다.
        //둘 다 만족해야 wait 을 벗어날 수 있다.
        //조건을 만족하지 못한다면, lock 을 풀고 다시 대기한다.
        cv.wait(lock, []() { return !q.empty(); });
        
        int data = q.front();
        q.pop();

        //Lock 을 잡고, 콘솔 출력을 하는 것은 좋은 습관은 아님.
        std::cout<<data<<std::endl;
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

## Conclusion

표준으로 제공되는 `Condition Variable` 을 사용하면, 플렛폼에 종속적이지 않은 코드를 작성할 수 있다.

`Event` 와 `Lock` 이 개별적으로 동작하는 문제점을 해결할 수 있다.

`Spuirous Wakeup` 을 방지하기 위해 `Predicate` 조건을 정의하는 등의 추가적인 작업이 필요하다.

## Reference

- [Condition Variable](https://en.cppreference.com/w/cpp/thread/condition_variable)
- [Spurious Wakeup](https://en.wikipedia.org/wiki/Spurious_wakeup)
