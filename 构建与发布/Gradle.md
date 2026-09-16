### 简介

首先，用两个单词形容它是什么，它是一个 Build Tool。和其它构建工具的区别：

**1）Ant**

Apache Ant 是一种基于 XML 的构建工具，通过 build.xml 定义编译、打包、部署等 task。缺点：无依赖管理能力，需手动管理 JAR 包；XML 冗长、可读性差，构建逻辑难以复用；无标准生命周期和项目结构约定，维护成本高。现已过时。

**2）Maven**

Apache Maven 是一种基于 POM（pom.xml）的项目管理和构建工具，核心理念是**约定大于配置**，提供标准的项目结构（源代码必须在 `src/main/java` 下、资源在 `src/main/resources` 下，测试在 `src/test/java` 下）和构建生命周期（clean → compile → test → package → install → deploy）。原生支持依赖管理，通过坐标（groupId:artifactId:version）自动从中央仓库下载依赖并处理传递依赖。缺点：XML 配置仍然冗长；构建流程固定、灵活性不足，复杂自定义构建场景写起来笨重；大型多模块项目构建速度较慢。后被 Gradle 在灵活性和性能上超越，但在传统 Java 后端项目中仍广泛使用。

**3）Gradle**

Gradle 旨在结合 Ant 的灵活性和 Maven 的依赖管理与项目结构约定。使用 Groovy 或 Kotlin 编写构建脚本，相比 XML 具有更高可读性，定制构建逻辑更简洁强大。完全兼容 Maven 仓库和依赖管理机制，同时采用标准化的项目目录结构。性能方面，引入增量构建（仅编译变更部分）、构建缓存和并行构建，在三者中最优。

**总结**

+ Ant = 构建（无依赖管理）
+ Maven = 构建 + 依赖管理（标准生命周期，约定大于配置，灵活性一般）
+ Gradle = 构建 + 依赖管理 + 可编程脚本（兼具两者优点，性能更强）

### 常用命令

下面都会降到，这里做一个汇总，方便查阅：

bash，地址

+  `./gradlew assembleDebug`：构建所有渠道的 Debug 变体。也可以，构建指定渠道的 Debug 变体，比如有 local8155 渠道，则 `./gradlew assemblelocal8155Debug`。
+ `./gradlew clean`：清理编译缓存。经常重复编译就会报错，可能是有些缓存冲突，需要用到这个。
+ `./gradlew test`：运行所有模块的单元测试。
+ `./gradlew :drive:dependencies`：查看 Drive 模块的依赖树。
+ `./gradlew :domain:publishToMavenLocal`：将 domain 模块的构建产物发布到本地 Maven 仓库（~/.m2/repository/），本地其它项目只需配置 `mavenLocal()` 即可直接引用，无需发布到远程 Maven 仓库。
### Android Gradle 构建系统（AGP）

AGP = Android Gradle Plugin，即 Android 的 Gradle 插件。就是每个模块 build.gradle 顶部 apply 的这个东西：
```groovy
apply plugin: 'com.android.application'   // 用来生成APK
apply plugin: 'com.android.library'       // 用来生成AAR
```
Gradle 本身只是构建工具，AGP 提供了 `android {}` DSL（Domain Specific Language，领域特定语言，专门为某个领域设计的一套"配置语法"）让 Gradle 能构建 Android 项目。核心功能如下：

+ 编译：`.java` / `.kt` → `.class` → `.dex`。
+ 资源处理：编译 `res/`、`assets/`、`AndroidManifest.xml`，生成 `R.java`。
+ 打包：代码 + 资源 + SO 库 → APK/AAR。
+ 签名：对 APK 签名。
+ 混淆/压缩：集成 ProGuard / R8。
+ 依赖管理：解析 Maven 依赖、处理冲突。
+ 构建变体：Flavor × BuildType 生成不同版本。
+ 测试：单元测试、覆盖率报告。
+ 增量构建：缓存 + 增量编译，加速构建。

整体结构大概如下：

```
android {
    defaultConfig { }           →  所有变体的基线默认值
    signingConfigs { }          →  签名配置
    flavorDimensions            →  定义几个维度
    productFlavors { }          →  每个维度下有哪些flavor
    buildTypes { }              →  怎么打包（debug/release/自定义）
    sourceSets { }              →  每个flavor用哪套源码
    buildFeatures { }           →  功能开关（BuildConfig/ViewBinding等）
    variantFilter { }           →  排除不需要的组合
    missingDimensionStrategy    →  依赖库没有匹配flavor时的降级策略
    compileOptions { }          →  Java兼容版本
    packagingOptions { }        →  打包资源冲突处理
    testOptions { }             →  测试配置
    lint { }                    →  Lint检查配置
}

dependencies {
    flavor + 依赖项              →  flavor专属
    buildType + 依赖项           →  构建类型专属
    flavor + buildType + 依赖项  →  组合专属
}
```

#### 构建变体（Build Variant）

构建变体是 AGP 中用于从同一套代码产出不同版本 APK/AAR 的机制，也是最核心的功能。它由两个维度组合而成：Build Variant = Product Flavor × Build Type（构建变体 = 产品变体 × 构建类型）

**1）Build Type**

用于定义构建和打包应用时的不同配置，常见结构如下：

```groovy
signingConfigs {
    release {
        storeFile file("keystore/release.jks")
        storePassword "xxx"
        keyAlias "xxx"
        keyPassword "xxx"
    }
}

buildTypes {
    debug {
        applicationIdSuffix ".debug"
        versionNameSuffix "-debug"
    }

    release {
        minifyEnabled true
        shrinkResources true
        proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        signingConfig signingConfigs.release
    }
}
```

常见配置项：

+ debuggable：是否允许 Java/Kotlin 层 Debug（debug 默认为 true，release 默认为 false）

+ jniDebuggable：是否允许 Native 层（C/C++）Debug（debug 默认 true，release 默认 false），关闭的话包体积更小（编译器会移除 .so 文件中的调试信息，比如变量名、函数名、源码行号映射表等）

+ signingConfig：指定使用哪个签名配置来对 APK 进行签名。每个 APK 必须有数字签名才能安装和运行（AAR 用不着签名）。

  ```java
  // 签名是应用的唯一身份证明，决定应用能否覆盖安装（签名不同=不同应用）；同时也是signature级别权限的授予依据，只有证书匹配才能获得对应系统权限；此外还用于确保APK内容未被篡改
  signingConfigs {
      debug {
          storeFile file("keystore/debug.jks")  // 密钥库文件路径，一个keystore文件里可以存放多对密
          storePassword "debug123"  // 密钥库密码
          keyAlias "debug"  // 密钥库其中一个密钥
          keyPassword "debug123"  // 该密钥密码
      }
      release {
          storeFile file("keystore/release.jks")
          storePassword "secure_password"  // 不一定要写在项目中，还可以系统读取的方式System.getenv("KEY_ALIAS")
          keyAlias "myapp"
          keyPassword "secure_password"
      }
  }
  
  buildTypes {
      debug {
          signingConfig signingConfigs.debug
      }
      release {
          signingConfig signingConfigs.release
      }
  }
  ```

+ minifyEnabled：是否开启代码混淆/压缩（两者默认都关），搭配 proguardFiles 使用，它会指定混淆的规则。

+ shrinkResources：是否移除未使用资源（两者默认都关，依赖 minify 开启，因为在做 minify 时会分析所有代码路径，生成一个"哪些资源 ID 被使用了"的精确清单，shrinkResources 拿到这个清单才能安全地判断哪些资源可以删）

+ packaging：控制 APK 打包时的资源处理行为，哪些文件包含、哪些排除、native 库如何处理等。

  ```groovy
  // 以前是packagingOptions
  packaging {
      // 排除指定文件（减小包体积、解决冲突）
      resources {
          excludes += [
              'META-INF/INDEX.LIST',
              'META-INF/LICENSE',
              'META-INF/NOTICE'
          ]
      }
  
      // Native .so库相关
      jniLibs {
          keepDebugSymbols += ["**/*.so"]  // 允许所有.so Debug（以前是doNotStrip），而jniDebuggable只影响自己编译的native代码
          pickFirsts += ["**/libc++_shared.so"]  // 多个库包含同名.so时取第一个
      }
  }
  ```
  
+ applicationIdSuffix：包名后缀，在 applicationId 后面自动追加一段字符串，比如 com.patac.hmi.settings.debug。

+ versionNameSuffix：版本名后缀，在 versionName 后面追加一段字符串，仅用于展示，不影响安装行为（用户在"应用信息"页面能看到版本号，比如 1.0.0-debug）。

**2）Product Flavor**

Build Type 决定怎么构建（混淆、签名等），Product Flavor 决定构建什么版本（不同品牌、地区等）。常见的标准结构如下（把其他常见的内容也加了进来）：

```groovy
android {
    compileSdk 34

    defaultConfig {
        applicationId "com.example.myapp"
        minSdk 24
        targetSdk 34
        versionCode 1
        versionName "1.0"
    }

    flavorDimensions = ["tier", "env"]

    productFlavors {
        // 1）层级
        // 免费版
        free {
            dimension "tier"
            applicationIdSuffix ".free"
            buildConfigField "boolean", "IS_PRO", "false"
        }
        // 付费版
        pro {
            dimension "tier"
            applicationIdSuffix ".pro"
            buildConfigField "boolean", "IS_PRO", "true"
        }
        // 2）环境
        // 开发环境
        dev {
            dimension "env"
            buildConfigField "String", "BASE_URL", '"https://dev.api.com"'
        }
        // 生产环境
        prod {
            dimension "env"
            buildConfigField "String", "BASE_URL", '"https://api.com"'
        }
    }
    // 等同于下面这样写（当然，sourceSets也得变）
    // productFlavors {
    //     freeDev   { ... }
    //     freeProd  { ... }
    //     proDev    { ... }
    //     proProd   { ... }
    // }
  
    sourceSets {
        free {
            java.srcDirs = ['src/main/java', 'src/free/java', 'src/shared_promo/java']
            res.srcDirs = ['src/main/res', 'src/free/res']
        }
        pro {
            java.srcDirs = ['src/main/java', 'src/pro/java']
            res.srcDirs = ['src/main/res', 'src/pro/res']
        }
        dev {
            java.srcDirs = ['src/main/java', 'src/dev/java']
            res.srcDirs = ['src/main/res', 'src/dev/res']
        }
        prod {
            java.srcDirs = ['src/main/java', 'src/prod/java']
            res.srcDirs = ['src/main/res', 'src/prod/res']
        }
    }

    variantFilter { variant ->
        def names = variant.flavors*.name
        // 排除free+prod的组合
        if (names.contains("free") && names.contains("prod")) {
            setIgnore(true)
        }
    }

    buildTypes {
        release {
            minifyEnabled true
        }
        debug {
            applicationIdSuffix ".debug"
        }
    }
}

dependencies {
    // 所有flavor都会引入
    implementation 'androidx.core:core-ktx:1.12.0'
    // 仅paid引入
    proImplementation 'com.stripe:stripe-android:20.0.0'
    // 仅free引入
    freeImplementation 'com.google.ads:ads:1.0.0'
}
```

拆开来讲，首先 flavorDimensions 用于声明 flavor 的维度，可定义多个维度，Gradle 会自动对各维度的 flavor 做笛卡尔积，生成所有组合 variant。例如两个维度 tier（free / pro）× env（dev / prod），会构建出 freeDev、freeProd、proDev、proProd 四个 variant。

defaultConfig、productFlavors 定义了各个 flavor，可以自定义如下内容：

+ applicationId：应用包名（com.patac.hmi.settings），区分设备上不同的应用。

+ applicationIdSuffix：在 applicationId 后面自动拼接一段后缀。

+ versionName：给用户看的字符串版本号，纯展示用，在手机的“应用信息”可以看到。

+ versionCode：给系统看的整数版本号，新版本 versionCode 必须 > 已安装版本，每次发布必须递增，不可重复。

+ minSdk：声明你的应用最低支持哪个 Android 版本，低于这个版本的设备无法安装。也决定了决定了代码中能直接使用哪些 API。另外应用商店会根据这个值过滤，不向低版本设备展示你的应用。

+ buildConfigField：在编译时往 BuildConfig 类中注入自定义常量，让代码能读到构建配置中的值。`buildConfigField "类型", "字段名", "值"`，可以有多个。

  ```java
  buildConfigField "String", "API_URL", "\"https://api.com\"
  // Java调用
  String url = BuildConfig.API_URL
  ```

+ resValue：类似 buildConfigField，在代码和布局中都能引用，`resValue "资源类型", "资源名", "值"`，`R.string.XXX `。

==这些属性，包括 Build Type 中介绍的，在 defaultConfig、productFlavors、buildTypes 中都能配置（buildConfigField 和 resValue 除外，只能放 defaultConfig 和 productFlavors）。整体是覆盖的关系 flavor > buildType > defaultConfig（applicationIdSuffix 是拼接而非覆盖）==

**3）sourceSets**

每个 flavor 可以拥有独立的源码、资源、Manifest。

**4）dependencies**

使用 `<flavorName>Implementation` 为特定 flavor 添加依赖。

**5）variantFilter**

多维度组合可能产生无意义的 variant，可以用 variantFilter 排除

**6）missingDimensionStrategy**

当引入定义了 productFlavors 的库依赖（AAR 或本地模块）时，有以下几种情况：

+ 维度名相同，有同名 flavor → 自动匹配，编译成功
+ 维度名相同，没有同名 flavor → 编译失败，需用 missingDimensionStrategy 指定
+ 维度名不同（AAR 有而项目没有）→ 编译失败，需用 missingDimensionStrategy 指定

missingDimensionStrategy 就是用来解决 Gradle 无法自动匹配 AAR 的 flavor 时，手动指定使用哪个 flavor 的问题。

```groovy
android {
    defaultConfig {
        // 告诉Gradle：A维度选a1 flavor，B维度选b1 flacor
        missingDimensionStrategy "A", "a1"
        missingDimensionStrategy "B", "b1"
    }
}
```

如果 AAR 没有定义 productFlavors，就完全不需要管 missingDimensionStrategy，直接用就行。

**7）常用命令**

最后这块常用的命令如下：

+ `./gradlew assembleDebug`：构建所有 flavor 的 Debug 变体（Gradle 最终构建的最小单元是 Variant，它是 Flavor 和 BuildType 的组合。构建命令格式为 `assemble<Flavor><BuildType>`）。
+ `./gradlew assembleRelease`：构建所有 flavor 的 Release 变体。
+ `./gradlew assembleVcuproDebug`：仅构建 vcupro 的 Debug 变体。
+ `./gradlew assemble`：构建全部变体。

#### APK 与 AAR





```
settings（APK）
    ├── common（AAR）
    ├── drive（AAR）
    ├── display（AAR）
    ├── ambientlight（AAR）
    ├── connections（AAR）
    ├── energy（AAR）
    ├── adassettings（AAR）
    ├── vehiclecontrol（AAR）
    ├── vehiclestatus（AAR）
    ├── userguide（AAR）
    ├── mcp（AAR）
    ├── vehicledomain（AAR）
    ├── feature-api（AAR）
    ├── feature-cn（AAR）
    ├── feature-manager（AAR）
    └── feature-overseas（AAR）
         ↓
    合并签名 → PATACSettings-xxx.apk
```











#### Maven Publish 依赖管理





`apply plugin: 'maven-publish'`

```shell
JAVA_HOME="/Users/mato/Library/Java/JavaVirtualMachines/corretto-18.0.2/Contents/Home" bash ./gradlew :domain:publishToMavenLocal

JAVA_HOME="/Users/mato/Library/Java/JavaVirtualMachines/corretto-18.0.2/Contents/Home" bash ./gradlew :model:publishToMavenLocal
```



#### 如何验证 compileOnly 依赖在目标设备上是否可用

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







#### Gradle 性能？增量构建、构建缓存、并行构建





### 签名机制

1. 用 storePassword 打开 keystore 文件
2. 用 keyAlias 找到对应的私钥
3. 用 keyPassword 解锁该私钥
4. 用私钥对 APK 进行签名



![image-20260913175639037](../../../../Library/Application Support/typora-user-images/image-20260913175639037.png)

![image-20260913175927393](../../../../Library/Application Support/typora-user-images/image-20260913175927393.png)

![image-20260913180021331](../../../../Library/Application Support/typora-user-images/image-20260913180021331.png)

![image-20260913180102251](../../../../Library/Application Support/typora-user-images/image-20260913180102251.png)







#### 多项目构建实战







这么设计的目的是什么

![image-20260915135944569](../../../../Library/Application Support/typora-user-images/image-20260915135944569.png)



![image-20260915140343959](../../../../Library/Application Support/typora-user-images/image-20260915140343959.png)





![image-20260915142249317](../../../../Library/Application Support/typora-user-images/image-20260915142249317.png)





Gradle生命周期

![image-20260915153131819](../../../../Library/Application Support/typora-user-images/image-20260915153131819.png)







![image-20260915155220732](../../../../Library/Application Support/typora-user-images/image-20260915155220732.png)
