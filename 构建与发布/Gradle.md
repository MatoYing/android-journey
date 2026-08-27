#### 简介
首先，用两个单词形容它是什么，它是一个 Build Tool。
Ant 可以自动化打包逻辑（javac、创建目录、复制文件等），Maven 也可以自动化打包，相比于 Ant 写起来比较麻烦，更多是去做 Jar 包的版本管理。而 Gradle 是就是它俩的结合。而且 Gradle 基于 Groovy 语言，比起 XML 更加精简、灵活。
和其它构建工具的区别：
**1）Ant**
它本质上是用 XML 编写构建脚本，定义一系列的任务，比如编译、拷贝资源、压缩 JAR/WAR、部署等。
- 优点：比较灵活，可以精确控制构建过程的每一个细节。
- 缺点：缺乏依赖管理，必须手动下载所有第三方 Jar 包，并将它们添加到项目路径。
**2）Maven**
Maven 旨在解决 Ant 缺乏依赖管理和项目结构混乱的问题。
在 Maven 只需要在 `pom.xml` 中声明需要的库和版本，Maven 就会自动从远程仓库下载这些 Jar 包。
并且，Maven 规定了标准的项目目录结构，源代码必须在 `src/main/java` 下、资源在 `src/main/resources` 下，测试在 `src/test/java…` 下。它引入**约定大于配置的理念**，就是先给你一套合理的默认规则，只在你要改变默认行为时才配置。基于约定，你不用配置编译路径，就可以直接 `mvn package`，自动编译、测试、打包。
Maven 引入了“约定优于配置”（Convention over Configuration）的理念，极大地简化了项目管理。它通过规定一套标准的默认目录结构（如源代码在 `src/main/java`、资源文件在 `src/main/resources` 等）来建立合理的默认规则。基于这些约定，开发者无需手动配置编译路径和打包参数，系统即可自动进行处理，根据相应生命周期（validate、compile、test、package、install、deploy）里的操作，就能自动完成编译、测试、打包等流程。只有当你需要偏离这些默认行为时，才需要编写额外的配置。
- 优点：强大的依赖管理，和项目结构清晰。
- 缺点：灵活性差，因为声明周期是固定的，一些定制化的构建流程比较难操作。
**3）Gradle**
Gradle 旨在结合 Ant 的灵活性和 Maven 的依赖管理与项目目录结构。
Gradle 使用 Groovy 或 Kotlin 语言编写脚本，脚本语言比 XML 具有更高的表达力，这使得定制构建逻辑变得非常简单和强大。
并且，它完全支持 Maven 的依赖管理机制，可以使用 Maven 仓库。它也采用了 Maven 的标准化项目结构约定。
性能上，Gradle 引入了增量构建（只编译发生变化的部分）等，性能在三者间也更强些。
- 优点：既有 Maven 优秀的依赖管理和项目结构，又有 Ant 强大的自定义能力，脚本编写更简洁高效。
- 缺点：学习曲线陡峭。
**总结**
- Ant = 自动化构建脚本工具（无依赖管理）
- Maven = 构建 + 依赖管理（生命周期固定，可扩展但灵活性一般）
- Gradle = 构建 + 依赖管理 + 可编程脚本（性能更好，表达力更强）
#### 常用命令
+  `./gradlew assembleDebug`：构建所有渠道的 Debug 变体。也可以，构建指定渠道的 Debug 变体，比如有 local8155 渠道，则 `./gradlew assemblelocal8155Debug`。
+ `./gradlew clean`：清理编译缓存。经常重复编译就会报错，可能是有些缓存冲突，需要用到这个。
+ `./gradlew test`：运行所有模块的单元测试。
+ `./gradlew :drive:dependencies`：查看 Drive 模块的依赖树。
+ `./gradlew :domain:publishToMavenLocal`：将 domain 模块的构建产物发布到本地 Maven 仓库（~/.m2/repository/），本地其它项目只需配置 `mavenLocal()` 即可直接引用，无需发布到远程 Maven 仓库。
#### Android Gradle 构建系统（AGP）
AGP = Android Gradle Plugin，即 Android 的 Gradle 插件。就是每个模块 build.gradle 顶部 apply 的这个东西：
```groovy
apply plugin: 'com.android.library'
```
它提供了 `android {}` 这个 DSL 代码块（Domain Specific Language，领域特定语言，专门为某个领域设计的一套"配置语法"），里面的 compileSdk、defaultConfig、productFlavors、buildTypes、missingDimensionStrategy 等全部是 AGP 定义的标准 API。
简单理解：Gradle 本身只是构建工具，AGP 让 Gradle 能构建 Android 项目。
Android 多模块项目中，每个模块可以定义自己的 flavor dimension（维度）和 product flavor（变体）。当模块 A 依赖模块 B 时：
维度名相同 → Gradle 自动匹配同名 flavor，无需额外配置
维度名不同 → Gradle 无法匹配，构建直接报错


```groovy
drive 模块：dimension = "drive"，flavors = cadi, vcupro, clea8775 ...
common 模块：dimension = "common"，flavors = cadi, vcupro, clea8775 ...
vehicledomain 模块：dimension = "vehicledomain"，flavors = vcupro, local8155 ...
```


#### 构建变体（Build Variant）
它的核心公式是：Build Variant = Product Flavor + Build Type（构建变体 = 产品变体 + 构建类型）
**1）Build Type**
https://aistudio.google.com/prompts/1GJ4Y3JNaQcy1HUC2_wgrQrxR8UQZBb2e


![[../assets/Pasted image 20260819111419.png]]
![[../assets/Pasted image 20260819111452.png]]
![[../assets/Pasted image 20260819111616.png]]
![[../assets/Pasted image 20260819111709.png]]
![[../assets/Pasted image 20260819111756.png]]
![[../assets/Pasted image 20260819111839.png]]


![[../assets/Pasted image 20260819112009.png]]
![[../assets/Pasted image 20260819112031.png]]
![[../assets/Pasted image 20260819112053.png]]











#### missingDimensionStrategy

Android 多模块项目中，每个模块可以定义自己的 flavor dimension（维度）和 product flavor（变体）。当模块 A 依赖模块 B 时：
维度名相同 → Gradle 自动匹配同名 flavor，无需额外配置
维度名不同 → Gradle 无法匹配，构建直接报错


Gradle 不知道该选目标模块的哪个 flavor → 编译失败。

当依赖的目标模块有当前模块没有的 flavor dimension

触发条件
依赖的目标模块有当前模块没有的 flavor dimension
作用
为缺失的维度指定默认 flavor，实现跨模块自动匹配
```groovy
// 所有flavor都依赖:common
implementation project(path: ':common')

productFlavors {
	// 各flavor自己指定依赖:common的哪个flavor
	vcupro {
	    missingDimensionStrategy "common", "vcupro"
	}
}
```



#### Maven 发布
`apply plugin: 'maven-publish'`

```shell
JAVA_HOME="/Users/mato/Library/Java/JavaVirtualMachines/corretto-18.0.2/Contents/Home" bash ./gradlew :domain:publishToMavenLocal

JAVA_HOME="/Users/mato/Library/Java/JavaVirtualMachines/corretto-18.0.2/Contents/Home" bash ./gradlew :model:publishToMavenLocal
```




### 如何验证 compileOnly 依赖在目标设备上是否可用
最直接有效的方式是通过 Class.forName 运行时验证。
```Java
try {
    Class<?> clazz = Class.forName("com.cls.vehicle.v1.OSRVM2", true, getClassLoader());
    Log.d("Verify", "类加载成功，ClassLoader: " + clazz.getClassLoader());
} catch (ClassNotFoundException e) {
    Log.e("Verify", "类不存在 (ClassNotFoundException)", e);
} catch (NoClassDefFoundError e) {
    Log.e("Verify", "类定义缺失或其依赖项缺失 (NoClassDefFoundError)", e);
} catch (Throwable t) {
    Log.e("Verify", "初始化或加载失败", t);
}
```
