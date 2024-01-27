---
title: "Game Server Development #10 : Future, Promise, Packaged Task"
summary: "Future, Promise, Packaged Task"
categories: ["Korean Post", "Programming"]
tags: ["C++", "Server", "Thread", "Future", "Promise", "Packaged Task"]
date: 2024-01-20
draft: false
showauthor: false
authors:
  - bbagwang
series: ["Game Server Development"]
series_order: 10
---

## Material
**[[C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈] Part4: 게임 서버](https://inf.run/8Chk)**

### Synchronous

동기는 순차적으로 실행되는 것을 의미한다.

즉, A라는 작업이 끝나야 B라는 작업이 실행되는 것이다.

아래의 코드를 보자.

```cpp
int Calculate()
{
  int sum = 0;

  for (int i = 0; i < 1000000; ++i)
  {
	  sum += i;
  }

  return sum;
}

int main()
{
	int sum = Calculate();
	std::cout << sum << std::endl;
}
```

메인 스레드에서 `sum` 을 계산하는 `Calculate()`` 함수가 끝나야만, `sum` 을 출력하는 코드가 실행될 수 있다.

이것이 동기적 실행이다.

### Asynchronous

비동기는 간단하게 보면 ***동기적이지 않다는 뜻***이다.

순차적으로 실행되지 않는다는 뜻이다.

조금 더 자세히 설명하자면, A라는 작업이 끝나지 않아도 B라는 작업이 실행될 수 있다는 것이다.

아래의 코드를 보자.

```cpp
#include <iostream>
#include <chrono>
#include <thread>

int Calculate()
{
	int sum = 0;

	for (int i = 0; i < 100; ++i)
	{
		sum += i;
	}

	std::cout << sum << std::endl;

	return sum;
}

int main()
{
	std::thread t(Calculate);

	std::this_thread::sleep_for(std::chrono::milliseconds(1));

	std::cout << "Hello World" << std::endl;

	t.join();
}
```

메인 스레드에서 `Calculate()` 함수를 새로운 스레드 `t` 에서 실행한다.

그 후, 1 밀리 세컨드 동안 메인 스레드를 잠시 멈추고, `Hello World` 를 출력한다.

1 밀리 세컨드를 멈춘 이유는, 비동기적 실행을 더 티나게 보여주기 위해서이다.

`Calculate()` 함수와 메인 스레드의 `Hello World` 출력은 서로 다른 스레드에서 실행되기 때문에, 순차적으로 실행되지 않는다.

***어쩔때는 `Calculate()` 함수가 먼저 실행되고, 어쩔때는 `Hello World` 가 먼저 실행될 수 있다.***

이처럼 비동기적 실행은 순차적 실행과 달리, 실행 순서가 절대적으로 보장되지 않는다.

작업 B 가 작업 A 에 종속되도록 하여 A 가 끝난 후, B 가 실행되도록 종속성을 만들거나, `mutex` 같은 동기화 객체를 사용하여 실행 순서 제어할 수는 있다.

***비동기가 언제나 멀티 스레드 작업을 의미하지는 않지만***, 대부분의 경우 멀티 스레드 작업을 의미한다.

## Future

퓨쳐는 미래에 계산되어올 값을 나타내는 객체이다.

조금 더 자세히 설명하자면, 퓨쳐는 비동기 연산의 결과를 나타내는 객체이다.

퓨쳐는 비동기 연산의 결과를 나타내기 때문에, 비동기 연산이 끝나기 전까지는 값을 알 수 없다.

비동기 연산이 끝나야지만, 퓨쳐에서 결과를 가지고 올 수 있다.

아래 코드는 간단히 퓨쳐를 사용하는 예제이다.

```cpp
#include <iostream>
#include <future>

int Calculate()
{
	int sum = 0;

	for (int i = 0; i < 1000000; ++i)
	{
		sum += i;
	}

	return sum;
}

int main()
{
	std::future<int> f = std::async(std::launch::async, Calculate);
	
	std::cout << f.get() << std::endl;
}
```

`std::async` 를 통해 `Calculate()` 함수를 비동기적으로 실행한다.

실행된 `std::async` 는 `std::future` 를 반환한다.

퓨쳐는 `get()` 함수를 통해 비동기 연산의 결과를 가져올 수 있다.

***`get()` 함수는 한번만 호출할 수 있다.*** 재사용이 불가능하므로 두번 이상 호출하면 안된다.

`get()` 으로 값을 가져오는 것은 비동기 연산이 끝났을 때만 가능하다.

작업이 완료되어있지 않다면, `get()` 함수는 작업이 완료될 때까지 기다린다.

퓨쳐는 `wait_for()` 와 `wait_until()` 함수들을 통해 작업이 완료되었는지 아닌지를 알 수 있다.

`wait()` 함수는 작업이 완료될 때까지 기다리는 함수라 조금 다르다. `get()` 을 하는 것이 사실상 값을 전달하는 `wait()` 과 같다고 보면 된다.

아래 코드는 `wait_for()` 를 사용하는 예제이다.

```cpp
#include <iostream>
#include <future>
#include <chrono>

int main()
{
	std::future f = std::async(std::launch::async, []() {
		std::this_thread::sleep_for(std::chrono::seconds(3));
		return 8;
	});

	std::future_status status;

	do 
	{
		status = f.wait_for(std::chrono::seconds(1));
		std::cout<< "Waiting..." << std::endl;

	} while (status != std::future_status::ready);

	std::cout<< "Ready" << std::endl;

	std::cout << "Result: " << f.get() << std::endl;

	return 0;
}
```

결과는 다음과 같다.

```
Waiting...
Waiting...
Waiting...
Ready
Result: 8
```

퓨쳐는 다음과 같이 특정 객체의 함수 호출에 대해서도 사용할 수 있다.
```cpp
#include <iostream>
#include <future>

class Knight
{
public:
	int GetHP() { return 100; }
};

int main()
{
	Knight knight;

	std::future f = std::async(std::launch::async, &Knight::GetHP, knight);

	std::cout << "Knight HP: " << f.get() << std::endl;

	return 0;
}
```

## Promise

프로미스는 퓨쳐에 값을 전달하기로 약속(Promise) 하는 객체이다.

프로미스는 비동기 연산의 결과를 퓨쳐에 전달할 때 사용한다.

프로미스에 값을 전달하면, 프로미스에서 연결한 퓨쳐에 그 값을 가져올 수 있다.

아래는 간단한 예제이다.

```cpp
#include <iostream>
#include <future>

void PromiseWorker(std::promise<int>&& p)
{
	p.set_value(123);
}

int main()
{
	std::promise<int> p;
	std::future<int> f = p.get_future();

	std::thread t(PromiseWorker, std::move(p));

	std::cout << f.get() << std::endl;

	t.join();

	return 0;
}
```

위 코드를 해설해보면 다음과 같다.

1. `std::promise<int> p` 를 통해 프로미스를 생성한다.
2. `p.get_future()` 를 통해 프로미스에 연결된 퓨쳐를 가져와 메인 스레드의 퓨쳐 `f` 에 저장한다.
3. `PromiseWorker()` 함수를 새로운 스레드 `t` 에서 실행한다. 이때, 프로미스 `p` 를 `std::move()` 를 통해 이동시켜 메인 스레드로부터 소유권을 이전한다.
4. 메인 스레드로부터 독립적인 스레드에서 `p.set_value(123)` 를 통해 프로미스에 `123` 이라는 값을 전달한다.
5. 메인 스레드에서 `f.get()` 을 통해 `PromiseWorker()` 에서 프로미스를 통해 전달한 값이 올 때 까지 대기하고, 값을 가져온다.

이처럼 프로미스는 비동기 상태에서 퓨쳐에 값을 전달할 때 사용할 수 있다.

## Packaged Task

`Packaged Task` 는 퓨쳐에 비동기 연산을 연결할 수 있는 객체이다.

퓨쳐에 비동기 연산을 연결할 때, `std::async` 를 사용할 수도 있지만, `Packaged Task` 를 사용할 수도 있다.

`std::async` 는 비동기 연산을 실행하고, 그 결과를 퓨쳐에 전달하는 것을 한번에 수행한다.

이때 `std::async` 는 비동기 연산을 실행하는 스레드를 알아서 생성하고, 그 스레드에서 비동기 연산을 실행한다.

`Packaged Task` 는 실행할 비동기 연산을 캡슐화하는 객체이다.

실행할 비동기 연산을 캡슐화하는 것만 할 뿐, 비동기 연산을 실행하는 스레드를 생성하거나, 비동기 연산을 실행하는 것은 `Packaged Task` 가 아니다.

말로하니 어렵다. 코드로 살펴보자.

```cpp
#include <iostream>
#include <future>
#include <thread>

int Compute(int x, int y)
{
	return x * y;
}

int main()
{
	std::packaged_task<int(int, int)> task(Compute);

	std::future<int> f = task.get_future();

	std::thread t(std::move(task), 10, 20);

	std::cout << "Result: " << f.get() << std::endl;

	thread.join();

	return 0;
}
```

흐름대로 설명해보면 다음과 같다.

1. `Compute()` 함수는 두 개의 정수를 곱하는 간단한 작업을 수행한다.

2. 메인 스레드에서 `std::packaged_task` 객체는 이 `Compute()` 함수를 캡슐화한다.

3. 메인 스레드에서 퓨쳐 `f` 에게 `task` 의 `get_future()` 를 호출하여 퓨쳐 객체를 얻는다.

4. 메인 스레드에서 `std::thread` 객체 `t` 를 생성하고, `task` 를 `std::move()` 를 통해 이동시켜 소유권을 독립 스레드에 이전한다.

5. 독립 스레드에서 `Compute()` 함수가 실행되면서 `task` 에 캡슐화된 비동기 연산이 실행된다.

6. 메인 스레드에서 `f.get()` 을 통해 비동기 연산의 결과를 기다리고, 가져온다.

이처럼 `Packaged Task` 는 비동기 연산을 캡슐화하여 퓨쳐에 연결할 수 있게 해준다.

## std::launch

끝내기 전에 `std::async` 의 인자로 사용한 `std::launch` 에 대해 좀 더 알아보자.

`std::launch` 는 두가지 값이 있다.

`std::launch::async` 와 `std::launch::deferred` 이다.

### std::launch::async

`std::launch::async` 는 비동기 연산을 실행하기 위해 스레드를 생성한다.

연산을 위한 스레드를 만들어 비동기 연산을 실행하고, 그 결과를 퓨쳐에 전달하기 위해 사용한다.

### std::launch::deferred

`std::launch::deferred` 는 사실 비동기 연산은 아니다.

지연된 연산(Lazy Evaluation) 이나 조건에 따른 동기적 실행을 위해 존재한다.

`std::launch::deferred` 를 사용하면, 비동기 연산을 실행하는 스레드를 생성하지 않는다.

퓨쳐와 함께 사용할 때, `get()` 이나 `wait()` 함수가 호출될 때 그제서야 연산을 실행하도록 미루기 위해 사용한다.

## Conclusion

`std::future` 는 비동기 연산의 결과를 나타내는 객체이다.

`std::future` 는 `get()` 함수를 통해 비동기 연산의 결과를 가져올 수 있다.

`std::future` 는 `wait_for()` 와 `wait_until()` 함수를 통해 비동기 연산의 결과가 올 때까지 기다릴 수 있고, 상태를 관찰할 수 있다.

`std::promise` 는 퓨쳐에 비동기 연산의 결과를 전달하기 위한 객체이다.

`std::promise` 는 `get_future()` 함수를 통해 퓨쳐를 생성할 수 있다.

`std::promise` 는 `set_value()` 함수를 통해 비동기 연산의 결과를 퓨쳐에 전달할 수 있다.

`std::packaged_task` 는 퓨쳐에 비동기 연산을 연결하기 위한 객체이다.

`std::packaged_task` 는 `get_future()` 함수를 통해 퓨쳐를 생성할 수 있다.

`std::packaged_task` 는 비동기 연산을 캡슐화할 수 있다.

`std::launch::async` 는 비동기 연산을 실행하기 위해 스레드를 생성한다.

`std::launch::deferred` 는 동기 연산이되, 실행 시점이 조절되는 지연된 연산(Lazy Evaluation) 을 위해 존재한다.
