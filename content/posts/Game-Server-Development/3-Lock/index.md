---
title: "Game Server Development #3 : Lock"
summary: "Lock"
categories: ["Korean Post", "Programming"]
tags: ["C++", "Server", "Thread", "Lock"]
date: 2023-08-12
draft: false
showauthor: false
authors:
  - bbagwang
series: ["Game Server Development"]
series_order: 3
---

## Material
**[[C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈] Part4: 게임 서버](https://inf.run/8Chk)**

## What is Lock

`Lock(락)` 은 동시에 여러 스레드에 의해 접근될 수 있는 데이터나 자원인 `공유 자원`의 동시 접근을 제어하고자 할 때 사용한다.

락을 쉽게 이해하는 방법은, 실생활에서 사용하는 자물쇠를 생각해보면 된다.

예를 들어, 두 사람이 한 개의 화장실을 사용하고 싶을 때, 먼저 도착한 사람이 화장실을 사용하게 되고, 그 사람이 사용하는 동안 다른 사람은 기다려야 한다.

이때 화장실에 자물쇠를 걸어 다른 사람이 사용하지 못하게 한다고 생각하면 된다.

비슷하게, 한 스레드가 공유 자원에 접근하기 위해 락을 획득(잠금)하면, 그 스레드만 해당 자원에 접근할 수 있고, 다른 스레드들은 그 락이 해제(잠금 해제)될 때까지 기다려야 한다.

이렇게 락을 사용하면 여러 스레드가 동시에 같은 자원에 접근하는 것을 막아 데이터의 무결성을 유지할 수 있다.

## Problem

필요성을 느끼기 위해 문제가 되는 상황을 먼저 확인 해보자.

```cpp
#include <iostream>
#include <thread>
#include <vector>

std::vector<int> v;

void Push()
{
  for (int i = 0; i < 1'0000; ++i)
  {
    v.push_back(i);
  }
}

int main()
{
  std::thread t1(Push);
  std::thread t2(Push);

  t1.join();
  t2.join();

  std::cout << v.size() << std::endl;

  return 0;
}
```

가장 먼저 이 코드는 런타임 에러. 즉, 크래시가 발생한다.

이유는 다음과 같다.

`v` 의 값이 채워지며, `capacity` 를 넘는 데이터 삽입 요청이 발생하는 경우, **`v` 의 `capacity` 를 늘리기 위해 메모리를 재할당한다.**

`t1` 스레드가 요청한 삽입을 처리하기 위해 `v` 의 `capacity` 를 늘리기 위해 메모리를 재할당하고 있는 도중 `t2` 스레드가 `v` 에 데이터를 삽입한다면, `t2` 스레드가 접근하려는 메모리는 해제된 메모리이기 때문에, 재할당한 메모리에 접근할 수 없게 되어 런타임 에러가 발생한다.

또 한편으로는 동시에 두 스레드 모두가 재할당을 요청하는 문제도 생길수도 있다.

그럼 만약 `v` 가 재할당이 일어나지 않았다면 문제가 없을까?

결론부터 말하자면, 그렇지 않다.

두 스레드가 동시에 `v` 의 `size` 를 확인하고, 그 값에 1을 더한 값을 `v` 에 추가하려고 할 것이다.

이때, 두 스레드가 동시에 `size` 를 확인하면, 서로 같은 값을 반환할 것이다.

그 상태에서 두 스레드가 `size` 에 1을 더한 값을 `v` 에 추가하려고 할 것이다.

이러면 `v` 에는 1개의 값만 추가되었을 것이다.

2 가지의 다른 문제가 있지만 결국 이러한 문제를 `Race Condition(경쟁 상태)` 라고 한다.

## Mutual Exclusion

`Lock` 을 사용하여 문제를 해결해보자.

C++ 에서 락을 사용하기 위해서는 `mutex` 라는 클래스를 사용한다.

`mutex` 는 `Mutual Exclusion(상호 배제)` 라는 의미를 가지고 있다.

`mutex` 는 `lock` 과 `unlock` 이라는 직관적인 함수를 통해 사용할 수 있다.

```cpp
#include <iostream>
#include <thread>
#include <vector>
#include <mutex>

std::vector<int> v;
std::mutex m;

void Push()
{
  for (int i = 0; i < 1'0000; ++i)
  {
    m.lock();

    v.push_back(i);

    m.unlock();
  }
}

int main()
{
  std::thread t1(Push);
  std::thread t2(Push);

  t1.join();
  t2.join();

  std::cout << v.size() << std::endl;

  return 0;
}
```

1. `<mutex>` 헤더를 추가하고, `std::mutex` 객체를 생성한다.
2. 경쟁이 발생할 위험이 있는 부분에 `lock` 함수를 호출하여 락을 획득한다.
3. 락을 획득한 후, 공유 자원에 접근한다. 만약 락을 획득하지 못했다면, 그 스레드는 접근하려는 락이 해제될 때까지 대기한다.
4. 공유 자원에 접근이 끝나면 `unlock` 함수를 호출하여 락을 해제한다.

더 쉽게 보면, 자물쇠(`mutex`) 를 잠그고 (`lock`) 사용하고, 사용이 끝나면 자물쇠를 푼다 (`unlock`) 는 개념이다.

`mutex`의 `lock` 안의 구간은 싱글 스레드로 동작하는 것과 같다. 단 하나의 스레드의 접근만 허용하기 때문이다.

## Dead Lock

락을 잠궈두고 `unlock` 을 하지 않는다면, 그 락은 영원히 해제되지 않는다.

그리고 그 락이 풀리길 기다리는 다른 스레드들은 영원히 대기하는 상태에 빠지고 만다.

이와 같이 비정상적인 락의 상황 때문에 스레드들이 무한정 대기하는 상황을 `Dead Lock(데드락)` 이라고 한다.

`Dead Lock` 은 두 개 이상의 프로세스나 스레드가 서로의 자원을 기다리며 영원히 진행되지 못하는 상태를 의미한다.

다음과 같은 코드를 보자.

```cpp
#include <thread>
#include <vector>
#include <mutex>

std::vector<int> v;
std::mutex m;

void Push()
{
  for (int i = 0; i < 1'0000; ++i)
  {
    m.lock();

    v.push_back(i);

    if (i == 5000)
      break;

    m.unlock();
  }
}
```

개발을 하다보면 위처럼 다양한 분기와 처리를 진행하게 된다.

이럴때 실수로 위와같이 `i` 가 `5000` 에 도달할 경우, 락을 해제하지 않고 함수를 빠져 나가는 실수를 할 수 있다.

조건문 안에 `m.unlock()` 을 넣어 해결할 수도 있지만, 만약 저 코드가 다양한 내용을 처리하는 100줄 이상의 함수였다면, 그 조건문을 찾기도 힘들고, 또 다른 조건문을 추가할 때마다 `unlock` 을 추가해야 한다.

이는 매우 귀찮고, 번거로우며, 실수하기 딱 좋은 상황이 된다.

위처럼 단순한 예는 그나마 금방 찾을 수 있지만, 실제로는 더 복잡한 `Dead Lock` 상황이 발생할 수 있으므로 조심해야한다.

## Lock Guard

위와 같은 `Dead Lock` 상황을 미연에 방지하기 위해 `std::lock_guard` 라는 클래스를 사용할 수 있다.

`std::lock_guard` 는 `mutex` 를 생성자에서 획득하고, 소멸자에서 해제하는 클래스이다.

이렇게 하면 스코프를 벗어나는 순간 `mutex` 가 해제되기 때문에, `unlock` 을 신경쓰지 않아도 된다.

이러한 방식을 `RAII(Resource Acquisition Is Initialization)` 라고 한다.

`std::lock_guard` 를 사용하면 다음과 같이 코드를 작성할 수 있다.

```cpp
#include <thread>
#include <vector>
#include <mutex>

std::vector<int> v;
std::mutex m;

void Push()
{
  for (int i = 0; i < 1'0000; ++i)
  {
    std::lock_guard<std::mutex> lockGuard(m);

    v.push_back(i);

    if (i == 5000)
      break;
  }
}
```

명시적인 `unlock` 을 신경쓸 필요가 없게 되어 더 안전한 코드를 작성할 수 있다.

RAII 는 꽤나 간단한 개념이므로, 아래와 같이 직접 Wrapper 를 만들어 Lock Guard 를 만들어 쓸 수도 있다.

```cpp
#include <thread>
#include <vector>
#include <mutex>

std::vector<int> v;
std::mutex m;

template<typename T>
class LockGuard
{
public:
  LockGuard(T& mutex) : m_mutex(mutex)
  {
    m_mutex.lock();
  }

  ~LockGuard()
  {
    m_mutex.unlock();
  }

private:
  T* m_mutex;
}

void Push()
{
  for (int i = 0; i < 1'0000; ++i)
  {
    LockGuard<std::mutex> lockGuard(m);

    v.push_back(i);

    if (i == 5000)
      break;
  }
}
```

## Unique Lock

위에서 보았던 `std::lock_guard` 는 `mutex` 를 생성자에서 획득하고, 소멸자에서 해제하는 클래스이다.

이러한 방식은 생성하자마자 `mutex` 를 획득하기 때문에 `lock` 을 거는 시점을 제어할 수 없다는 단점이 있다.

이러한 점을 보완하기 위해 `std::unique_lock` 을 사용할 수 있다.

`std::unique_lock` 은 원한다면 락을 거는 시점을 조절할 수 있다.

`unique_lock` 을 생성할 때, `std::defer_lock` 을 옵션으로 제공하면, 바로 `mutex` 를 획득하려 하지 않는다.

명시적으로 `lock` 함수를 호출하는 순간에 `mutex` 를 획득한다.

만약 아무 옵션도 없이 그냥 `unique_lock` 을 `mutex` 를 넣어 생성하면, 곧바로 락을 획득한다.

```cpp
#include <thread>
#include <vector>
#include <mutex>

std::vector<int> v;
std::mutex m;

void Push()
{
  for (int i = 0; i < 1'0000; ++i)
  {
    std::unique_lock<std::mutex> uniqueLock(m, std::defer_lock);
    
    if(i == 0)
      continue;

    uniqueLock.lock();

    v.push_back(i);

    if (i == 5000)
      break;
  }
}
```

## Lock Guard vs Unique Lock

`std::lock_guard` 와 `std::unique_lock` 의 차이점은 다음과 같다.

**std::lock_guard**

간단하고 빠른 `mutex` 락을 위한 클래스이다.

객체를 생성할 때 `mutex`가 자동으로 잠기고, 객체가 소멸될 때 자동으로 잠금이 해제된다.

경량 클래스이며, 별도의 조작(잠금 해제/재잠금)이 필요 없을 때 사용하는 것이 좋다.

**std::unique_lock**

`std::lock_guard` 보다 더 많은 유연성을 제공한다.

`std::unique_lock`을 사용하면 임의의 위치에서 수동으로 `mutex` 를 잠글 수 있다.

더 유연하기 때문에 기능을 위한 구현이 추가되어, `std::lock_guard` 보다는 조금 더 무겁고 느린 구현이다.
