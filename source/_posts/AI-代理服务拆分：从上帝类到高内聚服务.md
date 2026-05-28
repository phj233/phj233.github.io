---
title: AI 代理服务拆分：从上帝类到高内聚服务
date: 2026-05-28 10:30:00
categories:
- Program
tags:
- 重构
- 架构
- Java
- Spring Boot
- AI
---

在一个 AI SaaS 项目中，`AiProxyService` 曾经是后端的绝对核心——文本对话、图片生成、视频生成、3D 生成、任务查询、取消、退款、供应商路由全部堆在这一类里，一度膨胀到两千多行。经过几轮拆分，主类降到一千行出头，图片/视频协议各自独立，查询状态机和取消链路也有了清晰边界。

## 一、拆分前的上帝类

拆分前的 `AiProxyService` 长这样：

```java
@Service
@RequiredArgsConstructor
public class AiProxyService {
    private final BillingService billingService;
    private final AccountMapper accountMapper;
    private final AiTaskMapper aiTaskMapper;
    private final RestTemplate restTemplate;
    private final SystemConfigService configService;
    private final ProviderRouterService providerRouter;
    // ... 一共注入了十几个依赖

    // 创建任务 —— 包含了参数校验、幂等、任务落库、扣费、路由、上游提交
    public JsonData<?> createGenericTask(GenericAiTaskDTO dto, String productCode) { /* 200+ 行 */ }

    // 图片任务创建
    private JsonData<?> createImageTask(...) { /* 150+ 行 */ }

    // 视频任务创建（普通视频、native video、OpenAI video、Veo...）
    private JsonData<?> createVideoTask(...) { /* 300+ 行 */ }

    // 查询任务 —— 用户查询 + 后台补查 + 完成落库
    public JsonData<?> queryGenericTask(...) { /* 200+ 行 */ }

    // 取消任务 —— 资源校验 + 上游取消 + 退款
    public JsonData<?> cancelTask(...) { /* 150+ 行 */ }

    // 文本生成
    public JsonData<?> createTextTask(...) { /* 80+ 行 */ }

    // 3D 生成
    private JsonData<?> create3DTask(...) { /* 100+ 行 */ }

    // 再加上一堆私有工具方法：
    private String buildUrl(...) { ... }
    private HttpEntity<?> buildRequest(...) { ... }
    private String extractResultUrl(...) { ... }
    // ... 总计 2000+ 行
}
```

所有东西都堆在一个类里，改一处要小心翼翼地检查会不会影响其他地方。

---

## 二、拆分原则

不追求一步到位式的"完美重构"，拆分的三条铁律：

**1. 先拆低耦合，再拆高耦合。** 先把 URL 拼接、请求头组装、类型转换这类纯工具逻辑拆出去。这类代码几乎没有状态依赖，拆出去不会引入 bug。

**2. 每个 stage 都能单独回滚。** 不一次搬动图片、视频、3D、取消和查询全链路。拆完一个、验证一个、上线一个。

**3. 不变的边界不动。** 扣费时机不变、退款逻辑不变、供应商路由算法不变、对外接口响应格式不变。重构是代码组织方式的变化，不是行为的变化。

---

## 三、落位判断：合入还是新建？

不是每组方法都要新建一个 service。在决定"新建文件"前先问三个问题：

1. 这个能力是否已有现成的 service 可以合入？
2. 合入后现有 service 的职责是否清晰？
3. 这个能力是否有独立的业务生命周期和多处复用需求？

只有三个都满足才新建文件。具体规则：

```java
// 工具类 → 优先合入已有 service
// URL 提取/补全/路径拼接/媒体 URL 判断 → AiMediaUrlService
// 请求头/baseUrl/endpoint/default_params 读取 → AiProviderHeaderService
// DTO extra/Map 参数/类型转换 → AiRequestValueService
// 模型 guard/提示词 guard/参数校验 → AiGuardService
// 下载/转存/transfer_pending → AiResultTransferService

// 需要独立文件的 service 必须满足：
// 1) 独立的业务生命周期
// 2) 清晰的状态边界
// 3) 多处复用需求
// 否则优先合入已有 service

// 反例：不允许只为了少写一行转调而新建
// ❌ class AiTaskUrlBuilder { String build(String url) { return mediaUrlService.build(url); } }
```

---

## 四、拆分内容详解

### 第一阶段：小瘦身

在拆 service 之前，先清理冗余代码——这些都是零风险的：

```java
// 删除只做一层转调的薄包装方法
// ❌ 删除前：
private Map<String, Object> toStringObjectMap(Object obj) {
    return CommonUtil.toStringObjectMap(obj);
}
// ✅ 调用点直接使用 CommonUtil.toStringObjectMap(obj)

// 合并功能重复的私有方法
// ❌ 删除前有两个方法：
private Map<String, Object> buildDirectImageResult(String url) { ... }
private Map<String, Object> buildDirectVideoResult(String url) { ... }
// ✅ 合并为一个：
private Map<String, Object> buildDirectResult(String url, boolean video) { ... }

// 统一工具方法
// ❌ UUID.randomUUID()
// ✅ IdUtil.randomUUID()
// ❌ str.substring(0, Math.min(str.length(), n))
// ✅ StrUtil.subPre(str, n)
// ❌ Integer.parseInt(str)
// ✅ Convert.toInt(str)
```

### 第二阶段：工具类拆出

把 URL 拼接、请求头组装等工具代码拆到专门的服务：

```java
// AiMediaUrlService —— URL 提取、补全、拼接
@Service
public class AiMediaUrlService {
    public String joinApiUrl(String baseUrl, String path) { ... }
    public String extractResultUrl(JsonNode response, String apiFormat) { ... }
    public boolean isDirectUrl(String url) { ... }
    // ...
}

// AiProviderHeaderService —— 请求头、baseUrl、配置读取
@Service
public class AiProviderHeaderService {
    public String getBaseUrl(JsonNode config) { ... }
    public HttpHeaders buildJsonHeaders(JsonNode config) { ... }
    public String getConfigString(JsonNode config, String key, String fallback) { ... }
    public JsonNode mergeProviderConfig(ProductProviderDO provider, JsonNode config) { ... }
    // ...
}

// AiGuardService —— 模型参数 guard、提示词 guard
@Service
public class AiGuardService {
    public void validateResolution(GenericAiTaskDTO dto, String apiType, String model) { ... }
    public void normalizeDuration(GenericAiTaskDTO dto, String apiType, String model) { ... }
    public void applyVideoReferenceConsistencyGuard(...) { ... }
    // ...
}
```

这些工具类本身就很薄，拆出去后主类少了很多私有工具方法，但职责还没变。

### 第三阶段：独立业务 service

真正需要新建文件的几个核心能力：

**AiTaskMetaService** —— 任务 meta 的读写

```java
@Service
public class AiTaskMetaService {
    /**
     * 从 params_json 读取任务 meta，包括 client dedup scope、
     * 上游 requestId/taskId 和结果 meta 信息。
     */
    public Map<String, Object> readParams(AiTaskDO task) { ... }
    public void writeParams(AiTaskDO task, Map<String, Object> params) { ... }
    public Map<String, Object> buildClientDedupScope(GenericAiTaskDTO dto) { ... }
    public void mergeResultMeta(AiTaskDO task, Map<String, Object> resultMeta) { ... }
}
```

拆前，主类直接用 `objectMapper.readValue(task.getParamsJson())` 到处读写 JSON 字段。拆后所有 JSON 操作收敛到这里，方便后续引入 JSONB 或换序列化方式。

**AiTaskCancelService** —— 取消任务的完整链路

```java
@Service
public class AiTaskCancelService {

    public JsonData<?> cancelTask(String taskId,
            BiFunction<String, String, JsonData<?>> schedulerQuery) {

        // 1. 校验任务归属 —— 不是本人的任务不能取消
        AiTaskDO task = findTask(taskId);
        if (currentUserNotOwner(task, taskId))
            return JsonData.buildError("无权操作此任务");

        // 2. 状态前置检查 —— 已完成/已取消/已退款 直接返回
        if (isTaskCompleted(task)) return JsonData.buildError("任务已完成，无法取消");
        if (isTaskCancelled(task) || Objects.equals(task.getRefunded(), 1))
            return buildCancelSuccessResult(task, "任务已取消或已退款");

        // 3. 取消前补查 —— 确认任务没有在取消请求到达时恰好完成
        JsonData<?> preflight = queryBeforeCancel(task, schedulerQuery);
        if (preflight != null && taskCompletedInPreflight(preflight))
            return JsonData.buildError("任务已完成，无法取消");

        // 4. 调上游取消 API
        UpstreamCancelResult upstreamResult = cancelUpstreamTask(task);
        if (!upstreamResult.success())
            return JsonData.buildCodeMsgData(-2, upstreamResult.message(), ...);

        // 5. CAS 更新本地状态 + 退款
        return refundCancelledTask(task, upstreamResult);
    }

    private JsonData<?> refundCancelledTask(AiTaskDO task,
            UpstreamCancelResult upstreamResult) {
        // CAS: 防止并发取消或状态已变
        UpdateWrapper<AiTaskDO> casUpdate = new UpdateWrapper<>();
        casUpdate.eq("id", task.getId())
                 .eq("refunded", 0)
                 .ne("status", "completed")
                 .set("refunded", 1)
                 .set("status", "cancelled")
                 .set("error_msg", savedMsg)
                 .set("gmt_modified", LocalDateTime.now());

        int rows = aiTaskMapper.update(null, casUpdate);
        if (rows <= 0) {
            // CAS 失败 → 重新读取，可能已经被并发操作改掉了
            AiTaskDO refreshed = aiTaskMapper.selectById(task.getId());
            if (isCancelledOrRefunded(refreshed))
                return buildCancelSuccessResult(refreshed, "任务已取消或已退款");
            return JsonData.buildError("任务状态已变化，取消结果未落库，请刷新后重试");
        }

        // CAS 成功 → 退款
        billingService.refundBalance(task.getAccountId(), task.getCost(), task.getTraceId());
        taskAccountingService.markConsumptionStatus(task.getTraceId(), "refunded");
        return buildCancelSuccessResult(task, upstreamResult.message());
    }
}
```

**AiTaskQueryService** —— 查询状态机

```java
@Service
public class AiTaskQueryService {

    /**
     * 用户前端轮询 fast path：
     * 本地已完成的直接返回，未完成的提交后台补查后立即返回当前状态。
     */
    public JsonData<?> queryForUser(String productCode, String taskId) {
        AiTaskDO task = findTask(taskId);
        if (isCompletedOrTransferPending(task))
            return buildCompletedResult(task);

        // 提交后台补查，立即返回当前状态
        submitBackgroundCheck(task);
        return buildProcessingResult(task);
    }

    /**
     * 后台补查（由定时调度器调用）：
     * 保留上游重试、下载/上传转存和完成落库能力。
     */
    public void queryForScheduler(String productCode, String taskId) {
        AiTaskDO task = findTask(taskId);
        // 已完成的跳过
        if (isTaskCompleted(task)) return;

        try {
            // 调用上游查询
            Map<String, Object> result = queryUpstream(task);
            if (result != null && result.containsKey("url")) {
                // 上游已生成 → 先写 transfer_pending，再异步转存
                markTransferPending(task, result);
                transferResult(task, result);
            }
        } catch (RetryableException e) {
            // 查询异常 → 可重试，不退款
            return;
        } catch (UpstreamFailure e) {
            // 上游失败 → 退款
            markFailedAndRefund(task, e.getMessage());
        }
    }
}
```

**AiImageProviderClient** 和 **AiVideoProviderClient** —— 协议适配

```java
@Service
public class AiImageProviderClient {
    // 统一的图片创建入口
    public Map<String, Object> createImageTask(AiTaskDO task, ProductProviderDO provider,
            GenericAiTaskDTO dto, String apiType, String modelName) {
        return switch (apiType) {
            case "openai" -> createOpenAIImage(task, provider, dto, modelName);
            case "gemini" -> createGeminiImage(task, provider, dto, modelName);
            case "volcengine" -> createVolcengineImage(task, provider, dto, modelName);
            default -> createDefaultImage(task, provider, dto, modelName);
        };
    }

    // 统一的图片查询入口
    public Map<String, Object> queryImageTask(AiTaskDO task, ProductProviderDO provider) {
        // 不同供应商查询协议差异，收敛在这里
    }
}
```

```java
@Service
public class AiVideoProviderClient {
    public boolean isNativeVideoFormat(String apiFormat) {
        return "native_video".equals(apiFormat);
    }

    // 多协议分支：普通视频、native video、OpenAI video、Veo、Volcengine...
    public Map<String, Object> createVideoTask(AiTaskDO task, ProductProviderDO provider,
            GenericAiTaskDTO dto, String apiType, String modelName) {
        // 1500+ 行的视频协议适配代码，从主类中完全迁出
    }
}
```

---

## 五、拆分后主类的样子

拆分后的 `AiProxyService` 从两千多行降到一千行出头，只保留核心编排：

```java
@Service
@RequiredArgsConstructor
public class AiProxyService {

    // 注入的是拆分出去的业务 service，不再是原始依赖
    private final BillingService billingService;
    private final ProviderRouterService providerRouter;
    private final AiTaskCancelService taskCancelService;
    private final AiTaskQueryService taskQueryService;
    private final AiImageProviderClient imageProviderClient;
    private final AiVideoProviderClient videoProviderClient;
    private final AiTaskAccountingService taskAccountingService;
    private final AiTaskMetaService taskMetaService;
    private final AiProviderHeaderService providerHeaderService;
    private final AiMediaUrlService mediaUrlService;
    private final AiGuardService aiGuardService;
    private final AiRequestValueService requestValueService;
    // ... 依赖数量没少，但每个依赖的职责清晰了

    /**
     * 创建任务 —— 主类只负责编排，不再写具体协议代码
     */
    public JsonData<?> createGenericTask(GenericAiTaskDTO dto, String productCode) {
        // 1. 维护模式 / 空参校验
        // 2. 获取产品配置
        // 3. 参数规范化与 guard → aiGuardService
        // 4. 幂等检查 → taskMetaService
        // 5. 任务落库
        // 6. 扣费 → billingService
        // 7. 供应商路由 → providerRouter
        // 8. 根据 apiType 分发 →
        //     图片 → imageProviderClient.createImageTask(...)
        //     视频 → videoProviderClient.createVideoTask(...)
        //     3D   → tripoProtocolService.create3DTask(...)
        // 9. 上游失败 → 退款 → billingService.refundBalance(...)
    }

    // 取消任务 —— 直接委托
    public JsonData<?> cancelTask(String taskId) {
        return taskCancelService.cancelTask(taskId,
                this::queryGenericTaskForScheduler);
    }

    // 查询任务 —— 复用拆出去的查询服务
    public JsonData<?> queryGenericTask(String productCode, String taskId) {
        return taskQueryService.queryForUser(productCode, taskId);
    }

    // 对外的薄门面方法保留，但逻辑已迁出
}
```

---

## 六、几个踩过的坑

### 1. 计费/退款的顺序不能动

创建任务必须先有本地 `ai_task` 锚点 → 再扣费 → 再调上游。这个顺序一旦打乱，会出现"扣了钱但没任务"或"上游失败了但钱已经扣了"。

```java
// ✅ 正确顺序
AiTaskDO task = insertTask(dto);           // 1. 落库
consumption = billingService.charge(...);  // 2. 扣费
try {
    result = callUpstream(task, provider);  // 3. 调上游
} catch (Exception e) {
    billingService.refundBalance(           // 4. 上游失败，退款
        task.getAccountId(), task.getCost(), task.getTraceId());
    throw e;
}
```

### 2. 取消任务的 CAS 是关键

只在上游明确返回取消成功后才允许本地退款。中间用 CAS 更新避免并发问题：

```java
// CAS: 只有 refunded=0 且 status != completed 才执行取消退款
int rows = aiTaskMapper.update(null,
    new UpdateWrapper<AiTaskDO>()
        .eq("id", task.getId())
        .eq("refunded", 0)
        .ne("status", "completed")
        .set("refunded", 1)
        .set("status", "cancelled")
        .set("gmt_modified", now));

if (rows <= 0) {
    // 被并发了，重新读取判断状态
    AiTaskDO refreshed = aiTaskMapper.selectById(task.getId());
    if (isCancelledOrRefunded(refreshed)) return success;
    return error("状态已变化，请刷新后重试");
}
```

### 3. 不要为了拆而拆

每个新 service 都必须减少主类的职责面，不是把同一段逻辑换个文件名。新 service 内仍要优先复用已有 service，不能把主类的重复逻辑搬过去后继续重复。

---

## 七、拆分后的收益

1. **新接入视频供应商**：只需要在 `AiVideoProviderClient` 里加分支，完全不用碰主类
2. **修改取消逻辑**：只改 `AiTaskCancelService`，不影响创建和查询
3. **调整任务 meta 存储格式**：只改 `AiTaskMetaService`，对上下游透明
4. **测试更聚焦**：每个新 service 可以单独做单元测试

拆分不是终点，后续按真实维护痛点再决定是否继续拆。当前主类约一千行，已进入观察期——除非出现真实维护痛点，否则不再机械拆分。
