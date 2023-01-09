# 说明

master 分支能用，但是只支持到安卓7

functional 是最新分支，但是代码不全

反编译 [svxp64_210531_tissot.apk](https://github.com/asLody/SandVXposed/releases/download/V2.1.4/svxp64_210531_tissot.apk) 看到有混淆

# 编译 master 分支

## 各种版本

- Android Studio 3.6.1
- Gradle 5.6.4
- JDK 1.8
- NDK 21.4

```sh
cd /path/to/SandVXposed/
gradlew.bat build
```

这样编译出来可以在【小米note1(魔趣)】上运行成功