# 【Gradle】3.0后新的依赖关系

2017 年后 google Android studio 版本更新至 3.0，更新中，连带着com.android.tools.build:gradle 工具也升级到了3.0.0，在 3.0.0 中使用了最新的Gralde 4.0 里程碑版本作为gradle的编译版本，该版本gradle编译速度有所加速，更加欣喜的是，完全支持Java8。当然，对于Kotlin的支持，在这个版本也有所体现，Kotlin 插件默认是安装的。

旧版本的 Gradle 项目在 `Module` 中的 `dependencies` 中的变化。

```php
dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])
    implementation 'com.android.support:appcompat-v7:26.1.0'
    implementation 'com.android.support.constraint:constraint-layout:1.0.2'
    testImplementation 'junit:junit:4.12'
    androidTestImplementation 'com.android.support.test:runner:1.0.1'
    androidTestImplementation 'com.android.support.test.espresso:espresso-core:3.0.1'
}
```

下面我们来看看他们之前的差异：

**3.x版本及之前的依赖方式**

```
Compile
Provided
APK
Test compile
Debug compile
Release compile
```

***4.0之后新的依赖方式***

```
Implementation
API
Compile only
Runtime only
Unit Test implementation
Test implementation
Debug implementation
Release implementation
```

可以看到在 `gradle 4.0` 中，`compile`依赖关系已被弃用，被`implementation`和`api`替代，`provided`被`compile only`替代，`apk`被`runtime only`替代。

> 因为不做 android 开发, 所以 原来 3.0 中的 APK 的作用并不了解

## 区别和使用

### implementation 和 api

`implementation` 和 `api` 是取代之前的 `compile` 的，其中 `api` 和 `compile` 是一样的效果，`implementation` 有所不同，通过 `implementation` 依赖的库只能自己库本身访问，举个例子，A依赖B，B依赖C，如果B依赖C是使用的`implementation` 依赖，那么在 A中是访问不到C中的方法的，如果需要访问，请使用`api`依赖

> **那为什么要用它们来区分开原来的 compile 呢 ？**
>
> 1. 加快编译速度。
> 2. 隐藏对外不必要的接口。
>
> **为什么能加快编译速度呢？**
>
> 这对于大型项目含有多个`Module`模块的， 以上图为例，比如我们改动 `LibraryC` 接口的相关代码，这时候编译只需要单独编译`LibraryA`模块就行， 如果使用的是`api`或者旧时代的`compile`，由于`App Module` 也可以访问到 `LibraryC`,所以 `App Module`部分也需要重新编译。当然这是在全编的情况下。

```
目前在gradle-5.6.4 中测试, api 找不到了,似乎已被废弃
implementation 确实只将依赖的作用范围限定在当前模块内

建议一些公共使用的依赖, 例如slf4j, apache commons 下的工具包等还是用 compile 依赖

对于针对性功能较强的依赖, 如 spring, flink, 这些建议使用 implementation依赖
```

### compile only

`compile only` 和 `provided` 效果是一样的，只在编译的时候有效， 不参与打包

### runtime only

`runtimeOnly` 和 `apk`效果一样，只在打包的时候有效，编译不参与 ( 用得非常少 )

### test implementation

`testImplementation` 和 `testCompile` 效果一样，在单元测试和打包测试apk的时候有效

### debug implementation

`debugImplementation` 和 `debugCompile` 效果相同， 在 `debug` 模式下有效

### release implementation

`releaseImplementation`和 `releaseCompile`效果相同，只在 `release` 模式和打包release包情况下有效

