---
title: "How To Iterate Over UENUM"
summary: "Advanced Use of Unreal Engine Enumerator"
categories: ["English_Post", "Programming", "Unreal Engine"]
tags: ["C++", "Unreal Engine", "Enum"]
date: 2021-12-11
draft: false
showauthor: false
authors:
  - bbagwang
---

Sometimes we want to iterate over enum values.

In the Unreal Engine, We already have macros about making rages for UENUM.



First, Let's make a test enum type.

``` cpp
UENUM(BlueprintType)
enum class ETest : uint8
{
	ZERO,
	FIRST,
	SECOND,
	THIRD,
	FOURTH,
	FIFTH
};
```

That looks decent enum type.

We also need to print the enum value name for the test.

``` cpp
template<typename TEnum>
static FString EnumToString(const FString& Name, TEnum Value)
{
	const UEnum* EnumPtr = FindObject<UEnum>(ANY_PACKAGE, *Name, true);

	if (UNLIKELY(!IsValid(EnumPtr)))
		return FString("Invalid");

	return EnumPtr->GetNameStringByValue((int32)Value);
}
```

Okay, I think we're ready! Let's get into the topic.

There are 3 types of enum range macro.

## ENUM_RANGE_BY_COUNT

This is a simple one.

It makes iterator range from the initial value to counted value.

If we declare a range like below, We'll Iterator 3 elements from the initial value.

``` cpp
ENUM_RANGE_BY_COUNT(ETest, 3);
```

Now we can iterate **UENUM** value with **TEnumRange<>** like this.

``` cpp
for (ETest Iter : TEnumRange<ETest>())
	{
		GEngine->AddOnScreenDebugMessage(-1, 999.f, FColor::Cyan, EnumToString<ETest>("ETest",Iter));
	}
```



![ENUM_RANGE_BY_COUNT](img/ENUM_RANGE_BY_COUNT.png)

You can see that we print only 3 elements.

## ENUM_RANGE_BY_FIRST_AND_LAST

This version allows us to set the range scope ourselves.

If we declare a range like below, We'll Iterator elements that from value to value.

**from 2(ETest::SECOND) to 5(ETest::FIFTH)**

``` cpp
ENUM_RANGE_BY_FIRST_AND_LAST(ETest, 2, 5);
```

![ENUM_RANGE_BY_FIRST_AND_LAST](img/ENUM_RANGE_BY_FIRST_AND_LAST.png)

You can see that we print from 2 to 5 value.

## ENUM_RANGE_BY_VALUES

This version is quite interesting.

Let's say we make our enum values **non-contiguous** like this.

``` cpp
UENUM(BlueprintType)
enum class ETest : uint8
{
	ZERO = 0,
	FIRST = 3,
	SECOND = 5,
	THIRD = 10,
	FOURTH = 99,
	FIFTH = 128
};
```

We can't use the macro that was explained before. **Because this enum is non-contiguous.**

If we declare macro like below, **We'll Iterator non-contiguous enum range with specific individual values.**

``` cpp
ENUM_RANGE_BY_VALUES(ETest, ETest::FIRST, ETest::FOURTH);
```

![ENUM_RANGE_BY_VALUES](img/ENUM_RANGE_BY_VALUES.png)

## If you want to go deeper

You can find more details on these macro in **EnumRange.h**

**Engine\Source\Runtime\Core\Public\Misc\EnumRange.h**
