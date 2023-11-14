---
title: "Lua Tonic #1 : Variable"
summary: "Variable"
categories: ["Korean Post", "Programming"]
tags: ["Lua"]
date: 2023-11-12
draft: true
showauthor: false
authors:
  - bbagwang
series: ["Lua Tonic"]
series_order: 1
---


## Lua Types

Lua에는 여덟 가지 기본 타입이 있다.

- nil
- boolean
- number
- string
- function
- userdata
- thread
- table

## nil

nil 타입은 단일 값 nil을 가지며, 값의 부재를 표현하는 데 사용된다.

## boolean

boolean 타입은 true와 false 이렇게 오직 두 값만 있다.

nil과 false는 조건문에서 거짓을 나타내며, 다른 모든 값은 참을 나타낸다.

## number

`number` 타입은 정수와 실수를 모두 나타내며, 64비트 정수와 64비트 실수를 기본으로 사용하지만, 32비트 정수나 32비트 실수로 컴파일하는 것도 가능하다.

루아는 정수와 실수 사이를 필요에 따라 자동으로 변환한다.

프로그래머는 두 서브타입 사이의 차이를 무시하거나 각 숫자의 표현을 완전히 제어할 수 있다.

## string

string 타입은 변경 불가능한 바이트 시퀀스(Byte Sequence) 를 나타낸다.

Lua 문자열은 `8-Bit Clean` 이며, 인코딩에 대한 가정이 없다.

### 8-Bit Clean

문자열 처리에서 8비트 값을 모두 지원한다는 것을 의미한다.

루아의 문자열이 어떠한 8비트 값도 포함할 수 있고, 그 중에서도 특히 내장된 null 문자(ASCII 코드 0, C 언어에서의 '\0')를 포함할 수 있다는 것을 나타낸다.

다른 언어에서는 문자열이 `null` 문자를 만나면 종료되는 경우가 많다. 

그러나, 루아에서는 문자열 내에 `null` 문자가 포함되어 있어도 문자열의 나머지 부분을 정상적으로 처리하고 저장할 수 있다.

이러한 특성은 루아가 `Binary Data`나 특수한 텍스트 인코딩을 다루는데 유용하게 사용될 수 있도록 한다.

간단히 다시 말해보면, 루아에서는 문자열이 단순히 바이트의 연속으로 간주되며, 이 바이트들에 대한 특별한 제한이 없다는 것이다.

이러한 특성은 루아가 다양한 데이터 유형과 외부 시스템과의 통합에서 유연성을 제공하는 데 도움을 준다.

## function

`Lua` 및 `C(C++)`에서 작성된 함수 모두 `function` 타입으로 표현된다.

## userdata

`userdata` 타입은 루아 변수에 임의의 `C(C++)` 데이터를 저장하기 위해 제공된다.

메모리 블록이나 C 포인터 값을 나타낼 수 있습니다.

## thread

`thread` 타입은 독립적인 실행 스레드를 나타내며, 코루틴 구현에 사용된다.

## table

`table` 타입은 연관 배열을 구현하며, 다양한 루아 값들을 인덱스로 사용할 수 있다.

테이블은 루아의 유일한 데이터 구조화 메커니즘으로, 다양한 데이터 구조를 나타내는 데 사용된다.

## Type Conversions

`table`, `function`, `thread`, `userdata` 값은 객체이므로, 변수는 이러한 값들을 실제로 포함하지 않고 참조만 가진다.

할당, 매개변수 전달, 함수 반환은 이러한 값들에 대한 참조를 조작하며, 이러한 작업은 어떠한 복사도 수반하지 않는다.

## Lua's Concept of Value, Type and Variable

`Value` 는 값이다.

`Type` 은 말 그대로 타입이다.

그리고 마지막으로, `Variable` 은 변수이다.

이 각각의 개별적인 요소들은 서로 밀접한 관계를 가지고 있다.

`Lua` 는 동적 타입 언어로, **`변수(Variable)` 가 아니라 `값(Value)` 에 타입이 있다.**

이게 무슨 말이냐면, 루아에서는 변수에 미리 정의된 타입을 지정하지 않고, 변수에 값을 할당할 때, 그 값의 타입을 자동으로 취하게 된다는 것이다.

## All Values Carry Their Own Type

> variables do not have types; only values do. There are no type definitions in the language.  
All values carry their own type.<br>
> — <cite>Lua Reference Manual[^1]</cite>
[^1]: Lua 5.4 Reference Manual - [2.1 – Values and Types](https://www.lua.org/manual/5.4/manual.html#2.1)

루아에서 타입은 값에 내재되어 있으며, 프로그램 실행 중에 값의 타입을 변경할 수 없다.

각 값이 자신의 타입 정보를 내장하고 있다는 의미다.

즉, 값이 생성될 때 그 타입이 결정되며, 이 타입은 해당 값의 전체 생명주기 동안 변경되지 않는다.

이를 통해 두 가지 중요한 특성이 나타난다.

### Immutability of Values

Lua에서 값은 생성될 때 부여받은 타입을 변경할 수 없다.

예를 들어, 숫자 10은 항상 `number` 타입이며, 이를 다른 타입으로 변경할 수 없다.

문자열 "hello"도 마찬가지로 항상 `string` 타입이다.

이는 값의 불변성을 의미하며, `타입`이 `값`과 긴밀하게 연결되어 있다는 것이다.

### Flexibility of Variables

반면에, 변수는 할당된 값에 따라 타입이 변할 수 있다.

변수에 새로운 값이 할당되면, 그 변수는 새로운 값의 타입을 취한다.

예를 들어, 변수 `a`에 숫자를 할당하면 `a`는 `number` 타입이 되고, 나중에 같은 `a`에 문자열을 할당하면 `a`는 `string` 타입이 된다.

이는 변수가 유동적으로 참조된 값의 타입을 따른다는 것을 의미한다.

## First-Class Values

루아의 값은 `일급 값(First-Class Value)` 으로, 변수에 저장되거나 함수의 인자로 전달되고 결과로 반환될 수 있다.

조금 더 정확히 보자면, 일급 값은 다음과 같은 특성을 가진다.

- 변수에 `저장`할 수 있다.
- 함수의 인자로 `전달` 및 `반환`할 수 있다.
- 다른 값에 `할당` 및 `복사`할 수 있다.
- 필요에 따라 `생성` 및 `파괴` 될 수 있다.

## Reference

- [Lua 5.4 Reference Manual](https://www.lua.org/manual/5.4/)
- [Lua 5.4 Reference Manual (한국어)](https://wariua.github.io/lua-manual/5.4/)