# Puppeteer：从零出发，全面掌握浏览器自动化神器

> 全景式详解 Puppeteer 核心概念，带你轻松入门网页抓取与自动化测试

## 框架介绍

Puppeteer 译为木偶，是一个 Node.js 库，内部通过 DevTools 协议提供控制 Chrome 或 Firefox 的一系列 API。通过定义可以看出 Puppeteer 的核心在于提供用户控制浏览器行为的方法，以下是一些自动化入门示例：

* 自动提交表单、UI 测试、键盘输入等；
* 使用最新的 JavaScript 和 浏览器特性创建自动化环境；
* 捕获网站的时间线跟踪，帮助诊断性能问题；
* 测试 Chrome 扩展程序；
* 对页面截图和生成 PDF；
* 对 SPA 应用爬取并生成预渲染内容；

## 安装指引

Puppeteer 从 v1.7.0+ 开始同时提供 `puppeteer` 和 `puppeteer-core` 两个包：

* `puppeteer` 是在 `puppeteer-core` 基础上提供了更加完整的浏览器自动化产品：
  * 安装期间会下载与 Puppeteer 版本匹配的 Chrome for Testing；
  * `puppeteer@v21.6.0+` 会同时下载 `chrome-headless-shell` 二进制文件；
  * 默认安装位置：`$HOME/.cache/puppeteer`；
  * 提供合理的默认选项；

* `puppeteer-core` 是通过 DevTools 协议提供编程接口驱动的核心库：
  * 安装期间不会下载 Chrome for Testing 及 `chrome-headless-shell`；
  * 不提供任何默认选项；


```shell
npm i puppeteer # 完整版
npm i puppeteer-core # 核心库，需要显示指定远程/本地浏览器的连接地址
```

### 入门示例：

先快速初始化一个示例项目：

```shell
mkdir caniuse-puppeteer && cd caniuse-puppeteer
npm init -y && npm i puppeteer
echo 'console.log("Hello World!");' > index.mjs
npm pkg set type="module" 
npm pkg set scripts.dev="node index.mjs" 
```

PS：在使用 `npm pkg set` 时要注意 npm 的版本。

在示例中我尝试模拟用户在 caniuse.com 检索 **Flexible** 关键词，并打印出的第一条信息的描述内容：

```javascript
import puppeteer from 'puppeteer';

(async () => {
  	// ① 启动浏览器并打开一个新的页签
    const browser = await puppeteer.launch()
    const page = await browser.newPage()
    await page.setViewport({ width: 1920, height: 1080 })
  	// ② 跳转到 https://caniuse.com/ 地址
    await page.goto('https://caniuse.com/')
		
  	// ③ 通过 locator 定位 input 元素，并输入 Flexible 关键词
    await page.locator('input[name="search"]').fill('Flexible')
  	// ④ 等待查询结果对应的元素出现
    await page.waitForSelector('ciu-feature-list >>> .section__tables .feature-list-wrap ciu-feature >>> header div p')

  	// ⑤ 利用 evaluate 方法到目标页面的上下文获取第一条描述内容
    const textContent = await page.evaluate(() => {
        const element = document.querySelector("#main > main > ciu-feature-list")
            .shadowRoot.querySelector("#flexbox")
            .shadowRoot.querySelector("header > div > p")
        return element.textContent.trim()
    })
    console.log(textContent)
  
		// ⑥ 关闭浏览器
    await browser.close()
})()
```

PS：**>>>** 是 Puppeteer 提供的查询 Shadow DOM 元素的组合器。

### 运行环境：

* Node 18+，Puppeteer 遵循 Node 最新维护的 LTS 版本；
* 如果要使用 TypeScript，需要安装  TypeScript 4.7.4 +；
* 可以在 Windows（x64）、MacOS（x64/arm64）、Debian/Ubuntu Linux（x64）；

## 核心概念

Puppeteer 拥有 4 个核心概念，分别是：

| 核心概念        | 描述                                                         |
| :-------------- | ------------------------------------------------------------ |
| 浏览器管理      | Puppeteer 提供了启动、关闭和连接已启动的浏览器等主要功能。   |
| JavaScript 执行 | Puppeteer 在其驱动的页面上下文中执行 JavaScript 函数。       |
| 网络日志        | Puppeteer 默认监听所有的网络请求和响应，并在 `page` 上派发对应的事件 |
| 页面交互        | Puppeteer 允许使用鼠标、触摸事件和键盘输入与页面元素交互，通常应首先使用 CSS 选择器查询 DOM 元素。 |

### 浏览器管理：

在**入门示例**中已经使用过了启动和关闭浏览器的 API，这里主要了解一下浏览器上下文（包含权限）和如何连接到正在运行的浏览器两部分。

1. 浏览器上下文及上下文权限：

   * 浏览器上下文的作用是隔离自动换任务，保证 Cookie 和本地存储不会在浏览器上下文之间共享；
   * 浏览器上下文所关联的页面会在关闭上下文时一同被关闭；
   * 浏览器上下文支持权限配置，例如在首次访问高德地图需要提供 `geolocation` 权限；

   获取和创建浏览器上下文 API：

   ```javascript
    // 获取默认的浏览器上下文
    await browser.defaultBrowserContext()
    
    // 创建一个新的浏览器上下文
    await browser.createBrowserContext()
   ```

   为默认浏览器上下文重写访问高德地图的定位权限：


   ```JavaScript
   import puppeteer from 'puppeteer';
   
   (async () => {
       const browser = await puppeteer.launch({
           headless: false,
       })
       const context = await browser.defaultBrowserContext()
   
       const url = 'https://ditu.amap.com/'
       await context.overridePermissions(url, ['geolocation'])
   
       const page = await context.newPage()
       await page.goto(url)
   })()
   ```

2. 如何连接到正在运行的浏览器：

   * 除了入门示例是用到的启动浏览器的方式外，还可以使用 `connect` 直接连接到已启动的浏览器。

   使用 launch 启动浏览器：
   
   ```javascript
   import puppeteer from 'puppeteer';
   
   (async () => {
       const browser = await puppeteer.launch({
           headless: false,
           // 设置远程调试端口
           args: ['--remote-debugging-port=9222']
       })
       const page = await browser.newPage()
       // /json/version 获取 webSocketDebuggerUrl
       await page.goto('http://localhost:9222/json/version')
   })()
   ```

   连接上一个浏览器并打印 caniuse.com 网站的 news 元素：
   
   ```JavaScript
   import puppeteer from 'puppeteer';
   
   (async () => {
       const browser = await puppeteer.connect({
           browserWSEndpoint: "ws://localhost:9222/devtools/browser/5bce4998-1ba5-4f96-b512-d0fb8d49187d"
       })
       const page = await browser.newPage()
       await page.goto('https://caniuse.com/')
       const textContext = await page.locator('.news').map(el => el.textContent).wait()
       console.log(textContext)
   
       // 断开连接并不会关闭浏览器
       await browser.disconnect()
   })()
   ```


### JavaScript 执行：

### 网络日志：

### 页面交互：