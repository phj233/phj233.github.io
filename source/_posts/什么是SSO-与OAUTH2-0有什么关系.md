---
title: 什么是SSO 与OAuth2.0有什么关系
date: 2023-02-15 09:37:39
categories:
- Program
tags:
- 笔记
- OAuth2.0
- SSO
- Spring Security
---

`OAuto2.0`使用场景通常为**联合登陆**。一处注册，多处使用。

`SSO`使用场景为**单点登录**。一处登录，多处同时使用。eg：淘宝登录/退出后，天猫同时登陆退出。

> `SSO`的实现关键是将Session信息集中存储。

