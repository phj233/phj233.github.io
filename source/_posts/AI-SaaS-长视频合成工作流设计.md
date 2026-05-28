---
title: AI SaaS 长视频合成工作流设计
date: 2026-05-28 11:11:00
categories:
- Program
tags:
- 架构
- 后端
- AI
- 长视频
- SSE
---

一个 AI 创作平台的长视频功能：用户输入一个创意，后端自动完成意图分析 → 剧本生成 → 分镜拆分 → 视频生成 → 最终合成，全程通过 SSE 向用户推送进度。这里记录一下整体架构设计。

## 一、整体流程

```
用户输入意图
    ↓
Phase 1: 意图识别 → 生成剧本大纲 + 时长规划
    ↓   (文本模型，内部免费)
Phase 2: 剧本生成 → 完整的分幕剧情
    ↓   (文本模型，内部免费)
Phase 3: 分镜拆分 → 拆成 N 个镜头，每个镜头有画面描述 + 动作 + 对白 + 情绪 + 出图提示词
    ↓   (文本模型，内部免费)
Phase 4: 分镜渲染 → 每个镜头独立生成视频
    ↓   (视频模型，每个镜头独立扣费，失败退款)
Phase 5: 最终合成 → ffmpeg 本地合成所有已通过审核的分镜视频
    ↓   (本地处理，不额外收费)
输出: 封面 + 视频 → 进入作品列表
```

每个阶段之间通过 SSE 向用户推送进度，用户可以在任意阶段确认或修改后继续。

## 二、核心数据模型

### 项目主表

```sql
CREATE TABLE long_video (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  trace_id VARCHAR(64) NOT NULL COMMENT '项目追踪ID',
  account_id BIGINT NOT NULL,

  title VARCHAR(255),
  status VARCHAR(32) NOT NULL COMMENT
    'draft/processing/waiting_user/rendering/completed/failed/cancelled',
  current_stage VARCHAR(64) COMMENT
    'intent/script/storyboard/render_plan/shot_render/compose/done',

  video_model VARCHAR(128) NOT NULL COMMENT '用户选择的生视频模型',
  aspect_ratio VARCHAR(32) DEFAULT '16:9',
  duration INT COMMENT '目标总时长',

  stages JSON NOT NULL COMMENT '完整阶段状态（意图、剧本、分镜、渲染记录）',
  billing_summary JSON COMMENT '项目级费用汇总',
  final_result_url TEXT COMMENT '最终成片地址',
  cover_url TEXT COMMENT '封面地址',

  version INT NOT NULL DEFAULT 1 COMMENT '乐观锁版本',
  UNIQUE KEY uk_trace_id (trace_id),
  KEY idx_account_status (account_id, status)
);
```

### 项目与 AI 任务关系表

```sql
CREATE TABLE long_video_ai_task (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  long_video_id BIGINT NOT NULL,   -- 关联项目
  ai_task_id BIGINT NOT NULL,      -- 关联 AI 任务

  task_role VARCHAR(32) NOT NULL COMMENT
    'shot_render|final_render|reference_frame|shot_reference_frame',
  asset_scope VARCHAR(32) NOT NULL DEFAULT 'project_asset' COMMENT
    'project_asset|final_work|hidden',

  -- 分镜相关
  shot_index INT,
  render_id VARCHAR(64),           -- 渲染版本 ID: render_001_1
  selected TINYINT DEFAULT 0,     -- 用户是否选择此版本
  need_rerender TINYINT DEFAULT 0,
  billing_mode VARCHAR(32),       -- per_req/per_sec

  UNIQUE KEY uk_render_id (long_video_id, render_id),
  KEY idx_long_video_role (long_video_id, task_role)
);
```

设计要点：
- **不复用 `ai_task.task_type` 表达业务角色**。`task_type` 只保持 `image/video/3d` 的媒体类型语义，业务角色（shot_render、final_render 等）放在 `long_video_ai_task.task_role`
- **`asset_scope` 控制素材可见范围**。`project_asset` = 中间分镜素材不进入作品列表，`final_work` = 最终成片进入作品列表
- **乐观锁 `version`**：所有对 `stages` 的写入都带版本条件

## 三、分镜渲染与断点续跑

### 批量提交

用户确认分镜后，`render-all` 接口处理：

```java
// LongVideoService
// 批量渲染不是整体事务，每个分镜单独提交
// 串行模型（Seedance 1.5/2.0）→ 尾帧接力，前一个完成再提交下一个
// 非串行模型 → 后台批量模式，按 maxConcurrency 分批提交

public void renderAll(LongVideoDO project) {
    // 1. 判断模型类型
    boolean serialMode = isSerialModel(project.getVideoModel());

    // 2. 写入每个 shot 的 shotRender 为 rendering 状态
    //    记录 batchActive=true 或 chainActive=true
    writeShotRenderPending(project, serialMode);

    // 3. 串行模式：提交第一个分镜，上游完成后通过尾帧接力提交下一个
    //    批量模式：入队后快速返回，异步分批提交
    if (serialMode) {
        submitChainNext(project);
    } else {
        submitBatchFirstChunk(project);
    }
}
```

### 分镜创建 AI 任务

每个分镜走独立的 AI 任务创建流程：

```java
// AiProxyService.createGenericTaskForLongVideo
// 内部专用入口：
// - 显式传入 accountId（不依赖当前请求上下文）
// - 显式传入 traceId（每个分镜独立）
// - 显示传入 dedupBizKey（按 long_video_id + shot_id + render_id 去重）
// - 先落库 → 扣费 → 调上游 → 失败退款

// 扣费成功后上游失败，通过 ai_task.trace_id 退款
// consumption.request_id == ai_task.trace_id
// balance_log.ref_id == ai_task.trace_id
```

### 断点续跑

后端重启或前端重新进入时，需要恢复未完成的任务：

```java
// 定时扫描 + SSE/详情主动补查
// 1. 扫描 long_video.status=rendering AND current_stage=shot_render
// 2. 基于 long_video_ai_task 关系表找到 pending/processing 的分镜
// 3. 补交前先去重：
//    按 long_video_id + shot_id 查活动中的 shot_render
//    按 long_video_id + render_id 查已有关系
// 4. 命中 → 只同步快照，不重复提交
// 5. 未命中 → 提交上游
// 6. 自动恢复不重试失败分镜（避免重复扣费）
// 7. 用户显式点击 render-all 时可以重试失败的仍需渲染的分镜
```

---

## 四、SSE 事件推送

长视频使用 SSE 向前端实时推送进度。事件流设计：

```java
// LongVideoSseService
// 事件类型：
// - project_snapshot: 项目全量快照（核心事件）
// - heartbeat: 心跳保活
// - stage_updated: 阶段更新
// - shot_render_started: 分镜开始渲染
// - shot_render_updated: 分镜渲染进度更新
// - project_completed: 项目完成
// - project_failed: 项目失败
// - project_deleted: 项目已删除
//
// payload 是 projectResponse 的完整项目快照
// 不再使用旧的增量事件包络

// SSE 发送异常处理：
// 浏览器刷新/切换任务/网络断开 → Broken pipe → 清理连接，降级为 debug 日志
// 响应不可写时 → 不交给全局异常处理器写 JsonData
```

前端接收端处理：

```ts
// useSseChat 或长视频专用 SSE composable
// 1. 建立 EventSource 连接到 /api/v1/ai/long-video/events?traceId=xxx
// 2. 监听 project_snapshot 事件 → 更新本地项目快照
// 3. 监听 heartbeat → 仅保活，心跳间隔内无其他事件时不更新 UI
// 4. 连接断开 → 自动重连（带指数退避）
// 5. 页面不可见时 → 暂停重连，恢复后立即重连
```

---

## 五、最终合成

所有分镜视频完成后，ffmpeg 本地合成：

```java
// LongVideoFinalComposeService
// 1. 检查合成条件：
//    - 每个分镜 currentRenderId 对应 render 均为 completed
//    - 每个完成的分镜都有 resultUrl
// 2. 下载已选分镜视频到本地临时目录
// 3. ffmpeg concat 合成
// 4. 上传到 COS
// 5. 创建 final_render 的 ai_task 记录
//    写入 long_video_ai_task.task_role=final_render, asset_scope=final_work
// 6. 最终成片进入普通作品列表
// 
// ffmpeg 本地合成不额外收费
// 后续 Vue 剪辑器如果允许用户剪辑单镜头并上传新镜头，剪辑后上传再单独计费
```

---

## 六、核心不可变规则

1. **每个分镜 render 必须有独立 `ai_task.trace_id`**。决不允许多个分镜共用同一个 traceId
2. **先创建 `ai_task` → 再扣费 → 再调上游**。扣费成功后上游失败，必须通过 `ai_task.trace_id` 退款
3. **`consumption.request_id` == `ai_task.trace_id`**，`balance_log.ref_id` == `ai_task.trace_id`
4. **中间分镜素材不进入普通作品列表**（`asset_scope=project_asset`）
5. **最终成片进入普通作品列表**（`asset_scope=final_work`）
6. **复刻 `long_video.stages` 必须带乐观锁版本**
7. **SSE 只推送事务提交后的完整快照**，不推增量包络
8. **断点续跑必须基于 `long_video_ai_task` 关系表恢复**，不能只依赖 `project.status/currentStage`
9. **文本阶段（意图/剧本/分镜）默认免费，调用 `TextCompletionService` 但不写消费流水**

---

## 七、并发安全

```java
// 1. long_video.version 是乐观锁版本
// 2. 更新 stages 时带版本条件：
//    UPDATE long_video SET stages = ?, version = version+1
//    WHERE id = ? AND version = ?
// 3. 同步更新 long_video_ai_task 和 stages 时尽量在同一事务
// 4. 同一个 shot 选择不同 render 时，先取消旧 selected
// 5. 长视频详情和 SSE 建连都会对 pending/processing 分镜做限频补查
//    但复用 AiProxyService.queryGenericTask()
//    不绕过 ai_task、consumption、COS 转存和失败退款
```

这套方案从前端到后端经过了多轮迭代，当前已在线上稳定运行。
