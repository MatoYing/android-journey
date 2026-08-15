#### 常用命令
+  `./gradlew assembleDebug`：构建 Debug APK，可输出日志。比如有 local8155 渠道，则 `./gradlew assemblelocal8155Debug`
+ `./gradlew assembleRelease`：构建 Release APK，供上线使用。
+ `./gradlew clean`：清理编译缓存。经常重复编译就会报错，可能是有些缓存冲突，需要用到这个。
