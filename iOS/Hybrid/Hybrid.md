# iOS Hybrid

自下而上梳理 iOS 端 Hybrid 方案

## 方法交互：原生 ↔ JS

### JS 方法

- 全局对象存储公共方法，实现与原生交互。`callNative`、`callJS`……
- 业务方法通过调用公共方法实现传参（ JSON 函数用 `callbackId` 表示）。回调函数通过维护一个 `callback` 字典实现（ key 为 `callbakId`）
- 业务方法由客户端通过代码规范（遵守协议等）利用 AST（抽象语法树）在编译时直出 JS 方法（文件）

### JS 注入

##### UIWebView

通过 `JSContext` 注入上述生成的 JS 文件（公共方法、业务方法），同时使用 `JSExport` 协议，生成原生与 JS 的桥接类（用于 JS 在公共方法中实现与原生的交互）。

##### WKWebView

通过 `WKUserContentController`、`WKUserScript` 实现 JS 文件的注入，通过`WKScriptMessageHandler`协议实现 JS 事件监听（无法同步返回数据），通过不太规范的拦截`webView:runJavaScriptTextInputPanelWithPrompt:defaultText:initiatedByFrame:completionHandler:`回调实现同步数据返回。

### 参数解析

为了兼容 `UIWebView` 和 `WKWebView`，使用解析类统一解析参数，通过运行时分发至具体业务类（上述用于编译直出 JS 方法的类），完成 **JS → 原生**的调用。

业务类完成业务逻辑后，同步返回结果（有返回值则返回，无则返回`null`），如有异步回调则在完成时回调给JS：`UIWebView` 通过 `JSContext`，`WKWebView`直接通过实例方法`evaluateJavaScript:completionHandler:`调用公共方法中的 `callJS`。完成 **原生 → JS**的调用、回调。

## 资源文件管理

### 文件下载、缓存

使用全量包 + 增量包的模式进行资源包（HTML、JS、CSS）的异步下载，使用哈希算法（MD5 ）进行文件完整性校验，写入磁盘缓存。

### 请求拦截

使用 `NSURLProtocol` 拦截前端页面的网络请求，用于添加 `cookie`、缓存资源（无则请求网络后缓存、有则直接返回数据）、处理重定向。

### 缓存设计

文件缓存（上述），接口缓存。

