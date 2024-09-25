---
title: 在原生HTML及Vue中方便使用图标
date: 2024-09-24 22:21:49
categories:
- Program
tags:
- VUE
- 前端
- 笔记
---

# # 准备工作

- 技术栈: HTML + CSS 
- Node.js VUE3 (可选)

# 1.打开iconify design

[Iconify](https://icon-sets.iconify.design/)  

https://icon-sets.iconify.design

# 2.  选择喜欢的图标集

![image-20240924224832352](https://s2.loli.net/2024/09/24/LNeaizPIjQUC4Yh.png)

# 3. 进入图标集后选择喜欢的图标

图标的size、color属性可直接在此窗口自定义

![image-20240924225228154](https://s2.loli.net/2024/09/24/azkPqYdj9AVmwMy.png)



# 4. 代码操作

## 4.1 对于原生 HTML

点击 `SVG` 中的 **SVG** 按钮后再点击下方文本块中的 `SVG` 代码，之后在html中粘贴

想自定义样式之类的可以在外面包 `div` 或者 `a` 标签，这里不再赘述

![image-20240924225735941](https://s2.loli.net/2024/09/24/lOhT39AtPeYzsm1.png)

```html
<!DOCTYPE html>
<html lang="zh">
<style>
</style>
<head>
    <title>Test</title>
    <meta charset="UTF-8">
</head>
<body>
<svg xmlns="http://www.w3.org/2000/svg" width="64" height="64" viewBox="0 0 24 24">
    <path fill="black" d="M6 17c0-2 4-3.1 6-3.1s6 1.1 6 3.1v1H6m9-9a3 3 0 0 1-3 3a3 3 0 0 1-3-3a3 3 0 0 1 3-3a3 3 0 0 1 3 3M3 5v14a2 2 0 0 0 2 2h14a2 2 0 0 0 2-2V5a2 2 0 0 0-2-2H5a2 2 0 0 0-2 2" />
</svg>
</body>
</html>

```

## 4.2 对于原生CSS

点击 `CSS` 中的 **CSS **按钮

点击 `CSS` 代码块复制 **CSS** 代码

点击 `HTML` 代码块复制 **HTML** 代码

![image-20240924230732771](https://s2.loli.net/2024/09/24/wTfGhokQiSymeq4.png)

![image-20240924231041218](https://s2.loli.net/2024/09/24/J92uN6SCft4dTUK.png)

```html
<!DOCTYPE html>
<html lang="zh">
<style>
    .mdi--account-box {
        display: inline-block;
        width: 64px;
        height: 64px;
        background-repeat: no-repeat;
        background-size: 100% 100%;
        background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 24 24'%3E%3Cpath fill='black' d='M6 17c0-2 4-3.1 6-3.1s6 1.1 6 3.1v1H6m9-9a3 3 0 0 1-3 3a3 3 0 0 1-3-3a3 3 0 0 1 3-3a3 3 0 0 1 3 3M3 5v14a2 2 0 0 0 2 2h14a2 2 0 0 0 2-2V5a2 2 0 0 0-2-2H5a2 2 0 0 0-2 2'/%3E%3C/svg%3E");
    }
</style>
<head>
    <title>Test</title>
    <meta charset="UTF-8">
</head>
<body>
<span class="mdi--account-box"></span>
</body>
</html>

```



## 4.3  对于 VUE  (可选)

![image-20240924231316017](https://s2.loli.net/2024/09/24/obKfjU2dwa6LPzn.png)

个人建议使用 **iconify** 推荐的 [Iconify for Vue](https://iconify.design/docs/icon-components/vue/) 



### 4.3.1 安装 Iconify for Vue

在 `vue` 项目根路径下 输入

```bash
npm install --save-dev @iconify/vue
```

![image-20240924231558145](https://s2.loli.net/2024/09/24/v9ULiB4Oz3xhTYG.png)



### 4.3.2 在相关 **VUE** 组件中写入代码

点击复制 `4.3.1` 图片中的 `Icon` 代码

```vue
<script setup lang="ts">

import {Icon} from "@iconify/vue";
</script>

<template>
<n-card hoverable class="w-4/5 h-80">
  <n-flex>

  </n-flex>
</n-card>
</template>

<style scoped>

</style>

```

必须导入 `@iconify/vue` 中的 Icon

![image-20240924232402750](https://s2.loli.net/2024/09/24/9RzohuiVLxPcHZw.png)
