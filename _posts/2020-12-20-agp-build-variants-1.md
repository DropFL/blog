---
title: "[Android] 빌드 다양화"
tags:
  - Android
  - Gradle
  - AGP
show_author_profile: true
---

Gradle을 통해 Android 빌드 환경을 구성할 때, 빌드 다양화를 구현하는 방법을 정리한 내용입니다.

<!--more-->

## Android Gradle Plugin (AGP)

Android Studio에서는 프로젝트 구성 및 빌드에 Gradle을 사용합니다. 그리고 Android 프로젝트에 맞는 빌드 설정을 간편하게 하기 위해 플러그인을 같이 제공하는데, 이것을 **AGP**라고 부릅니다.

Android 앱을 개발할 때 필수적으로 사용하는 플러그인으로, gradle 파일 내에서 **`android`로 시작하는 코드 블럭**이 바로 AGP를 통해 빌드를 구성하는 부분입니다.

## 빌드 다양화

하나의 프로젝트가 여러 빌드를 구성하는 것은 매우 흔한 일이고, 이는 Android 프로젝트 또한 마찬가지입니다. 예를 들어, 개발하는 앱이 다음과 같이 여러 버전으로 나뉜다고 합시다.

1. 광고 유무에 따른 `free` 버전 / `pro` 버전
2. 테스트 결제 활성화 여부에 따른 `sandbox` 버전 / `product` 버전
3. 디버깅 가능 여부에 따른 `debug` 버전 / `release` 버전

그렇다면 이 앱에서 가능한 버전의 조합은 `free + sandbox + debug` 버전, `pro + product + release` 버전 등이 있습니다. 여기서 가능한 조합을 단순히 계산하면 총 8개인데, 각 버전 조합에 따른 프로젝트를 개별적으로 구성하는 것은 아주 비효율적입니다. 그보다는 **프로젝트는 하나로 두되, 각 버전마다 빌드 설정 및 리소스의 일부를 변경하는 방식**을 차용하는 것이 훨씬 간편할 것입니다.

이렇게 한 프로젝트가 여러 빌드 구성을 갖게끔 하는 것을 **빌드 다양화**라고 칭하겠습니다.

## AGP의 빌드 다양화

AGP에서는 [Travis CI의 Build Matrix](https://docs.travis-ci.com/user/build-matrix/)와 유사한 방식으로 빌드 다양화 기능을 제공합니다. 하지만 그 구조가 복잡하지는 않으므로 Build Matrix를 모르더라도 구조를 이해하는 데에는 큰 지장이 없습니다.

### `VariantDimension`과 `Variant`

{:red: style="color:red"}
{:green: style="color:green"}
{:blue: style="color:blue"}

**AGP에서 빌드 다양화는 N차원 행렬을 구성하듯이 이루어집니다.** 위의 예시에서는 **<span>광고 유무</span>{: red}, <span>테스트 결제</span>{: green}, <span>디버깅 가능</span>{: blue}** 총 3개 차원을 두고 행렬을 만들고, 각 차원에 존재할 수 있는 값이 다음과 같이 한정되는 셈입니다.

* **<span>광고 유무</span>{: red}**: `free`, `pro`
* **<span>테스트 결제</span>{: green}**: `sandbox`, `product`
* **<span>디버깅 가능</span>{: blue}**: `debug`, `release`

**[`VariantDimension`](https://developer.android.com/reference/tools/gradle-api/4.1/com/android/build/api/dsl/VariantDimension)은 각 차원에 존재할 수 있는 값입니다.** 여기서는 `free`, `sandbox`, `release` 등이 `VariantDimension`에 해당합니다. 실제로는 `VariantElement`에 가까운 역할이라고 보면 됩니다.

**[`Variant`](https://developer.android.com/reference/tools/gradle-api/4.1/com/android/build/api/variant/Variant)는 AGP에서 `VariantDimension`을 조합하여 빌드를 구성하는 최소 단위입니다.** 예를 들어, `pro + product + release` 버전에 대한 빌드는 유일한 `Variant`에 대응됩니다.

### `BuildType`

[`BuildType`](https://developer.android.com/reference/tools/gradle-api/4.1/com/android/build/api/dsl/BuildType)은 **프로젝트 개발 단계와 관련된 빌드 옵션을 제어하기 위한 `VariantDimension`**입니다. 예시에서는 **<span>디버깅 가능</span>{: blue}** 차원에 속하는 값들이 `BuildType`에 해당됩니다. Gradle 상에서는 다음과 같이 작성합니다.

```gradle
android {
    buildTypes {
        debug {
            // 디버깅 가능 세팅 예시
            debuggable true
        }
        release {
            // 디버깅 불가능 세팅 예시
            debuggable false
        }
    }
}
```

### `ProductFlavor`

[`ProductFlavor`](https://developer.android.com/reference/tools/gradle-api/4.1/com/android/build/api/dsl/ProductFlavor)는 **`BuildType`에 해당되지 않는 나머지 `VariantDimension`**입니다. 예시에서는 **<span>광고 유무</span>{: red}, <span>테스트 결제</span>{: green}** 차원에 속하는 값들이 `ProductFlavor`에 해당됩니다.

`ProductFlavor`는 다음과 같이 `flavorDimensions` 명령을 통해 **자체적인 차원을 정의하여 사용해야 합니다.**

```gradle
android {
    flavorDimensions 'advertise', 'payment'
    productFlavors {
        free {
            dimension 'advertise'
            // 광고를 포함하는 세팅
        }
        pro {
            dimension 'advertise'
            // 광고를 제외하는 세팅
        }

        sandbox {
            dimension 'payment'
            // 테스트 결제 세팅
        }
        product {
            dimension 'payment'
            // 실제 결제 세팅
        }
    }
}
```

### `DefaultConfig`

[`DefaultConfig`](https://developer.android.com/reference/tools/gradle-api/4.1/com/android/build/api/dsl/DefaultConfig)는 **모든 빌드에 공통으로 적용되는 특수한 `ProductFlavor`**입니다. 모든 빌드에 적용되므로 별도의 차원을 정의할 필요가 없으며, `ProductFlavor`에 적용 가능한 대부분의 세팅을 동일하게 적용할 수 있습니다. Gradle 상에서는 다음과 같이 작성합니다.

```gradle
android {
    defaultConfig {
        // 모든 빌드 공통 세팅
    }
}
```

`Variant`를 구성하는 `BuildType`, `ProductFlavor`, `DefaultConfig`의 빌드 세팅은 서로 합쳐지거나, 우선순위가 가장 높은 것으로 덮어쓰게 됩니다. 이와 관련된 내용은 추후 다른 포스트에서 다루겠습니다.

### `VariantFilter`

[`VariantFilter`](https://developer.android.com/reference/tools/gradle-api/4.1/com/android/build/api/variant/VariantFilter)는 **특정 `Variant`를 빌드 대상에서 제외하는 함수**입니다.

AGP에서는 기본적으로 Gradle에서 정의된 `BuildType`과 `ProductFlavor`들을 바탕으로 가능한 모든 조합의 `Variant`를 생성합니다. 하지만 이는 빌드를 원치 않는 조합도 생성해낼 수 있는데, 그러한 조합을 제외할 때 `VariantFilter`를 사용합니다.

다음은 `product` 빌드에서 `debug` 빌드를 비활성화하는 Gradle 스크립트입니다.

```gradle
android {
    ...

    variantFilter { variant ->
        def isProduct = (variant.flavors.get(1).name == 'product') // flavorDimensions[1] == 'payments'
        def isDebug = (variant.buildType.name == 'debug')

        if (isProduct && isDebug)
            setIgnore(true)
    }
}
```

## Android Studio와 연동

Android Studio 내에서 빌드 별 설정을 적용하는 방법은 다음과 같습니다.

1. Gradle에서 빌드 다양화를 구성합니다.
2. **Sync Project with Gradle Files** 버튼을 눌러 수정 사항을 적용합니다.
3. **Build Variants** 탭을 띄웁니다. 기본적으로 좌측 하단에 위치하며, **상단 메뉴 > View > Tool Windows > Build Variants**를 통해 진입할 수도 있습니다.
4. **Active Build Variant** 하단의 셀을 눌러 빌드 대상을 수정합니다.

## 마무리

지금까지 AGP에서 빌드를 구성하는 방법에 대해 알아보았습니다. 이후 포스팅에서는 여기서 소개한 기능을 더 자세히 알아보면서 Gradle과 AGP의 동작 방식에 대해 깊이 이해해보겠습니다.
