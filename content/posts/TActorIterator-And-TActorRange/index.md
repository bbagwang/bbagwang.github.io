---
title: "TActorIterator and TActorRange"
summary: "TActorIterator and TActorRange"
categories: ["English Post", "Programming", "UnrealEngine"]
tags: ["C++", "Unreal Engine"]
date: 2022-04-22
draft: false
showauthor: false
authors:
  - bbagwang
---

If you want to collect class-specific actors that spawned in the world you can use TActorIterator.

```cpp
for (TActorIterator<ACharacter> Iter(GetWorld()); Iter; ++Iter)
{
    ACharacter* Character = *Iter;
    //If IsValid
    Character->Func();
}
```

There is also a version of the Ranged For feature available in C++11.

It's called TActorRange. With this, we can iterate it safer and easier.

```cpp
for (ACharacter* Character : TActorRange<ACharacter>(GetWorld()))
{
    //If IsValid
    Character->Func();
}
```
