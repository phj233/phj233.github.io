---
title: Git 提交合并与大文件历史清理记录
date: 2026-06-24 23:30:00
categories:
- Program
tags:
- git
- 笔记
- 工程化
---

最近在一个前端项目里做了一次 Git 提交整理：本来已经按文档、构建、素材、功能拆了几次 commit，但后来发现素材提交里有一个问题——早期提交进来了一批 PNG 大图，后续又改成 WebP 并删除了旧 PNG。虽然最终工作区里已经没有旧图了，但只要这些 PNG 曾经出现在可达历史里，第一次 push 仍然可能把它们一起传上去。

所以这次不是普通的“改 commit message”，而是做了一次本地历史重写：把最终文件树重新整理成几条干净提交，让 `main` 历史里只保留当前真正需要的文件。

## 一、先看问题

开始前先看工作区和提交历史：

```bash
git status --short --branch
git log --oneline --reverse
git count-objects -vH
```

当时历史大概是这样：

```text
docs(project): 初始化协作与产品设计文档
chore(build): 初始化前端构建配置
chore(assets): 添加修仙游戏视觉资源
feat(frontend): 搭建修仙 Roguelike 可玩界面
chore(assets): 添加 RPG WebP 视觉素材
feat(rpg): 重构单局系统与战斗表现
chore(assets): 清理旧 PNG 素材并更新索引
docs(project): 更新 RPG 系统与架构说明
```

表面上看已经有一个 `chore(assets): 清理旧 PNG 素材并更新索引`，最终文件也干净。但 Git 的问题在于：**删除文件只是让最新提交不再引用它，不代表历史里没有它。**

如果这些提交还没有 push，最省事的处理方式就是在本地重写历史。

## 二、确认 dist 和 node_modules 没入库

大文件处理前，先确认构建产物和依赖目录没有被提交：

```bash
git ls-files 'frontend/dist' 'frontend/node_modules' 'dist' 'node_modules'
git status --short --ignored
```

`git ls-files` 没有输出，说明这些路径不是 tracked 文件。

`git status --ignored` 里看到：

```text
!! frontend/dist/
!! frontend/node_modules/
```

这表示它们只是被 `.gitignore` 忽略了，没有进 Git。后面又补了一条：

```gitignore
.vite
```

避免 Vite 缓存目录也冒出来。

## 三、为什么不是简单 squash

如果只是把相邻的几个资产 commit squash 到一起，有时确实能解决问题。但这里有两个容易混淆的点：

1. **合并提交信息**不等于清理历史对象。
2. 如果最终合并后的 commit 仍然能从历史路径访问到旧文件，大文件还是会被 push。

这次仓库还没有正式推送这些提交，所以直接选择更干净的方式：用 orphan 分支按最终工作树重新建一条历史。

这种方式适合：

- 本地提交还没 push。
- 想把初始历史重新整理得更干净。
- 不想在交互式 rebase 里一条条处理大量素材提交。

如果提交已经被别人拉取过，就要谨慎了。重写公共分支需要 force push，会影响协作者。

## 四、用 orphan 分支重建历史

核心命令是：

```bash
git switch --orphan codex/rewrite-assets-history
```

`--orphan` 会创建一个没有父提交的新分支。它适合用来“保留当前文件结果，但重新安排提交历史”。

切 orphan 分支后，如果工作区文件出现变化，不要慌。可以从旧 `main` 把最终文件树检出来：

```bash
git checkout main -- .
```

然后先清空索引，只保留工作区文件：

```bash
git rm --cached -r .
```

这一步不会删除真实文件，只是把暂存区清掉，方便重新分组提交。

## 五、重新按模块提交

接下来按模块重新 `git add`。

第一组：文档和架构说明。

```bash
git add .impeccable/design.json \
  AGENTS.md CLAUDE.md DESIGN.md PRODUCT.md README.md \
  backend/README.md frontend/README.md frontend/src/README.md \
  frontend/src/features/rpg/ARCHITECTURE.md

git commit -m "docs(project): 初始化项目与 RPG 架构文档"
```

第二组：构建配置。

```bash
git add .gitignore .vscode/extensions.json \
  frontend/index.html frontend/package-lock.json frontend/package.json \
  frontend/public/favicon.svg \
  frontend/tsconfig.app.json frontend/tsconfig.json frontend/tsconfig.node.json \
  frontend/vite.config.ts

git commit -m "chore(build): 初始化前端构建配置"
```

第三组：当前真正需要的素材。

```bash
git add frontend/src/assets
git commit -m "chore(assets): 添加 RPG 当前视觉素材"
```

这一步的重点是“当前”。旧的 PNG 不再经历“先添加、再删除”的历史链路，最终 `main` 里只保留当前要用的 WebP、少量 PNG 源素材和资产说明。

第四组：前端 RPG 源码。

```bash
git add frontend/src ':!frontend/src/assets'
git commit -m "feat(rpg): 搭建修仙 Roguelike 单局系统"
```

最后补一条 Vite 缓存忽略：

```bash
git add .gitignore
git commit -m "chore(build): 忽略 Vite 缓存目录"
```

最终历史变成：

```text
067f597 docs(project): 初始化项目与 RPG 架构文档
4a26153 chore(build): 初始化前端构建配置
d036f9d chore(assets): 添加 RPG 当前视觉素材
cb76c14 feat(rpg): 搭建修仙 Roguelike 单局系统
894e24c chore(build): 忽略 Vite 缓存目录
```

## 六、替换本地 main

orphan 分支确认没问题后，把它切回 `main`：

```bash
git branch -D main
git branch -m main
```

这里删除的是本地旧 `main` 引用，不是删除工作区文件。新 `main` 指向刚刚整理好的干净历史。

如果后续要推远端，并且远端已经有旧历史，需要：

```bash
git push --force-with-lease origin main
```

如果远端还是空的，普通 push 就行：

```bash
git push origin main
```

`--force-with-lease` 比 `--force` 稍微安全一点，它会检查远端分支是否被别人更新过。

## 七、检查旧大文件是否还在 main 历史

重写后不要只看 `git status`，还要查可达历史：

```bash
git rev-list --objects main | rg 'frontend/src/assets/(dao-battle-spirits|dao-event-atlas|dao-rpg-atlas)\.png'
```

没有输出，说明这些旧 PNG 路径不在 `main` 可达历史里。

如果使用：

```bash
git rev-list --objects --all
```

可能仍然会看到旧文件。这不一定代表 `main` 有问题，因为本地工具可能会保留一些内部 ref，比如：

```text
refs/codex/turn-diffs/...
```

这些不是业务分支，正常 `git push origin main` 不会推它们。判断要 push 的内容时，应优先看 `main` 或目标分支的可达历史，而不是盲目看 `--all`。

## 八、最终验证

最后跑构建和状态检查：

```bash
cd frontend
npm run build
```

再回到仓库根目录：

```bash
git status --short --branch --ignored
git log --oneline --reverse
```

状态里只剩 ignored 文件：

```text
!! frontend/.vite/
!! frontend/dist/
!! frontend/node_modules/
```

这就说明依赖、缓存和构建产物都没有进 Git。

## 九、小结

这次操作可以总结成一句话：

> 如果大文件只是被删除了，但曾经出现在提交历史里，push 时仍然可能被上传；在未发布的本地分支上，可以通过重写历史把最终文件树整理成干净提交。

常用判断：

- 只是改最近一条提交信息：`git commit --amend`
- 合并最近几条普通提交：`git rebase -i`
- 初始历史混乱且还没 push：`git switch --orphan` 重建历史
- 已经 push 且有协作者：先沟通，再考虑 `--force-with-lease`

Git 的提交合并不是为了“看起来少几条 commit”，更重要的是让历史表达真实意图：文档是文档，构建是构建，资产是当前资产，源码是源码。这样以后 review、回滚、push 都轻松很多。
