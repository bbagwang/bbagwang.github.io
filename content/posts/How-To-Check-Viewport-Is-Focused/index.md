---
title: "How To Check Viewport Is Focused"
summary: "How To Check Viewport Is Focused"
categories: ["English_Post", "Programming", "UnrealEngine"]
tags: ["C++", "Unreal Engine"]
date: 2022-03-05
draft: false
showauthor: false
authors:
  - bbagwang
---

When you have multiple PIE viewports, you might have some viewport focus issues.

For example, having spectator to player and input mode change make viewports do ping pong each other.

It only happens in PIE mode because it's not a Standalone Game.

Here's the code on how to check my viewport is in focus.

```cpp
#if UE_EDITOR
UWorld* World = GetWorld();
if (IsValid(World) && World->WorldType == EWorldType::PIE)
{
    UGameViewportClient* GameViewportClient = World->GetGameViewport();
    if (!IsValid(GameViewportClient))
        return;

    if (IsValid(GEngine) && IsValid(GEngine->GameViewport) && (GEngine->GameViewport->Viewport != nullptr))
    {
        const bool bIsFocusedViewport = GameViewportClient->IsFocused(GEngine->GameViewport->Viewport);
        
        UE_LOG(LogTemp, Log, TEXT("%s"), (bIsFocusedViewport ? TEXT("FOCUSED") : TEXT("NOT FOCUSED")));
    }
}
#endif //UE_EDITOR
```

Cheers!
