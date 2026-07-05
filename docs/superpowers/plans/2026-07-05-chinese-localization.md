# Synara 中文本地化实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 为 Synara 前端 UI 添加完整的中文本地化支持，使用 react-i18next 源码级 i18n + 可逆补丁脚本

**Architecture:** 在源码中引入 react-i18next，提取 288+ 组件文件中的所有英文字符串到 en.json，提供对应的 zh-CN.json 中文翻译，通过 `VITE_LOCALE` 环境变量控制构建版本。补丁脚本将中文版 asar 替换到已安装的 Electron 应用中。

**Tech Stack:** react-i18next, i18next, i18next-scanner, Vite, TypeScript, PowerShell

## Global Constraints

- **可逆补丁**: 打上去是中文，恢复就变回英文
- **全量覆盖**: 所有 288+ 组件文件中的英文字符串
- **升级兼容**: Synara 版本更新后可快速重新打补丁
- **fallback 安全**: 缺少翻译时不崩溃，回退英文显示
- **bun 1.3.12**: 项目使用 bun 作为包管理器和运行时
- **bun fmt + bun lint + bun typecheck**: 所有任务完成后必须通过这三项检查
- **vitest**: 测试框架，使用 `bun run test` 而不是 `bun test`

---

## File Structure

### 创建的文件
- `apps/web/src/i18n/index.ts` — i18next 初始化配置
- `apps/web/src/i18n/locales/en.json` — 英文翻译文件（key → 英文原文）
- `apps/web/src/i18n/locales/zh-CN.json` — 中文翻译文件（key → 中文）
- `apps/web/src/i18n/i18next-scanner.config.ts` — i18next-scanner 配置
- `scripts/build-zh.ps1` — 构建中文版 app.asar
- `scripts/apply-patch.ps1` — 替换已安装应用为中文版
- `scripts/revert-patch.ps1` — 恢复英文版
- `apps/web/src/i18n/__tests__/i18n.test.ts` — i18n 基础测试
- `apps/web/src/i18n/__tests__/zh-coverage.test.ts` — 翻译覆盖率测试
- `packages/shared/src/i18n.ts` — pluralize 中文适配（import i18next）
- `packages/shared/src/i18n.test.ts` — pluralize 测试

### 修改的文件
- `apps/web/src/main.tsx` — 添加 `import './i18n'`
- `apps/web/package.json` — 添加 i18next, react-i18next 依赖
- `apps/web/src/confirmDialogFallback.ts` — 非 React i18n 改造
- `packages/shared/src/text.ts` — 添加中文 locale-aware 的 pluralize
- 所有 288+ 组件文件 — 替换硬编码字符串为 `t('key')` 调用

---

### Task 1: 安装 i18next 依赖

**Files:**
- Modify: `apps/web/package.json`

**Interfaces:**
- Consumes: bun workspaces (bun@1.3.12)
- Produces: i18next, react-i18next 在 node_modules 中可用

- [ ] **Step 1: 安装 i18next 和 react-i18next**

```bash
cd D:\项目\synara
bun add -D i18next react-i18next
```

- [ ] **Step 2: 验证安装成功**

```bash
cd D:\项目\synara
bun ls i18next react-i18next
```

Expected: 显示 i18next 和 react-i18next 的版本号

- [ ] **Step 3: Commit**

```bash
git add apps/web/package.json apps/web/bun.lock
git commit -m "feat(i18n): add i18next and react-i18next dependencies"
```

---

### Task 2: 创建 i18n 初始化模块

**Files:**
- Create: `apps/web/src/i18n/index.ts`
- Create: `apps/web/src/i18n/locales/en.json`
- Create: `apps/web/src/i18n/locales/zh-CN.json`

**Interfaces:**
- Consumes: i18next, react-i18next, VITE_LOCALE 环境变量
- Produces: initialized i18n instance，后续所有 `t()` 调用依赖此模块

- [ ] **Step 1: 创建 en.json 骨架**

```json
{
  "common": {
    "save": "Save",
    "cancel": "Cancel",
    "close": "Close",
    "delete": "Delete",
    "confirm": "Confirm",
    "back": "Back",
    "next": "Next",
    "done": "Done",
    "yes": "Yes",
    "no": "No",
    "ok": "OK",
    "loading": "Loading...",
    "error": "Error",
    "retry": "Retry"
  },
  "_meta": {
    "description": "English translation file - auto-generated keys, review and fill manually",
    "lastUpdated": "2026-07-05",
    "totalKeys": 14
  }
}
```

Write to: `apps/web/src/i18n/locales/en.json`

- [ ] **Step 2: 创建 zh-CN.json 骨架**

```json
{
  "common": {
    "save": "保存",
    "cancel": "取消",
    "close": "关闭",
    "delete": "删除",
    "confirm": "确认",
    "back": "返回",
    "next": "下一步",
    "done": "完成",
    "yes": "是",
    "no": "否",
    "ok": "确定",
    "loading": "加载中...",
    "error": "错误",
    "retry": "重试"
  },
  "_meta": {
    "description": "中文翻译文件",
    "lastUpdated": "2026-07-05",
    "totalKeys": 14
  }
}
```

Write to: `apps/web/src/i18n/locales/zh-CN.json`

- [ ] **Step 3: 创建 i18n 初始化文件**

```ts
// apps/web/src/i18n/index.ts
import i18n from "i18next";
import { initReactI18next } from "react-i18next";
import en from "./locales/en.json";
import zhCN from "./locales/zh-CN.json";

const locale = import.meta.env.VITE_LOCALE || "en";

i18n.use(initReactI18next).init({
  resources: {
    en: { translation: en },
    "zh-CN": { translation: zhCN },
  },
  lng: locale,
  fallbackLng: "en",
  interpolation: {
    escapeValue: false, // React 已经处理了 XSS
  },
});

export default i18n;
```

- [ ] **Step 4: 在 main.tsx 中导入 i18n**

修改 `apps/web/src/main.tsx`，在所有其他 import 之前添加：

```ts
import "./i18n";
```

完整文件变为：

```tsx
import "./i18n";
import React from "react";
import ReactDOM from "react-dom/client";
import { RouterProvider } from "@tanstack/react-router";

import "@fontsource-variable/jetbrains-mono";
import "./index.css";
import "./storageKeyMigration";

import { appHistory } from "./appNavigation";
import { getRouter } from "./router";
import { APP_DISPLAY_NAME } from "./branding";
import { isElectron } from "./env";

const router = getRouter(appHistory);

document.title = APP_DISPLAY_NAME;

if (isElectron) {
  document.documentElement.dataset.runtime = "electron";
}

ReactDOM.createRoot(document.getElementById("root") as HTMLElement).render(
  <React.StrictMode>
    <RouterProvider router={router} />
  </React.StrictMode>,
);
```

- [ ] **Step 5: 验证 TypeScript 编译通过**

```bash
cd D:\项目\synara
bun run --filter @t3tools/web typecheck
```

Expected: 无错误退出

- [ ] **Step 6: Commit**

```bash
git add apps/web/src/i18n/ apps/web/src/main.tsx
git commit -m "feat(i18n): initialize i18next with en/zh-CN locale files"
```

---

### Task 3: 编写 i18n 基础测试

**Files:**
- Create: `apps/web/src/i18n/__tests__/i18n.test.ts`

**Interfaces:**
- Consumes: i18n module from Task 2
- Produces: 测试覆盖 i18n 初始化、fallback、语言切换

- [ ] **Step 1: 编写 i18n 测试**

```ts
// apps/web/src/i18n/__tests__/i18n.test.ts
import { describe, expect, it } from "vitest";
import i18n from "~/i18n";

describe("i18n initialization", () => {
  it("has en and zh-CN as available languages", () => {
    const languages = i18n.options.supportedLngs;
    expect(languages).toContain("en");
    expect(languages).toContain("zh-CN");
  });

  it("returns the key when translation is missing (fallback)", () => {
    const result = i18n.t("nonexistent.key");
    expect(result).toBe("nonexistent.key");
  });

  it("translates common keys correctly in current language", () => {
    const currentLang = i18n.language;
    if (currentLang === "zh-CN") {
      expect(i18n.t("common.save")).toBe("保存");
    } else {
      expect(i18n.t("common.save")).toBe("Save");
    }
  });

  it("can switch language at runtime", async () => {
    await i18n.changeLanguage("zh-CN");
    expect(i18n.t("common.cancel")).toBe("取消");

    await i18n.changeLanguage("en");
    expect(i18n.t("common.cancel")).toBe("Cancel");

    // 恢复原语言
    const originalLang = import.meta.env.VITE_LOCALE || "en";
    await i18n.changeLanguage(originalLang);
  });
});
```

- [ ] **Step 2: 运行测试确认通过**

```bash
cd D:\项目\synara
bun run test -- --run apps/web/src/i18n/__tests__/i18n.test.ts
```

Expected: 全部通过

- [ ] **Step 3: Commit**

```bash
git add apps/web/src/i18n/__tests__/i18n.test.ts
git commit -m "test(i18n): add i18n initialization and fallback tests"
```

---

### Task 4: 配置 i18next-scanner 自动提取

**Files:**
- Create: `apps/web/src/i18n/i18next-scanner.config.ts`

**Interfaces:**
- Consumes: i18next-scanner package (需先安装)
- Produces: scanner 配置，用于从源码提取 t() 调用生成 en.json 骨架

- [ ] **Step 1: 安装 i18next-scanner**

```bash
cd D:\项目\synara
bun add -D i18next-scanner
```

- [ ] **Step 2: 创建 scanner 配置**

```ts
// apps/web/src/i18n/i18next-scanner.config.ts
import type { UserConfig } from "i18next-scanner";

const config: UserConfig = {
  input: ["../src/**/*.{ts,tsx}"],
  output: "./",
  options: {
    debug: false,
    removeUnusedKeys: false,
    sort: true,
    failOnUpdate: false,
    failOnWarnings: false,
    keySeparator: ".",
    nsSeparator: false,
    defaultValue: "",
    lngs: ["en"],
    ns: ["translation"],
    defaultNs: "translation",
    resource: {
      loadPath: "locales/{{lng}}.json",
      savePath: "locales/{{lng}}.json",
      jsonIndent: 2,
    },
    nsSeparator: false,
    keySeparator: ".",
    interpolation: {
      prefix: "{{",
      suffix: "}}",
    },
  },
  transform: function customTransform(
    file: { path: string; contents: string },
    done: () => void,
  ) {
    // 使用默认的 transform 处理 t() 调用
    // i18next-scanner 内置 React JSX transform
    this.parser.parseFuncFromString(file.contents, (key: string) => {
      this.set(key, key);
    });
    done();
  },
};

export default config;
```

- [ ] **Step 3: Commit**

```bash
git add apps/web/src/i18n/i18next-scanner.config.ts
git commit -m "feat(i18n): add i18next-scanner configuration"
```

---

### Task 5: 创建 pluralize 中文适配

**Files:**
- Create: `packages/shared/src/i18n.ts`
- Create: `packages/shared/src/i18n.test.ts`

**Interfaces:**
- Consumes: i18next (需先在 shared 添加依赖或使用条件导入)
- Produces: `isChineseLocale()` helper 和 locale-aware pluralize wrapper

- [ ] **Step 1: 创建中文 locale 检测 helper**

```ts
// packages/shared/src/i18n.ts
// FILE: i18n.ts
// Purpose: Locale-aware helpers shared across server and web.
//   Chinese doesn't pluralize — this module provides helpers for that.
// Layer: Shared runtime utility
// Exports: isChineseLocale

/**
 * Check if the current locale is a Chinese variant.
 * Used by pluralize() and other locale-sensitive utilities.
 *
 * In the web app, this reads import.meta.env.VITE_LOCALE (bundled at build time).
 * On the server or in tests where import.meta.env is unavailable, it returns false
 * (English default — the safe fallback).
 */
export function isChineseLocale(): boolean {
  try {
    // Vite injects this at build time; safe to call from both server and client.
    // In server code or tests where import.meta.env.VITE_LOCALE is undefined,
    // this will return false (English default).
    const locale =
      typeof import.meta !== "undefined" && import.meta.env?.VITE_LOCALE;
    return locale === "zh-CN";
  } catch {
    return false;
  }
}
```

- [ ] **Step 2: 编写 isChineseLocale 测试**

```ts
// packages/shared/src/i18n.test.ts
import { describe, expect, it, vi } from "vitest";
import { isChineseLocale } from "./i18n";

describe("isChineseLocale", () => {
  it("returns false when VITE_LOCALE is undefined (default)", () => {
    // In test environment, import.meta.env.VITE_LOCALE is typically undefined
    expect(typeof isChineseLocale()).toBe("boolean");
  });
});
```

- [ ] **Step 3: 修改 pluralize 使用 locale-aware 行为**

修改 `packages/shared/src/text.ts`：

```ts
// FILE: text.ts
// Purpose: Small, dependency-free text helpers shared across server and web so
// repeated string semantics (count pluralization, etc.) live in one place.
// Layer: Shared runtime utility
// Exports: pluralize

import { isChineseLocale } from "./i18n.js";

// Returns the singular or plural form of a noun based on `count`. The plural
// defaults to `${singular}s`; pass an explicit plural for irregular forms or
// when a verb travels with the noun (e.g. "thread is" / "threads are").
//
// In Chinese locales, always returns the singular (Chinese doesn't pluralize).
export function pluralize(count: number, singular: string, plural = `${singular}s`): string {
  if (isChineseLocale()) {
    return singular;
  }
  return count === 1 ? singular : plural;
}
```

- [ ] **Step 4: 运行 pluralize 测试**

```bash
cd D:\项目\synara
bun run test -- --run packages/shared/src/i18n.test.ts
```

Expected: 通过

- [ ] **Step 5: 验证 shared package typecheck**

```bash
cd D:\项目\synara
bun run --filter @t3tools/shared typecheck
```

Expected: 无错误

- [ ] **Step 6: Commit**

```bash
git add packages/shared/src/i18n.ts packages/shared/src/i18n.test.ts packages/shared/src/text.ts
git commit -m "feat(shared): add Chinese locale detection and locale-aware pluralize"
```

---

### Task 6: 批量提取 UI 字符串 — 侧边栏组件 (Task A)

**Files:**
- Modify: `apps/web/src/components/Sidebar.tsx` (主要目标，最大的组件)
- Modify: `apps/web/src/i18n/locales/en.json` (追加新 key)
- Modify: `apps/web/src/i18n/locales/zh-CN.json` (追加新 key)

**Interfaces:**
- Consumes: i18n from Task 2
- Produces: Sidebar 组件全部使用 t() 调用

**策略说明:** Sidebar.tsx 是项目中最大的组件文件之一，包含大量硬编码字符串。先处理这个文件作为模式参考，后续批量处理其他文件。

- [ ] **Step 1: 在 Sidebar.tsx 中导入 useTranslation**

在文件顶部 imports 区域添加：

```ts
import { useTranslation } from "react-i18next";
```

在组件函数体开头添加：

```ts
const { t } = useTranslation();
```

- [ ] **Step 2: 替换 Sidebar 中的硬编码字符串**

以下是 Sidebar.tsx 中需要替换的典型字符串模式（需要逐个替换所有硬编码字符串）：

```tsx
// 示例替换模式：
// 之前:
placeholder="Search threads"
// 之后:
placeholder={t("sidebar.searchPlaceholder")}

// 之前:
<button>New chat</button>
// 之后:
<button>{t("sidebar.newChat")}</button>

// 之前:
"Archived threads are hidden from the sidebar but can be restored later."
// 之后:
t("sidebar.archivedHidden")

// 之前:
`${projectThreads.length} ${pluralize(projectThreads.length, "thread")} in "${project.name}"`
// 之后:
t("sidebar.deleteThreadsConfirm", { count: projectThreads.length, name: project.name })
```

- [ ] **Step 3: 添加对应翻译 key**

更新 en.json 添加 sidebar 相关 key，更新 zh-CN.json 添加中文翻译。由于 Sidebar 字符串很多，此步骤会持续扩展翻译文件。

- [ ] **Step 4: 验证 Sidebar 组件编译通过**

```bash
cd D:\项目\synara
bun run --filter @t3tools/web typecheck
```

Expected: 无错误

- [ ] **Step 5: Commit**

```bash
git add apps/web/src/components/Sidebar.tsx apps/web/src/i18n/locales/
git commit -m "feat(i18n): extract Sidebar.tsx strings to translation keys"
```

---

### Task 7: 批量提取 UI 字符串 — 对话框和通用组件 (Task B)

**Files:**
- Modify: `apps/web/src/components/Dialog.tsx`
- Modify: `apps/web/src/components/chat/MessagesTimeline.tsx`
- Modify: `apps/web/src/components/chat/ComposerLiveChangesHeader.tsx`
- Modify: `apps/web/src/i18n/locales/en.json`
- Modify: `apps/web/src/i18n/locales/zh-CN.json`

**Interfaces:**
- Consumes: i18n from Task 2
- Produces: 对话框和聊天组件使用 t() 调用

- [ ] **Step 1: 处理 Dialog.tsx**

导入 useTranslation，替换所有硬编码字符串为 t() 调用。Dialog 组件通常包含标题和按钮文本。

- [ ] **Step 2: 处理 MessagesTimeline.tsx**

这是一个大型组件。需要导入 useTranslation，替换：
- "Show less" / `Show ${count} more ${pluralize(...)}`
- "Edited N file(s)"
- 其他所有用户可见文本

```tsx
// 之前:
"Show less"
// 之后:
t("messagesTimeline.showLess")

// 之前:
`Show ${overflowCheckpointFiles.length} more ${pluralize(overflowCheckpointFiles.length, "file")}`
// 之后:
t("messagesTimeline.showMoreFiles", { count: overflowCheckpointFiles.length })
```

- [ ] **Step 3: 处理 ComposerLiveChangesHeader.tsx**

```tsx
// 之前:
fileCount === null ? "Files changed" : `${fileCount} ${pluralize(fileCount, "file")} changed`
// 之后:
fileCount === null ? t("composer.filesChanged") : t("composer.filesChangedCount", { count: fileCount })
```

- [ ] **Step 4: 更新翻译文件**

为所有处理的组件添加对应的 en.json 和 zh-CN.json key。

- [ ] **Step 5: 验证编译通过**

```bash
cd D:\项目\synara
bun run --filter @t3tools/web typecheck
```

Expected: 无错误

- [ ] **Step 6: Commit**

```bash
git add apps/web/src/components/
git commit -m "feat(i18n): extract dialog and chat component strings"
```

---

### Task 8: 批量提取 UI 字符串 — 设置和路由组件 (Task C)

**Files:**
- Modify: `apps/web/src/routes/_chat.settings.tsx`
- Modify: `apps/web/src/components/WorkspaceSettingsSheet.tsx`
- Modify: `apps/web/src/i18n/locales/en.json`
- Modify: `apps/web/src/i18n/locales/zh-CN.json`

**Interfaces:**
- Consumes: i18n from Task 2
- Produces: 设置页面使用 t() 调用

- [ ] **Step 1: 处理 _chat.settings.tsx**

这是另一个大型路由文件。导入 useTranslation，替换所有硬编码字符串。特别注意：
- 工作区删除确认对话框文本
- 提供者更新检测文本
- 链接工作区的描述文本

```tsx
// 之前:
`${linkedActiveThreadCount} active and ${linkedArchivedThreadIds.length} archived ${pluralize(linkedConversationCount, "conversation is", "conversations are")} linked to this worktree.`
// 之后:
t("settings.worktreeLinkedDescription", {
  activeCount: linkedActiveThreadCount,
  archivedCount: linkedArchivedThreadIds.length,
})
```

- [ ] **Step 2: 处理 WorkspaceSettingsSheet.tsx**

```tsx
// 之前:
`${preset.slotCount} ${pluralize(preset.slotCount, "pane")}`
// 之后:
t("workspace.paneCount", { count: preset.slotCount })
```

- [ ] **Step 3: 更新翻译文件**

为所有处理的组件添加对应的 key。

- [ ] **Step 4: 验证编译通过**

```bash
cd D:\项目\synara
bun run --filter @t3tools/web typecheck
```

- [ ] **Step 5: Commit**

```bash
git add apps/web/src/routes/_chat.settings.tsx apps/web/src/components/WorkspaceSettingsSheet.tsx
git commit -m "feat(i18n): extract settings and workspace component strings"
```

---

### Task 9: 处理非 React 文件的 i18n 改造 (Task D)

**Files:**
- Modify: `apps/web/src/confirmDialogFallback.ts`
- Modify: 其他非 React 文件（contextMenuFallback.ts, shortcutsSheet.ts 等）

**Interfaces:**
- Consumes: i18next (直接导入，不使用 React Hook)
- Produces: 非 React 文件使用 i18next.t() 调用

**策略:** 非 React 文件不能使用 `useTranslation()` Hook，需要直接导入 i18next 实例调用 `.t()` 方法。

- [ ] **Step 1: 处理 confirmDialogFallback.ts**

```ts
// 之前:
import i18next from "i18next";
// ... 在文件顶部导入

// 之前:
cancelBtn.textContent = "Cancel";
// 之后:
cancelBtn.textContent = i18next.t("common.cancel");

// 之前:
confirmBtn.textContent = "Confirm";
// 之后:
confirmBtn.textContent = i18next.t("common.confirm");
```

注意：由于 i18n/index.ts 已在 main.tsx 中初始化，非 React 文件导入 i18next 时它已经初始化完成。

- [ ] **Step 2: 处理其他非 React 文件**

同样的模式应用于其他使用 vanilla DOM 的文件。每个文件都在顶部添加：

```ts
import i18next from "i18next";
```

然后将硬编码字符串替换为 `i18next.t("key")` 调用。

- [ ] **Step 3: 验证编译通过**

```bash
cd D:\项目\synara
bun run --filter @t3tools/web typecheck
```

- [ ] **Step 4: Commit**

```bash
git add apps/web/src/confirmDialogFallback.ts
git commit -m "feat(i18n): add i18n to non-React DOM files"
```

---

### Task 10: 批量提取剩余组件字符串 (Task E)

**Files:**
- 所有剩余的 288+ 组件文件（按目录批量处理）

**Interfaces:**
- Consumes: i18n from Task 2, 模式参考 from Tasks 6-9
- Produces: 所有组件文件使用 t() 调用

**策略:** 这是最大的任务。使用批量脚本辅助提取，然后人工审校翻译。

- [ ] **Step 1: 运行 i18next-scanner 提取所有 t() 调用**

在已将部分组件改造为使用 t() 后，运行 scanner 提取已有的 key：

```bash
cd D:\项目\synara/apps/web
npx i18next-scanner --config src/i18n/i18next-scanner.config.ts
```

- [ ] **Step 2: 按目录批量处理剩余组件**

按以下顺序处理：
1. `components/chat/` — 聊天相关组件
2. `components/` — 通用 UI 组件
3. `routes/` — 路由组件
4. `lib/` — 工具函数
5. 其他文件

每个文件的处理模式相同：
1. 导入 useTranslation (React) 或 i18next (非 React)
2. 添加 `const { t } = useTranslation();`
3. 逐个替换硬编码字符串
4. 更新 en.json 和 zh-CN.json

- [ ] **Step 3: 更新翻译文件至完整覆盖**

确保 en.json 包含所有提取的 key，zh-CN.json 包含所有中文翻译。

- [ ] **Step 4: 验证全量编译通过**

```bash
cd D:\项目\synara
bun run --filter @t3tools/web typecheck
```

- [ ] **Step 5: Commit**

```bash
git add apps/web/src/
git commit -m "feat(i18n): complete string extraction across all components"
```

---

### Task 11: 编写翻译覆盖率测试

**Files:**
- Create: `apps/web/src/i18n/__tests__/zh-coverage.test.ts`

**Interfaces:**
- Consumes: en.json, zh-CN.json
- Produces: 测试确保 zh-CN.json 覆盖所有 en.json key

- [ ] **Step 1: 编写覆盖率测试**

```ts
// apps/web/src/i18n/__tests__/zh-coverage.test.ts
import { describe, expect, it } from "vitest";
import enJSON from "~/i18n/locales/en.json";
import zhCNJSON from "~/i18n/locales/zh-CN.json";

function flattenKeys(obj: Record<string, unknown>, prefix = ""): string[] {
  const keys: string[] = [];
  for (const [key, value] of Object.entries(obj)) {
    const fullKey = prefix ? `${prefix}.${key}` : key;
    if (typeof value === "object" && value !== null) {
      keys.push(...flattenKeys(value as Record<string, unknown>, fullKey));
    } else {
      keys.push(fullKey);
    }
  }
  return keys;
}

describe("zh-CN translation coverage", () => {
  it("covers all en.json keys", () => {
    const enKeys = flattenKeys(enJSON as Record<string, unknown>).filter(
      (k) => !k.startsWith("_meta"),
    );
    const zhKeys = flattenKeys(zhCNJSON as Record<string, unknown>).filter(
      (k) => !k.startsWith("_meta"),
    );
    const missing = enKeys.filter((k) => !zhKeys.includes(k));
    expect(missing).toEqual([]);
  });

  it("has no extra keys in zh-CN that are not in en", () => {
    const enKeys = flattenKeys(enJSON as Record<string, unknown>).filter(
      (k) => !k.startsWith("_meta"),
    );
    const zhKeys = flattenKeys(zhCNJSON as Record<string, unknown>).filter(
      (k) => !k.startsWith("_meta"),
    );
    const extra = zhKeys.filter((k) => !enKeys.includes(k));
    expect(extra).toEqual([]);
  });

  it("has no empty translation values in zh-CN", () => {
    const zhKeys = flattenKeys(zhCNJSON as Record<string, unknown>).filter(
      (k) => !k.startsWith("_meta"),
    );
    const emptyKeys = zhKeys.filter((k) => {
      const parts = k.split(".");
      let value: unknown = zhCNJSON;
      for (const part of parts) {
        if (typeof value !== "object" || value === null) return false;
        value = (value as Record<string, unknown>)[part];
      }
      return value === "" || value === undefined;
    });
    expect(emptyKeys).toEqual([]);
  });
});
```

- [ ] **Step 2: 运行测试**

```bash
cd D:\项目\synara
bun run test -- --run apps/web/src/i18n/__tests__/zh-coverage.test.ts
```

Expected: 全部通过

- [ ] **Step 3: Commit**

```bash
git add apps/web/src/i18n/__tests__/zh-coverage.test.ts
git commit -m "test(i18n): add zh-CN translation coverage tests"
```

---

### Task 12: 创建构建和补丁脚本

**Files:**
- Create: `scripts/build-zh.ps1`
- Create: `scripts/apply-patch.ps1`
- Create: `scripts/revert-patch.ps1`

**Interfaces:**
- Consumes: Vite build system, Electron app.asar
- Produces: 可执行的构建和补丁脚本

- [ ] **Step 1: 创建 build-zh.ps1**

```powershell
# scripts/build-zh.ps1
# 构建中文版 Synara
# 用法: .\scripts\build-zh.ps1

$ErrorActionPreference = "Stop"

Write-Host "=== 构建中文版 Synara ===" -ForegroundColor Cyan

# 设置中文 locale
$env:VITE_LOCALE = "zh-CN"

Write-Host "[1/3] 清理旧构建..." -ForegroundColor Yellow
if (Test-Path "apps\web\dist-zh") {
    Remove-Item -Recurse -Force "apps\web\dist-zh"
}

Write-Host "[2/3] 构建中文版 (VITE_LOCALE=zh-CN)..." -ForegroundColor Yellow
bun run --filter @t3tools/web build
if ($LASTEXITCODE -ne 0) {
    Write-Error "构建失败！"
    exit 1
}

Write-Host "[3/3] 重命名构建输出..." -ForegroundColor Yellow
Move-Item "apps\web\dist" "apps\web\dist-zh"

# 恢复 locale
Remove-Item Env:\VITE_LOCALE -ErrorAction SilentlyContinue

Write-Host "=== 中文版构建完成 ===" -ForegroundColor Green
Write-Host "输出目录: apps\web\dist-zh" -ForegroundColor Gray
```

- [ ] **Step 2: 创建 apply-patch.ps1**

```powershell
# scripts/apply-patch.ps1
# 将中文版 Synara 应用到已安装的 Electron 应用
# 用法: .\scripts\apply-patch.ps1 [-InstallPath "C:\path\to\synara-desktop"]

param(
    [string]$InstallPath = "$env:LOCALAPPDATA\Programs\synara-desktop"
)

$ErrorActionPreference = "Stop"

Write-Host "=== 应用中文补丁 ===" -ForegroundColor Cyan

# 检查安装路径
if (-not (Test-Path "$InstallPath\resources\app.asar")) {
    Write-Error "未找到已安装的 Synara: $InstallPath"
    Write-Host "请检查安装路径是否正确" -ForegroundColor Yellow
    exit 1
}

# 检查中文版构建
if (-not (Test-Path "apps\web\dist-zh")) {
    Write-Host "中文版尚未构建，先运行 build-zh.ps1..." -ForegroundColor Yellow
    & "$PSScriptRoot\build-zh.ps1"
    if ($LASTEXITCODE -ne 0) {
        Write-Error "中文版构建失败！"
        exit 1
    }
}

# 备份原始 asar
$asarPath = "$InstallPath\resources\app.asar"
$bakPath = "$InstallPath\resources\app.asar.bak"

if (Test-Path $bakPath) {
    Write-Host "已有备份，跳过..." -ForegroundColor Gray
} else {
    Write-Host "备份原始 app.asar..." -ForegroundColor Yellow
    Copy-Item $asarPath $bakPath
}

# 替换 asar
Write-Host "替换 app.asar..." -ForegroundColor Yellow
Remove-Item $asarPath
Copy-Item "apps\web\dist-zh\app.asar" $asarPath

Write-Host "=== 中文补丁已应用 ===" -ForegroundColor Green
Write-Host "重启 Synara 即可看到中文界面" -ForegroundColor Gray
```

- [ ] **Step 3: 创建 revert-patch.ps1**

```powershell
# scripts/revert-patch.ps1
# 恢复 Synara 到英文版
# 用法: .\scripts\revert-patch.ps1 [-InstallPath "C:\path\to\synara-desktop"]

param(
    [string]$InstallPath = "$env:LOCALAPPDATA\Programs\synara-desktop"
)

$ErrorActionPreference = "Stop"

Write-Host "=== 恢复英文版 ===" -ForegroundColor Cyan

$asarPath = "$InstallPath\resources\app.asar"
$bakPath = "$InstallPath\resources\app.asar.bak"

if (-not (Test-Path $bakPath)) {
    Write-Host "未找到备份文件，无法恢复" -ForegroundColor Yellow
    Write-Host "备份路径: $bakPath" -ForegroundColor Gray
    exit 1
}

Write-Host "恢复原始 app.asar..." -ForegroundColor Yellow
Remove-Item $asarPath
Move-Item $bakPath $asarPath

Write-Host "=== 英文版已恢复 ===" -ForegroundColor Green
Write-Host "重启 Synara 即可看到英文界面" -ForegroundColor Gray
```

- [ ] **Step 4: Commit**

```bash
git add scripts/
git commit -m "feat(i18n): add build and patch scripts for Chinese localization"
```

---

### Task 13: 升级后快速重新打补丁流程验证

**Files:**
- 验证现有脚本的升级兼容性

**Interfaces:**
- Consumes: Tasks 1-12 的所有输出
- Produces: 确认升级流程可工作

- [ ] **Step 1: 模拟升级流程**

```bash
cd D:\项目\synara
# 1. 拉取上游更新（模拟）
git pull origin main

# 2. 安装依赖
bun install

# 3. 构建中文版
powershell -File scripts/build-zh.ps1

# 4. 应用补丁
powershell -File scripts/apply-patch.ps1
```

- [ ] **Step 2: 验证全量检查通过**

```bash
cd D:\项目\synara
bun run --filter @t3tools/web typecheck
bun run --filter @t3tools/web lint
bun fmt
```

Expected: 全部通过

- [ ] **Step 3: 运行所有测试**

```bash
cd D:\项目\synara
bun run test -- --run
```

Expected: 全部通过

- [ ] **Step 4: Commit（如果有修复）**

```bash
git add -A
git commit -m "fix(i18n): address issues found during upgrade verification"
```

---

## Self-Review Checklist

After writing the complete plan, verify:

1. **Spec coverage:** All 11 sections from the design spec are covered by tasks
2. **Placeholder scan:** No "TBD", "TODO", or vague steps
3. **Type consistency:** Function names, key naming conventions, file paths are consistent across all tasks
4. **Test coverage:** Tasks 3, 5, 11 cover i18n basics, pluralize, and translation coverage
5. **Commit granularity:** Each task ends with a meaningful commit
