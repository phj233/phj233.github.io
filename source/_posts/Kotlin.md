---
title: Kotlin
date: 2024-07-08 17:43:39
categories:
- Program
tags:
- Gradle
- Kotlin
- 笔记
---

# 1.引用 `kotlinx-serialization-json` 或其他kotlinx序列化时无法使用注解

在 `build.gradle.kts` 中添加

```kotlin
plugins {
    kotlin("plugin.serialization") version "你的kotlin编译器版本"
}
...
```



# 2.kotlin + gradle 打包 `jar` 并包含依赖

在 `build.gradle.kts` 中添加 `shadow` 插件，并为打包添加主清单(程序入口类)

```kotlin
plugins {
    id("com.github.johnrengelman.shadow") version "8.1.1"
}
...
tasks.jar {
    manifest {
        attributes(mapOf("Main-Class" to "top.phj233.simbot.SimBotKt"))
    }
}
...
```



3.在idea使用gradle时自动下载依赖的源文件以及文档

在 `build.gradle.kts` 中添加

```kotlin
idea {
    module {
        isDownloadJavadoc = true
        isDownloadSources = true
    }
}
```

