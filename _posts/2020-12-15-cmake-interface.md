---
title: "[CMake] INTERFACE 라이브러리"
tags:
  - CMake
show_author_profile: true
---

최근 Android Studio에서 CMake 3.10.2를 통해 JNI 프로젝트를 구성해봤는데, 그 과정에서 `INTERFACE` 라이브러리에 관해 알게된 내용입니다.

<!--more-->

{:red: style="color:red"}

## CMake 내 라이브러리

CMake에서 빌드를 구성할 수 있는 항목은 크게 **실행 파일**과 **라이브러리**로 나눌 수 있습니다. 그 중에서 라이브러리는 세부적으로 다음과 같이 나뉩니다.

* `SHARED`, `STATIC`, `MODULE` : 실제로 바이너리를 생성하는 라이브러리 타입
* `OBJECT` : 오브젝트 파일(`.o`)의 집합으로 취급되는 라이브러리 타입
* `IMPORTED` : 외부에서 빌드된 라이브러리 타입
* `ALIAS` : 읽기 전용으로 이름만 바뀐 라이브러리 타입
* **`INTERFACE`**

## `INTERFACE` 라이브러리?

`INTERFACE` 라이브러리는 어떠한 바이너리를 타게팅하는 것이 아니라, **빌드 설정의 집합**을 관리하는 특별한 라이브러리입니다. 이로 인해 해당 라이브러리가 갖게 되는 특성은 다음과 같습니다.

* `.c`나 `.cpp` 등 컴파일 대상이 되는 소스들을 포함할 수 없습니다.[^1] (헤더 등은 포함할 수 있습니다.)
* 모든 속성이 `PUBLIC`처럼 취급되어, 다른 라이브러리가 `INTERFACE` 라이브러리를 링크할 때 해당 라이브러리에 적용된 설정이 모두 주입됩니다.
* 일부 커맨드에 대해 `INTERFACE` 키워드를 명시해야 합니다.

[^1]: CMake 3.19 이상부터는 컴파일 대상이 되는 소스를 포함할 수 있으나, `INTERFACE` 라이브러리 만으로는 여전히 컴파일 대상에 포함되지 않습니다.

제 경우에는 하위 라이브러리의 집합을 다음과 같이 `INTERFACE` 라이브러리로 관리했습니다.

```cmake
# CMakeLists.txt
# MyLibs.cmake
# MyLibs
#   ├─ foo
#   │   ├─ CMakeLists.txt (for static library libfoo.a)
#   │   └─ ...sources...
#   ├─ bar
#   │   ├─ CMakeLists.txt (for static library libbar.a)
#   │   └─ ...sources...
#   └─ ...


# MyLibs.cmake
add_library(MyLibs INTERFACE)

foreach(lib IN ITEMS foo bar ...)
  add_subdirectory(MyLibs/${lib})
  target_link_libraries(MyLibs INTERFACE ${lib})
endforeach()


# CMakeLists.txt
add_library(AppJNIPart SHARED)
include(MyLibs.cmake)

target_link_libraries(
  AppJNIPart

  PRIVATE MyLibs
  PRIVATE SysLib1
  PRIVATE SysLib2
)
```

## 링크 순서 문제

그런데 이 상태에서 빌드를 돌리면 계속 링크에 실패해서, verbose 모드로 내부 커맨드를 확인해봤습니다.

`... -lSysLib1 -lSysLib2 -lMyLibs/libfoo.a -lMyLibs/libbar.a ...`

어? 이건 기대한 링크 순서와 다른데요? 분명 CMake 상에서 `AppJNIPart`에 `target_link_libraries` 커맨드를 적용할 때 `MyLibs`, `SysLib1`, `SysLib2` 순으로 링크했는데요...

```cmake
target_link_libraries(
  AppJNIPart

  PRIVATE MyLibs    # 1st
  PRIVATE SysLib1   # 2nd
  PRIVATE SysLib2   # 3rd
)
```

그럼 당연히 **`MyLibs`에 속한 라이브러리들이 <span>먼저</span>{: red} 링크되어야 할텐데**, 왜 멋대로 순서가 뒤바뀐 걸까요?

더 이상한 것은 최상위 `CMakeLists.txt`를 다음과 같이 고쳤을 때 일어났습니다.

```cmake
add_library(AppJNIPart SHARED)
include(MyLibs.cmake)

target_link_libraries(
  AppJNIPart

  PRIVATE "-Wl,-("    # added
  PRIVATE MyLibs
  PRIVATE "-Wl,-)"    # added
  PRIVATE SysLib1
  PRIVATE SysLib2
)
```

`MyLibs`에 속한 라이브러리 간 순환 참조를 해결하기 위해 넣은 플래그로, 의도한 바는 다음과 같습니다.

**의도** : `... -Wl,-( -lMyLibs/libfoo.a -lMyLibs/libbar.a -Wl,-) -lSysLib1 -lSysLib2 ...`

그런데 실제 최종 커맨드는 다음과 같았습니다.

**실제** : `... -Wl,-( -Wl,-) -lSysLib1 -lSysLib2 -lMyLibs/libfoo.a -lMyLibs/libbar.a ...`

**`-(`, `-)` 옵션을 내버려둔 채 `MyLibs`만 맨 뒤로 가버려**, 순환 참조를 지정할 수 없게 되어버렸습니다!

이후 CMake 삽질을 거듭하며 경험적으로 알게 된 사실은 다음과 같습니다.

* **일반적인 링크 옵션은 추가되는 즉시 반영됩니다.**
* **`INTERFACE` 라이브러리를 통해 주입되는 링크 옵션은 가장 나중에 추가됩니다.** [^2]
* **한 `INTERFACE` 라이브러리 내 링크 옵션 순서는 보존됩니다.**
* **여러 `INTERFACE` 라이브러리에 대해 링크 옵션이 추가되는 순서는 보존됩니다.**

[^2]: `INTERFACE` 라이브러리의 경우 CMake 구성이 모두 끝났을 때 비로소 빌드 설정을 전이할 수 있는 것으로 추측됩니다.

이러한 사항을 반영하여, 의도한대로 CMake 스크립트를 수정하면 다음과 같습니다.

```cmake
# MyLibs.cmake
add_library(MyLibs INTERFACE)

target_link_libraries(MyLibs INTERFACE "-Wl,-(") # added
foreach(lib IN ITEMS foo bar ...)
  add_subdirectory(MyLibs/${lib})
  target_link_libraries(MyLibs INTERFACE ${lib})
endforeach()
target_link_libraries(MyLibs INTERFACE "-Wl,-)") # added
```

순환 참조 플래그를 `MyLibs` 라이브러리에 추가하여 하위 라이브러리 링크 옵션을 감싸도록 수정했습니다.

이렇게 되면 `MyLibs`가 포함하는 링크 옵션에 순환 참조 플래그가 포함되어, 이전처럼 라이브러리 링크 옵션이 플래그를 탈출하는 현상을 막을 수 있습니다.

```cmake
# CMakeLists.txt
add_library(AppJNIPart SHARED)
include(MyLibs.cmake)

# added
add_library(SysLibs INTERFACE)
target_link_libraries(
  SysLibs

  INTERFACE SysLib1
  INTERFACE SysLib2
)

target_link_libraries(
  AppJNIPart

  PRIVATE MyLibs
  PRIVATE SysLibs  # modified
)
```

`SysLibs` 라이브러리를 추가로 두어, `MyLibs` 라이브러리보다 링크 옵션이 먼저 오지 않도록 수정했습니다.

`SysLibs`가 `MyLibs`가 포함하는 링크 옵션에 순환 참조 플래그가 포함되어, 이전처럼 라이브러리 링크 옵션이 플래그를 탈출하는 현상을 막을 수 있습니다.
