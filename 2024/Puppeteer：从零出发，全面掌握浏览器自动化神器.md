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

PS：`>>>` 是 Puppeteer 提供的查询 Shadow DOM 元素的组合器。

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
   
   ```javascript
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

在 Puppeteer 驱动的页面上下文中执行 JavaScript 函数同样在入门示例中有过使用，但没有提到如何传递参数和其中的一个缺陷。

传参：`evaluate` 第二个参数支持传递一个 **ElementHandle** 对象：

```javascript
import puppeteer from 'puppeteer';

(async () => {
    const browser = await puppeteer.launch()
    const page = await browser.newPage()
    await page.goto('https://caniuse.com/')

    const handle = await page.locator('.news').waitHandle()
    const textContent = await page.evaluate(el => el.textContent, handle)
    console.log(textContent)

    await browser.close()
})()
```

缺陷：上面示例中 textContent 被成功的输出，说明 **el** 是个有效的对象，但如果直接返回 **el** 对象，你会看到不一样的结果，终端输出了 `{}` 。造成这个现象的原因是 Puppeteer 会将对象序列化导致得到了不正确的结果，为了处理返回的对象，Puppeteer 提供了通过引用返回对象的方法：

```javascript
import puppeteer from 'puppeteer';

(async () => {
    const browser = await puppeteer.launch()
    const page = await browser.newPage()
    await page.goto('https://caniuse.com/')

    const handle = await page.locator('.news').waitHandle()
    const element = await page.evaluateHandle(el => el, handle)
    console.log(element instanceof ElementHandle)

    await browser.close()
})()
```

PS：在实际使用时要格外注意，多加测试。

### 网络日志：

`page` 提供了一个 `on(event, handler)` 函数，允许对 Puppeteer 派发的事件进行监听。

```javascript
import puppeteer from 'puppeteer';

(async () => {
    const browser = await puppeteer.launch()
    const page = await browser.newPage()
    await page.goto('https://caniuse.com/')

    page.on('request', request => {
        console.log('request : ', request.url())
    })

    page.on('response', response => {
        console.log('response : ', response.url())
    })
})()
```

### 页面交互：

前面的示例中或多或少都使用到了Puppeteer 提供与页面交互的 API，页面交互也是 Puppeteer 核心概念中内容最多的一块，所以放到这个小节的最后来讲。

#### 定位器：

Puppeteer 推荐使用定位器 API 选择元素并与之交互，定位器 API 会等待元素在 DOM 中处于可操作的正确状态。但是如果定位器 API 无法满足时仍可以使用低级别的 API，如：`page.waitForSelector()` 或 `ElementHandle`。

普通操作：

| 操作类型     | API 示例                                                     | 默认检查项目                                                 |
| ------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 点击元素     | `await page.locator('button').click();`                      | 1 确保元素位于视口中<br />2 等待元素可见或隐藏<br />3 等待元素启用<br />4 等待元素在两个连续的动画帧上具有稳定边界框 |
| 录入文本     | `await page.locator('input').fill('hello world');`           | 1 确保元素位于视口中<br />2 等待元素可见或隐藏<br />3 等待元素启用<br />4 等待元素在两个连续的动画帧上具有稳定边界框 |
| 鼠标悬停     | `await page.locator('div').hover();`                         | 1 确保元素位于视口中<br />2 等待元素可见或隐藏<br />3 等待元素在两个连续的动画帧上具有稳定边界框 |
| 滚动元素     | `await page.locator('div').scroll({ scrollTop: 10, scrollLeft: 20 });` | 1 确保元素位于视口中<br />2 等待元素可见或隐藏<br />3 等待元素在两个连续的动画帧上具有稳定边界框 |
| 等待元素可见 | `await page.locator('.loading').wait();`                     | 等待元素可见或隐藏                                           |

配置自检项：

```javascript
await page.locator('button')
  .setEnsureElementIsInTheViewport(false) // 禁用后无法保证操作前元素位于视口中
  .setVisibility(null)                    // 设置忽略操作前检查元素可见或隐藏状态
  .setWaitForEnabled(false)               // 禁用后无法保证操作前元素可用
  .setWaitForStableBoundingBox(false)     // 禁用后将不等待元素在两个连续动画帧上具有稳定边界框
  .click();
```

配置超时时间：

```javascript
await page.locator('button').setTimeout(5 * 1000).click();
```

PS：由于网页的响应速度存在差异，默认的超时时间不满足需要的情况下，可使用 `setTimeout()` 函数适当延长，超时时将抛出 TimeoutError 异常。

获取元素值或 `ElementHandle` ：

```javascript
// 使用 map 函数将元素映射为 JavaScript 值，调用 wait() 将返回序列化的 JavaScript 值
const enabled = await page.locator('button').map(el => !el.disabled).wait();

// 调用 waitHandle() 函数返回 ElementHandle
const buttonHandle = await page.locator('button').waitHandle();
await buttonHandle.click();
```

添加过滤器：

```javascript
await page.locator('button')
	.filter(el = el.innerText().includes('Click Me'))
  .click();
```

PS：通过过滤器来匹配所有按钮元素中符合特定文本的按钮元素。

添加事件监听：

```javascript
await page.locator('button')
  .on(LocatorEvent.Action, () => {
      console.log('clicked');
  }).click();
```

PS：目前定位器仅支持一个单独的事件，事件会在定位器准备执行动作前触发，以此表示所有前提条件已经得到满足。

自定义等待函数：

```javascript
await page.locator(() => {
    let resolve;
    const promise = new Promise((res) => {
        return (resolve = res)
    });
    const observer = new MutationObserver((records) => {
        for (const record of records) {
            if (record.target instanceof HTMLCanvasElement) {
                resolve(record.target);
            }
        }
    });
    observer.observe(document);
    return promise;
}).wait()
```

PS：示例中借助 **MutationObserver** 订阅 document 中出现 **HTMLCanvasElement** 元素的功能。

#### 等待选择器：

等待选择器（`waitForSelector`）与定位器相比是一个较低级别的 API，允许等待元素在 DOM 中可用。如果操作失败不具备重试特性，且需要手动释放生成 ElementHandle 以防止内存泄漏。

```javascript
import pprt from "puppeteer"

(async () => {
    const browser = await pprt.launch()
    const page = await browser.newPage()
    await page.goto("URL_ADDRESS")

    // 使用 waitForSelector 查询句柄
    const element = await page.waitForSelector("div > .class-name")
    await element.click();

    // 注意释放资源
    await element.dispose();

    await browser.close();
})()
```

#### 立即选择器：

在明确已知元素位于页面上时，可以直接使用立即选择器。

| API           | 描述                                                     |
| :------------ | -------------------------------------------------------- |
| page.$()      | 返回与选择器匹配的单个元素                               |
| page.$$()     | 返回与选择器匹配的多个元素                               |
| page.$eval()  | 返回与选择器匹配的第一个元素上运行 JavaScript 函数的结果 |
| page.$$eval() | 返回与选择器匹配的每一个元素上运行 JavaScript 函数的结果 |

#### 扩展选择器：

XPath 选择器（`-p-path`）：

```javascript
import pptr from 'puppeteer'
      
(async () => {
    const browser = await pptr.launch({ headless: false })
    const page = await browser.newPage()
    await page.setViewport({ width: 1080, height: 1024 })
    await page.goto('https://developer.mozilla.org/zh-CN/')
      
  	// XPath 选择器
    const textContent = await page.locator('::-p-xpath((//*[@class="tile-container"]/div/h3/a)[1])')
      .map(el => el.textContent)
      .wait()
    console.log(textContent)
      
    await page.screenshot({ path: 'screenshot.png' })
    await browser.close()
})()
```



Text 选择器（`-p-text`）：

```javascript
import pptr from 'puppeteer'
      
(async () => {
    const browser = await pptr.launch({ headless: false })
    const page = await browser.newPage()
    await page.setViewport({ width: 1080, height: 1024 })
    await page.goto('https://developer.mozilla.org/zh-CN/')
      		
  	// 文本选择器
    const textContent = await page.locator('::-p-text(Developer essentials: JavaScript console methods)')
    	.map(el => el.textContent)
    	.wait()
		console.log(textContent)
      
    await page.screenshot({ path: 'screenshot.png' })
    await browser.close()
})()
```



ARIA 选择器（`-p-aria`）：

```javascript
import pptr from 'puppeteer'
      
(async () => {
    const browser = await pptr.launch({ headless: false })
    const page = await browser.newPage()
    await page.setViewport({ width: 1080, height: 1024 })
    await page.goto('https://developer.mozilla.org/zh-CN/')
      
  	// 无障碍属性选择器
    const innerHTML = await page.locator('::-p-aria(MDN homepage)')
      .map(el => el.innerHTML)
      .wait()
    console.log(innerHTML)
      
    await page.screenshot({ path: 'screenshot.png' })
    await browser.close()
})()
```



Pierce 选择器（`pierce/`）：

```javascript
import pptr from 'puppeteer'
      
(async () => {
    const browser = await pptr.launch({ headless: false })
    const page = await browser.newPage()
    await page.setViewport({ width: 1080, height: 1024 })
    await page.goto('https://mdn.github.io/web-components-examples/composed-composed-path/')
        
  	// 穿透 shadow DOM 选择器，深度组合器，同 pierce/p
    const textContent = await page.locator('& >>> p')
    	.map(el => el.textContent)
    	.wait()
    console.log(textContent)
      
    await page.screenshot({ path: 'screenshot.png' })
    await browser.close()
})()
```



自定义选择器（如 `-p-class`）：

```JavaScript
import puppeteer, { Puppeteer } from "puppeteer"
      
(async () => {
    // 注册 class 选择器 
    Puppeteer.registerCustomQueryHandler('class', {
        queryOne: (node, selector) => {
            return node.querySelector(`.${selector}`)
        },
        queryAll: (node, selector) => {
            return [...node.querySelectorAll(`.${selector}`)]
        }
    })
      
    const browser = await puppeteer.launch()
    const page = await browser.newPage()
    await page.goto("https://developer.mozilla.org/zh-CN/")
      
    // 使用 class 选择器
    const innerHTML = await page.locator('::-p-class(tile-title)').map(el => el.innerText).wait()
    console.log(innerHTML)
      
    await browser.close()
})()
```



## 配置说明

Puppeteer 支持通过配置文件和环境变量两种方式来改变默认配置项，且环境变量的优先级要高于配置文件。配置文件支持的格式如下：

- .puppeteerrc.cjs
- .puppeteerrc.js
- .puppeteerrc (YAML、JSON)
- .puppeteerrc.json
- .puppeteerrc.config.js
- .puppeteerrc.config.cjs

配置文件使用示例：

```JavaScript
const { join } = require('path');

/**
 * @type {import('puppeteer').Configuration}
 */
module.exports = {
  	// 修改缓存目录后需要重新安装 Puppeteer，以保证新的缓存目录中包含的运行的必要文件
    cacheDirectory: join(__dirname, '.cache', 'puppeteer')
}
```

配置选项及对应的环境变量：

| 配置项                          | 类型                  | 环境变量                                      | 描述                                                         |
| ------------------------------- | --------------------- | --------------------------------------------- | ------------------------------------------------------------ |
| browserRevision                 | string                | PUPPETEER_BROWSER_REVISION                    | 指定浏览器版本号，默认值为当前 Puppeteer 内置的浏览器版本号  |
| cacheDirectory                  | string                | PUPPETEER_CACHE_DIR                           | 指定 Puppeteer 使用的缓存目录，默认通过 path.join(os.homedir(), ‘.cache’, ‘puppeteer’) 配置路径 |
| defaultProduct                  | chrome、firefox       | PUPPETEER_PRODUCT                             | 指定浏览器产品，默认为 chrome 浏览器                         |
| downloadBaseUrl                 | string                | PUPPETEER_DOWNLOAD_BASE_URL                   | 指定下载浏览器的前缀地址，不同的浏览器产品对应的下载路径不同：https://storage.googleapis.com/chrome-for-testing-public or https://archive.mozilla.org/pub/firefox/nightly/latest-mozilla-central |
| executablePath                  | string                | PUPPETEER_EXECUTABLE_PATH                     | 指定 [puppeteer.launch](https://pptr.dev/api/puppeteer.puppeteernode.launch) 启动路径，默认会自动查找安装路径 |
| experiments                     | Record<string, never> | --                                            | 指定 Puppeteer 的实验选项                                    |
| logLevel                        | silent、error、warn   | --                                            | 指定日志输出级别，默认为 warn 级别                           |
| skipChromeDownload              | boolean               | PUPPETEER_SKIP_CHROME_DOWNLOAD                | 安装 Puppeteer 时跳过 Chrome 下载                            |
| skipChromeHeadlessShellDownload | boolean               | PUPPETEER_SKIP_CHROME_HEADLESS_SHELL_DOWNLOAD | 安装 Puppeteer 时跳过 chrome-headless-shell 下载             |
| skipDownload                    | boolean               | PUPPETEER_SKIP_DOWNLOAD                       | 安装 Puppeteer 时跳过下载                                    |
| temporaryDirectory              | string                | PUPPETEER_TMP_DIR                             | 指定 Puppeteer 使用的临时文件目录，默认通过 os.tmpdir() 配置路径 |

PS：环境变量还包含 HTTP_PROXY、HTTPS_PROXY、NO_PROXY 用于定义下载和运行浏览器的代理设置。

## 调试说明

由于 Puppeteer 设计浏览器的许多不同组件，因此没有统一的方式调试所有的可能得问题，Puppeteer 尽可能的提供多种调试方法来涵盖所有可能得问题。

一般来说在使用 Puppeteer 的时候主要的问题来自两个来源：在 Node.js 上运行的代码（称之为服务端代码）和在浏览器端运行的代码（称之为客户端代码）。

### 基础配置：

因为调试往往发生在开发环境中，所以提供一个环境变量来动态启动调试的基础配置还是有很帮助的：

1. 禁用无头模式：可以查看浏览器显示的内容，主观的观察内容变化；
2. 延长执行时间：通过延长执行时间来观察正在发生的情况；
3. 启用浏览器调试：调试时会自动启动开发者工具；
4. 打印浏览器日志：启用后可以接管浏览器意外崩溃或无法正常启动时的日志信息。

```javascript
import puppeteer from 'puppeteer'

const production = process.env.NODE_ENV === 'production';

(async () => {
    const browser = await puppeteer.launch({
        // 开发环境中不使用无头模式
        headless: production,
        // 开发环境中延长执行时间
        slowMo: production ? 0 : 250,
        // 开发环境中打开开发者工具
        devtools: production ? false : true,
        // 开发环境中输出浏览器进程信息
        dumpio: production ? false : true,
    })
})()
```

### 客户端代码调试：

1. 捕获客户端代码中的 `console.*` 的输出：

   ```javascript
   // 监听页面的 console.* 输出 
   page.on('console', msg => {
       console.log('PAGE LOG: ', msg.text())
   })
      
   // 模拟浏览器环境中 console.* 输出
   await page.evaluate(() => {
       console.log('PAGE EVAL: ', 'Hello World!')
   })
   ```

2. 添加 `debugger;` 关键字中断代码：

   ```javascript
   // 注意启用 devtools 选项
   await page.evaluate(() => {
     	// 模拟客户端代码中使用 debugger; 关键字中断代码执行
     	debugger;
       console.log('PAGE EVAL: ', 'Hello World!')
   })
   ```

### 服务端代码调试：

在 Node.js 中使用调试器仅限于 Chrome 和 Chromium 中使用。

1. 在关闭无头模式的前提下，需要在运行服务端代码的脚本中添加 `--inspect-brk` 选项，如：

```shell
npm pkg set scripts.debug="cross-env NODE_ENV=development node --inspect-brk index.mjs" // v7.24.2 +
```

1. 在 Chrome 或 Chromium 中打开 `chrome://inspect/#devices` ，在新页面中的 Remote Target 菜单下找到对应的 Target 并启动调试。
2. 在新打开的浏览器中，按 `F8` 可以恢复测试执行；
3. 添加的 `debugger;` 关键字也会被命中并中断程序执行；

### 记录 DevTools 协议流量：

以上的调试方法都不起作用时，则可能是 Puppeteer 和 DevTools 协议之间可能存在着问题，那这时候可以通过设置 DEBUG 环境变量来进一步调试：

```shell
# 基本详细日志记录
cross-env DEBUG="puppeteer:*" node script.js

# 防止截断长消息
cross-env DEBUG="puppeteer:*" env DEBUG_MAX_STRING_LENGTH=null node script.js

# 协议通信可能相当繁杂。此示例过滤掉所有网络域消息
cross-env DEBUG="puppeteer:*" env DEBUG_COLORS=true node script.js 2>&1 | grep -v '"Network'

# 过滤掉所有协议消息，但保留所有其他日志记录
cross-env DEBUG="puppeteer:*,-puppeteer:protocol:*" node script.js
```

### 记录待处理的协议调用：

如果遇到 Puppeteer 异步任务未能变为 **Fulfilled** 状态时，可以尝试使用 debugInfo 借口记录被挂起的回调，并查看导致的原因：

```javascript
console.log(browser.debugInfo.pendingProtocolErrors);
```
