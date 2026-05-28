---
title: Vue3 大型项目 Liquid Glass 主题与组件化重构
date: 2026-05-28 11:00:00
categories:
- Program
tags:
- Vue3
- TypeScript
- 前端
- 架构
- Tailwind
---

一个 AI 创作平台的前端从旧版原生 HTML 迁移到了 Vue3 + Vite + TypeScript + Element Plus + Pinia + Tailwind CSS。项目当前约 121 个 Vue SFC、95 个 TS 文件、8 个 CSS 模块，合计约 5.2 万行。这里记录一下主题体系和组件化重构的思路。

## 一、设计令牌（Design Tokens）

所有颜色、阴影、模糊、边框统一收敛到 CSS 自定义属性，通过主题变量控制全局外观。`index.css` 定义了完整的令牌体系：

```css
:root {
  color-scheme: dark;
  background: #08080f;
  color: #f1f5f9;

  /* 背景层级 */
  --yy-bg: #08080f;
  --yy-bg-soft: #0f1020;
  --yy-panel: rgba(255, 255, 255, 0.04);          /* 底层面板 */
  --yy-panel-strong: rgba(255, 255, 255, 0.075);    /* 强调面板 */
  --yy-panel-elevated: rgba(20, 20, 30, 0.96);      /* 浮层面板 */
  --yy-panel-solid: rgba(10, 12, 20, 0.84);

  /* 文字层级 */
  --yy-text: #f1f5f9;
  --yy-text-muted: #94a3b8;
  --yy-text-faint: #64748b;

  /* 品牌色 */
  --yy-brand: #7c3aed;         /* 紫色 */
  --yy-brand-2: #3b82f6;       /* 蓝色 */
  --yy-success: #22c55e;
  --yy-warning: #f59e0b;
  --yy-danger: #ef4444;
  --yy-info: #60a5fa;

  /* 玻璃效果 */
  --yy-glass-blur: blur(18px);
  --yy-glass-saturate: saturate(1);
  --yy-glass-shadow: 0 24px 80px rgba(0, 0, 0, 0.22);

  /* 控件 */
  --yy-control-bg: rgba(255, 255, 255, 0.055);
  --yy-control-bg-hover: rgba(124, 58, 237, 0.12);
  --yy-control-ring: rgba(124, 58, 237, 0.42);

  /* 导航栏 */
  --yy-navbar-bg: linear-gradient(180deg,
    rgba(8, 8, 15, 0.52), rgba(8, 8, 15, 0.38));

  /* 选择器 */
  --yy-select-glass-bg: var(--yy-panel-elevated);
  --yy-select-glass-border: var(--yy-border);
  --yy-select-glass-hover: var(--yy-control-bg);

  /* 背景光晕 */
  --yy-bg-orb-primary: rgba(124, 58, 237, 0.16);
  --yy-bg-orb-secondary: rgba(59, 130, 246, 0.12);

  /* 将 Element Plus 的主题色链接到我们的令牌 */
  --el-color-primary: var(--yy-brand);
  --el-color-success: var(--yy-success);
  --el-color-warning: var(--yy-warning);
  --el-color-danger: var(--yy-danger);
}
```

Element Plus 的主题色通过赋值给 `--el-color-primary` 来统一，避免第三方组件和自定义组件颜色不一致。

---

## 二、Liquid Glass 面板组件

核心面板组件 `LiquidGlassPanel`，承载透明玻璃质感的容器：

```vue
<script setup lang="ts">
withDefaults(
  defineProps<{
    as?: string                       // 渲染标签，默认 'div'
    interactive?: boolean             // 悬停动效
    tone?: 'soft' | 'strong' | 'media' // 强调级别
  }>(),
  { as: 'div', interactive: false, tone: 'soft' },
)
</script>

<template>
  <component
    :is="as"
    class="liquid-glass-panel"
    :class="[`tone-${tone}`, { interactive }]"
  >
    <slot />
  </component>
</template>

<style scoped>
.liquid-glass-panel {
  position: relative;
  overflow: hidden;
  isolation: isolate;
  color: var(--yy-text);
  background:
    linear-gradient(135deg,
      color-mix(in srgb, var(--yy-panel-strong) 74%, transparent),
      color-mix(in srgb, var(--yy-panel) 92%, transparent)),
    color-mix(in srgb, var(--yy-bg) 4%, transparent);
  border: 1px solid color-mix(in srgb,
    var(--yy-border) 78%, var(--yy-on-accent) 12%);
  -webkit-backdrop-filter: var(--yy-glass-blur) var(--yy-glass-saturate);
  backdrop-filter: var(--yy-glass-blur) var(--yy-glass-saturate);
  box-shadow: var(--yy-glass-shadow);
}

/* 顶部高光伪元素 —— 模拟玻璃的反光 */
.liquid-glass-panel::before {
  content: '';
  position: absolute;
  inset: 0;
  z-index: 0;
  pointer-events: none;
  background:
    linear-gradient(120deg, rgba(255, 255, 255, 0.28), transparent 28%),
    linear-gradient(180deg, rgba(255, 255, 255, 0.12), transparent 44%);
  opacity: 0.44;
}

:slotted(*) {
  position: relative;
  z-index: 1;
}

/* Strong 模式：更强调的面板 */
.liquid-glass-panel.tone-strong {
  background: linear-gradient(135deg,
    color-mix(in srgb, var(--yy-panel-elevated) 78%, transparent),
    color-mix(in srgb, var(--yy-panel-strong) 92%, transparent));
  border-color: color-mix(in srgb, var(--yy-brand) 20%, var(--yy-border));
}

/* 交互态：悬停浮起 */
.liquid-glass-panel.interactive {
  transition: border-color 0.25s ease, box-shadow 0.25s ease, transform 0.25s ease;
}
.liquid-glass-panel.interactive:hover {
  border-color: color-mix(in srgb, var(--yy-brand) 32%, var(--yy-border));
  box-shadow: var(--yy-glass-shadow),
    0 14px 42px color-mix(in srgb, var(--yy-brand) 16%, transparent);
  transform: translateY(-2px);
}
</style>
```

用 `::before` 伪元素做顶部高光、`backdrop-filter` 做背景模糊、`color-mix` 做半透明混合，三件套实现玻璃效果。`interactive` 模式下悬停会浮起并向品牌色靠近。

---

## 三、Liquid Glass Select 组件

Element Plus 原生的 `ElSelect` 在玻璃背景下有对比度问题、弹层被裁切、长模型名显示不佳。自己写了一个 `LiquidGlassSelect`：

```vue
<script setup lang="ts">
export interface LiquidGlassSelectOption {
  value: string
  label: string
  description?: string
  icon?: string
  iconClass?: string
  disabled?: boolean
}

const props = withDefaults(
  defineProps<{
    modelValue: string
    options: LiquidGlassSelectOption[]
    menuLabel?: string
    placeholder?: string
    variant?: 'param' | 'model'    // param=参数选择，model=模型选择
    wide?: boolean
    compact?: boolean
    disabled?: boolean
    placement?: 'top' | 'bottom'
  }>(),
  { menuLabel: '', placeholder: '请选择', variant: 'param', /* ... */ }
)

const emit = defineEmits<{
  'update:modelValue': [value: string]
  change: [value: string]
}>()

const rootRef = ref<HTMLElement | null>(null)
const menuRef = ref<HTMLElement | null>(null)
const open = ref(false)
const menuStyle = ref<Record<string, string>>({})
const activeOptionIndex = ref(-1)

// 键盘导航
function handleTriggerKeydown(event: KeyboardEvent) {
  if (event.key === 'ArrowDown' || event.key === 'ArrowUp') {
    event.preventDefault()
    if (!open.value) openMenu()
    else moveActiveOption(event.key === 'ArrowDown' ? 1 : -1)
    return
  }
  if (event.key === 'Enter' || event.key === ' ') {
    event.preventDefault()
    if (!open.value) openMenu()
    else pickActiveOption()
  }
}

// 弹层位置自适应：根据 viewport 自动选择上/下弹出，不超出屏幕
function updateMenuPosition() {
  if (!open.value || !rootRef.value) return

  const rect = rootRef.value.getBoundingClientRect()
  const viewportHeight = window.innerHeight
  const gap = 8
  const edge = 16
  const minWidth = isWide.value ? 260 : props.compact ? 132 : 200
  const maxAllowedWidth = Math.max(minWidth, window.innerWidth - edge * 2)
  const naturalWidth = Math.max(rect.width, minWidth)
  const width = Math.min(naturalWidth,
    isWide.value ? Math.min(420, maxAllowedWidth) : maxAllowedWidth)
  const left = Math.min(Math.max(edge, rect.left),
    window.innerWidth - width - edge)
  const menuHeight = menuRef.value?.offsetHeight || (props.variant === 'model' ? 320 : 260)
  const belowTop = rect.bottom + gap
  const aboveTop = rect.top - menuHeight - gap
  const shouldOpenUp = props.placement === 'top'
    || (belowTop + menuHeight > viewportHeight - edge && aboveTop >= edge)
  const top = shouldOpenUp
    ? Math.max(edge, aboveTop)
    : Math.min(belowTop, Math.max(edge, viewportHeight - menuHeight - edge))

  menuStyle.value = {
    position: 'fixed',
    top: `${top}px`,
    left: `${left}px`,
    width: `${width}px`,
  }
}
</script>

<template>
  <div ref="rootRef" class="liquid-glass-select" :class="[/* ... */]">
    <button class="lg-select-trigger" @click.stop="toggle"
            @keydown="handleTriggerKeydown" aria-haspopup="listbox">
      <span v-if="triggerIcon" class="lg-select-icon" :class="triggerIconClass">
        {{ triggerIcon }}
      </span>
      <span class="lg-select-value">{{ triggerLabel }}</span>
      <span class="lg-select-arrow">▾</span>
    </button>

    <!-- Teleport 到 body 避免被父容器裁切 -->
    <Teleport to="body">
      <div v-if="open" ref="menuRef" class="lg-select-menu dd show"
           :style="menuStyle" role="listbox">
        <div v-if="menuLabel" class="dd-label">{{ menuLabel }}</div>
        <div class="lg-select-scroll">
          <button v-for="(option, index) in options" :key="option.value"
                  :id="`${selectId}-option-${index}`" class="dd-item"
                  :class="{ sel: option.value === modelValue,
                            active: index === activeOptionIndex }"
                  @click="pick(option)"
                  @mouseenter="activeOptionIndex = option.disabled
                    ? activeOptionIndex : index">
            <!-- variant=model 时展示模型名+描述 -->
            <span v-if="variant === 'model'" class="lg-model-text">
              <strong>
                <span v-if="option.icon" class="lg-option-icon"
                      :class="option.iconClass">{{ option.icon }}</span>
                {{ option.label }}
              </strong>
              <small v-if="option.description">{{ option.description }}</small>
            </span>
            <span v-else>{{ option.label }}</span>
            <span v-if="option.value === modelValue" class="check">✓</span>
          </button>
        </div>
      </div>
    </Teleport>
  </div>
</template>
```

关键设计决策：
- **Teleport 到 body**：弹层不再受父容器 `overflow: hidden` / `backdrop-filter` / `fixed` 影响
- **自适应位置**：优先向下弹出，空间不足则向上；水平方向约束在 viewport 内
- **键盘导航**：支持 `ArrowDown/Up/Enter/Space/Escape`
- **ARIA 属性**：`role="listbox"` / `aria-expanded` / `aria-selected`

---

## 四、模块化样式治理

项目样式按模块拆分到独立的 CSS 文件中，由 `index.css` 统一引入：

```css
/* src/styles/index.css */
@import 'tailwindcss';
@import './home.css';
@import './center.css';
@import './invite.css';
@import './comic-workflow.css';
@import './media-generation.css';
@import './membership.css';

/* 全局设计令牌和 reset 在这里 */
/* Element Plus 主题适配在这里 */
```

每个模块 CSS 使用 Tailwind 的 `@layer components` 包装，示例来自 `center.css`：

```css
@layer components {
  .center-panel-head {
    @apply mb-4 flex items-start justify-between gap-3;
  }

  .center-panel-title {
    @apply flex min-w-0 items-center gap-2 text-base font-bold
           leading-[1.35] text-yy-text;
  }

  .center-tab {
    @apply cursor-pointer whitespace-nowrap rounded-lg border-0 px-3.5
           py-1.5 text-xs font-medium;
    background: var(--glass-strong);
    color: var(--yy-text-faint);
  }

  .center-tab-active {
    background: rgba(124, 58, 237, 0.15);
    color: var(--yy-text);
  }
}
```

而 Liquid Glass 的原生效果（`color-mix`、`backdrop-filter`、多层渐变、变量阴影）保持在原生 CSS 中，避免 Tailwind 无法表达的玻璃质感在 Safari/iOS 下降级。

---

## 五、页面级组织：以 AI 创作为例

`AiCreatePage.vue` 目前约 1489 行，拆成了多个层级：

**页面壳层**：
```vue
<script setup lang="ts">
// 注入 composable 函数，每个 composable 负责一个独立关注点
import { useCosUpload } from '../../composables/useCosUpload'
import { useAiCreateMediaPreview } from '../../composables/useAiCreateMediaPreview'
import { useAiCreateReferenceUploads } from '../../composables/useAiCreateReferenceUploads'
import { useAiCreateSubmission } from '../../composables/useAiCreateSubmission'
import { useAiCreateTaskActions } from '../../composables/useAiCreateTaskActions'
import { useAiCreateTaskLifecycle } from '../../composables/useAiCreateTaskLifecycle'
// ...
</script>
```

**组件层级**（全部从页面壳拆出）：
```
AiCreatePage.vue
├── AiCreateWorkspaceShell.vue    —— 工作区滚动壳 + 侧栏/主面板 slot
├── AiCreateSidePanel.vue         —— 左侧设置面板 + 参考素材面板
├── AiCreateIntroPanel.vue        —— 顶部统计
├── AiCreateFloatingComposer.vue  —— 底部悬浮输入卡 + 媒体条 + 提示词
├── AiCreateTaskOverview.vue      —— 任务历史列表（loading/empty/卡片）
└── LongVideoChatPanel.vue        —— 长视频二级工作台的聊天区
```

**Composable 层级**（每个 composable 负责一个独立关注点）：
```
useCosUpload.ts               —— COS 上传
useAiCreateMediaPreview.ts    —— 媒体预览
useAiCreateReferenceUploads.ts —— 参考素材上传
useAiCreateSubmission.ts      —— 任务提交
useAiCreateTaskActions.ts     —— 任务操作（预览/下载/删除）
useAiCreateTaskLifecycle.ts   —— 任务生命周期（倒计时/轮询/取消）
useAiCreateTaskRegenerate.ts  —— 再次编辑
useAiCreateTaskRemoval.ts     —— 任务删除
useMediaTaskPolling.ts        —— 通用媒体任务轮询（Map 管理多任务并行轮询）
```

**工具层**（纯函数，无副作用）：
```
aiTaskCards.ts      —— 本地任务、远程作品、长视频项目合并去重
aiCreateTaskUi.ts   —— 状态文案、进度计算、样式 class
happyHorse.ts       —— HappyHorse 模型参数约束
seedance.ts         —— Seedance 模型参数约束
modelPricing.ts     —— 价格计算（按用户会员等级读不同价格）
```

---

## 六、Pinia 持久化策略

使用 `pinia-plugin-persistedstate` 做持久化，但不是所有状态都需要持久化。只持久化跨刷新必要的状态：

**`creationTasks` store**：
```ts
const TASKS_KEY = 'ai_create_tasks_v1'
const REFS_KEY = 'ai_create_refs'
const FRAME_ASSETS_KEY = 'ai_create_frame_assets'
const IMAGE_REF_KEY = 'ai_create_image_ref'
```

- 持久化：任务卡、参考素材、输入草稿、模型偏好
- 不持久化：用户余额、模型配置、账单——这些后端可恢复且易过期

**`auth` store**：
- 持久化 `token` / `refreshToken`，底层 key 兼容旧代码
- 登录态刷新/过期不再用 `window` 自定义事件，改用类型化信号和 `useAuthSessionBridge`

---

## 七、路由结构

```ts
const routes = [
  { path: '',           component: HomePage,         meta: { title: '首页' } },
  { path: 'center',     component: PersonalCenterPage, meta: { requiresAuth: true } },
  { path: 'comic-workflow', component: ComicWorkflowPage },
  { path: 'gallery',    component: GalleryPage },
  { path: 'models',     component: ModelsPage },
  { path: 'membership', component: MembershipPage },
  { path: 'video-generation', component: VideoGenerationPage },
  { path: 'image-generation-create', component: ImageGenerationPage },
  { path: 'ai-create',  component: AiCreatePage },
  { path: 'ai-create/long-video', component: AiCreatePage },
  { path: 'ai-create/3d', component: Model3DWorkbenchPage },
  { path: 'chat',       component: ChatPage },
  { path: 'invite',     component: InvitePage },
  { path: 'legal/privacy',   component: PrivacyPage },
  { path: 'legal/terms',     component: TermsPage },
]
```

所有页面使用懒加载 `() => import(...)`，`MainLayout` 和 `FullscreenLayout` 两套布局壳。

---

## 八、核心约定

1. **样式先拆到模块 CSS**：`src/styles/<module>.css`，使用 `@layer components` 包装
2. **Liquid Glass 原生效果保持原生 CSS**：`color-mix` / `backdrop-filter` / 玻璃阴影不用 Tailwind
3. **一个 composable 一个关注点**：不把图片参数和视频轮询混在一个 composable 里
4. **持久化最小化**：只存跨刷新必要的状态，其余后端可恢复
5. **组件 Teleport**：弹层/下拉框 Teleport 到 body，避免被父容器裁切
6. **无 `as any`、无生产 `console.log`**：全量扫描确认过
