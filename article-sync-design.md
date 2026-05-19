# 文章同步助手 — 完整设计方案

## 一、项目概述

### 1.1 产品定位

参考快稿助手 / Wechatsync，实现一个**文章同步助手浏览器扩展**，将自有系统中的文章一键同步到多个新媒体平台（微信公众号、头条号、知乎、掘金、CSDN、百家号、小红书等）。

### 1.2 核心设计目标

| 目标 | 说明 |
|------|------|
| **脚本远程维护** | 所有平台脚本由自有系统维护和提供，扩展本身不内置任何平台逻辑 |
| **插件零改动扩展** | 新增媒体平台时，只需在自有系统新增脚本，浏览器扩展无需任何代码变更和重新发布 |
| **脚本快速更新** | 平台调整后，只需更新自有系统中的脚本，扩展下次运行自动拉取最新版本 |
| **安全合规** | 遵循 Chrome Manifest V3 安全规范，所有敏感操作在沙箱中执行 |

### 1.3 核心技术挑战

**Chrome MV3 禁止远程代码执行**：Manifest V3 要求所有 JavaScript 逻辑必须打包在扩展内部，不允许 `eval()`、`new Function()` 或动态加载远程 JS。

**解决方案**：使用 MV3 的 `sandbox` 机制。在 `manifest.json` 中声明沙箱页面，沙箱页面不受 CSP `script-src 'self'` 限制，允许 `eval()` 执行远程获取的脚本代码。扩展通过 `postMessage` 与沙箱通信，沙箱内脚本通过扩展提供的桥接能力操作目标平台。

---

## 二、系统整体架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                        自有系统（Server）                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐  │
│  │  文章管理服务  │  │  平台脚本服务  │  │  平台注册中心（Registry） │  │
│  │  /api/articles│  │  /api/scripts │  │  /api/platforms          │  │
│  └──────┬───────┘  └──────┬───────┘  └────────────┬─────────────┘  │
│         │                 │                        │                 │
└─────────┼─────────────────┼────────────────────────┼─────────────────┘
          │ HTTP API        │ HTTP API               │ HTTP API
          │                 │                        │
┌─────────┼─────────────────┼────────────────────────┼─────────────────┐
│         ▼                 ▼                        ▼                 │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │              浏览器扩展（Article Sync Extension）              │    │
│  │                                                              │    │
│  │  ┌────────────┐    ┌─────────────┐    ┌──────────────────┐   │    │
│  │  │  Popup /   │    │  Service    │    │  Sandbox Page    │   │    │
│  │  │  Sidepanel │◄──►│  Worker     │◄──►│  (eval 远程脚本)  │   │    │
│  │  │  (用户界面) │    │  (调度中心)  │    │                  │   │    │
│  │  └────────────┘    └──────┬──────┘    └──────────────────┘   │    │
│  │                           │                                   │    │
│  │                    ┌──────▼──────┐                            │    │
│  │                    │ Content     │                            │    │
│  │                    │ Script      │                            │    │
│  │                    │ (DOM桥接层)  │                            │    │
│  │                    └──────┬──────┘                            │    │
│  └───────────────────────────┼──────────────────────────────────┘    │
│                              │ DOM 操作 / API 调用                   │
│              ┌───────────────┼───────────────┐                       │
│              ▼               ▼               ▼                       │
│        ┌──────────┐   ┌──────────┐   ┌──────────┐                   │
│        │  知乎     │   │  掘金     │   │  头条号   │  ...             │
│        │  编辑器   │   │  编辑器   │   │  编辑器   │                   │
│        └──────────┘   └──────────┘   └──────────┘                   │
└──────────────────────────────────────────────────────────────────────┘
```

### 2.1 架构分层说明

| 层次 | 组件 | 职责 |
|------|------|------|
| **服务层** | 自有系统 Server | 文章管理、平台脚本存储与分发、平台注册中心 |
| **调度层** | Service Worker | 扩展的核心调度中心，协调各组件通信、管理同步任务生命周期 |
| **执行层** | Sandbox Page | 在沙箱环境中执行远程获取的平台脚本，支持 eval |
| **桥接层** | Content Script | 注入目标平台页面，执行 DOM 操作和 API 调用 |
| **展示层** | Popup / Sidepanel | 用户界面，展示文章列表、平台选择、同步状态 |

---

## 三、浏览器扩展详细设计

### 3.1 项目结构

```
article-sync-extension/
├── manifest.json
├── src/
│   ├── background/
│   │   └── service-worker.ts        # Service Worker 调度中心
│   ├── sandbox/
│   │   ├── sandbox.html             # 沙箱页面
│   │   └── sandbox.ts               # 沙箱运行时（eval 执行器）
│   ├── content/
│   │   ├── bridge.ts                # Content Script 桥接层
│   │   └── dom-operator.ts          # DOM 操作工具集
│   ├── popup/
│   │   ├── App.vue                  # Popup 入口
│   │   ├── views/
│   │   │   ├── ArticleList.vue      # 文章列表
│   │   │   ├── PlatformSelect.vue   # 平台选择
│   │   │   └── SyncProgress.vue     # 同步进度
│   │   └── components/
│   ├── sidepanel/
│   │   └── SidePanel.vue            # 侧边栏面板（可选）
│   ├── shared/
│   │   ├── message-types.ts         # 消息类型定义
│   │   ├── constants.ts             # 常量
│   │   └── utils.ts                 # 工具函数
│   └── sdk/
│       ├── base-adapter.ts          # 脚本框架基类定义
│       ├── script-loader.ts         # 脚本加载器
│       └── runtime-context.ts       # 运行时上下文
├── public/
│   ├── icons/
│   └── sandbox.html
├── package.json
├── vite.config.ts
└── tsconfig.json
```

### 3.2 manifest.json

```json
{
  "manifest_version": 3,
  "name": "文章同步助手",
  "version": "1.0.0",
  "description": "将自有系统文章一键同步到多个新媒体平台",

  "permissions": [
    "storage",
    "tabs",
    "activeTab",
    "scripting",
    "cookies"
  ],

  "host_permissions": [
    "<all_urls>"
  ],

  "background": {
    "service_worker": "src/background/service-worker.ts",
    "type": "module"
  },

  "action": {
    "default_popup": "src/popup/index.html",
    "default_icon": {
      "16": "public/icons/icon16.png",
      "48": "public/icons/icon48.png",
      "128": "public/icons/icon128.png"
    }
  },

  "side_panel": {
    "default_path": "src/sidepanel/index.html"
  },

  "sandbox": {
    "pages": ["src/sandbox/sandbox.html"]
  },

  "content_scripts": [
    {
      "matches": ["<all_urls>"],
      "js": ["src/content/bridge.ts"],
      "run_at": "document_idle"
    }
  ]
}
```

### 3.3 核心组件设计

#### 3.3.1 Service Worker（调度中心）

Service Worker 是整个扩展的调度中心，负责：

1. **平台注册表管理**：从自有系统拉取支持的平台列表，缓存到本地
2. **脚本获取与分发**：按需从自有系统获取平台脚本，传递给沙箱执行
3. **同步任务调度**：管理同步任务的生命周期（创建 → 执行 → 监控 → 完成）
4. **组件间通信**：协调 Popup、Sandbox、Content Script 之间的消息传递

```typescript
// service-worker.ts 核心逻辑伪代码

const SERVER_BASE = 'https://your-server.com/api'

// 平台注册表缓存
let platformRegistry: PlatformInfo[] = []

// 同步任务管理
const syncTasks = new Map<string, SyncTask>()

// 初始化：拉取平台列表
async function initPlatformRegistry() {
  const resp = await fetch(`${SERVER_BASE}/platforms`)
  platformRegistry = await resp.json()
  chrome.storage.local.set({ platformRegistry })
}

// 创建同步任务
async function createSyncTask(articleId: string, platformIds: string[]) {
  const task: SyncTask = {
    id: crypto.randomUUID(),
    articleId,
    platformIds,
    status: 'pending',
    results: {}
  }
  syncTasks.set(task.id, task)

  for (const platformId of platformIds) {
    await executePlatformSync(task, platformId)
  }
}

// 执行单个平台同步
async function executePlatformSync(task: SyncTask, platformId: string) {
  // 1. 获取文章内容
  const article = await fetchArticle(task.articleId)
  // 2. 获取平台脚本
  const script = await fetchPlatformScript(platformId)
  // 3. 发送到沙箱执行
  const result = await executeInSandbox(script, {
    article,
    platformId,
    action: 'sync'
  })
  // 4. 处理结果
  task.results[platformId] = result
}

// 与沙箱通信
async function executeInSandbox(scriptCode: string, context: SyncContext) {
  return new Promise((resolve) => {
    const sandboxFrame = getSandboxFrame()
    const requestId = crypto.randomUUID()

    const handler = (event: MessageEvent) => {
      if (event.data.requestId === requestId) {
        window.removeEventListener('message', handler)
        resolve(event.data.result)
      }
    }
    window.addEventListener('message', handler)

    sandboxFrame.contentWindow.postMessage({
      type: 'EXECUTE_SCRIPT',
      requestId,
      scriptCode,
      context
    }, '*')
  })
}
```

#### 3.3.2 Sandbox Page（沙箱执行器）

沙箱页面是 MV3 下执行远程脚本的关键。它在独立的环境中运行，不受扩展 CSP 限制，允许 `eval()`。

```html
<!-- sandbox.html -->
<!DOCTYPE html>
<html>
<head><meta charset="utf-8"></head>
<body>
<script src="sandbox.js"></script>
</body>
</html>
```

```typescript
// sandbox.ts

// 注入运行时 SDK 到全局
self.ScriptSDK = {
  $: jQuery,
  axios,
  CryptoJS,
  turndownService,
  domOperator: null, // 由 content script 桥接提供
  cache: {
    set: (key: string, value: any) => { /* ... */ },
    get: (key: string) => { /* ... */ },
  },
  modifyRequestHeaders: (config: any) => { /* ... */ },
  bridge: {
    call: (method: string, params: any) => { /* 通过 postMessage 调用 content script */ },
  }
}

window.addEventListener('message', async (event) => {
  const { type, requestId, scriptCode, context } = event.data

  if (type !== 'EXECUTE_SCRIPT') return

  try {
    // 在沙箱中 eval 执行远程脚本
    const scriptFactory = eval(scriptCode)
    // 实例化适配器
    const adapter = typeof scriptFactory === 'function'
      ? scriptFactory(ScriptSDK)
      : new scriptFactory.default(ScriptSDK)

    // 按生命周期执行
    const result = await runAdapterLifecycle(adapter, context)

    event.source.postMessage({
      type: 'EXECUTE_RESULT',
      requestId,
      result: { success: true, data: result }
    }, { targetOrigin: '*' })
  } catch (error) {
    event.source.postMessage({
      type: 'EXECUTE_RESULT',
      requestId,
      result: { success: false, error: error.message }
    }, { targetOrigin: '*' })
  }
})

async function runAdapterLifecycle(adapter: any, context: SyncContext) {
  const results: LifecycleResult = {}

  // Step 1: 获取平台元数据
  results.metaData = await adapter.getMetaData()

  // Step 2: 内容预处理
  const processedPost = await adapter.preEditPost(context.article)

  // Step 3: 创建文章
  const addResult = await adapter.addPost(processedPost)
  results.addPost = addResult

  // Step 4: 上传图片（如有）
  if (processedPost.images && processedPost.images.length > 0) {
    for (const image of processedPost.images) {
      const uploadResult = await adapter.uploadFile(image)
      processedPost.content = processedPost.content.replace(
        image.originalUrl,
        uploadResult.url
      )
    }
  }

  // Step 5: 更新文章（替换图片链接后）
  if (adapter.editPost) {
    results.editPost = await adapter.editPost(addResult.post_id, processedPost)
  }

  return results
}
```

#### 3.3.3 Content Script（DOM 桥接层）

Content Script 注入到目标平台页面中，为沙箱中的脚本提供 DOM 操作能力。沙箱本身无法直接操作目标页面的 DOM，需要通过 Content Script 中转。

```typescript
// bridge.ts

// 监听来自 Service Worker 的 DOM 操作请求
chrome.runtime.onMessage.addListener((message, sender, sendResponse) => {
  if (message.type === 'DOM_OPERATION') {
    handleDomOperation(message.operation).then(sendResponse)
    return true
  }
})

async function handleDomOperation(operation: DomOperation) {
  switch (operation.action) {
    case 'navigate':
      window.location.href = operation.url
      break
    case 'click':
      document.querySelector(operation.selector)?.click()
      break
    case 'fill':
      const el = document.querySelector(operation.selector) as HTMLInputElement
      if (el) {
        el.focus()
        el.value = operation.value
        el.dispatchEvent(new Event('input', { bubbles: true }))
        el.dispatchEvent(new Event('change', { bubbles: true }))
      }
      break
    case 'fillRichText':
      const editor = document.querySelector(operation.selector)
      if (editor) {
        editor.innerHTML = operation.html
        editor.dispatchEvent(new Event('input', { bubbles: true }))
      }
      break
    case 'uploadFile':
      await triggerFileUpload(operation.selector, operation.fileData)
      break
    case 'waitForElement':
      await waitForElement(operation.selector, operation.timeout)
      break
    case 'getElementText':
      return document.querySelector(operation.selector)?.textContent
    case 'screenshot':
      // 返回页面关键信息
      break
  }
}

// 等待元素出现
function waitForElement(selector: string, timeout = 10000): Promise<Element> {
  return new Promise((resolve, reject) => {
    const existing = document.querySelector(selector)
    if (existing) return resolve(existing)

    const observer = new MutationObserver(() => {
      const el = document.querySelector(selector)
      if (el) {
        observer.disconnect()
        resolve(el)
      }
    })
    observer.observe(document.body, { childList: true, subtree: true })
    setTimeout(() => {
      observer.disconnect()
      reject(new Error(`Element ${selector} not found within ${timeout}ms`))
    }, timeout)
  })
}

// 触发文件上传
async function triggerFileUpload(selector: string, fileData: ArrayBuffer) {
  const input = document.querySelector(selector) as HTMLInputElement
  if (!input) throw new Error('File input not found')

  const file = new File([fileData], 'image.png', { type: 'image/png' })
  const dataTransfer = new DataTransfer()
  dataTransfer.items.add(file)
  input.files = dataTransfer.files
  input.dispatchEvent(new Event('change', { bubbles: true }))
}
```

---

## 四、平台脚本框架（Script SDK）设计

### 4.1 核心设计理念

脚本框架是整个系统最关键的设计，它定义了平台脚本的**编写规范**和**运行时能力**。好的框架设计应该做到：

1. **接口统一**：所有平台脚本遵循相同的接口规范
2. **能力丰富**：提供充足的运行时工具（HTTP 请求、DOM 操作、文件上传等）
3. **隔离安全**：脚本在沙箱中运行，无法直接访问扩展内部 API
4. **两种模式**：支持 API 调用模式和 DOM 自动化模式

### 4.2 BaseAdapter 接口定义

```typescript
// base-adapter.ts

interface ArticlePost {
  title: string
  content: string              // HTML 格式正文
  markdown?: string            // Markdown 格式正文（可选）
  summary?: string             // 摘要
  tags?: string[]              // 标签
  category?: string            // 分类
  coverImage?: string          // 封面图 URL
  images?: ArticleImage[]      // 文章中的图片列表
  originalUrl?: string         // 原文链接
  author?: string              // 作者
  publishType?: 'draft' | 'publish'  // 发布类型
  extra?: Record<string, any>  // 平台特有字段
}

interface ArticleImage {
  originalUrl: string          // 原始图片 URL
  alt?: string                 // 图片描述
  width?: number
  height?: number
}

interface PlatformMetaData {
  uid: string                  // 用户 ID
  title: string                // 用户昵称
  avatar: string               // 头像 URL
  platformId: string           // 平台标识（如 zhihu, juejin）
  displayName: string          // 平台显示名称
  home: string                 // 平台主页 URL
  icon: string                 // 平台图标 URL
  supportTypes: ('html' | 'markdown' | 'richtext')[]  // 支持的内容格式
  logged_in: boolean           // 是否已登录
}

interface AddPostResult {
  status: 'success' | 'failed'
  post_id: string              // 平台文章 ID
  draftLink?: string           // 草稿链接
  publishedLink?: string       // 已发布链接
  error?: string               // 错误信息
}

interface UploadFileResult {
  status: 'success' | 'failed'
  url: string                  // 上传后的图片 URL
  error?: string
}

interface EditPostResult {
  status: 'success' | 'failed'
  draftLink?: string
  publishedLink?: string
  error?: string
}

abstract class BaseAdapter {
  protected sdk: ScriptSDKContext

  constructor(sdk: ScriptSDKContext) {
    this.sdk = sdk
  }

  // ===== 必须实现的生命周期方法 =====

  // 获取平台登录状态和用户信息
  abstract getMetaData(): Promise<PlatformMetaData>

  // 创建文章（发布到草稿箱）
  abstract addPost(post: ArticlePost): Promise<AddPostResult>

  // 上传图片到平台
  abstract uploadFile(file: { data: ArrayBuffer; name: string; type: string }): Promise<UploadFileResult>

  // ===== 可选覆写的生命周期方法 =====

  // 内容预处理（平台格式适配）
  async preEditPost(post: ArticlePost): Promise<ArticlePost> {
    return post
  }

  // 更新文章（替换图片链接后）
  async editPost(postId: string, post: ArticlePost): Promise<EditPostResult> {
    return { status: 'success' }
  }

  // 获取草稿列表（可选）
  async getPosts(): Promise<any[]> {
    return []
  }

  // 删除草稿（可选）
  async deletePost(postId: string): Promise<void> {}

  // ===== 平台配置 =====

  // 平台标识
  static platformId: string

  // 平台显示名称
  static displayName: string

  // 平台编辑器 URL（用于 DOM 自动化模式）
  static editorUrl: string

  // 运行模式：api 或 dom
  static mode: 'api' | 'dom' = 'api'
}
```

### 4.3 ScriptSDKContext（运行时上下文）

脚本运行时可访问的 SDK 能力：

```typescript
interface ScriptSDKContext {
  // HTTP 请求
  axios: AxiosInstance

  // DOM 操作桥接（通过 content script 执行）
  dom: {
    navigate(url: string): Promise<void>
    click(selector: string): Promise<void>
    fill(selector: string, value: string): Promise<void>
    fillRichText(selector: string, html: string): Promise<void>
    uploadFile(selector: string, fileData: ArrayBuffer, fileName: string): Promise<void>
    waitForElement(selector: string, timeout?: number): Promise<void>
    getElementText(selector: string): Promise<string>
    getElementAttribute(selector: string, attr: string): Promise<string>
    waitFor(ms: number): Promise<void>
    scrollIntoView(selector: string): Promise<void>
    screenshot(): Promise<string>
  }

  // 缓存
  cache: {
    set(key: string, value: any, ttl?: number): void
    get(key: string): any
    delete(key: string): void
  }

  // HTML ↔ Markdown 转换
  turndown: {
    htmlToMarkdown(html: string): string
  }

  // 加密工具
  crypto: {
    md5(data: string): string
    sha256(data: string): string
    hmac(key: string, data: string): string
  }

  // 请求头修改（处理 CORS）
  modifyRequestHeaders(config: {
    url: string
    headers: Record<string, string>
  }): void

  // Cookie 操作
  cookie: {
    get(url: string, name: string): Promise<string>
    getAll(url: string): Promise<Record<string, string>>
  }

  // 日志
  logger: {
    info(msg: string, data?: any): void
    warn(msg: string, data?: any): void
    error(msg: string, data?: any): void
  }
}
```

### 4.4 脚本两种运行模式

#### 模式一：API 模式

通过调用平台的后端 API 完成文章发布。需要分析平台的 API 接口，直接发起 HTTP 请求。

**适用场景**：平台有可用的后端 API（如头条号、百家号、WordPress 等）

```typescript
// 示例：头条号适配器（API 模式）
exports.adapter = function(sdk) {
  return {
    platformId: 'toutiao',
    displayName: '头条号',
    mode: 'api',

    async getMetaData() {
      const res = await sdk.axios.get('https://mp.toutiao.com/mp/agw/media/get_media_info')
      return {
        uid: res.data.user.id,
        title: res.data.user.screen_name,
        avatar: res.data.user.https_avatar_url,
        platformId: 'toutiao',
        displayName: '头条号',
        home: 'https://mp.toutiao.com/profile_v3/graphic/publish',
        icon: 'https://sf1-ttcdn-tos.pstatp.com/obj/ttfe/pgcfe/sz/mp_logo.png',
        supportTypes: ['html'],
        logged_in: true
      }
    },

    async preEditPost(post) {
      // 头条号不支持 Markdown，确保使用 HTML
      return { ...post, content: post.content }
    },

    async addPost(post) {
      const res = await sdk.axios.post(
        'https://mp.toutiao.com/mp/agw/article/publish',
        {
          title: post.title,
          content: post.content,
          save: 0
        }
      )
      return {
        status: 'success',
        post_id: res.data.pgc_id,
        draftLink: `https://mp.toutiao.com/profile_v3/graphic/publish?pgc_id=${res.data.pgc_id}`
      }
    },

    async uploadFile(file) {
      const formData = new FormData()
      formData.append('image', new Blob([file.data]), file.name)
      const res = await sdk.axios.post(
        'https://mp.toutiao.com/mp/agw/article/upload_image',
        formData
      )
      return { status: 'success', url: res.data.url }
    }
  }
}
```

#### 模式二：DOM 自动化模式

通过模拟用户操作（点击、填写、上传）完成文章发布。适用于没有公开 API 或 API 难以逆向的平台。

**适用场景**：平台无可用 API，需要模拟用户在编辑器中的操作（如部分平台的新版编辑器）

```typescript
// 示例：某平台适配器（DOM 模式）
exports.adapter = function(sdk) {
  return {
    platformId: 'example-platform',
    displayName: '示例平台',
    mode: 'dom',
    editorUrl: 'https://example.com/editor/new',

    async getMetaData() {
      await sdk.dom.navigate('https://example.com/profile')
      await sdk.dom.waitForElement('.user-name', 5000)
      const userName = await sdk.dom.getElementText('.user-name')
      const avatar = await sdk.dom.getElementAttribute('.user-avatar', 'src')
      return {
        uid: userName,
        title: userName,
        avatar,
        platformId: 'example-platform',
        displayName: '示例平台',
        home: 'https://example.com/editor/new',
        icon: 'https://example.com/favicon.ico',
        supportTypes: ['html'],
        logged_in: !!userName
      }
    },

    async addPost(post) {
      // 1. 打开编辑器
      await sdk.dom.navigate(this.editorUrl)
      await sdk.dom.waitForElement('#title-input', 10000)

      // 2. 填写标题
      await sdk.dom.fill('#title-input', post.title)

      // 3. 填写正文
      await sdk.dom.fillRichText('.editor-content', post.content)

      // 4. 填写标签
      if (post.tags && post.tags.length > 0) {
        await sdk.dom.click('#tag-input')
        for (const tag of post.tags) {
          await sdk.dom.fill('#tag-search', tag)
          await sdk.dom.waitForElement('.tag-suggestion', 3000)
          await sdk.dom.click('.tag-suggestion:first-child')
        }
      }

      // 5. 保存草稿
      await sdk.dom.click('#save-draft-btn')
      await sdk.dom.waitFor(2000)

      return {
        status: 'success',
        post_id: 'dom-mode-no-id',
        draftLink: 'https://example.com/editor'
      }
    },

    async uploadFile(file) {
      await sdk.dom.click('#upload-image-btn')
      await sdk.dom.waitForElement('input[type="file"]', 3000)
      await sdk.dom.uploadFile('input[type="file"]', file.data, file.name)
      await sdk.dom.waitForElement('.uploaded-image', 10000)
      const url = await sdk.dom.getElementAttribute('.uploaded-image', 'src')
      return { status: 'success', url }
    }
  }
}
```

### 4.5 脚本导出规范

平台脚本必须遵循统一的导出格式：

```javascript
// 方式一：工厂函数（推荐）
exports.adapter = function(sdk) {
  return {
    platformId: 'xxx',
    displayName: 'XXX',
    mode: 'api',  // 或 'dom'
    // ... 实现 BaseAdapter 定义的所有方法
  }
}

// 方式二：类导出
exports.Adapter = class SomePlatformAdapter {
  constructor(sdk) {
    this.sdk = sdk
  }
  async getMetaData() { /* ... */ }
  async addPost(post) { /* ... */ }
  async uploadFile(file) { /* ... */ }
}
exports.adapter = function(sdk) {
  return new exports.Adapter(sdk)
}
```

### 4.6 脚本元数据声明

每个脚本必须包含元数据头，用于脚本管理和版本控制：

```javascript
/**
 * @platformId zhihu
 * @displayName 知乎
 * @version 1.2.0
 * @mode api
 * @editorUrl https://zhuanlan.zhihu.com/write
 * @icon https://static.zhihu.com/heifetz/favicon.ico
 * @author your-team
 * @description 知乎专栏文章同步适配器
 * @supportTypes html,markdown
 * @requiredCookies zhihu.com
 */
exports.adapter = function(sdk) {
  // ...
}
```

---

## 五、自有系统 API 设计

### 5.1 平台注册中心 API

#### 获取支持的平台列表

```
GET /api/platforms
```

**Response:**
```json
{
  "code": 0,
  "data": [
    {
      "platformId": "zhihu",
      "displayName": "知乎",
      "icon": "https://static.zhihu.com/heifetz/favicon.ico",
      "editorUrl": "https://zhuanlan.zhihu.com/write",
      "mode": "api",
      "supportTypes": ["html", "markdown"],
      "scriptVersion": "1.2.0",
      "scriptUrl": "/api/scripts/zhihu",
      "enabled": true,
      "requiredCookies": ["zhihu.com"],
      "sortOrder": 1
    },
    {
      "platformId": "juejin",
      "displayName": "掘金",
      "icon": "https://lf-web-assets.juejin.cn/obj/juejin-web/xitu_juejin_web/favicon.ico",
      "editorUrl": "https://juejin.cn/editor/drafts/new",
      "mode": "api",
      "supportTypes": ["markdown"],
      "scriptVersion": "1.0.3",
      "scriptUrl": "/api/scripts/juejin",
      "enabled": true,
      "requiredCookies": ["juejin.cn"],
      "sortOrder": 2
    }
  ]
}
```

#### 获取单个平台信息

```
GET /api/platforms/:platformId
```

### 5.2 平台脚本 API

#### 获取平台脚本代码

```
GET /api/scripts/:platformId
```

**Query Parameters:**
| 参数 | 类型 | 说明 |
|------|------|------|
| version | string | 指定版本号，不传则返回最新版本 |

**Response:**
```json
{
  "code": 0,
  "data": {
    "platformId": "zhihu",
    "version": "1.2.0",
    "code": "exports.adapter = function(sdk) { ... }",
    "hash": "sha256:abc123...",
    "updatedAt": "2026-05-15T10:30:00Z",
    "changelog": "修复知乎编辑器更新后的标题填写问题"
  }
}
```

#### 批量获取脚本（可选优化）

```
POST /api/scripts/batch
```

**Request Body:**
```json
{
  "platformIds": ["zhihu", "juejin", "csdn"],
  "currentVersions": {
    "zhihu": "1.1.0",
    "juejin": "1.0.3"
  }
}
```

**Response:** 只返回有更新的脚本代码，未变化的返回空（节省带宽）

```json
{
  "code": 0,
  "data": {
    "zhihu": {
      "version": "1.2.0",
      "code": "exports.adapter = function(sdk) { ... }",
      "hash": "sha256:abc123...",
      "updated": true
    },
    "juejin": {
      "version": "1.0.3",
      "updated": false
    },
    "csdn": {
      "version": "1.1.1",
      "code": "exports.adapter = function(sdk) { ... }",
      "hash": "sha256:def456...",
      "updated": true
    }
  }
}
```

### 5.3 文章管理 API

#### 获取文章列表

```
GET /api/articles
```

**Query Parameters:**
| 参数 | 类型 | 说明 |
|------|------|------|
| page | number | 页码 |
| pageSize | number | 每页数量 |
| status | string | 筛选状态：published/draft |
| keyword | string | 搜索关键词 |

**Response:**
```json
{
  "code": 0,
  "data": {
    "total": 100,
    "list": [
      {
        "id": "art_001",
        "title": "深入理解 Chrome 扩展架构",
        "summary": "本文详细介绍了...",
        "coverImage": "https://cdn.example.com/cover.jpg",
        "status": "published",
        "createdAt": "2026-05-10T08:00:00Z",
        "syncStatus": {
          "zhihu": { status: "synced", syncedAt: "2026-05-10T09:00:00Z" },
          "juejin": { status: "pending" }
        }
      }
    ]
  }
}
```

#### 获取文章详情（含正文）

```
GET /api/articles/:articleId
```

**Response:**
```json
{
  "code": 0,
  "data": {
    "id": "art_001",
    "title": "深入理解 Chrome 扩展架构",
    "content": "<h1>深入理解...</h1>...",
    "markdown": "# 深入理解...\n...",
    "summary": "本文详细介绍了...",
    "tags": ["Chrome", "浏览器扩展"],
    "category": "技术",
    "coverImage": "https://cdn.example.com/cover.jpg",
    "images": [
      { "originalUrl": "https://cdn.example.com/img1.jpg", "alt": "架构图" }
    ],
    "author": "张三",
    "originalUrl": "https://blog.example.com/chrome-extension",
    "status": "published"
  }
}
```

### 5.4 同步结果回调 API

扩展在同步完成后，将结果回传给自有系统：

```
POST /api/sync-results
```

**Request Body:**
```json
{
  "articleId": "art_001",
  "results": [
    {
      "platformId": "zhihu",
      "status": "success",
      "postId": "zh_12345",
      "draftLink": "https://zhuanlan.zhihu.com/p/12345/edit",
      "syncedAt": "2026-05-19T10:30:00Z"
    },
    {
      "platformId": "juejin",
      "status": "failed",
      "error": "登录已过期，请重新登录",
      "syncedAt": "2026-05-19T10:30:05Z"
    }
  ]
}
```

### 5.5 脚本管理 API（管理后台使用）

#### 创建/更新平台脚本

```
PUT /api/scripts/:platformId
```

**Request Body:**
```json
{
  "code": "exports.adapter = function(sdk) { ... }",
  "version": "1.3.0",
  "changelog": "适配知乎新版编辑器",
  "metadata": {
    "platformId": "zhihu",
    "displayName": "知乎",
    "mode": "api",
    "editorUrl": "https://zhuanlan.zhihu.com/write",
    "icon": "https://static.zhihu.com/heifetz/favicon.ico",
    "supportTypes": ["html", "markdown"],
    "requiredCookies": ["zhihu.com"]
  }
}
```

#### 启用/禁用平台

```
PATCH /api/platforms/:platformId/status
```

**Request Body:**
```json
{
  "enabled": true
}
```

---

## 六、同步流程详细设计

### 6.1 完整同步流程

```
用户                    Popup              Service Worker         Sandbox           Content Script        自有系统
 │                       │                      │                    │                    │                    │
 │  点击"同步文章"        │                      │                    │                    │                    │
 │──────────────────────►│                      │                    │                    │                    │
 │                       │  fetchPlatforms()    │                    │                    │                    │
 │                       │─────────────────────►│                    │                    │                    │
 │                       │                      │  GET /api/platforms │                    │                    │
 │                       │                      │───────────────────────────────────────────────────────────────►│
 │                       │                      │◄───────────────────────────────────────────────────────────────│
 │                       │  platformList        │                    │                    │                    │
 │                       │◄─────────────────────│                    │                    │                    │
 │  选择平台+确认         │                      │                    │                    │                    │
 │──────────────────────►│                      │                    │                    │                    │
 │                       │  createSyncTask()    │                    │                    │                    │
 │                       │─────────────────────►│                    │                    │                    │
 │                       │                      │  GET /api/articles/:id                  │                    │
 │                       │                      │───────────────────────────────────────────────────────────────►│
 │                       │                      │◄───────────────────────────────────────────────────────────────│
 │                       │                      │  GET /api/scripts/zhihu                 │                    │
 │                       │                      │───────────────────────────────────────────────────────────────►│
 │                       │                      │◄───────────────────────────────────────────────────────────────│
 │                       │                      │                    │                    │                    │
 │                       │                      │  EXECUTE_SCRIPT    │                    │                    │
 │                       │                      │───────────────────►│                    │                    │
 │                       │                      │                    │  eval(scriptCode)  │                    │
 │                       │                      │                    │  adapter.getMetaData()               │                    │
 │                       │                      │                    │  DOM_OP(click)     │                    │
 │                       │                      │                    │───────────────────►│                    │
 │                       │                      │                    │◄───────────────────│                    │
 │                       │                      │                    │  adapter.addPost() │                    │
 │                       │                      │                    │  HTTP request ─────────────────────────────────────────────────►│
 │                       │                      │                    │◄──────────────────────────────────────────────────────────────│
 │                       │                      │                    │  adapter.uploadFile()                │                    │
 │                       │                      │                    │  ...               │                    │
 │                       │                      │  EXECUTE_RESULT    │                    │                    │
 │                       │                      │◄───────────────────│                    │                    │
 │                       │                      │                    │                    │                    │
 │                       │                      │  POST /api/sync-results                │                    │
 │                       │                      │───────────────────────────────────────────────────────────────►│
 │                       │  syncComplete        │                    │                    │                    │
 │                       │◄─────────────────────│                    │                    │                    │
 │  显示同步结果          │                      │                    │                    │                    │
 │◄──────────────────────│                      │                    │                    │                    │
```

### 6.2 脚本缓存策略

为减少对自有系统的请求压力，扩展采用多级缓存策略：

```
┌─────────────────────────────────────────────┐
│ Level 1: 内存缓存（Service Worker 运行时）    │
│ TTL: 当前 Service Worker 生命周期             │
├─────────────────────────────────────────────┤
│ Level 2: chrome.storage.local（持久化）       │
│ TTL: 24 小时                                 │
│ 存储: 脚本代码 + 版本号 + hash               │
├─────────────────────────────────────────────┤
│ Level 3: 远程获取（自有系统 API）             │
│ 条件: 缓存过期 / 版本不一致 / 强制刷新        │
└─────────────────────────────────────────────┘
```

缓存更新流程：
1. 启动时检查本地缓存的脚本版本
2. 调用批量接口 `POST /api/scripts/batch`，传入当前版本号
3. 只下载有更新的脚本，未变化的跳过
4. 更新本地缓存

### 6.3 错误处理与重试

```typescript
interface SyncError {
  platformId: string
  phase: 'getMetaData' | 'preEditPost' | 'addPost' | 'uploadFile' | 'editPost'
  errorType: 'network' | 'auth' | 'dom' | 'script' | 'unknown'
  message: string
  retryable: boolean
}

// 重试策略
const RETRY_CONFIG = {
  maxRetries: 3,
  baseDelay: 1000,      // 1s
  maxDelay: 30000,       // 30s
  retryableErrors: ['network', 'dom'],
  nonRetryableErrors: ['auth']  // 认证失败不重试，提示用户重新登录
}
```

---

## 七、安全设计

### 7.1 脚本执行安全

| 安全措施 | 说明 |
|----------|------|
| **沙箱隔离** | 远程脚本在 sandbox iframe 中执行，无法访问扩展内部 API |
| **脚本签名验证** | 自有系统对脚本代码进行签名，扩展验证签名后再执行 |
| **HTTPS 传输** | 所有脚本获取走 HTTPS，防止中间人篡改 |
| **hash 校验** | 脚本下载后校验 hash，确保完整性 |
| **权限最小化** | 脚本只能通过 SDK 提供的能力操作，无法直接调用 chrome.* API |

### 7.2 脚本签名方案

```
自有系统发布脚本时:
1. 计算脚本代码的 SHA-256 hash
2. 使用私钥对 hash 进行 RSA-SHA256 签名
3. 将签名和公钥证书随脚本一起下发

扩展验证时:
1. 下载脚本代码 + 签名 + 证书
2. 使用内置公钥验证证书有效性
3. 使用证书中的公钥验证签名
4. 验证通过后才执行脚本
```

### 7.3 Cookie 安全

- 扩展复用浏览器已登录的 Cookie，不存储账号密码
- Cookie 仅用于平台 API 请求，不回传到自有系统
- 用户可在扩展设置中随时清除缓存的 Cookie

---

## 八、扩展 UI 设计

### 8.1 Popup 页面

```
┌──────────────────────────────────────┐
│  📝 文章同步助手                      │
│                                      │
│  ┌────────────────────────────────┐  │
│  │ 🔍 搜索文章...                  │  │
│  └────────────────────────────────┘  │
│                                      │
│  📄 深入理解 Chrome 扩展架构          │
│     2026-05-10 | 已同步 2/5 平台     │
│                                      │
│  📄 Vue3 组合式 API 最佳实践         │
│     2026-05-08 | 未同步              │
│                                      │
│  📄 微服务架构设计模式总结            │
│     2026-05-05 | 已同步 5/5 平台     │
│                                      │
│  ─────────────────────────────────── │
│                                      │
│  选择同步平台:                        │
│                                      │
│  ☑ 知乎      ☑ 掘金      ☐ CSDN     │
│  ☑ 头条号    ☐ 百家号    ☐ 公众号    │
│  ☐ 小红书    ☐ B站       ☐ 简书      │
│                                      │
│  ─────────────────────────────────── │
│                                      │
│  [  开始同步  ]                      │
│                                      │
│  ⚙️ 设置                            │
└──────────────────────────────────────┘
```

### 8.2 同步进度页面

```
┌──────────────────────────────────────┐
│  🔄 正在同步: 深入理解 Chrome 扩展架构 │
│                                      │
│  ✅ 知乎     已保存到草稿箱  12:30:05 │
│  ✅ 掘金     已保存到草稿箱  12:30:12 │
│  🔄 头条号   正在上传图片...  12:30:18│
│  ⏳ CSDN     等待中                  │
│  ❌ 百家号   登录已过期              │
│                                      │
│  ─────────────────────────────────── │
│  进度: 2/5 完成                       │
│  ████████████░░░░░░░░░░░  40%       │
│                                      │
│  [  取消  ]         [  查看详情  ]   │
└──────────────────────────────────────┘
```

### 8.3 设置页面

```
┌──────────────────────────────────────┐
│  ⚙️ 设置                            │
│                                      │
│  服务器地址                           │
│  ┌────────────────────────────────┐  │
│  │ https://your-server.com       │  │
│  └────────────────────────────────┘  │
│                                      │
│  脚本更新频率                         │
│  ○ 每次同步前检查更新                 │
│  ○ 每天检查一次                      │
│  ○ 手动检查更新                      │
│                                      │
│  默认发布方式                         │
│  ○ 保存为草稿                        │
│  ○ 直接发布                          │
│                                      │
│  同步失败时                           │
│  ☑ 自动重试（最多3次）               │
│  ☑ 显示桌面通知                      │
│                                      │
│  [  检查脚本更新  ]                  │
│  [  清除缓存  ]                      │
│  [  保存设置  ]                      │
└──────────────────────────────────────┘
```

---

## 九、平台脚本开发工作流

### 9.1 新增平台脚本流程

```
1. 分析目标平台
   ├── 打开平台编辑器页面
   ├── F12 分析网络请求（API 模式）
   └── 或分析 DOM 结构（DOM 模式）

2. 编写脚本
   ├── 复制脚本模板
   ├── 实现 getMetaData / addPost / uploadFile 等方法
   └── 添加元数据头

3. 本地测试
   ├── 在自有系统管理后台上传脚本
   ├── 在扩展中测试同步功能
   └── 检查日志输出

4. 发布上线
   ├── 在管理后台设置版本号和更新日志
   ├── 启用平台
   └── 扩展自动拉取新脚本
```

### 9.2 脚本模板

```javascript
/**
 * @platformId {{PLATFORM_ID}}
 * @displayName {{PLATFORM_NAME}}
 * @version 1.0.0
 * @mode api
 * @editorUrl {{EDITOR_URL}}
 * @icon {{ICON_URL}}
 * @supportTypes html
 * @requiredCookies {{DOMAIN}}
 */

exports.adapter = function(sdk) {

  const API_BASE = '{{API_BASE_URL}}'

  return {
    platformId: '{{PLATFORM_ID}}',
    displayName: '{{PLATFORM_NAME}}',
    mode: 'api',

    async getMetaData() {
      try {
        const res = await sdk.axios.get(`${API_BASE}/user/info`)
        return {
          uid: String(res.data.id),
          title: res.data.nickname,
          avatar: res.data.avatar,
          platformId: '{{PLATFORM_ID}}',
          displayName: '{{PLATFORM_NAME}}',
          home: '{{EDITOR_URL}}',
          icon: '{{ICON_URL}}',
          supportTypes: ['html'],
          logged_in: true
        }
      } catch (e) {
        return {
          uid: '',
          title: '',
          avatar: '',
          platformId: '{{PLATFORM_ID}}',
          displayName: '{{PLATFORM_NAME}}',
          home: '{{EDITOR_URL}}',
          icon: '{{ICON_URL}}',
          supportTypes: ['html'],
          logged_in: false
        }
      }
    },

    async preEditPost(post) {
      return post
    },

    async addPost(post) {
      // TODO: 实现文章创建逻辑
      throw new Error('addPost not implemented')
    },

    async uploadFile(file) {
      // TODO: 实现图片上传逻辑
      throw new Error('uploadFile not implemented')
    },

    async editPost(postId, post) {
      return { status: 'success' }
    }
  }
}
```

---

## 十、自有系统管理后台设计

### 10.1 功能模块

```
┌─────────────────────────────────────────────────────┐
│                  自有系统管理后台                      │
├──────────┬──────────┬──────────┬────────────────────┤
│  文章管理  │  平台管理  │  脚本管理  │  同步记录        │
├──────────┼──────────┼──────────┼────────────────────┤
│ 文章列表  │ 平台列表  │ 脚本编辑器│ 同步历史          │
│ 文章编辑  │ 启用/禁用 │ 版本管理  │ 失败重试          │
│ 文章导入  │ 新增平台  │ 在线调试  │ 数据统计          │
│ 批量操作  │ 排序配置  │ 灰度发布  │                   │
└──────────┴──────────┴──────────┴────────────────────┘
```

### 10.2 脚本在线编辑器

管理后台提供一个在线脚本编辑器，支持：

- **语法高亮**：基于 Monaco Editor
- **代码提示**：ScriptSDK 类型提示
- **在线调试**：一键部署到测试用户的扩展中
- **版本对比**：Diff 查看不同版本间的变更
- **灰度发布**：先对部分用户生效，确认无问题后全量发布

### 10.3 脚本版本管理

```
脚本版本状态流转:

  draft ──► testing ──► canary(灰度) ──► stable(正式)
    │          │             │                │
    │          │             │                │
    ▼          ▼             ▼                ▼
  编辑中    内部测试     10%用户生效      全量生效
```

---

## 十一、技术选型

| 组件 | 技术选型 | 说明 |
|------|---------|------|
| 扩展框架 | Vue 3 + TypeScript | 轻量、生态丰富 |
| 构建工具 | Vite + CRXJS | 快速开发，热更新支持 |
| UI 框架 | Naive UI | 轻量、支持 Tree-shaking |
| 状态管理 | Pinia | Vue 3 官方推荐 |
| HTTP 客户端 | Axios | 拦截器、请求/响应转换 |
| 数据存储 | chrome.storage.local + IndexedDB | 轻量数据用 storage，大量数据用 IndexedDB |
| Markdown | marked + highlight.js | 文章格式转换 |
| 服务端 | Node.js / Go / Java | 根据团队技术栈选择 |
| 数据库 | PostgreSQL / MySQL | 脚本存储、版本管理 |
| 缓存 | Redis | 脚本缓存、频率限制 |

---

## 十二、扩展发布与更新策略

### 12.1 扩展本身更新

扩展本身代码变更极少（仅框架层调整），更新频率低。通过 Chrome Web Store 正常发布更新。

### 12.2 平台脚本更新

脚本更新完全由自有系统控制，无需 Chrome Web Store 审核：

1. 管理员在后台更新脚本代码
2. 扩展在同步前检查脚本版本
3. 发现新版本后自动下载并缓存
4. 下次同步使用新版本脚本

### 12.3 新增平台

1. 管理员在后台编写新平台脚本
2. 在平台注册中心新增平台配置
3. 启用平台
4. 扩展下次拉取平台列表时自动出现新平台

**全程无需修改扩展代码。**

---

## 十三、后续扩展能力

| 能力 | 说明 |
|------|------|
| **定时同步** | 支持设置定时任务，自动在指定时间同步文章 |
| **模板适配** | 不同平台可配置不同的内容模板（如自动添加"原文链接"水印） |
| **数据回采** | 同步后自动回采各平台阅读量、评论数等数据 |
| **评论同步** | 将各平台评论统一展示和管理 |
| **短视频同步** | 扩展脚本框架支持视频上传类型 |
| **团队协作** | 支持多用户、子账号权限管理 |
| **Webhook 通知** | 同步结果通过 Webhook 推送到企业微信/钉钉 |
| **CLI / MCP 支持** | 提供命令行工具和 MCP 协议，支持 AI 工具调用 |
