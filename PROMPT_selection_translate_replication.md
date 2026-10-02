# PROMPT：为你的项目从零实现「划选翻译」功能（自包含规格，无需访问参考仓库）

> 本 prompt 自包含：架构、算法、请求格式、关键代码全部内嵌，**不依赖阅读任何参考仓库即可实现**。
> 文末「参考实现」仅作对照用（MyReader/Readest fork，公开仓库，链接可能随版本漂移）。
> `【适配】`标记处需按你的项目替换。

---

## 0. 目标与最终形态

在阅读器（或任何长文本界面）中实现：用户**划选文本** → 点工具栏「翻译」按钮（或 Ctrl/Cmd+T）→ 弹出**锚定在选区附近**的翻译弹窗 → 调用**免 API key 的翻译接口**展示译文。要求：

- 四层解耦（selection / trigger / service / popup），以后换翻译服务只改 service 一层
- 译文缓存（同一文本第二次弹出秒出）
- 断网/失败有可读错误态，不崩溃

## 1. 架构总览

```
┌─ Selection 层 ──┐   ┌─ Trigger 层 ──┐   ┌─ Service 层 ────────────┐   ┌─ Popup 层 ─┐
│ selectionchange │ → │ 工具栏按钮     │ → │ provider 注册表          │ → │ 锚定弹窗    │
│ 有效性过滤/去重  │   │ Ctrl/Cmd+T    │   │  ├ edge（免key，默认）    │   │ 原文/译文   │
│ 文本提取+锚点矩形│   │ quick action  │   │  ├ （预留新 provider）    │   │ 语言切换    │
└────────┬────────┘   └───────────────┘   │  缓存(内存+IndexedDB)    │   └────────────┘
         │ 锚点矩形（滚动时重算并广播）      │  preprocess → polish     │
         ▼                                └──────────┬───────────────┘
   弹窗定位消费坐标                                    │ 读写 Settings：
                                        translationProvider / translateTargetLang
```

## 2. Selection 层 —— 选词检测（核心代码骨架）

### 2.1 监听与过滤

```ts
/** 挂在阅读器分节 document 上（每节一个实例），卸载时必须移除 */
function attachSelectionListener(doc: Document, onSelect: (sel: SelectionInfo | null) => void) {
  let timer: ReturnType<typeof setTimeout> | null = null
  let lastKey = ''
  const onChange = () => {
    if (timer) clearTimeout(timer)
    timer = setTimeout(() => {            // debounce ~200ms：selectionchange 高频触发
      const info = extractSelection(doc)
      const key = info ? `${info.text}|${info.range.startContainer}_${info.range.startOffset}_${info.range.endOffset}` : ''
      if (key === lastKey) return         // 去重：同一次划选触发多次 selectionchange
      lastKey = key
      onSelect(info)
    }, 200)
  }
  doc.addEventListener('selectionchange', onChange)
  return () => doc.removeEventListener('selectionchange', onChange)
}
```

有效性过滤（`extractSelection` 内）：

- `selection.isCollapsed` → null（非划选）
- 文本 trim 后为空 / 长度 > 5000 → null
- `range.commonAncestorContainer` 落在 `input`/`textarea`/`[contenteditable]` 内 → null
- 选区起点终点不在同一渲染节（分页阅读器每节一个 iframe/document）→ null

### 2.2 文本提取（重要）

`range.toString()` 会把 Ruby 注音、脚注上标、连字符断词的排版文本混进来。规则：

- **阅读器项目**：复用阅读器已有的注释/高亮文本提取函数（foliate-js 系的阅读器都有 `getAnnotationText(range)`，能跨节点合并、还原断词、剥注音）。划词查词、批注、翻译必须**共用同一个提取入口**。
- 普通网页/无现成函数：`range.toString()` 兜底即可。

### 2.3 锚点矩形（弹窗定位依据）

```ts
function selectionAnchor(range: Range): DOMRect {
  const rects = range.getClientRects()
  if (rects.length === 0) return range.getBoundingClientRect()
  const first = rects[0], last = rects[rects.length - 1]
  // 弹窗锚在「末行」：末行左边缘，底边为弹窗上缘参考
  return new DOMRect(first.left, last.top, last.right - first.left, last.bottom - first.top)
}
```

**滚动/翻页/字号/边距变化后必须重算**：把「重新计算 anchor → 通知弹窗更新位置」做成回调广播；弹窗自己不监听滚动，只消费最新坐标。

### 2.4 移动端兜底（可选）

touch 场景 `selectionchange` 不可靠：`touchstart` 记录起点 → `touchmove` 判定是选择（水平+纵向位移 > 阈值）而非滚动 → `touchend` 后 300ms 读一次选区。PDF/固定版式可省略划选，用右键菜单直接把当前选区喂给触发层。

## 3. Trigger 层 —— 触发入口（至少做前两个）

| 入口 | 行为 |
|---|---|
| 工具栏「翻译」按钮 | 划选后可用（无选区时禁用）；点击才弹，**不是划选即弹**（避免打扰阅读） |
| 快捷键 Ctrl/Cmd+T | 对当前选区生效 |
| 长按/双击 quick action（可选） | 直接弹 |
| 右键菜单（可选） | 选区右键直接弹翻译框 |

**所有入口汇聚到同一个 handler**（禁止两套逻辑）：

```ts
function handleTranslation() {
  if (!currentSelection) return
  closeOtherAnchoredPopups()      // 锚定型弹窗（批注/查词/翻译）单弹窗仲裁：开一关一
  openTranslatorPopup({ text: currentSelection.text, anchor: currentSelection.anchor })
}
```

## 4. Service 层 —— 翻译服务（provider 可插拔 + 免 key 的 Edge 接口）

### 4.1 Provider 接口与注册表

```ts
export interface TranslationProvider {
  name: string
  translate(texts: string[], from: string, to: string): Promise<string[]>  // 批量，返回顺序一一对应
}

const providers: Record<string, TranslationProvider> = { edge: edgeProvider }
// 以后加 google/deepl/openai 只在这里注册，UI 与上层代码零改动

function resolveProvider(name?: string): TranslationProvider {
  return (name && providers[name]) || Object.values(providers)[0]
}
```

### 4.2 默认 provider：Microsoft Edge 免费接口（无鉴权、无 key）

这是 Edge 浏览器内置翻译服务，**未见于微软官方文档**、可能随版本变动——这正是 provider 必须可插拔的原因（付费稳定替代：Azure Translator，见 https://learn.microsoft.com/azure/ai-services/translator/ ）。

```ts
// 完整实现，可直接使用
const EDGE_UA =
  'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36 Edg/120.0.0.0'

export const edgeProvider: TranslationProvider = {
  name: 'edge',
  async translate(texts, from, to) {
    const url = new URL('https://edge.microsoft.com/translate/translatetext')
    url.searchParams.set('to', to)
    url.searchParams.set('isEnterpriseClient', 'false')
    if (from && from !== 'auto') url.searchParams.set('from', from)
    const res = await fetch(url, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', 'User-Agent': EDGE_UA },  // UA 必须伪装 Edg
      body: JSON.stringify(texts),           // ⚠️ 裸字符串数组，不是 {text:...} 对象包裹
    })
    if (!res.ok) throw new Error(`translate HTTP ${res.status}`)
    const data = await res.json()
    return data.map((item: any) => item?.translations?.[0]?.text ?? '')
  },
}
```

请求/响应实测规格（照抄即可）：

```
POST https://edge.microsoft.com/translate/translatetext?to=zh-CN&isEnterpriseClient=false[&from=en]
Content-Type: application/json
User-Agent: ... Edg/120.0.0.0
body: ["Hello world"]                              ← 纯 JSON 数组
→ [ { "translations": [ { "text": "你好，世界", "to": "zh-CN" } ] } ]
```

### 4.3 CORS 通道【适配】

- **纯网页**：该端点返回带 `Access-Control-Allow-Origin`，`fetch` 直连可行。
- **Tauri**：用 `@tauri-apps/plugin-http` 的 fetch（免 CORS、UA 可控）。
- **Electron**：主进程 `net.fetch` 转发。
- **浏览器扩展**：background/service worker fetch + host 权限。
- 若目标环境两个都不行，退化为自有代理转发。

### 4.4 编排 hook：缓存 → 翻译 → polish

```ts
async function translate(texts: string[], opts: { provider?: string; from?: string; to: string }) {
  const active = resolveProvider(opts.provider)
  const from = opts.from ?? 'auto'
  // 1) 读缓存（key 含 provider 与语言，provider 换了缓存自然失效）
  const hits = await Promise.all(texts.map((t) => cacheGet(active.name, from, opts.to, t.trim())))
  if (hits.every(Boolean)) return hits
  // 2) 未命中的批量调用（划选场景永远只有 1 条，但签名保持数组——整段/整页翻译直接复用）
  const missing = texts.filter((_, i) => !hits[i])
  const out = await active.translate(missing, from, opts.to)
  // 3) 写缓存 + polish 后处理
  let j = 0
  return Promise.all(texts.map(async (t, i) => {
    if (hits[i]) return hits[i]
    const polished = polish(out[j], opts.to)
    await cacheSet(active.name, from, opts.to, t.trim(), polished)
    j++
    return polished
  }))
}
```

**缓存**：两层——内存 Map（会话内）+ IndexedDB/localStorage（跨会话）；key = `provider:from:to:text`（from 为 auto 也照写）；文本进 key 前 trim。

**polish 后处理**（只做「确定无害」的清理，不改语义）：

- 目标语言是中文：折叠词间多余空格（Edge 译文常带英文式空格）、统一标点
- 目标语言是英文：保留空格、规范首字母/标点
- 永远不做语序改写

### 4.5 语言码归一化

UI 内部语言码（`zh`/`en`，常缺地区）→ 接口全码（`zh-Hans`/`zh-Hant`/`pt-BR`…）必须走**一张显式映射表**，集中一个 `normalizeToFullLang()`；目标语言下拉的选项就是这张表。禁止在调用点散写转换、禁止拿接口返回的语言码反猜。

## 5. Popup 层 —— 翻译弹窗

**组件规格**（锚定型 Popup，贴选区）：

- 定位：优先放锚点矩形（末行）**下方** 8px；视口放不下翻到**上方**；水平夹在视口内（左右各留 12px）。消费 Selection 层广播的最新 anchor。
- 双窗格布局：
  - 原文窗格（可滚动，显示提取的原始文本）
  - 分隔线
  - 译文窗格（loading 态显示 spinner/骨架；错误态显示 `_( 'Unable to fetch the translation. Try again later.')` + 重试按钮）
- 控件内嵌在窗格里：
  - 源语言下拉：AUTO + 语言表
  - 目标语言下拉：语言表（**切换即重发请求，弹窗不关闭**）
  - 底部 provider 下拉 + 「Translated by Edge.」致谢行
- 行为：挂载即 `translate([text])`；点弹窗外关闭；Esc 关闭；单弹窗仲裁（见 §3）。

## 6. Settings

- 两项进阅读设置：`translationProvider`（默认 `'edge'`）、`translateTargetLang`（默认 `'EN'`）
- 设置面板的语言页提供两个下拉，保存走项目现有的 view-settings 通道
- 弹窗内的语言选择可临时覆盖全局设置，但**不写回**（除非明确做了「设为默认」按钮）

## 7. 已知的坑（全部来自真实实现，必读）

1. **变量命名会骗人**：如果历史上用过 DeepL 后换 Edge，把 `showDeepLPopup` 这类残留命名一起改掉，否则后来者会误判技术栈。
2. **弹窗锚点必须在滚动时重算**，否则滚轮/翻页后弹窗飘在页外（这是划选类弹窗最高频的 bug）。
3. **iframe/分节渲染的阅读器里，`selectionchange` 是每个 document 一个**：挂载/卸载必须跟节生命周期对齐，切节后旧监听必须摘掉，否则幽灵回调读旧 document。
4. **中日共用汉字**：目标语言别写死。中→日与中→英是完全不同的查询，用户必须能现场切换。
5. **批量签名留数组**：划选永远单条，但接口签名保持 `texts: string[]`——将来整段/整页翻译（甚至全书翻译）直接复用同一 service。
6. **错误要可读**：网络失败/HTTP 4xx/5xx/超时/不支持的语言对分别给文案，不要笼统「翻译失败」。
7. **合成点击触发不了手势类逻辑**：自动化测试里 `element.click()` 派发的事件没有 user activation；用真实输入事件（`focus()` + 键盘 Space/Enter）或真机点击验证。
8. **划选即弹是反模式**：用户滚动选择阅读位置时极易误触；一律「选完 → 显式点按钮/快捷键」。

## 8. 验收标准

- [ ] 划选单词/句子/段落 → 点工具栏翻译或 Ctrl/Cmd+T → 弹窗出现且贴住选区
- [ ] 滚动、翻页、调字号后弹窗仍贴着选区（或已正确关闭，无「飘出页外」）
- [ ] 切换目标语言立即重翻、弹窗不关；同一文本第二次弹出秒出（缓存命中）
- [ ] 断网/接口 5xx：可读错误态 + 重试，不崩溃、弹窗可关
- [ ] 中→英、中→日、日→中、英→中 四条路径各验证一次
- [ ] 新增一个 provider 只需注册一个对象，UI/service 代码零改动
- [ ] 设置项持久化，重开应用生效

## 9. 参考实现（可选对照，非必需）

MyReader（Readest fork，公开仓库 `https://github.com/PoxenStudio/myreader`，develop 分支；本地路径 `/vol1/1000/docker/myreader-src`）：

| 环节 | 文件（app/src/ 下） | 链接 |
|---|---|---|
| 选词检测 | `app/src/app/reader/hooks/useTextSelector.ts` | https://github.com/PoxenStudio/myreader/blob/develop/app/src/app/reader/hooks/useTextSelector.ts |
| 编排（选区→弹窗→触发） | `app/src/app/reader/components/annotator/Annotator.tsx` | https://github.com/PoxenStudio/myreader/blob/develop/app/src/app/reader/components/annotator/Annotator.tsx |
| 翻译弹窗 | `app/src/app/reader/components/annotator/TranslatorPopup.tsx` | https://github.com/PoxenStudio/myreader/blob/develop/app/src/app/reader/components/annotator/TranslatorPopup.tsx |
| 翻译 hook | `app/src/hooks/useTranslator.ts` | https://github.com/PoxenStudio/myreader/blob/develop/app/src/hooks/useTranslator.ts |
| provider 注册表 | `app/src/services/translators/providers/index.ts` | https://github.com/PoxenStudio/myreader/blob/develop/app/src/services/translators/providers/index.ts |
| Edge provider | `app/src/services/translators/providers/edge.ts` | https://github.com/PoxenStudio/myreader/blob/develop/app/src/services/translators/providers/edge.ts |
| 缓存 | `app/src/services/translators/cache.ts` | https://github.com/PoxenStudio/myreader/blob/develop/app/src/services/translators/cache.ts |
| 后处理 | `app/src/services/translators/polish.ts` | https://github.com/PoxenStudio/myreader/blob/develop/app/src/services/translators/polish.ts |
| 语言码工具 | `app/src/utils/lang.ts` | https://github.com/PoxenStudio/myreader/blob/develop/app/src/utils/lang.ts |
| 设置面板 | `app/src/components/settings/LangPanel.tsx` | https://github.com/PoxenStudio/myreader/blob/develop/app/src/components/settings/LangPanel.tsx |
| 快捷键 | `app/src/helpers/shortcuts.ts` | https://github.com/PoxenStudio/myreader/blob/develop/app/src/helpers/shortcuts.ts |

> 注意：参考实现里的 `deepl.ts` 与 `showDeepLPopup` 命名是历史遗留死代码，实际唯一 provider 是 Edge——**不要照抄这两个名字**。
