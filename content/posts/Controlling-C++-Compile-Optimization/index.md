---
title: "Controlling C++ Compile Optimization"
summary: "Controlling C++ Compile Optimization"
categories: ["English_Post", "Programming"]
tags: ["C++"]
date: 2022-04-03
draft: false
showauthor: false
authors:
  - bbagwang
---

If you want to debug your code while you didn't compile your project with the DebugGame setting.

You might end up with see an optimized variable in which you can't see what's in it.

In that case, you can disable optimization on a specific section of your code.

Like this.

```cpp
#pragma optimize("", off)
void AVCharacter::NotOptimized()
{
	//Some Codes
}
#pragma optimize("", on)

void AVCharacter::Optimized()
{
	//Some Codes
}
```
