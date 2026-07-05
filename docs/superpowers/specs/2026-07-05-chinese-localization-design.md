# Synara 中文本地化设计文档

**日期：** 2026-07-05
**状态：** 已批准
**方案：** react-i18next 源码级 i18n + 可逆补丁脚本

---

## 1. 目标

为 Synara 前端 UI 添加完整的中文本地化支持，满足以下约束：

- **可逆补丁**：打上去是中文，恢复就变回英文
- **全量覆盖**：所有 288+ 组件文件中的英文字符串
- **升级兼容**：Synara 版本更新后可快速重新打补丁
- **fallback 安全**：缺少翻译时不崩溃，回退英文显示

## 2. 架构总览

```
D:\项目\synara\
├── apps/web/src/
│   ├── i18n/                    # i18n 配置目录
│   │   ├── index.ts             # i18next 初始化
│   │   └── locales/
│   │       ├── en.json          # 英文翻译（key→英文原文）
│   │       └── zh-CN.json       # 中文翻译（key→中文）
│   ├── components/              # 288+ 文件逐步替换 → t('key')
│   └── confirmDialogFallback.ts # 非 React 场景用 i18next.t()
│
├── scripts/
│   ├── build-zh.ps1             # 构建中文版 app.asar
│   ├── apply-patch.ps1          # 替换已安装应用的 asar → 中文
│   └── revert-patch.ps1         # 恢复英文 asar
│
└── packages/shared/             # pluralize() 等工具需适配中文
```

## 3. 技术选型

**i18n 框架：react-i18next**

选择理由：

- 最成熟的 React i18n 方案，社区最大
- `i18next-scanner` 可自动从源码提取字符串
- `i18next.t()` 可脱离 React 使用，覆盖非 React 文件
- 翻译文件是纯 JSON，天然适合补丁方案
- Vite 原生支持，开箱即用

## 4. 字符串提取策略

### 4.1 四种字符串类型

| 类型 | 占比 | 提取方式 |
|------|------|---------|
| JSX 内联文本 | ~70% | `<h1>Settings</h1>` → `<h1>{t('settings.title')}</h1>` |
| 模板字符串/变量 | ~15% | `` `Update ${provider}` `` → `t('update', { provider })` |
| aria-label 属性 | ~10% | `aria-label="Close"` → `aria-label={t('aria.close')}` |
| 非 React 文件 | ~5% | 直接 `i18next.t('key')` |

### 4.2 非 React 文件（30个）

通过 `document.createElement` + `.textContent` grep 识别的文件：

- `confirmDialogFallback.ts` — 对话框按钮
- `contextMenuFallback.ts` — 右键菜单
- `composerPastedText.ts` — 粘贴文本标签
- `shortcutsSheet.ts` — 快捷键描述
- `settingsNavigation.ts` — 设置导航标签
- 以及 25 个其他文件

这些文件直接 `import i18next from 'i18next'` 调用 `.t()` 方法。

### 4.3 Key 命名规则

格式：`{模块}.{组件}.{元素}`

```json
{
  "common": { "save": "Save", "cancel": "Cancel", "close": "Close" },
  "sidebar": { "search": "Search threads", "newChat": "New chat" },
  "settings": { "title": "Settings", "general": "General" },
  "chat": { "send": "Send message", "stop": "Stop generation" },
  "aria": { "closeDialog": "Close dialog", "sendMessage": "Send message" }
}
```

### 4.4 自动化提取流程

```
1. 安装 i18next-scanner
2. 运行扫描 → 从源码提取所有 t() 调用 → 生成 en.json 骨架
3. 人工审校翻译 → 补齐 zh-CN.json
4. TypeScript 编译时检查确保所有 key 都有类型定义
```

## 5. i18n 初始化

```ts
// apps/web/src/i18n/index.ts
import i18n from 'i18next';
import { initReactI18next } from 'react-i18next';
import en from './locales/en.json';
import zhCN from './locales/zh-CN.json';

i18n.use(initReactI18next).init({
  resources: {
    'en': { translation: en },
    'zh-CN': { translation: zhCN },
  },
  lng: import.meta.env.VITE_LOCALE || 'en',
  fallbackLng: 'en',
  interpolation: { escapeValue: false },
});

export default i18n;
```

在 `main.tsx` 中 `import './i18n'` 即可激活。

## 6. 构建管线

### 6.1 构建命令

```bash
# 中文版
VITE_LOCALE=zh-CN pnpm --filter @t3tools/web build

# 英文版（默认）
pnpm --filter @t3tools/web build
```

### 6.2 补丁脚本

**build-zh.ps1：** 构建中文版 → 输出到 `dist-zh/`

**apply-patch.ps1：**
1. 检测已安装路径（默认 `C:\Users\Mi\AppData\Local\Programs\synara-desktop`）
2. 备份 `resources/app.asar` → `resources/app.asar.bak`
3. 复制 `dist-zh/app.asar` → 覆盖

**revert-patch.ps1：**
1. `resources/app.asar.bak` → 恢复为 `resources/app.asar`
2. 清理备份

### 6.3 升级流程

```
Synara 新版本发布
→ git pull
→ pnpm install
→ pnpm --filter @t3tools/web build (英文构建)
→ scripts/build-zh.ps1 (中文构建)
→ scripts/apply-patch.ps1 (替换已安装版本)
```

## 7. 错误处理

### 7.1 翻译缺失 fallback 链

```
zh-CN.json → en.json → key 本身（如 "settings.title"）
```

i18next 内置此机制，无需额外代码。

### 7.2 翻译文件加载失败

- i18next 会 `console.error` 但不阻塞渲染
- UI 保持可操作，显示英文 key 名

## 8. 测试策略

| 测试类型 | 内容 | 工具 |
|---------|------|------|
| 类型安全 | `t()` 的 key 都在 en.json 中存在 | TypeScript 类型生成 |
| 翻译完整性 | zh-CN.json 覆盖率 100% | vitest 自定义测试 |
| 构建验证 | 中文构建正常完成 | CI 中运行 `build-zh.ps1` |

**翻译完整性测试：**

```ts
test('zh-CN.json covers all en.json keys', () => {
  const enKeys = Object.keys(enJSON);
  const zhKeys = Object.keys(zhJSON);
  const missing = enKeys.filter(k => !zhKeys.includes(k));
  expect(missing).toEqual([]);
});
```

## 9. 边界情况处理

### 9.1 confirmDialogFallback.ts（非 React）

直接 `import i18next` 调用 `.t()`，不依赖 React Hook。i18next 原生支持此用法。

### 9.2 含插值的字符串

```ts
// 英文：`${count} items selected`
// i18n：t('selected', { count })
// en.json: "{{count}} items selected"
// zh-CN.json: "已选择 {{count}} 个项目"
```

中文不区分复数，直接使用数字。

### 9.3 pluralize() 函数

`packages/shared/src/text.ts` 中的 `pluralize()` 函数在英文中追加 "s" 变复数。中文不需要复数变化。

方案：在中文 locale 下 `pluralize()` 直接返回原词。

### 9.4 What's New / Release Notes

`whatsNew/entries.ts` 中的发布说明是营销文案，变动频繁。**策略：暂不翻译**，保持英文。后续按需添加。

## 10. 补丁文件结构

```
synara-zh-patch/
├── apply-patch.ps1          # 一键打补丁（中文）
├── revert-patch.ps1         # 一键恢复（英文）
├── build-zh.ps1             # 重新构建中文版
├── dist-zh/
│   └── app.asar             # 预构建的中文版 asar
├── locales/
│   ├── en.json              # 英文原文（备份）
│   └── zh-CN.json           # 中文翻译
└── README.md                # 使用说明
```

## 11. 不在范围内

- `whatsNew/entries.ts` 发布说明文案
- Electron 自身的 `locales/*.pak`（Chromium 右键菜单等）
- 服务端 `apps/server` 的错误消息（非 UI）
- `packages/contracts` 中的类型定义（无用户可见文本）
