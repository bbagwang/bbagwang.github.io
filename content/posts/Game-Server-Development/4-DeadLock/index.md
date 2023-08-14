---
title: "Game Server Development #4 : Dead Lock"
summary: "Dead Lock"
categories: ["Korean Post", "Programming"]
tags: ["C++", "Server", "Thread", "Lock"]
date: 2023-08-13
draft: false
showauthor: false
authors:
  - bbagwang
series: ["Game Server Development"]
series_order: 4
---

## Material
**[[C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈] Part4: 게임 서버](https://inf.run/8Chk)**

아래 글의 #Dead-Lock 섹션에 기본적인 문제를 설명해 두었었다. 먼저 읽어본다면 지금 글을 이해하는데 도움이 될 것이다.

{{< article link="/posts/game-server-development/3-lock/" >}}

## 4 Necessary Conditions for Dead Lock

**`Dead Lock` 이 발생하려면 다음의 4 가지 조건이 동시에 만족되어야 한다.**

## 상호 배제 (Mutual Exclusion)

한 번에 한 프로세스만 공유 자원을 사용할 수 있다.

## 점유 및 대기 (Hold and Wait)

프로세스가 최소한 하나의 자원을 점유하고 있으면서, 다른 프로세스에 의해 점유 중인 자원을 추가로 기다린다.

## 비선점 (No Preemption)

자원을 점유한 프로세스는 그 자원을 스스로만 방출할 수 있다.

## 환형 대기 (Circular Wait)

두 개 이상의 프로세스나 스레드가 서로를 기다리는 사이클이 형성한다.

## Problem Example

아직 데드락을 탐지하는 방법은 배우지 않았고, 추후 작성할 `Dead Lock Detection` 글에서 문제 해결 방법을 이어 설명할 예정이다.

이번 글에서는 데드락이 일어나는 또 다른 경우와 그 해결법에 대해 설명한다.

`Account` 라는 정보를 관린하는 `Account Manager` 클래스가 존재하고, `User` 라는 정보를 관리하는 `User Manager` 가 존재한다고 가정해보자.

`~Manager` 들은 `Singleton` 으로 구현되어 있으며, MMORPG 게임의 유저와 계정 정보를 관리하는 클래스이다.

코드로 작성해보면 다음과 같다.

**Account Manager**
```cpp
class Account
{
	//TEMP
}

class AccountManager
{
public:
	static AccountManager* Instance()
	{
		static AccountManager instance;
		return &instance;
	}
	
	Account* GetAccount(int32 id)
	{
		return nullptr;
	}
	
	void ProcessLogin();
	
private:
	mutex _mutex;
	//map<int32, Account*> _accounts;
}
```

**User Manager**
```cpp
class User
{
	//TEMP
}

public:
	static UserManager* Instance()
	{
		static UserManager instance;
		return &instance;
	}
	
	User* GetUser(int32 id)
	{
		return nullptr;
	}
	
	void ProcessSave();
	
private:
	mutex _mutex;
	//map<int32, User*> _users;
}
```

이처럼 각각의 매니저는 `mutex` 를 갖고있고, `ProcessLogin` 과 `ProcessSave` 라는 기능이 있다.

여기서 `AccountManager` 의 `ProcessLogin` 함수에 다음과 같은 로직이 구현된다.

```cpp
void AccountManager::ProcessLogin()
{
	//AccountManager 의 mutex 획득 시도
	std::lock_guard<mutex> guard(_mutex);
	
	//UserManager 의 mutex 획득 시도
	User* user = UserManager::Instance()->GetUser(100);
	
	//Something...
}
```

`AccountManager` 의 `mutex` 를 잡은 다음에, `UserManager` 의 `mutex` 또한 획득하려 한다.

그런데 이제 필요에 의해 `UserManager` 의 `ProcessSave` 또한 다음과 같은 로직이 구현된다.

```cpp
void UserManager::ProcessSave()
{
	//UserManager 의 mutex 획득 시도
	std::lock_guard<mutex> guard(_mutex);
	
	//AccountManager 의 mutex 획득 시도
	User* user = AccountManager::Instance()->GetAccount(100);
	
	//Something...
}
```

`UserManager` 의 `mutex` 를 잡은 다음에, `AccountManager` 의 `mutex` 를 획득하려 한다.

이제 다음과 같이 여러번 호출하는 상황을 만들어 문제가 없는지 확인해보자.

```cpp
#include <iostream>
#include <thread>
#include "AccountManager.h"
#include "UserManager.h"

void Func1()
{
	for(int i = 0; i < 1000; ++i)
	{
		UserManager::Instance()->ProcessSave();
	}	
}

void Func2()
{
	for(int i = 0; i < 1000; ++i)
	{
		AccountManager::Instance()->ProcessLogin();
	}
}

int main()
{
	std::thread t1(Func1);
	std::thread t2(Func2);
	
	t1.join();
	t2.join();
	
	std::cout << "Jobs Done" << std::endl;
}
```

실행해보니 Jobs Done 이 출력되지 않는다.

- UserManager 는 AccountManager 의 락을 획득하려 한다.

- AccountManager 는 UserManager 의 락을 획득하려 한다.

**서로가 서로의 락이 풀리기를 기다리는 교착 상태에 빠져버렸다.**

## Solution

위와 같은 상황은 락을 잡는 순서를 맞춰주면 해결할 수 있다.

일단 `UserManager` 의 `ProcessSave` 가 다음과 같이 바뀐다면 해결된다.

```cpp
void UserManager::ProcessSave()
{
	//AccountManager 의 mutex 획득 시도
	User* user = AccountManager::Instance()->GetAccount(100);
	
	//UserManager 의 mutex 획득 시도
	std::lock_guard<mutex> guard(_mutex);
	
	//Something...
}
```

`AccountManager` 의 `ProcessLogin` 과 락을 잡는 순서를 맞춰주면 되는 것이다.

반대로 `AccountManager` 가 `UserManager` 의 `ProcessSave` 와 동일한 락 순서를 만들어도 된다.

하지만, 이건 이렇게 간단한 상황일때나 가능한 이야기고, 사용하는 스레드와 뮤텍스가 많아지면 일일히 순서를 관리하는 것은 너무 어렵다.

## Conclusion

락을 거는 순서에 유의하여 락을 걸어야 한다.

추후 락을 거는 순서에 싸이클이 발생하는지 탐지하는 알고리즘을 이용하여 이러한 문제 상황을 쉽게 확인할 수 있도록 해야한다.

## std::lock

`std::lock`은 C++11에서 도입된 기능으로, 여러 개의 `std::mutex`를 `Dead Lock` 없이 한번에 잠그기 위해 사용한다.

`std::lock`은 여러 개의 `std::mutex` 객체를 인자로 받아, 그 중 어떤 것도 잠기지 않은 상태에서 모든 뮤텍스를 한 번에 잠근다.

만약 일부 뮤텍스가 이미 잠겼다면, `std::lock` 은 필요한 모든 뮤텍스가 풀릴 때까지 기다린 다음, 그것들을 한번에 잠근다.

한 번에 잠그기 때문에 교착 상태가 일어날 가능성이 줄어든다.

```cpp
std::mutex m1, m2;

std::lock(m1, m2); // 두 뮤텍스를 한번에 잠금 (잠기는 순서는 상관없음)
```

## std::adpot_lock

`std::lock` 을 이용해 한 번에 여러개의 `mutex` 를 획득 했다고 치자.

`std::lock` 공유 자원을 건드는 일을 마친 후 다시 락을 반환해 주어야 한다.

이미 `std::lock_guard` 를 통해 `RAII` 패턴으로 함수를 벗어나면 락을 풀어줄 수 있는 방법이 있다.

하지만, `std::lock_guard` 는 인자로 넘어온 `mutex` 를 즉시 잠구려고 시도한다.

이미 잠긴 `mutex` 를 다시 잠구려고 시도하면 또다시 Dead Lock 이 발생한다.

잠긴 `mutex` 를 풀어주기 전까진 다시 잠굴 수 없기 때문이다. (이를 위해 `std::recursive_lock` 이 있긴 한데 지금은 몰라도 된다.)

아래의 코드와 같은 상황이라고 보면 된다.

```cpp
#include <thread>
#include <mutex>

std::mutex mtx;

void DoubleLock()
{
	mtx.lock();
	mtx.lock();

	mtx.unlock();
	mtx.unlock();
}

int main()
{
	std::thread t1(DoubleLock);

	t1.join();

	return 0;
}
```

이럴때 `std::adopt_lock` 을 사용하면 문제를 해결할 수 있다.

**`std::adopt_lock` 은 인자로 넘어온 뮤텍스가 이미 잠겨 있음을 인지하고, 추가로 잠그지 말고 나중에 소멸될 때 락을 풀어주기만 해라 라는 의미이다.**

```cpp
std::mutex mtx1, mtx2;

void MultiMutex() {
    std::lock(mtx1, mtx2);  // mtx1과 mtx2를 동시에 잠근다.

    // 아래의 lock_guard는 mtx1과 mtx2가 이미 잠겼음을 "인지"하고, 추가로 잠그려고 시도하지 않는다.
    std::lock_guard<std::mutex> lock1(mtx1, std::adopt_lock);
    std::lock_guard<std::mutex> lock2(mtx2, std::adopt_lock);

    // ... 임의의 (공유 자원에 대한) 작업 ...

    // lock1과 lock2가 소멸되면 mtx1과 mtx2는 자동으로 해제된다.
}
```
