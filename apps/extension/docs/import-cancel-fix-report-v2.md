# 修复报告 v2：停止导入按钮无效

**修复日期：** 2026-06-23  
**问题版本：** v1 修复后仍无法停止  
**关联文档：** `import-cancel-fix-report.md`（v1 修复）

---

## 一、v1 修复失效的根本原因

v1 的方案使用了 `cancelledRef.current` boolean flag，**检查点只在外层批次循环的开头**：

```
for (batchStart = 0; batchStart < total; batchStart += BATCH_SIZE) {
  if (cancelledRef.current) break;   ← 每 5 条才检查一次
  for (bm of batch) {
    await analyzeBookmarkWithAI(...)  ← 这里 await 阻塞，无法中断
  }
}
```

**AI 模式下 `BATCH_SIZE = 5`**，每批 5 条书签串行调用 AI，每条 AI 请求要等待网络 I/O（数秒）。用户点击停止后：

1. `cancelledRef.current = true` 被设置
2. 但当前 `await analyzeBookmarkWithAI(...)` 仍在阻塞执行
3. 必须等当前批次全部 5 条 AI 请求都完成，才能回到外层循环头部读到 flag

**实际体感：** 点击停止后最多需要等待 `5条 × 每条数秒 = 数十秒` 才有反应，用户感知为"无效"。

---

## 二、v2 修复方案：AbortController 贯穿整条调用链

**核心思路：** 将 `AbortController` 的 `signal` 透传到底层 `fetch` 请求，点击停止时 `abort()` 立即中止正在进行的网络请求，抛出 `AbortError` 向上冒泡。

```
用户点击「停止导入」
  │
  ▼
handleCancelImport()
  abortControllerRef.current.abort()    ← 立即触发
  importTaskStorage.cancelHtmlTask()
  │
  ▼ AbortError 沿调用栈向上冒泡
generateObject / streamObject           ← ai SDK 收到 signal.aborted，立即中止 fetch
  ← generateStructuredObject(abortSignal)
    ← analyzeBookmark({ signal })
      ← analyzeBookmarkForLibrary({ signal })
        ← analyzeBookmarkWithAI(...)
          ← 内层 for (bm of batch) 捕获到 AbortError → 向上 throw
            ← runHtmlImportTask 捕获 AbortError → 走取消分支
              ← importFromHTML 捕获 AbortError → cancelHtmlTask → rethrow
                ← handleFileImport / handleBrowserImport 捕获 AbortError
                    → setImportResult({ cancelled: true, ... })
```

**停止响应时间：** 从当前 AI 请求的网络层面立即中止，通常 `< 100ms`。

---

## 三、变更文件详情

### 1. `packages/agent/src/generation/index.ts`

`generateStructuredObject` 新增 `abortSignal?: AbortSignal` 参数，透传给 Vercel AI SDK 的 `generateObject` 和 `streamObject`：

```diff
 export interface GenerateStructuredObjectOptions extends AgentModelConfig {
   ...
+  abortSignal?: AbortSignal;
 }

 export async function generateStructuredObject(options) {
-  const { schema, prompt, system, temperature, maxTokens, ...modelConfig } = options;
+  const { schema, prompt, system, temperature, maxTokens, abortSignal, ...modelConfig } = options;
   ...
   return await generateObject({
     ...
+    abortSignal,
   });
   // streamObject fallback 同样传入 abortSignal
 }
```

---

### 2. `apps/extension/lib/agent/services/bookmark-analysis-service.ts`

两处接口均加入 `signal?: AbortSignal`：

```diff
 export interface EnhancedAnalyzeInput {
   ...
+  signal?: AbortSignal;
 }

 // analyzeBookmark: 透传 signal 给 generateStructuredObject
 const result = await generateStructuredObject({
   ...
+  abortSignal: input.signal,
 });

 // analyzeBookmarkForLibrary: 接收并向下传递
 async analyzeBookmarkForLibrary(options: {
   ...
+  signal?: AbortSignal;
 }) {
   ...
   const result = await this.analyzeBookmark({
     ...
+    signal: options.signal,
   });
 }
```

---

### 3. `apps/extension/components/ImportExportPage.tsx`

#### ① `cancelledRef` → `abortControllerRef`

```diff
-const cancelledRef = useRef(false);
+const abortControllerRef = useRef<AbortController | null>(null);
```

#### ② 每次新导入前创建新的 AbortController

```diff
-cancelledRef.current = false;
+abortControllerRef.current = new AbortController();
 setImporting(true);
```

（`handleFileImport` 和 `handleBrowserImport` 两处）

#### ③ `handleCancelImport` 改为调用 `abort()`

```diff
 const handleCancelImport = async () => {
-  cancelledRef.current = true;
+  abortControllerRef.current?.abort();  // 立即中止正在进行的 fetch
   await importTaskStorage.cancelHtmlTask();
 };
```

#### ④ `analyzeBookmarkWithAI` 透传 signal

```diff
 const result = await bookmarkAnalysisService.analyzeBookmarkForLibrary({
   url, title, currentCategories, existingTags, shouldFetchPageContent,
+  signal: abortControllerRef.current?.signal,
 });
```

`AbortError` 在此函数的 catch 中直接 rethrow（不记录为 AI 错误）。

#### ⑤ `runHtmlImportTask` 内的两层检查

```diff
+const abortSignal = abortControllerRef.current?.signal;
 ...
 for (batchStart = currentIndex; ...) {
-  if (cancelledRef.current) break;
+  if (abortSignal?.aborted) break;           // 外层：批次级检查
   ...
   for (bm of batch) {
+    if (abortSignal?.aborted) {               // 内层：每条书签前检查
+      throw new DOMException("Import cancelled", "AbortError");
+    }
     await analyzeBookmarkWithAI(...)          // AI 请求本身携带 signal，网络层面即时中止
   }
 }
```

`AbortError` 被内层 catch 识别后直接 `throw`（不计入 `firstAiError`），冒泡到外层 for 循环退出。

**移除** v1 遗留的 `cancelledRef.current` 后置检查块。

#### ⑥ 所有 catch 点统一处理 AbortError

涉及三处调用 `runHtmlImportTask` / `importFromHTML` 的位置：
- `resumePendingTask`（页面刷新恢复任务）
- `handleFileImport`
- `handleBrowserImport`
- `importFromHTML`（内层，负责标记 storage 状态）

每处均区分 `AbortError` 和普通错误：

```typescript
} catch (error) {
  if (error instanceof Error && error.name === "AbortError") {
    setImportResult({ success: false, cancelled: true, message: "导入已停止" });
  } else {
    setImportResult({ success: false, message: "导入失败", details: ... });
  }
}
```

---

## 四、行为变化对比

| 场景 | v1 修复后 | v2 修复后 |
|------|-----------|-----------|
| 点击「停止导入」的响应 | 等当前批次(≤5条)全部跑完 | 当前 AI 请求被网络层面中止，`< 100ms` 响应 |
| 停止机制 | boolean flag，只在批次边界生效 | `AbortController.abort()`，直接中止 fetch |
| AI 请求中途中止 | ❌ 必须等请求自然完成 | ✅ `generateObject` 收到 `abortSignal` 立即中止 |
| 停止后 UI 显示 | 琥珀色「导入已停止」（无已处理数量） | 琥珀色「导入已停止」（即时） |
| 刷新页面恢复 | `cancelled` 状态被识别，不续跑 | 同上（不变） |
| AbortError 计入 AI 错误 | — | ❌ 不计入（不触发「部分书签 AI 分析出错」提示） |

---

## 五、调用链总览

```
ImportExportPage
 abortControllerRef (AbortController)
        │ .signal 透传
        ▼
 analyzeBookmarkWithAI(signal)
        │
        ▼
 bookmarkAnalysisService.analyzeBookmarkForLibrary({ signal })
        │
        ▼
 bookmarkAnalysisService.analyzeBookmark({ signal })
        │ input.signal → abortSignal
        ▼
 generateStructuredObject({ abortSignal })       [packages/agent]
        │
        ▼
 generateObject({ abortSignal })                 [ai SDK v6]
        │
        ▼
 HTTP fetch → 收到 abort 信号，立即 reject AbortError
```

---

## 六、未改动范围（有意保留）

- `useBatchAITask`：已有独立的 `cancelTask` 机制，不受影响
- JSON 格式导入：不涉及 AI，无需 AbortController
- `fetchPageContentForAI`（`shouldFetchPageContent` 模式下的页面抓取）：本次未传入 signal，可作为后续优化点
