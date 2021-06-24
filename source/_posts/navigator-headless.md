---
title: 无头浏览器初探
date: 2018-08-30 15:00:39
tags: [前端技术, Headless-Chrome, Puppeter]
---
无头浏览器是浏览器一种新模式，该模式下用户可以完全通过命令行运行浏览器。前端早期对headless的支持实现是PhantomJS和selenium-webdriver，2017年谷歌宣布推出Chrome headless模式和Puppeteer，随后PhantomJS 和 Selenium IDE for Firefox 作者宣布停止维护各自的项目。（内心OS： 你叫我们和人家专业做浏览器的怎么比。。。。。）Chrome 从 v59开始支持无头模式。支持所有Chromium和Blink的所有功能。无头浏览器在自动化测试、页面爬虫、批量截图、检测网络性能等领域都有广泛的应用。

对于爬虫而言，以往的爬虫需要关注数据报文、页面结构或者模拟发包，有了无头浏览器之后，只需要关注用户交互即可。

对于SPA（单页面应用）而言，由于页面内容通常用JS动态生成，爬虫和自动化流程都相对复杂。使用无头浏览器之后，因为能够模拟出整个页面的渲染，不用担心页面的结构变化问题。

使用无头浏览器，甚至可以追踪页面性能，输出数据，方便开发人员进行分析。


# Chrome无头模式

## 环境准备

* MacOS下升级chrome到60以上

* alias配置：

```
alias chrome="/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome"
# Chrome canary版
alias chrome-canary="/Applications/Google\ Chrome\ Canary.app/Contents/MacOS/Google\ Chrome\ Canary"
# Chromium
alias chromium="/Applications/Chromium.app/Contents/MacOS/Chromium"
```

## 实验开始

1 基本功能

```
chrome --headless --remote-debugging-port=9222 --crash-dumps-dir=/tmp 
```

打开远程调试

```
chrome --headless --crash-dumps-dir=/tmp --dump-dom https://www.baidu.com
```

输出站点的dom结构，此时不能启用remote debug功能。

```
chrome --headless --crash-dumps-dir=/tmp --screenshot --window-size=1280,1696 https://www.baidu.com

```
输出站点的截图。

2 REPL mode (read-eval-print loop)
加上--repl参数后，可以在命令行里运行JS表达式（类似于浏览器的Console）

```
$ chrome --headless --repl --crash-dumps-dir=./tmp 
>>> location.href
{"result":{"type":"string","value":"https://www.baidu.com/"}}
```


# Puppeter

Puppeter是Chrome团队开发的一个Node libary，可以通过DevTools Protocol调用Chrome，默认运行于headless模式下，也可以通过配置修改运行于完整模式下。

## 环境准备

```
$ npm i puppeteer
```

下载puppeteer会默认下载新版的Chromium（体积很大）。如果已经安装，可以通过配置环境变量跳过这一步骤。

## 实验开始

为了体验puppeteer的功能，我写了个简单的脚本，实现了某discuz论坛的自动签到功能。

```javascript
const puppeteer = require('puppeteer');
// const devices = require('puppeteer/DeviceDescriptors');
// const iPhone = devices['iPhone 6'];
// 模拟移动设备

(async () => {
  const browser = await puppeteer.launch({
    // headless: false,
    // 关闭无头模式，打开浏览器页面，调试用
    // executablePath: '/path/to/Chrome',
    // 如果在安装puppeteer的时候跳过了安装chrominum，可通过指定改路径创建浏览器实例
    // slowMo: 250 
    // 减慢浏览器渲染速度slow down by 250ms
    timeout: 50000
  });
  const page = await browser.newPage();
  page.setDefaultNavigationTimeout(50000)
  page.on('console', msg => console.log('PAGE LOG:', msg.text()));
  // 捕获事件，获得console输出
  // await page.emulate(iPhone);
  // 模拟IPHONE 6
  await page.goto('http://HOSTNAME/forum.php', { waitUntil: 'networkidle0'});
  let loginres = await page.evaluate(() => {
    return !!document.querySelector('a[href="home.php?mod=space&amp;uid=USERID"]')
  })
  console.log('loginres')
  // 检查是否登录
  if (!loginres) {
    await page.click("a[href='member.php?mod=logging&action=login']", { delay: 200 });
    await page.waitForSelector('input[name=username]')
    console.log('get loginres')
    await page.type('input[name=username]', 'USERNAME');
    await page.type('input[name=password]', 'PASSWORD');
    await page.click('.pc', { delay: 200 });
    const navigationPromise = page.waitForNavigation({ waitUntil : 'networkidle0' })
    await page.click('button.pn.pnc',  { delay: 200 });
    // login
    await navigationPromise;
  }
  // sign
  await page.goto('http://HOSTNAME/plugin.php?id=XXXXsign:sign', { waitUntil: 'networkidle0'});
  await page.click('li#kx')
  await page.type('input[name=todaysay]', '签到啦')
  await page.click('img[src="source/plugin/XXXXsign/img/qdtb.gif"]')
  await page.waitForNavigation({ waitUntil : 'networkidle0' })
  console.log('finish')
  await browser.close();
})();

```
相比于模拟发送报文的自动签到插件效率确实低了点，但是实现更简单直观不是吗（doge脸）

官网调试技巧，可以关掉浏览器的headless模式，这样调试更直接。


# 问题总结

1 MacOS下执行指令之后setxattr报错：`Operation not permitted`

在指令中增加参数`--crash-dumps-dir=/tmp`即可。初始化时执行setxattr org.chromium.crashpad.database.initialized on file /var/folders/xxxxxxxxx，
因为新版MacOS增加了SIP(System Integrity Protection)功能，导致许多目录如/var目录无法读写。通过设置目录到/tmp可解决，也可以通过关掉SIP解决该问题。

参考链接：

https://stackoverflow.com/questions/49103799/running-chrome-in-headless-mode


https://superuser.com/questions/1292863/chrome-crashpad-crashes-on-xattr/1307573

2 Puppeteer 中 evaluate 执行一个script脚本注入的全局函数报错。

```
Evaluation failed: TypeError: 'caller', 'callee', and 'arguments' properties may not beaccessed on strict mode functions or the arguments objects for calls to them
```

全局函数里有arguments的调用，puppeteer 的 evaluate 默认函数都是严格模式，不能调用caller、arguments、callee。（因为这个函数内部实现对外隐藏，最开始对puppeteer不熟悉，试了一阵子才反应过来是函数本身的问题。。）

3 Puppeteer 中的 evaluate 执行需要的所有参数都只能通过evaluate的参数传入，不能获得外面的变量。

```
  // page.evaluate(pageFunction, ...args)
  // 参数只能通过args传入，结果只能通过pageFunction的return传出
  // 例子
  const result = await page.evaluate(x => {
    return Promise.resolve(8 * x;
  }, 7)
  console.log(result); // prints "56"
```


# 参考文档

headless-chrome

https://developers.google.cn/web/updates/2017/04/headless-chrome

devtools-protocol

https://chromedevtools.github.io/devtools-protocol/

puppeteer API 文档(1.7)

https://github.com/GoogleChrome/puppeteer/blob/v1.7.0/docs/api.md

# 附注

调试http接口：
```
获取当前所有可调式页面信息
http://127.0.0.1:9222/json

获取调试目标 WebView/blink 的版本号
http://127.0.0.1:9222/json/version

创建新的 tab，并加载 url
http://127.0.0.1:9222/json/new?url

关闭 id 对应的 tab
http://127.0.0.1:9222/json/close/id

切换到目标Tab
http://127.0.0.1:9222/json/activate/69301801-d503-42a3-9335-3e448a780857 : 
```
