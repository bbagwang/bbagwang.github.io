---
title: "C++ Compile Optimization 일부분만 해제하는 방법"
summary: "C++ Compile Optimization 일부분만 해제하는 방법"
categories: ["Korean Post", "Programming"]
tags: ["C++", "Optimize"]
date: 2022-04-03
draft: false
showauthor: false
authors:
  - bbagwang
---

언리얼의 경우 빌드 환경이 DebugGame 이 아닌 경우 (Development. Shipping) 에 성능은 거의 그대로 유지한체, 일부 디버깅이 필요한 영역만 컴파일 최적화를 꺼서 디버깅할 수 있다.

일반 C++ 코드에도 그대로 적용 가능하다.

#pragma optimize("", off) 이후 작성한 모든 코드에 대해 컴파일 최적화를 끄고,

#pragma optimize("", on) 을 한 시점부터 컴파일 최적화가 다시 켜진다.

```cpp
//컴파일 최적화 비활성화
#pragma optimize("", off)
void AVCharacter::SetRotationMode(EVRotationMode NewRotationMode)
{

    //컴파일 최적화가 꺼져서 변수 내용들이 잘 보임
}
//컴파일 최적화 활성화
#pragma optimize("", on)

void AVCharacter::CalculateVisualScore()
{
    //컴파일 최적화가 위에서 켜져서 변수 내용들이 잘 안 보임
}
```
