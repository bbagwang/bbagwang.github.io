---
title: "How To Check Class Is From Native Or Blueprint"
summary: "How To Check Class Is From Native Or Blueprint"
categories: ["English Post", "Programming", "UnrealEngine"]
tags: ["C++", "Unreal Engine", "Class"]
date: 2021-12-12
draft: false
showauthor: false
authors:
  - bbagwang
---

It's just simple.
Just check with this code.

```cpp
GetClass()->IsNative()
```

You can test like this.

```cpp
FString Result = GetClass()->IsNative() ? TEXT(" : NATIVE") : TEXT(" : BLUEPRINT");
GEngine->AddOnScreenDebugMessage(-1, 999.f, GetClass()->IsNative() ? FColor::Yellow : FColor::Cyan, GetName() + Result);
```

![IsNative](img/IsNative.png)
