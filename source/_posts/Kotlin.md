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
```

