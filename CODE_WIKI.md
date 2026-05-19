# WechatSync - Code Wiki

> **项目**: 微信公众号同步助手 (WechatSync)  
> **版本**: 2.0.9  
> **仓库**: https://github.com/wechatsync/Wechatsync  
> **协议**: GPL-3.0  
> **生成日期**: 2026-05-19

---

## 目录

1. [项目概述](#1-项目概述)
2. [整体架构](#2-整体架构)
3. [包结构详解](#3-包结构详解)
4. [核心类型系统](#4-核心类型系统)
5. [关键类与函数](#5-关键类与函数)
6. [平台适配器体系](#6-平台适配器体系)
7. [运行时抽象层](#7-运行时抽象层)
8. [内容处理引擎](#8-内容处理引擎)
9. [MCP/AI 集成](#9-mcpai-集成)
10. [依赖关系](#10-依赖关系)
11. [项目运行方式](#11-项目运行方式)
12. [数据流与工作原理](#12-数据流与工作原理)

---

## 1. 项目概述

WechatSync 是一款**开源免费**的跨平台文章同步工具，以 Chrome 浏览器扩展为核心载体，支持将微信公众号文章一键同步到知乎、头条、掘金、CSDN 等 29+ 自媒体平台。

### 核心特性

- **一键批量发布**: 同步到 29+ 平台
- **网页转 Markdown**: 智能提取正文，过滤广告噪音
- **自建站支持**: WordPress、Typecho、博客园 (MetaWeblog API)
- **AI 集成**: 支持 Anthropic MCP 协议，可在 Claude Desktop / Claude Code 中使用
- **数据安全**: 所有操作在本地浏览器完成，不经过第三方服务器
- **草稿优先**: 默认保存为草稿，发布前人工确认

### 技术栈

| 层级 | 技术 |
|------|------|
| 语言 | TypeScript 5.x |
| 包管理 | pnpm + Yarn workspaces |
| 构建 | tsup (core/cli/mcp), Vite (extension) |
| 前端框架 | React 18 + Tailwind CSS + Zustand |
| 扩展框架 | Chrome Extension Manifest V3 |
| 测试 | Vitest |

---

## 2. 整体架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                        WechatSync 架构图                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐          │
│  │   CLI 工具    │    │  MCP Server  │    │  Claude Code │          │
│  │  @wechatsync │    │  @wechatsync │    │   Desktop    │          │
│  │     /cli     │    │  /mcp-server │    │              │          │
│  └──────┬───────┘    └──────┬───────┘    └──────┬───────┘          │
│         │                   │                   │                   │
│         │  WebSocket Bridge │                   │                   │
│         └─────────┬─────────┘                   │                   │
│                   │                             │                   │
│                   ▼                             ▼                   │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │              Chrome Extension (MV3)                          │   │
│  │           @wechatsync/extension                              │   │
│  │  ┌─────────┐  ┌──────────┐  ┌─────────┐  ┌──────────────┐  │   │
│  │  │ Popup   │  │ Content  │  │Background│  │  Editor      │  │   │
│  │  │  UI     │  │ Script   │  │Service   │  │  Page        │  │   │
│  │  │         │  │          │  │ Worker   │  │              │  │   │
│  │  └────┬────┘  └────┬─────┘  └────┬────┘  └──────┬───────┘  │   │
│  │       └─────────────┴─────────────┴──────────────┘          │   │
│  │                          │                                  │   │
│  │                          ▼                                  │   │
│  │              ┌──────────────────────┐                       │   │
│  │              │   @wechatsync/core   │                       │   │
│  │              │  ┌────────────────┐  │                       │   │
│  │              │  │ 适配器注册中心  │  │                       │   │
│  │              │  │ 运行时抽象层   │  │                       │   │
│  │              │  │ 内容处理引擎   │  │                       │   │
│  │              │  │ AI 处理器      │  │                       │   │
│  │              │  └────────────────┘  │                       │   │
│  │              └──────────────────────┘                       │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    目标平台 (29+)                            │   │
│  │  知乎  掘金  CSDN  微博  B站  百家号  语雀  豆瓣  搜狐 ...    │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

### 架构设计原则

1. **核心共享**: `@wechatsync/core` 包含所有平台适配器和内容处理逻辑，被 extension/cli/mcp-server 共享
2. **运行时抽象**: `RuntimeInterface` 抽象层使核心逻辑可在浏览器扩展和 Node.js 环境中复用
3. **适配器模式**: 每个平台一个适配器，统一接口，独立实现
4. **安全优先**: 使用浏览器原生 Cookie，不模拟登录，不经过第三方服务器

---

## 3. 包结构详解

### 3.1 根项目 (`/`)

```
/
├── package.json          # 根 package.json，定义 workspaces
├── pnpm-workspace.yaml   # pnpm workspace 配置
├── tsconfig.json         # 根 TypeScript 配置
├── README.md             # 项目说明文档
├── CHANGELOG.md          # 更新日志
├── CONTRIBUTING.md       # 贡献指南
├── docs/
│   └── adapter-spec.md   # 适配器开发规范
└── packages/             # 子包目录
```

### 3.2 Core 包 (`packages/core/`)

**职责**: 共享核心逻辑 - 平台适配器、运行时抽象、同步引擎、内容处理

```
packages/core/
├── src/
│   ├── index.ts              # 统一导出入口
│   ├── types.ts              # 核心类型定义 (Article, SyncResult, AuthResult 等)
│   ├── runtime/
│   │   ├── index.ts          # 运行时模块导出
│   │   └── interface.ts      # RuntimeInterface 定义
│   ├── adapters/
│   │   ├── base.ts           # BaseAdapter 基类
│   │   ├── types.ts          # 适配器类型定义
│   │   ├── registry.ts       # 适配器注册中心 (AdapterRegistry)
│   │   ├── code-adapter.ts   # 代码适配器工具
│   │   ├── index.ts          # 适配器模块导出
│   │   └── platforms/        # 各平台适配器实现
│   │       ├── index.ts      # 平台适配器统一导出
│   │       ├── zhihu.ts      # 知乎适配器
│   │       ├── juejin.ts     # 掘金适配器
│   │       ├── csdn.ts       # CSDN 适配器
│   │       ├── weibo.ts      # 微博适配器
│   │       ├── bilibili.ts   # B站适配器
│   │       ├── weixin.ts     # 微信公众号适配器
│   │       ├── yuque.ts      # 语雀适配器
│   │       ├── douban.ts     # 豆瓣适配器
│   │       ├── sohu.ts       # 搜狐号适配器
│   │       ├── xueqiu.ts     # 雪球适配器
│   │       ├── woshipm.ts    # 人人都是产品经理适配器
│   │       ├── baijiahao.ts  # 百家号适配器
│   │       ├── cto51.ts      # 51CTO 适配器
│   │       ├── imooc.ts      # 慕课网适配器
│   │       ├── oschina.ts    # 开源中国适配器
│   │       ├── segmentfault.ts # SegmentFault 适配器
│   │       ├── cnblogs.ts    # 博客园适配器
│   │       ├── eastmoney.ts  # 东方财富适配器
│   │       └── zip-download.ts # Markdown ZIP 下载适配器
│   ├── ai/
│   │   └── index.ts          # AI 处理器接口 (AIProcessor)
│   └── lib/
│       ├── index.ts          # 工具库导出
│       ├── turndown.ts       # HTML ↔ Markdown 转换引擎
│       ├── markdown-images.ts # Markdown 图片解析
│       ├── markdown-to-draft.ts # Markdown → Draft.js
│       ├── logger.ts         # 日志系统
│       └── aws4.ts           # AWS4 签名工具
├── package.json
├── tsconfig.json
└── tsup.config.ts
```

### 3.3 Extension 包 (`packages/extension/`)

**职责**: Chrome 浏览器扩展 (Manifest V3)，提供 UI、内容提取、后台同步

```
packages/extension/
├── src/
│   ├── popup/                # 弹出窗口 UI
│   │   ├── App.tsx           # Popup 主组件
│   │   ├── pages/            # 页面组件
│   │   │   ├── HomeNew.tsx   # 首页
│   │   │   ├── History.tsx   # 历史记录
│   │   │   ├── About.tsx     # 关于页面
│   │   │   └── AddCMS.tsx    # 添加 CMS 账户
│   │   ├── stores/           # Zustand 状态管理
│   │   │   ├── sync.ts       # 同步状态
│   │   │   └── cms.ts        # CMS 账户状态
│   │   └── styles/
│   │       └── globals.css   # 全局样式
│   ├── content/              # Content Scripts
│   │   ├── api.ts            # 页面 API 兼容层 ($syncer)
│   │   ├── extractor.ts      # 文章提取器
│   │   ├── weixin.ts         # 微信公众号特殊处理
│   │   ├── weixin-editor.ts  # 微信编辑器处理
│   │   └── toutiao.ts        # 头条号处理
│   ├── background/           # Service Worker
│   │   ├── index.ts          # 后台消息处理中心
│   │   └── sync-service.ts   # 同步服务
│   ├── editor/               # 独立编辑器页面
│   │   ├── EditorApp.tsx     # 编辑器应用
│   │   ├── main.tsx          # 编辑器入口
│   │   └── index.html        # 编辑器 HTML
│   ├── sync-dialog/          # 同步对话框页面
│   │   ├── SyncDialogPage.tsx
│   │   ├── main.tsx
│   │   └── index.html
│   ├── components/           # 共享组件
│   │   └── sync-dialog/      # 同步对话框组件
│   │       ├── SyncDialog.tsx
│   │       ├── ArticleCard.tsx
│   │       ├── PlatformList.tsx
│   │       └── ...
│   ├── adapters/             # 扩展端适配器初始化
│   │   ├── index.ts          # 适配器系统初始化
│   │   └── cms/              # CMS 适配器
│   │       ├── wordpress.ts
│   │       └── metaweblog.ts
│   ├── runtime/
│   │   └── extension.ts      # ExtensionRuntime 实现
│   ├── lib/                  # 扩展端工具库
│   │   ├── logger.ts         # 日志
│   │   ├── analytics.ts      # 埋点分析
│   │   ├── rate-limit.ts     # 频率限制
│   │   ├── content-processor.ts # 内容预处理
│   │   ├── reader/           # 文章阅读器
│   │   ├── utils.ts          # 通用工具
│   │   └── ...
│   └── mcp/
│       └── client.ts         # MCP WebSocket 客户端
├── public/                   # 静态资源
│   ├── inject-api.js         # 页面注入 API
│   └── lib/                  # Readability.js 等
├── assets/                   # 扩展图标
├── manifest.json             # Chrome 扩展清单 (MV3)
├── vite.config.ts            # Vite 构建配置
├── tailwind.config.js        # Tailwind 配置
└── package.json
```

### 3.4 CLI 包 (`packages/cli/`)

**职责**: 命令行工具，通过 WebSocket 桥接与 Chrome 扩展通信

```
packages/cli/
├── src/
│   └── index.ts              # CLI 入口 (Commander.js)
├── package.json
├── tsconfig.json
└── tsup.config.ts
```

**命令**:
- `wechatsync sync <file>` - 同步文章到平台
- `wechatsync platforms` - 列出支持的平台
- `wechatsync auth [platform]` - 检查登录状态
- `wechatsync extract` - 从浏览器提取文章

### 3.5 MCP Server 包 (`packages/mcp-server/`)

**职责**: Anthropic MCP 协议服务器，桥接 Claude Code 与 Chrome 扩展

```
packages/mcp-server/
├── src/
│   ├── index.ts              # MCP Server 入口 (stdio/SSE 双模式)
│   ├── server.ts             # SyncAssistantMcpServer 类
│   ├── ws-bridge.ts          # ExtensionBridge WebSocket 桥接
│   ├── types.ts              # 类型定义
│   └── exports.ts            # 桥接导出
├── package.json
├── tsconfig.json
└── tsup.config.ts
```

---

## 4. 核心类型系统

### 4.1 文章类型 (`packages/core/src/types.ts`)

```typescript
export interface Article {
  title: string           // 文章标题
  markdown: string        // Markdown 格式内容（主要）
  html?: string           // 原始 HTML（可选）
  summary?: string        // 摘要
  cover?: string          // 封面图 URL
  tags?: string[]         // 标签
  category?: string       // 分类
  source?: {              // 来源信息
    url: string
    platform: string
  }
}
```

### 4.2 同步结果类型

```typescript
export interface SyncResult {
  platform: string        // 平台 ID
  success: boolean        // 是否成功
  postId?: string         // 文章 ID
  postUrl?: string        // 文章 URL
  draftOnly?: boolean     // 是否只保存草稿
  error?: string          // 错误信息
  message?: string        // 额外提示
  timestamp: number       // 时间戳
}
```

### 4.3 认证结果类型

```typescript
export interface AuthResult {
  isAuthenticated: boolean  // 是否已登录
  username?: string         // 用户名
  userId?: string           // 用户 ID
  avatar?: string           // 头像
  error?: string            // 错误信息
}
```

### 4.4 平台元信息类型

```typescript
export interface PlatformMeta {
  id: string              // 平台唯一标识
  name: string            // 平台显示名称
  icon: string            // 平台图标 URL
  homepage: string        // 平台主页
  capabilities: PlatformCapability[]  // 支持的能力
}

export type PlatformCapability =
  | 'article'      // 发布文章
  | 'draft'        // 草稿支持
  | 'image_upload' // 图片上传
  | 'categories'   // 分类
  | 'tags'         // 标签
  | 'cover'        // 封面图
  | 'schedule'     // 定时发布
```

### 4.5 运行时接口类型 (`packages/core/src/runtime/interface.ts`)

```typescript
export interface RuntimeInterface {
  readonly type: 'extension' | 'node'
  
  // HTTP 请求
  fetch(url: string, options?: RequestInit): Promise<Response>
  
  // Cookie 管理
  cookies: {
    get(domain: string): Promise<Cookie[]>
    set(cookie: Cookie): Promise<void>
    remove(name: string, domain: string): Promise<void>
  }
  
  // 获取单个 Cookie
  getCookie?(domain: string, name: string): Promise<string | null>
  
  // 持久化存储
  storage: {
    get<T>(key: string): Promise<T | null>
    set<T>(key: string, value: T): Promise<void>
    remove(key: string): Promise<void>
  }
  
  // 会话存储
  session: {
    get<T>(key: string): Promise<T | null>
    set<T>(key: string, value: T): Promise<void>
  }
  
  // Header 规则管理 (仅扩展环境)
  headerRules?: {
    add(rule: HeaderRule): Promise<string>
    remove(ruleId: string): Promise<void>
    clear(): Promise<void>
  }
  
  // 文件下载 (仅扩展环境)
  downloads?: {
    download(blob: Blob, filename: string, saveAs?: boolean): Promise<number>
  }
  
  // Tab 管理 (仅扩展环境)
  tabs?: {
    query(urlPattern: string): Promise<Array<{ id: number; url?: string }>>
    create(url: string, active?: boolean): Promise<{ id: number }>
    waitForLoad(tabId: number, timeout?: number): Promise<void>
    executeScript<T, A extends unknown[]>(tabId: number, func: (...args: A) => T | Promise<T>, args: A): Promise<T>
  }
  
  // DOM 操作
  dom: {
    parseHTML(html: string): Promise<Document>
    querySelector(doc: Document, selector: string): Element | null
    querySelectorAll(doc: Document, selector: string): Element[]
    getTextContent(element: Element): string
    getInnerHTML(element: Element): string
  }
}
```

---

## 5. 关键类与函数

### 5.1 适配器基类 (`BaseAdapter`)

**位置**: `packages/core/src/adapters/base.ts`

```typescript
export abstract class BaseAdapter implements PlatformAdapter {
  abstract readonly meta: PlatformMeta
  protected runtime!: RuntimeInterface
  protected context: Record<string, unknown> = {}

  async init(runtime: RuntimeInterface): Promise<void>
  abstract checkAuth(): Promise<AuthResult>
  abstract publish(article: Article): Promise<SyncResult>
  
  // 通用 HTTP 请求
  protected async request<T>(url: string, options?: RequestInit): Promise<T>
  
  // 带重试的请求
  protected async requestWithRetry<T>(url: string, options?: RequestInit, maxRetries?: number): Promise<T>
  
  // 延迟工具
  protected delay(ms: number): Promise<void>
  
  // 创建同步结果
  protected createResult(success: boolean, data?: Partial<SyncResult>): SyncResult
}
```

**职责**: 为所有平台适配器提供通用的 HTTP 请求、重试、延迟、结果创建等基础能力。

### 5.2 适配器注册中心 (`AdapterRegistry`)

**位置**: `packages/core/src/adapters/registry.ts`

```typescript
class AdapterRegistry {
  setRuntime(runtime: RuntimeInterface): void
  register(entry: AdapterRegistryEntry): void
  registerAll(entries: AdapterRegistryEntry[]): void
  async get(platformId: string): Promise<PlatformAdapter | null>
  getAllMeta(): PlatformMeta[]
  has(platformId: string): boolean
  getRegisteredIds(): string[]
  clear(): void
  getPreprocessConfig(platformId: string): PreprocessConfig
  getPreprocessConfigs(platformIds: string[]): Record<string, PreprocessConfig>
}

// 全局实例
export const adapterRegistry = new AdapterRegistry()
```

**职责**: 管理所有平台适配器的注册、获取和生命周期。采用单例模式，缓存适配器实例。

### 5.3 扩展运行时 (`ExtensionRuntime`)

**位置**: `packages/extension/src/runtime/extension.ts`

```typescript
export class ExtensionRuntime implements RuntimeInterface {
  readonly type = 'extension' as const
  
  async fetch(url: string, options?: RequestInit): Promise<Response>
  cookies: { get, set, remove }
  async getCookie(domain: string, name: string): Promise<string | null>
  storage: { get, set, remove }
  session: { get, set }
  headerRules: { add, remove, clear }
  downloads: { download }
  tabs: { query, create, waitForLoad, executeScript }
  dom: { parseHTML, querySelector, querySelectorAll, getTextContent, getInnerHTML }
}

export function createExtensionRuntime(config?: RuntimeConfig): ExtensionRuntime
```

**职责**: 在 Chrome 扩展环境中实现 `RuntimeInterface`，封装 Chrome API。

### 5.4 WebSocket 桥接 (`ExtensionBridge`)

**位置**: `packages/mcp-server/src/ws-bridge.ts`

```typescript
export class ExtensionBridge {
  constructor(port: number = 9527, options?: { silent?: boolean })
  
  async start(): Promise<void>           // 启动服务（自动选择 PRIMARY/SECONDARY）
  stop(): void                           // 停止服务
  getMode(): 'primary' | 'secondary'     // 获取运行模式
  isConnected(): boolean                 // 检查扩展是否连接
  waitForConnection(timeoutMs?: number): Promise<void>
  async request<T>(method: string, params?: Record<string, unknown>): Promise<T>
  async uploadImageChunked(imageData: string, mimeType: string, platform?: string): Promise<{ url: string; platform: string }>
}
```

**职责**: 在 CLI/MCP Server 与 Chrome 扩展之间建立 WebSocket 通信桥接。支持多实例模式（PRIMARY/SECONDARY）和自动端口接管。

### 5.5 MCP Server (`SyncAssistantMcpServer`)

**位置**: `packages/mcp-server/src/server.ts`

```typescript
export class SyncAssistantMcpServer {
  constructor(wsPort: number = 9527, httpPort: number = 9528)
  
  private setupHandlers(): void          // 设置 MCP 工具处理器
  private setupHttpRoutes(): void        // 设置 HTTP 路由
  async start(): Promise<void>           // 启动服务器
}
```

**提供的 MCP 工具**:

| 工具名 | 说明 |
|--------|------|
| `list_platforms` | 列出所有支持的平台及登录状态 |
| `check_auth` | 检查指定平台登录状态 |
| `sync_article` | 同步文章到指定平台（草稿） |
| `extract_article` | 从当前浏览器页面提取文章 |
| `upload_image_file` | 上传本地图片到图床 |

### 5.6 后台消息处理中心

**位置**: `packages/extension/src/background/index.ts`

核心消息类型:

```typescript
type MessageAction =
  | { type: 'GET_PLATFORMS' }
  | { type: 'CHECK_ALL_AUTH'; payload?: { forceRefresh?: boolean } }
  | { type: 'CHECK_AUTH'; payload: { platformId: string } }
  | { type: 'SYNC_ARTICLE'; payload: { article: any; platforms: string[]; ... } }
  | { type: 'OPEN_SYNC_PAGE'; path?: string }
  | { type: 'TEST_CMS_CONNECTION'; payload: { type: CMSType; url: string; username: string; password: string } }
  | { type: 'SYNC_TO_CMS'; payload: { accountId: string; article: any } }
  | { type: 'MCP_ENABLE' | 'MCP_DISABLE' | 'MCP_STATUS' | 'MCP_SET_SERVER_URL' }
  | { type: 'UPLOAD_IMAGE'; payload: { src: string; platform?: string } }
  | { type: 'MAGIC_CALL'; payload: { methodName: string; data: any } }
  | ...
```

**职责**: 处理来自 popup、content script、MCP 客户端的所有消息，协调同步流程。

### 5.7 内容提取与转换引擎

**位置**: `packages/core/src/lib/turndown.ts`

```typescript
// HTML → Markdown (原生 DOM，推荐)
export function htmlToMarkdownNative(html: string, options?: TurndownOptions): string

// HTML → Markdown (正则回退，Service Worker 环境)
export function htmlToMarkdown(html: string, options?: TurndownOptions): string

// Markdown → HTML
export function markdownToHtml(markdown: string): string

// HTML 标准化 (HTML → Markdown → HTML 往返)
export function normalizeHtml(html: string, options?: TurndownOptions): string

// 修复代码块中未转义的 < 字符
export function fixUnescapedLtInCode(html: string): string

// 从 HTML 安全提取代码文本
export function extractCodeFromHtml(html: string): string
```

**职责**: 提供高质量的 HTML ↔ Markdown 双向转换，支持表格、代码块、LaTeX 公式等复杂元素。

---

## 6. 平台适配器体系

### 6.1 适配器接口 (`PlatformAdapter`)

**位置**: `packages/core/src/adapters/types.ts`

```typescript
export interface PlatformAdapter {
  readonly meta: PlatformMeta
  readonly preprocessConfig?: Partial<PreprocessConfig>
  
  init(runtime: RuntimeInterface): Promise<void>
  checkAuth(): Promise<AuthResult>
  publish(article: Article, options?: PublishOptions): Promise<SyncResult>
  
  // 可选能力
  uploadImage?(file: Blob, filename?: string): Promise<string>
  getCategories?(): Promise<Category[]>
  getDrafts?(): Promise<Draft[]>
  update?(postId: string, article: Article): Promise<SyncResult>
  delete?(postId: string): Promise<void>
}
```

### 6.2 预处理配置 (`PreprocessConfig`)

每个适配器可定义自己的内容预处理配置，Content Script 根据此配置在发送到 Service Worker 前进行预处理:

```typescript
export interface PreprocessConfig {
  outputFormat: 'html' | 'markdown'
  removeLinks?: boolean
  removeIframes?: boolean
  removeComments?: boolean
  removeSpecialTags?: boolean
  processCodeBlocks?: boolean
  processLazyImages?: boolean
  removeEmptyElements?: boolean
  convertTablesToText?: boolean
  keepStyles?: boolean
  // ... 更多选项
}
```

### 6.3 已支持平台 (23 个公开 + 私有)

| 平台 | ID | 适配器文件 |
|------|-----|-----------|
| 知乎 | zhihu | `zhihu.ts` |
| 掘金 | juejin | `juejin.ts` |
| CSDN | csdn | `csdn.ts` |
| 微博 | weibo | `weibo.ts` |
| B站专栏 | bilibili | `bilibili.ts` |
| 百家号 | baijiahao | `baijiahao.ts` |
| 语雀 | yuque | `yuque.ts` |
| 微信公众号 | weixin | `weixin.ts` |
| 豆瓣 | douban | `douban.ts` |
| 搜狐号 | sohu | `sohu.ts` |
| 雪球 | xueqiu | `xueqiu.ts` |
| 人人都是产品经理 | woshipm | `woshipm.ts` |
| 51CTO | cto51 | `cto51.ts` |
| 慕课网 | imooc | `imooc.ts` |
| 开源中国 | oschina | `oschina.ts` |
| SegmentFault | segmentfault | `segmentfault.ts` |
| 博客园 | cnblogs | `cnblogs.ts` |
| 东方财富 | eastmoney | `eastmoney.ts` |
| Markdown ZIP | zip-download | `zip-download.ts` |

私有适配器通过 `import.meta.glob` 动态加载，存储在 git submodule 中。

---

## 7. 运行时抽象层

### 7.1 设计目的

`RuntimeInterface` 抽象层使 `@wechatsync/core` 中的平台适配器和内容处理逻辑可以在不同环境中复用:

- **浏览器扩展环境**: 使用 Chrome API (cookies, storage, declarativeNetRequest, tabs)
- **Node.js 环境**: 使用 node-fetch, jsdom, 文件系统等替代实现

### 7.2 环境差异对比

| 能力 | 扩展环境 (`ExtensionRuntime`) | Node 环境 (未来) |
|------|------------------------------|-----------------|
| HTTP 请求 | `fetch` (自动携带 cookies) | `node-fetch` (需手动管理 cookies) |
| Cookie 管理 | `chrome.cookies` API | Cookie jar / 手动管理 |
| 存储 | `chrome.storage.local/session` | 文件系统 / 内存 |
| Header 规则 | `chrome.declarativeNetRequest` | 请求拦截器 |
| 文件下载 | `chrome.downloads` | 文件系统写入 |
| Tab 管理 | `chrome.tabs` + `chrome.scripting` | 无 / Puppeteer |
| DOM 操作 | 原生 DOMParser | jsdom / linkedom |

---

## 8. 内容处理引擎

### 8.1 HTML → Markdown 转换流程

```
原始 HTML
    │
    ▼
┌─────────────────┐
│ 1. 预处理        │
│    - 修复未转义的 < │
│    - 移除微信代码行号 │
│    - 处理懒加载图片   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 2. DOM 解析      │
│    - 原生 DOMParser │
│    - 或 linkedom   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 3. Turndown 转换 │
│    - 表格规则      │
│    - 代码块规则    │
│    - 链接清理      │
│    - LaTeX 公式   │
└────────┬────────┘
         │
         ▼
    Markdown
```

### 8.2 内容预处理流程

Content Script 根据各平台的 `preprocessConfig` 在发送前预处理 HTML:

```
原始 HTML (来自源平台)
    │
    ▼
┌─────────────────────────┐
│ 平台特定预处理            │
│ - 移除/保留特定标签       │
│ - 处理代码块              │
│ - 转换图片                │
│ - 清理空元素              │
│ - 链接处理                │
└───────────┬─────────────┘
            │
    ┌───────┼───────┐
    ▼       ▼       ▼
 平台A   平台B   平台C
 内容    内容    内容
```

---

## 9. MCP/AI 集成

### 9.1 MCP 协议支持

项目支持 [Anthropic Model Context Protocol (MCP)](https://modelcontextprotocol.io/)，允许 Claude Desktop / Claude Code 通过 AI 自然语言操作同步功能。

**连接方式**:
1. **stdio 模式** (推荐): `claude mcp add sync-assistant node dist/index.js`
2. **SSE 模式**: `claude mcp add --transport sse sync-assistant http://localhost:9528/sse`

### 9.2 AI 处理器接口

**位置**: `packages/core/src/ai/index.ts`

```typescript
export interface AIProcessor {
  optimizeTitle(title: string, platform: string): Promise<string[]>
  generateSummary(content: string, maxLength?: number): Promise<string>
  suggestTags(content: string, platform: string): Promise<string[]>
  adaptContent(content: string, sourcePlatform: string, targetPlatform: string): Promise<string>
}
```

当前实现为 `NoopAIProcessor`（空实现），预留接口供后续接入 OpenAI、Claude 等 AI 服务。

### 9.3 WebSocket 桥接架构

```
Claude Code          MCP Server          Chrome Extension
    │                    │                     │
    │  stdio/SSE         │                     │
    │◄──────────────────►│                     │
    │                    │   WebSocket         │
    │                    │◄───────────────────►│
    │                    │   (端口 9527)        │
    │                    │                     │
    │ "同步到知乎"        │                     │
    │───────────────────►│                     │
    │                    │  syncArticle        │
    │                    │────────────────────►│
    │                    │                     │ 调用知乎适配器
    │                    │                     │ 发布文章
    │                    │  返回结果            │
    │  显示结果           │◄────────────────────│
    │◄───────────────────│                     │
```

---

## 10. 依赖关系

### 10.1 包间依赖

```
@wechatsync/extension
    ├── @wechatsync/core (workspace:*)
    ├── react, react-dom
    ├── react-router-dom
    ├── zustand
    ├── tailwindcss
    ├── lucide-react
    ├── clsx, tailwind-merge
    ├── defuddle (文章提取)
    ├── juice (CSS 内联)
    └── @crxjs/vite-plugin

@wechatsync/cli
    ├── @wechatsync/mcp-server (workspace:*)
    ├── commander (CLI 框架)
    ├── chalk (终端颜色)
    ├── ora (加载动画)
    ├── open (打开浏览器)
    ├── juice
    └── ws

@wechatsync/mcp-server
    ├── @modelcontextprotocol/sdk
    ├── express
    ├── ws
    └── typescript

@wechatsync/core
    ├── turndown (HTML→Markdown)
    ├── marked (Markdown→HTML)
    ├── remarkable
    ├── markdown-draft-js
    ├── jszip
    ├── juice
    ├── linkedom
    ├── unified, rehype-*, remark-* (Unified 生态)
    ├── yaml
    ├── zod
    └── js-md5
```

### 10.2 外部服务依赖

| 服务 | 用途 | 是否必需 |
|------|------|---------|
| 各平台官方 API | 文章发布 | 是 |
| Chrome Web Store | 扩展分发 | 否 |
| www.wechatsync.com | 官网、远程配置、更新检查 | 否 |

---

## 11. 项目运行方式

### 11.1 环境要求

- **Node.js**: >= 20.0.0
- **pnpm**: 最新版
- **Chrome 浏览器**: 支持 Manifest V3

### 11.2 安装依赖

```bash
# 安装所有依赖
pnpm install

# 或使用 yarn
yarn install
```

### 11.3 开发模式

```bash
# 启动扩展开发服务器
pnpm dev

# 或单独启动
yarn workspace @wechatsync/extension dev
```

然后在 Chrome 中加载 `packages/extension/dist` 目录。

### 11.4 构建

```bash
# 构建所有包
pnpm build

# 构建单个包
pnpm build:core
pnpm build:extension
pnpm build:mcp
pnpm build:cli
```

### 11.5 测试

```bash
# 运行所有测试
pnpm test

# 类型检查
pnpm typecheck

# 代码检查
pnpm lint
```

### 11.6 MCP Server 启动

```bash
# stdio 模式（用于 Claude Desktop）
yarn workspace @wechatsync/mcp-server start

# SSE 模式
yarn workspace @wechatsync/mcp-server start --sse
```

### 11.7 CLI 使用

```bash
# 全局安装
npm install -g @wechatsync/cli

# 设置 Token（从扩展设置中获取）
export WECHATSYNC_TOKEN="your-token"

# 同步文章
wechatsync sync article.md -p zhihu,juejin,csdn

# 查看平台登录状态
wechatsync platforms --auth

# 从浏览器提取文章
wechatsync extract -o article.md
```

---

## 12. 数据流与工作原理

### 12.1 文章同步完整流程

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   用户操作   │     │  内容提取    │     │  平台同步    │
│             │     │             │     │             │
│ 1. 打开文章  │────►│ 2. 提取内容  │────►│ 3. 选择平台  │
│    页面      │     │    (Content) │     │             │
└─────────────┘     └─────────────┘     └──────┬──────┘
                                               │
                                               ▼
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   完成通知   │◄────│  结果汇总    │◄────│  并行发布    │
│             │     │  (Background)│     │  (Adapters)  │
│ 6. 显示结果  │     │ 5. 收集结果  │     │ 4. 调用各平台 │
└─────────────┘     └─────────────┘     └─────────────┘
```

### 12.2 内容提取流程

```
网页加载
    │
    ▼
┌─────────────────────┐
│ Readability.js      │  提取文章正文
│ (defuddle)          │  过滤广告/导航
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ htmlToMarkdownNative│  转换为 Markdown
│ (Turndown)          │  保留格式/代码块/表格
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ 图片处理             │  下载远程图片
│                     │  转为 base64/data URI
└──────────┬──────────┘
           │
           ▼
      Article 对象
      { title, markdown, html, cover }
```

### 12.3 安全模型

```
┌─────────────────────────────────────────┐
│           安全架构                        │
├─────────────────────────────────────────┤
│                                         │
│  ┌─────────┐    ┌─────────┐    ┌─────┐ │
│  │ 用户浏览器 │    │ 扩展    │    │ 平台 │ │
│  │ (已登录) │◄──►│ (MV3)  │◄──►│ API │ │
│  └─────────┘    └─────────┘    └─────┘ │
│       │                              │  │
│       │ Cookie (自动携带)             │  │
│       │                              │  │
│       ▼                              │  │
│  不经过任何第三方服务器                  │  │
│  所有请求直接从浏览器发往目标平台          │  │
│                                         │
└─────────────────────────────────────────┘
```

---

## 附录

### A. 目录结构总览

```
Wechatsync/
├── .claude/                    # Claude Code 配置
│   └── commands/               # 自定义命令
├── .claude-plugin/             # Claude 插件配置
├── docs/
│   └── adapter-spec.md         # 适配器开发规范
├── packages/
│   ├── cli/                    # 命令行工具
│   ├── core/                   # 核心逻辑共享包
│   ├── extension/              # Chrome 扩展
│   └── mcp-server/             # MCP 协议服务器
├── skills/
│   └── wechatsync/             # Claude Code Skill
├── package.json                # 根 package.json
├── pnpm-workspace.yaml         # pnpm workspace 配置
├── tsconfig.json               # 根 tsconfig
├── README.md                   # 项目说明
├── CHANGELOG.md                # 更新日志
└── CONTRIBUTING.md             # 贡献指南
```

### B. 关键文件速查

| 文件 | 说明 |
|------|------|
| `packages/core/src/types.ts` | 核心类型定义 |
| `packages/core/src/runtime/interface.ts` | 运行时接口 |
| `packages/core/src/adapters/base.ts` | 适配器基类 |
| `packages/core/src/adapters/registry.ts` | 适配器注册中心 |
| `packages/core/src/lib/turndown.ts` | HTML↔Markdown 引擎 |
| `packages/extension/src/background/index.ts` | 后台消息中心 |
| `packages/extension/src/runtime/extension.ts` | 扩展运行时实现 |
| `packages/extension/src/adapters/index.ts` | 适配器初始化 |
| `packages/mcp-server/src/ws-bridge.ts` | WebSocket 桥接 |
| `packages/mcp-server/src/index.ts` | MCP Server 入口 |
| `packages/cli/src/index.ts` | CLI 入口 |

### C. 版本历史

| 版本 | 日期 | 主要更新 |
|------|------|---------|
| v2.0.9 | 2026-03-24 | 文章提取增强，HTML 样式保留，UI 优化 |
| v2.0.8 | 2026-03-17 | 新增抖音图文，统一同步对话框 |
| v2.0.7 | 2026-03-10 | 新增什么值得买、网易号 |
| v2.0.6 | 2026-02-25 | 新增东方财富，悬浮同步按钮 |
| v2.0.5 | 2025-02-05 | Markdown 压缩包下载 |
