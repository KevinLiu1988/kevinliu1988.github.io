# 20201208 Android Gradle Plugin迎来7.X版本

## 前言

通过Android Weekly了解到，在12.1日Android Gradle Plugin（以下简称AGP）迎来了7.0版本。忽然一下子有些迷惑，之前不是还在3、4的版本吗？为什么一下子跳到了7，仔细看了下来大致了解到了。后续会将AGP和Android Studio的版本拆开来单独发展，之所以跳版本号是因为目前Gradle是7版本，乍以为以后都会跟随Gradle的主版本，但是官方又说此版本后续也会支持8。当前也是出于Alpha版本，想必日后的变数还会比较多吧。下边大概介绍下官方宣布的主要内容。



## 官宣内容

### 新版本控制

从7.0开始引入了 [语义化版本](https://semver.org/lang/zh-CN/) 的概念，其内容可以大意概括为：

- 版本号由【主版本】•【次版本】•【修订版本】构成
- 只要主版本不动，那么API都是可以兼容的，动了就代表API产生了不兼容的情况

更新的频率会控制在每年一个主版本，会正好在Gradle的年度大版本更新之后。



会提前一年对要删除的API做废弃（`@Deprecated`）标识。

### 需要Java11

AGP 7.X需要最小Java JDK 11的支持。

### API

包含酝酿中的API和重要的API变化。

在AGP4.1版本里孵化中的API将来也可能发生变化，这些API先不遵循上述说到的废弃原则。

#### 重要的API变化

- `onVariants` | `onProperties` | `onVariantProperties` 代码块将被删除（在AGP 4.2 beta中）
- 以上被替换成了`androidComponents`代码块中的`beforeVariants` | `onVariants`，`VariantSelector`可以用来生成以上DSL依赖的`variant`回调，这些回调会在AGP通过`afterEvcaluate`计算完变体组合后触发
- `onVariants`中嵌套的属性被删除了
- 其他的如`buildType` | `name` | `flavor`可以用`withBuildType` | `withName` | `withFlavor` 来类似的使用
- `androidComponents`中还添加了测试相关的`beforeUnitTest` | `unitTest` | `beforeAndroidTest` | `androidTest`
- 两个类重命名了：原来的`Variant`命名为`VariantBuilder`，在`beforeVariants`中使用。`VariantProperties`命名为`Variant`，作为`onVariants`代码块的入参

#### 示例

以`onVariants`和`beforeVariants`为例：

```groovy
android {
   ...
   //onVariants.withName("release") {
   //   ...
   //}
  
   //onVariantProperties {
   //   ... 
   //}
   ...
}
androidComponents {
   val release = selector().withBuildType("release")
   beforeVariants(release) { variant ->
      ...
   }
  
  onVariants { variant ->
  	...
  }
}
```

#### 注意

官方还提到，以上的改变应该是在插件内部代码中修改的，而不应该是在你的工程中的build.gradle。Gradle的插件作者需要在7.0稳定后尽快适配，但不应该依赖那些孵化中的API。

## 引用

- [AGP 7.0](https://android-developers.googleblog.com/2020/12/announcing-android-gradle-plugin.html)

