---
title: "Lua Tonic #1 : Hello Lua"
summary: "Hello Lua!"
categories: ["Korean Post", "Programming"]
tags: ["Lua", "Programming"]
date: 2023-11-12
draft: false
showauthor: false
authors:
  - bbagwang
series: ["Lua Tonic"]
series_order: 1
---

`Lua` 를 이용해 가장 먼저 해볼 것은 언어 학습의 국룰인 `Hello World!` 를 출력하는 것이다.

나는 루아를 배우기 시작한 만큼, `Hello Lua!` 를 출력해보려 한다.

그런데 일단 루아가 없다. 설치부터 진행해 보도록 하자.

**Disclaimer**

아래 내용은 정석적이긴 하지만, 꽤나 귀찮은 방식이긴 하다.

좀 더 빠르고 간편하게 루아를 설치하고 싶다면 `homebrew` 같은 것을 사용하는 것도 좋다.

## Install Lua

내가 평소에 사용하는 머신은 2 가지 이다.

- MacBook Pro M1 (Mac OS)
- HanSung TFX5075G (Windows 11)

두 노트북은 서로 다른 OS 를 사용하고, 루아를 설치하는 방법도 약간 다르다.

노트북을 번갈아 가면서 글을 쓰게될 것이므로, 각각의 OS 에 맞는 설치 방법을 적어두겠다.

현재 이 글을 쓰는 기준에서 가장 최신 버전인 `5.4.6` 버전의 루아를 설치하도록 한다.

### Mac

`Terminal` 을 연다.

`cd` 명령어를 이용해 루아를 다운로드할 디렉토리로 이동한다.

나는 `Downloads` 폴더를 추천한다.

명령어를 이용해 루아를 다운로드 받는다.

```bash
curl -R -O http://www.lua.org/ftp/lua-5.4.6.tar.gz
```

그 후 받아온 파일을 압축 해제한다.

```bash
tar zxf lua-5.4.6.tar.gz
```

`ls` 명령어로 파일 리스트를 보면 `lua-5.4.6` 이라는 폴더가 생성되어 있는 것을 확인할 수 있다.

`cd lua-5.4.6` 으로 해당 폴더로 이동한다.

다시 `ls` 명령어를 사용해보면, 압축을 해제한 폴더에 무엇이 있는지 확인할 수 있다.

여기서 `MakeFile` 이라는 파일이 있는 것을 확인할 수 있다.

이 파일은 루아를 빌드하고 설치하는 데 사용된다.

아래의 명령어를 입력해 루아를 빌드하고, 설치하자.

```bash
sudo make all install
```

큰 문제 없이 설치가 되었다면, `lua` 를 입력해 루아가 설치되었는지 확인해보자.

다음과 같은 메시지가 출력된다면, 빌드와 설치에 성공한 것이다.

```
Lua 5.4.6  Copyright (C) 1994-2023 Lua.org, PUC-Rio
```

`>` 가 출력되며 인터프리터가 실행되어 있다면, Control + C 를 눌러 나올 수 있다.

### Windows

윈도우즈에서 루아를 직접 빌드하고, 적용하는 것은 조금 복잡하다.

일단 `make` 부터 동작하지 않을 것이기 때문이다.

그래서 미리 빌드된 루아를 다운로드 받아서 사용하도록 하는 것이 속 편하다.

https://joedf.ahkscript.org/LuaBuilds/

위 링크로 들어가면, 미리 빌드된 루아 바이너리를 다운로드 받을 수 있다.

`lua-5.4.6_Win64_bin.zip` 를 다운로드 받아서, 압축을 해제한다.

압축을 해제한 폴더의 이름을 `lua` 로 변경한다.

`C:\Program Files` 위치에 `lua` 폴더를 이동시킨다.

그럼 최종적으로 `C:\Program Files\lua` 경로에 압축을 해제한 내용이 들어있을 것이다.

이제 `lua` 폴더의 경로를 환경 변수에 추가해야 한다.

`시스템 환경 변수 편집` 을 메뉴에서 검색해 실행한다.

오른쪽 아래 `환경 변수` 를 클릭한다.

`시스템 변수` 에서 `Path` 를 찾아서 편집한다.

`새로 만들기` 를 클릭하고, `C:\Program Files\lua` 를 입력한다.

`확인` 을 눌러서 저장한다.

이제 `cmd` 를 열고, `lua` 를 입력해보자.

다음과 같은 메시지가 출력된다면, 빌드와 설치에 성공한 것이다.

```
Lua 5.4.6  Copyright (C) 1994-2023 Lua.org, PUC-Rio
```

`>` 가 출력되며 인터프리터가 실행되어 있다면, Control + C 를 눌러 나올 수 있다.

## Hello Lua!

이제 루아를 설치했으니, `Hello World!` 를 출력해보자.

`Terminal` 혹은 `cmd` 를 열고, `lua` 를 입력해 인터프리터를 실행한다.

그리고 다음과 같이 입력해보자.

```lua
print("Hello Lua!")
```

그럼 다음과 같이 출력된다.

```
Hello Lua!
```

여기까지 루아를 설치해 보고, 간단한 출력 테스트를 진행해 보았다.

## Visual Studio Code

앞으로 사용할 코드 에디터는 `Visual Studio Code` 다.

확장을 설치하면 개발이 편해지므로, 확장 탭에서 `Lua` 를 검색해서 아래의 확장을 설치하면 된다.

https://marketplace.visualstudio.com/items?itemName=sumneko.lua
