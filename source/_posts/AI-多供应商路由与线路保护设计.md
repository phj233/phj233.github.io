---
title: AI 多供应商路由与线路保护设计
date: 2026-05-28 10:00:00
categories:
- Program
tags:
- AI
- 架构
- 后端
- SaaS
- 限流
---

在 AI SaaS 平台中，你不可能只接一家模型供应商。需要接多家供应商来保障稳定性、控制成本和备份容灾。但多供应商带来的问题是：当某条线路异常时，如何在不影响用户体验的前提下自动切换？如何避免已经被官方频控的线路继续接收请求？

下面是我在实际项目中落地的一套供应商路由与线路保护方案。

## 一、供应商路由核心逻辑

`ProviderRouterService` 是整个路由的决策中心。每次 AI 任务提交时，路由服务需要回答：用哪家供应商的哪个模型？

### 获取可用供应商

```java
public List<ProductProviderDO> getAvailableProviders(Long productId) {
    if (productId == null) return List.of();
    // 1. 从缓存中按产品ID加载所有启用的供应商
    List<ProductProviderDO> all = appCacheService.get(
            AppCacheNames.PROVIDER_ROUTE_ENABLED,
            productId,
            () -> loadEnabledProviders(productId));
    if (all.isEmpty()) {
        observabilityService.recordProviderRoute("route", "empty", 1);
        return all;
    }
    // 2. 过滤掉已熔断的供应商
    List<ProductProviderDO> available = all.stream()
            .filter(this::isNotCircuitOpen)
            .collect(Collectors.toList());
    if (!available.isEmpty()) {
        observabilityService.recordProviderRoute("route", "available", 1);
        return available;
    }
    // 3. 全部被保护时，放行半开探测
    observabilityService.recordProviderRoute("route", "all_protected", 1);
    return recoveryProbeProviders(productId, all);
}
```

### 智能路由排序

当存在多条候选线路时，按成功率和降权策略排序：

```java
public List<ProductProviderDO> getSmartRoutedProviders(Long productId) {
    List<ProductProviderDO> available = getAvailableProviders(productId);
    if (available.size() <= 1) return available;
    double degradeThreshold = runtimeConfigService.aiProviderDegradeSuccessRateThreshold();

    // 分两组：成功率 >= 阈值 的按默认优先级排，< 阈值的按成功率降序排到后面
    List<ProductProviderDO> good = new ArrayList<>();
    List<ProductProviderDO> degraded = new ArrayList<>();

    for (ProductProviderDO p : available) {
        double rate = getSuccessRate(p.getId());
        if (rate >= degradeThreshold) good.add(p);
        else degraded.add(p);
    }

    // 降权组内按成功率从高到低排（给它们恢复的机会）
    degraded.sort((a, b) ->
        Double.compare(getSuccessRate(b.getId()), getSuccessRate(a.getId())));

    List<ProductProviderDO> result = new ArrayList<>(good);
    result.addAll(degraded);
    return result;
}
```

### 滑动窗口成功率

使用 `ConcurrentLinkedDeque` 维护每个供应商最近 20 次调用的结果，只计算 10 分钟内的：

```java
private static final int WINDOW_SIZE = 20;

public double getSuccessRate(Long providerId) {
    ConcurrentLinkedDeque<WindowEntry> window = successWindows.get(providerId);
    if (window == null || window.isEmpty()) return 1.0; // 无数据默认100%

    long cutoff = System.currentTimeMillis() - 10 * 60 * 1000; // 只看10分钟内
    long total = 0, success = 0;
    for (WindowEntry e : window) {
        if (e.timestamp() >= cutoff) {
            total++;
            if (e.success()) success++;
        }
    }
    if (total == 0) return 1.0;
    return (double) success / total;
}
```

### 阶梯式熔断

失败越多，熔断越久。3 次失败起开始保护：

```java
private static final int FAIL_THRESHOLD = 3;

private long getCircuitOpenSeconds(int failCount) {
    if (failCount < FAIL_THRESHOLD) return 0;
    long baseSeconds = runtimeConfigService.aiProviderCircuitOpenSeconds();
    if (failCount < 10) return baseSeconds;
    if (failCount < 20) return baseSeconds * 10;
    if (failCount < 50) return baseSeconds * 60;
    return baseSeconds * 120;
}
```

---

## 二、错误分类服务

不是所有失败都该惩罚供应商。`AiProviderErrorService` 对上游异常做一次性分类，调用方根据返回的决策对象处理路由、退款和提示：

```java
@Service
public class AiProviderErrorService {

    public enum ProviderErrorType {
        CONTENT_POLICY,      // 内容安全拒绝
        AUTH_FAILURE,        // 鉴权失败
        QUOTA_INSUFFICIENT,  // 额度不足
        RATE_LIMITED,        // 频控
        PROVIDER_UNAVAILABLE,// 模型平台不可用
        INVALID_REQUEST,     // 用户参数错误
        EXTERNAL_NETWORK,    // 外部网络失败
        TEMPORARY,           // 上游临时不可用
        UNKNOWN              // 未知错误
    }

    /**
     * 分类后的决策对象，包含所有后续路由、退款和提示所需的信息。
     */
    public record ProviderErrorDecision(
            ProviderErrorType type,
            String userMessage,              // 面向用户的中文错误说明
            String refundedUserMessage,      // 已退款场景可直接展示的文案
            String logMessage,               // 面向日志的简短分类
            boolean recordProviderFailure,   // 是否记录供应商健康失败
            boolean longProviderProtection,  // 是否进入较长保护
            boolean stopProviderRetry,       // 是否停止当前供应商的后续重试
            boolean stopProviderSwitch,      // 是否停止切换其他供应商
            Long retryAfterSeconds           // 上游建议等待秒数，仅频控使用
    ) { }

    public ProviderErrorDecision decide(Throwable error) {
        // 内容安全拒绝 → 不记录失败，停止继续重试和切换
        if (isContentPolicyViolation(error)) {
            return decision(ProviderErrorType.CONTENT_POLICY,
                    contentPolicyMessage(error),
                    "内容安全拦截",
                    false,   // 不记录供应商失败
                    true,    // 停止重试当前供应商
                    true);   // 停止切换供应商
        }

        // 频率限制 → 进入短冷却，可切换供应商
        if (isRateLimited(error)) {
            return decision(ProviderErrorType.RATE_LIMITED,
                    rateLimitedMessage(error),
                    "供应商频率受限",
                    true,    // 记录失败（触发短冷却）
                    true,    // 停止重试
                    false,   // 可切换供应商
                    false,
                    retryAfterSeconds(error)); // 上游建议等待时间
        }

        // 额度不足 → 进入长保护，停止切换
        if (isQuotaInsufficient(error)) {
            return decision(ProviderErrorType.QUOTA_INSUFFICIENT,
                    quotaInsufficientMessage(),
                    "供应商额度不足",
                    true,    // 记录失败（进入长保护）
                    true,    // 停止重试
                    true);   // 停止切换——其他供应商大概率也在同一个账号下
        }

        // 鉴权失败 → 进入长保护
        if (isAuthFailure(error)) {
            return decision(ProviderErrorType.AUTH_FAILURE,
                    authFailureMessage(providerErrorText(error)),
                    "供应商鉴权配置异常",
                    true, true, true);
        }

        // 模型平台不可用 → 进入长保护
        if (isNonRetryableProviderFailure(error)) {
            return decision(ProviderErrorType.PROVIDER_UNAVAILABLE,
                    providerUnavailableMessage(error),
                    "供应商模型平台不可用",
                    true, true, true);
        }

        // 用户参数错误 → 不记录失败，停止重试
        if (isInvalidRequest(error)) {
            return decision(ProviderErrorType.INVALID_REQUEST,
                    invalidRequestMessage(error),
                    "用户输入或参数校验失败",
                    false, true, true);
        }

        // 外部网络失败 / 上游临时不可用 → 记录失败但只做基础熔断
        if (isExternalNetworkFailure(error)) {
            return decision(ProviderErrorType.EXTERNAL_NETWORK,
                    externalNetworkFailureMessage(),
                    "外部网络访问失败",
                    true, false, false);
        }
        if (isTemporaryFailure(error)) {
            return decision(ProviderErrorType.TEMPORARY, temporaryFailureMessage(),
                    "上游临时不可用", true, false, false);
        }

        // 兜底：未知错误，记录失败但不做强保护
        String text = providerErrorText(error);
        return decision(ProviderErrorType.UNKNOWN,
                "调用 AI 供应商失败，请稍后重试或切换供应商",
                text, true, false, false);
    }
}
```

### 频控检测与自适应退避

```java
public boolean isRateLimited(Throwable error) {
    return isRateLimitedText(providerErrorText(error));
}

public boolean isRateLimitedText(String text) {
    if (text == null) return false;
    String lower = text.toLowerCase(Locale.ROOT);
    return lower.contains("rate limit") || lower.contains("too many requests")
            || lower.contains("429") || lower.contains("throttle")
            || lower.contains("frequency") || lower.contains("too frequent")
            || lower.contains("api rate limit exceeded")
            || lower.contains("quota limit exceeded")
            || lower.contains("request limit reached");
}
```

### 用户可见错误文案

对上游返回的英文错误做"翻译"，生成可读的中文说明，同时不暴露敏感信息：

```java
public String translateUpstreamError(String raw) {
    if (raw == null || raw.isEmpty())
        return "错误阶段：AI 生成。生成失败，已自动退款。";
    String lower = raw.toLowerCase(Locale.ROOT);

    if (lower.contains("copyright"))
        return "视频内容被判定可能涉及版权保护（人物/IP/品牌等），"
             + "已自动退款。请调整提示词或更换参考图后重试。";

    if (lower.contains("face") && (lower.contains("audit")
            || lower.contains("detect") || lower.contains("recogn")))
        return "参考图中的人脸未通过安全审核，"
             + "已自动退款。请更换为清晰且非敏感人物的照片后重试。";

    if (lower.contains("timeout") || lower.contains("timed out"))
        return "上游服务响应超时，已自动退款。请稍后再试。";

    if (isRateLimitedText(raw))
        return rateLimitedMessage(raw) + "，已自动退款。";

    // ... 更多分支匹配各种错误模式 ...
}
```

---

## 三、记录成功与失败

调用完成后，分别走成功和失败记录路径：

```java
// 成功：清理故障计数，清除短冷却和频控状态
public void recordSuccess(Long providerId) {
    if (providerId == null) return;
    // DB: 成功即清理故障计数
    providerMapper.update(null, new UpdateWrapper<ProductProviderDO>()
            .eq("id", providerId)
            .set("fail_count", 0)
            .set("last_fail_time", null)
            .set("last_success_time", LocalDateTime.now()));
    // 本机内存状态也清理
    providerHealth.put(providerId, new ProviderHealth(0, null));
    rateLimitCooldownUntil.remove(providerId);
    rateLimitStates.remove(providerId);
    // 滑动窗口追加成功记录
    appendToWindow(providerId, true);
}

// 失败：根据错误决策分级处理
public void recordFailure(Long providerId, ProviderErrorDecision decision) {
    if (providerId == null || decision == null) return;
    // 内容安全问题/用户参数错误不记录供应商失败
    if (!decision.recordProviderFailure()) return;

    // 频控类走短冷却
    if (decision.type() == ProviderErrorType.RATE_LIMITED) {
        long cooldownMs = decision.retryAfterSeconds() != null
                ? decision.retryAfterSeconds() * 1000L
                : adaptiveCooldownMs(providerId);
        rateLimitCooldownUntil.put(providerId,
                System.currentTimeMillis() + cooldownMs);
    }

    // 更新 DB 失败计数和最近失败时间
    updateFailCount(providerId);
    // 追加到滑动窗口
    appendToWindow(providerId, false);
    // 记录最近错误摘要供后台展示
    recordRecentError(providerId, decision);
}
```

---

## 四、后台健康可视化

管理员在后台可以看到每条线路的实时健康快照：

```java
public record ProviderHealthSnapshot(
        Long providerId,
        String status,              // "normal" / "circuit_open" / "rate_limited"
        String statusLabel,         // "正常" / "已熔断" / "已限流"
        double successRate,         // 成功率
        int failCount,              // 当前失败计数
        LocalDateTime lastFailTime,
        LocalDateTime lastSuccessTime,
        long rateLimitRemainingSeconds,  // 频控剩余冷却秒数
        long circuitRemainingSeconds,     // 熔断剩余秒数
        long nextProbeInSeconds,          // 半开探测等待秒数
        long probeLeaseRemainingSeconds,  // 探测租约剩余秒数
        String recentErrorType,           // 最近错误类型
        String recentErrorMessage,        // 最近错误摘要（支持长错误折叠展开）
        LocalDateTime recentErrorTime     // 最近错误时间
) { }

public ProviderHealthSnapshot getProviderHealth(ProductProviderDO provider) {
    // ... 计算成功率、熔断剩余时间、频控剩余时间等
    // 返回完整快照供后台渲染
}
```

---

## 五、设计原则

1. **每条线路独立保护**：不能因为 Vidu 挂了就把所有视频生成一刀切锁死
2. **全保护态放行半开探测**：不会让模型永远不可用，探测成功自动恢复
3. **内容安全不污染供应商**：图片/视频不合规是模型能力边界，不是线路故障
4. **429 自适应退避**：连续 429 后延长冷却，尊重 `Retry-After` 头部
5. **用户错误提示不泄漏**：说明失败阶段，但不透出 API Key、Token 和完整响应

这套方案跑了一段时间后比较稳定，当前还在观察多实例部署下的短冷却同步问题——后面多实例同时 429 的话，可能需要在 Redis 中维护全局冷却状态。
