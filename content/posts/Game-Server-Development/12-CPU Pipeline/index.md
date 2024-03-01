---
title: "Game Server Development #12 : CPU Pipeline"
summary: "CPU Pipeline"
categories: ["Korean Post", "Programming"]
tags: ["C++", "Server", "Thread", "CPU", "Optimization"]
date: 2024-03-02
draft: false
showauthor: false
authors:
  - bbagwang
series: ["Game Server Development"]
series_order: 12
---

## Material
**[[C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈] Part4: 게임 서버](https://inf.run/8Chk)**

## CPU Pipeline




```cpp
#include <iostream>
#include <thread>

int x = 0;
int y = 0;
int r1 = 0;
int r2 = 0;

volatile bool ready;

void Thread1()
{
	while (!ready);

	y = 1;	//Store y
	r1 = x;	//Load x
}

void Thread2()
{
	while (!ready);

	x = 1;	//Store x
	r2 = y;	//Load y
}

int main()
{
	int count = 0;

	while (true)
	{
		++count;

		x = y = r1 = r2 = 0;

		std::thread t1(Thread1);
		std::thread t2(Thread2);

		ready = true;

		t1.join();
		t2.join();

		if (r1 == 0 && r2 == 0)
			break;
	}

	std::cout << count << " 번 만에 빠져나옴" << std::endl;
}
```