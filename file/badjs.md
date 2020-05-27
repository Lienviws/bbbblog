# 你知道自己的代码在线上有多少问题吗

> badjs，即前端异常的一个洋气的统称。指代那些「找不到对象」、「未定义」、「语法问题」等在前端抛出来的异常错误。

## 前言

本人所在的前端业务长期受到大量异常的困扰，又常常找不到原因。有时异常一下暴涨，又降了回去，定位不到问题，深受其扰。经过长时间的沉淀，分析总结出了一套结论和方法。

本文前半部分讲述了怎么收集和分析 badjs 以及为什么要做这件事，适合于没有系统接触过 badjs 的同学了解。后半部分对于一些异常信息，比如 Script error 做了较为详细的攻防介绍，另外也简单的介绍了一下后台的瓶颈之处，供有相同体会的同学参考和讨论。

希望对于屏幕面前的你能有所启发。

ps：此系列方法不适用于node.js

## 京东社交电商部的 badjs

先来看下这张图片：

![badjs](https://tva1.sinaimg.cn/large/007S8ZIlgy1gewsv2g4igj30e00army0.jpg)

这是我所负责的一个线上业务的 badjs 走势。不知道你看到这根刺是什么感觉，反正我看到是会非常紧张，不论手上有什么事都得立马扑向电脑检查问题，分析日志，跟老板汇报起因...

### 为什么我们会有这个

俗话说，技术服务于业务。我们的 badjs 系统诞生于必然。

以微信小程序商品详情业务为例。平日的流量峰值是2w/min，日pv有千万。

假如前端出了问题，有啥东西点不动，导致访问的用户变少了。最后的结果是单量少了，用户丢了，还影响了整个部门同学的饭碗。这个锅，背不起。

面临这些问题，试问一下，你要上线的时候怕不怕？

为了程序员的幸福生活，提前发现问题，把黑锅扼杀在摇篮中，这样一个系统是必须的选择。

## badjs原理和收集

我们无法预测哪一段代码会出问题，成本最小的方案是在一个集中的地方统一处理，然后收集起来。

### badjs 的由来

badjs 实质是 js 引擎执行了无法识别的逻辑，出现了异常。

比如在 undefined 上进行操作、用错了数据类型、解析异常、语法错误等：

![花式错误](https://tva1.sinaimg.cn/large/007S8ZIlgy1gf60yo87dbj30ib09twev.jpg)

### badjs的收集

当 JavaScript 运行发生异常时，并且未被捕获，`window` 下会触发一个 `ErrorEvent` 接口的 `error` 事件，并执行 `window.onerror()`

我们可以通过以下两种方式在全局处理异常：

```js
window.addEventListener('error', function(errorEvent) {
    const { message, filename, lineno, colno, error } = errorEvent
    ...
})
```

```js
window.onerror = function(message, source, lineno, colno, error) { ... }
```

这两种方式在写法上前者更优一点，因为 `error` 事件可以监听多个，`window.onerror` 只能订阅一个。

可惜 ie8 以前不支持 `ErrorEvent`，只能使用 `window.onerror`。 所以看业务需要合理使用。

另外还有 `try...catch` 的方式，也可以收集到特定场景下的全局错误信息，但这个方式有很多缺陷，关于它我们后面再谈。

通过 window 下的 error 事件，我们正常情况下能收集到五类信息，分别是「`message`: 错误信息」、「`filename(source)`：发生错误的脚本URL」、「`lineno`: 错误行」、「`colno`: 错误列」、「`error`: Error对象」

如图所示：

![error信息](https://tva1.sinaimg.cn/large/007S8ZIlgy1gexze85jhlj30hp06774b.jpg)

有了这些，就已经给了我们相当充分的信息来定位问题了。

这里报错的信息有一点挺有意思。message 里会带上 'Uncaught' 表示未捕获。而 error 内没有这个前缀词：

![Uncaught](https://tva1.sinaimg.cn/large/007S8ZIlgy1gf70kv267bj309r02mdfo.jpg)

以上说到的是理想场景，真实环境复杂很多，只能收集到一些不太有意义的信息。

![跨域error信息](https://tva1.sinaimg.cn/large/007S8ZIlgy1gexzhsszbxj3042053t8i.jpg)

这种是由于脚本跨域的问题导致的。这个问题文章后半部分会深入的谈谈。

window 下的 error 事件也并不是什么异常都能捕获的。比如我们直接在 console 里打入的错误、浏览器报的跨域错误、资源 404 等都不会触发。

**小程序badjs**

上面是html系的前端错误收集，小程序的有些差异。以微信小程序为例，

小程序的全局异常捕获，可以在`App`注册小程序的方法下订阅。[相关文档点这里](https://developers.weixin.qq.com/miniprogram/dev/reference/api/App.html#onError-String-error)

```js
App({
  onLaunch (options) {
    // Do something initial when launch.
  },
  ...,
  onError (msg) {
    console.log(msg)
  }
})
```

这里不太一样的是，onError订阅函数里只有一个`msg`参数。内容类似`error.stack`

![msg](https://tva1.sinaimg.cn/large/007S8ZIlgy1gez0s3o5usj30ze05ut95.jpg)

### badjs上报

由于依靠上报的信息定位问题，需要确定哪些信息是需要的。另外考虑到数据量的问题，只上报一些必要的信息。

**前端上报数据**

类别 | 来源 | 示例
- | - | -
stack | error.stack | "ReferenceError: test is not defined    at HTMLLIElement.<anonymous> (https://wq.360buyimg.com/wecteam/badjstest.js:8996:115)    at HTMLUListElement.delegator (https://wq.360buyimg.com/wecteam/badjstest.js:2894:43)    at HTMLUListElement.handler.proxy (https://wq.360buyimg.com/wecteam/badjstest.js:2729:31)";URL:https://wq.360buyimg.com/wecteam/badjstest.js;Line:8996;Column:115
其他必要的自定义数据 | / | balabalabala

另外有些数据是包含在报文当中的，可以在服务端一起收集和展示，前端不需要再单独上传

**服务端数据**

类别 | 来源 | 示例
- | - | -
ip | / | 58.20.191.9
time | / | May 20th 2020, 14:56:00.062
referer | Headers.Referer | https://wqsou.jd.com/wecTeam?key=%E9%9B%B6%E9%A3%9F&sf=14&as=0&version=V00--
UA | Headers.User-Agent | Mozilla/5.0 (iPhone; CPU iPhone OS 13_3_1 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Mobile/15E148 MicroMessenger/7.0.12(0x17000c2d) NetType/4G Language/zh_CN
user(用户信息) | Headers.Cookie | wecTeam

**数据上报**

前端把必要的数据拼接起来，大概长这个样子

https://wq.jd.com/wecteam/badjs?stack=%22ReferenceError%3A%20test%20is%20not%20defined%20%20%20%20at%20HTMLLIElement.%3Canonymous%3E%20(https%3A%2F%2Fwq.360buyimg.com%2Fwecteam%2Fbadjstest.js%3A8996%3A115)%20%20%20%20at%20HTMLUListElement.delegator%20(https%3A%2F%2Fwq.360buyimg.com%2Fwecteam%2Fbadjstest.js%3A2894%3A43)%20%20%20%20at%20HTMLUListElement.handler.proxy%20(https%3A%2F%2Fwq.360buyimg.com%2Fwecteam%2Fbadjstest.js%3A2729%3A31)%22%3BURL%3Ahttps%3A%2F%2Fwq.360buyimg.com%2Fwecteam%2Fbadjstest.js%3BLine%3A8996%3BColumn%3A115&exts=balabalabala


数据传输可以用`get`方式即可。更简单一些的用`img`传递。

```js
var _img = new Image();
_img.src = url;
```

需要注意的是，`get`方式因为拼在`url`上，有最大长度限制，注意裁剪。如果要不限长度，请用`post`方式上报。

**数据清洗**

真实的环境下会更复杂一些。因为一旦发生`badjs`，数据量可能会非常大，形成井喷，可能会拖挂数据服务器。但是也不能不上报，因此需要对上报的数据进行一层预处理。

1.去除重复错误
真实场景下大多是因为某个事件里面的代码出现badjs，比如点击按钮。当用户重复点击的时候，就会重复出现badjs。这里可以提前筛掉重复上报的数据，可以节省很多数据的存储量。

2.合并数据(延迟上报)
这一步目的也是为了减少上报量和节省存储数据。另外如果在同一个user view下出现多个错误，也方便分析问题相互的关联。

### 实例：bj-report.js

可能你会问：道理我都懂，但我不想努力了，有没有现成的工具？

有。

推荐这块鹅厂出品：bj-report.js [点击这里进入git主页](https://github.com/BetterJS/badjs-report.git)

## 数据分析

说完了数据上报，再说一下数据可视化系统，以及数据的分析。从数据中挖掘出来有用的信息也是一门学问。很多时候需要经验，有时候还需要一些直觉。

### 可视化系统

先看下京东社交电商部平时使用的监控系统：

![kibana系统](https://tva1.sinaimg.cn/large/007S8ZIlgy1geyztuwuiij31fz0u0wna.jpg)

我们本身有badjs的数据块，然后大数据同学基于 ElasticSearch， 用 kibana 搭建了这个日志可视化分析系统。

kibana 是一个比较专业的日志分析系统，使用起来根据多维度来查询信息。

需要省事一些的话，也可以搭建一个简单的报表系统：

![简单的报表系统](https://tva1.sinaimg.cn/large/007S8ZIlgy1gez1jthn4tj32940qi75u.jpg)

搭建的过程不属于本篇主题，这里就不介绍了。关于数据方面，大家可能会关注一些点，在后面会专门来介绍一下。

### 错误分析

怎么用这些数据去寻找错误的原因。这里简单介绍一下。

需要特别说明的是。badjs分两种，一种是因为代码缺陷而导致的，俗称bug。还有一种是人为故意的，比如刷子、爬虫、浏览器插件等嵌入的第三方的脚本，触发了本不存在的异常。我们需要特别关注前者，并尽可能的把它和后者区分开来。

**content信息**

content信息即`Error.stack`，里面是错误的堆栈信息。

一段完整的stack信息，包含了错误代码堆栈，文件和行列号。通过这些信息我们基本可以断定错误的位置和触发原因。

比如这一段 stack 信息：

![badjs](https://tva1.sinaimg.cn/large/007S8ZIlgy1gf714ewntvj30ca01mglg.jpg)

我们一行行来看：

`Uncaught TypeError: Cannot read property 'style' of undefined`

无法在 `undefined` 下访问 `style` 这个属性。大概场景就是说在某个对象下，某个属性空了，在这个基础上又访问 `style` 这个属性，因此报错了。

`at removeAD (badjs.js:4)`

该异常代码位于`badjs.js` 这个文件里的第 4 行，所属的方法名叫做 `removeAD`。

`at window.fireBadjs (badjs.js:8)`

调用 `removeAD` 的代码位于 `badjs.js` 第 8 行，所属的方法名叫做 `window.fireBadjs`。

`at badjs.html:16`

调用 `window.fireBadjs` 的代码位于 `badjs.html` 这个文件的第 16 行。

有这么详细的信息，对照源码查一下，对于 badjs 的原因基本就心知肚明了。

还有一种比较特别的content信息：

![anonymous](https://tva1.sinaimg.cn/large/007S8ZIlgy1gf71dw840cj30bx00y3yb.jpg)

发生异常的位置很奇怪，第一行第一列。但是我们没有什么代码是写在第一行第一列就有问题的。

这个其实是在浏览器的匿名函数空间中执行的方法，类似直接打在 console 中的代码，或者通过 `eval` 函数运行的代码。

上面图片里的这个 badjs 出现的背景，是 App 的 native 用 js 代码在 webview 里直接执行了一个 window 作用域下的 `callback`。但是这个函数不存在，所以报错。

有了上面这个基础，再来看看这个错误。

![](https://tva1.sinaimg.cn/large/007S8ZIlgy1gf3mzjgiusj31z40kqq84.jpg)

先看错误内容，大概知道是在匿名空间下访问了一个名为 SOHUZ 的属性，但是因为作用域链上这个属性不存在，所以报错了。又因为是在第一行第一列，我们代码里面没有这个，因此推断多半是某个 App 主动执行的 js 代码。

进一步的分析需要结合 ua 的信息。ua 里有 `kuaizhanAndroidWrapper` 这个字段，通过万能的互联网，发现"快站" app 的身份信息里包含这个字段。

因此结论出来了："快站" app 里访问了这个页面，它可能因为没有进行非空检查，直接访问了 `SOHUZ`，导致 badjs。对我们而言，是非错误。

**ua**

在上面一段的结尾提前介绍了下 ua 信息的作用。

我们能通过 ua，得知运行环境：比如微信里、某app，QQ浏览器等，还有用户标识：比如百度爬虫、阿里百川、恶意爬虫等。

ua还可以作为简单的用户指纹。笔者所负责的页面会经常被不知名的网友刷，在内部我们统一称之为：刷子。有时我们会遇到大量奇怪的报错，但是背后的 ua 都是一模一样的，基本可以认定是有网友在刷我们的的页面。

需要强调的一点是，ua可以伪造。很多有经验的刷子，每次的 ua 是通过生成器创造的，无法判断。因此 ua 信息是否真实，具体情况具体分析。

**script error**

还有一种更真实的情况，是有大量的 script error 信息。

![script error](https://tva1.sinaimg.cn/large/007S8ZIlgy1gf72ozscp6j31ao0git9x.jpg)

出现 script error，是因为引入了跨域脚本。比如通过 script 标签引入的其他非同域 js 文件，里面发生了 badjs，这时在 onerror 事件里的 message 信息就只有一句 `script error`，error 信息为 `null`

![跨域error信息](https://tva1.sinaimg.cn/large/007S8ZIlgy1gexzhsszbxj3042053t8i.jpg)

对于这些已经上报了的 script error 数据，能得到的数据非常有限，大多数时候只能放弃分析。能做的只有尽量在上报的时候就打开 script error 里的信息，再上报它。比如跨域脚本设置信任策略。

关于 script error 详细的应对策略，我们将在下面的"进阶"章节里进一步的讨论它。

## 进阶

开发人员往往在自己的象牙塔内进行改造和升级，但是真实的生产环境往往比预想的更复杂。各种各样背景的浏览器、 webview，还有非正常途径访问的流量、防不胜防的script error以及让人头疼的网络环境。

window 下的 error 事件并不是一个银弹。有一些特殊场景的错误它并不能抓到，但不代表这些抓不到的 badjs 不重要。可能在未来的某一天，运营同学找上来问你为什么数据不对、表现奇怪，你抓耳挠腮查了一整天问题所在，最后才发现来源是一个没有捕获到的错误。

这一节我们先讨论一下，怎么去尽可能的覆盖那些潜在的问题。最后再谈谈跟后台相关的一些小事。

### script error

文章之前的内容卖了不少关子，这一段深入一下，讨论怎么能挖出更多有效的信息。

**来源**

script error 本质是因为浏览器的跨域安全策略行为，保护非同域代码内容的安全。

所有通过 script 标签引入的跨域脚本，如果出现异常，window 下的 error 事件都只能得到 'script error'。

![script error](https://tva1.sinaimg.cn/large/007S8ZIlgy1gf3jw5io8cj30zk0b60t1.jpg)

**常规解决方案**

对于 script 标签引入的跨域脚本产生 'script error' 的问题，业内有解决方案。

大概思路就是：不就是安全问题嘛，只要双方都觉得可信，那我浏览器就给你放开这个限制。

具体实现如下：

1.response 头增加 Access-Control-Allow-Origin，表明受信任的域名

![Access-Control-Allow-Origin](https://tva1.sinaimg.cn/large/007S8ZIlgy1gf7ax5bmndj30a306ajre.jpg)

2.请求的 script 标签增加 crossorigin 属性

![crossorigin](https://tva1.sinaimg.cn/large/007S8ZIlgy1gf5xxx4bhaj30qb00lwec.jpg)

crossorigin 有两个值，分别是

- anonymous
- use-credentials

当 `crossorigin=''` ，或者其他字符，和设置 anonymous 的效果一样。anonymous 依赖 response 头的 Access-Control-Allow-Origin。需要注意的是，使用 anonymous 时，request 头不会带上用户信息，比如 cookie。相对的，使用 use-credentials 可以带上用户信息。

use-credentials 需要和 response 头的 Access-Control-Allow-Credentials 配合使用，当 Access-Control-Allow-Credentials 返回 true 的时候，浏览器才会允许运行脚本和信任来源。另外在这种情况下， Access-Control-Allow-Origin 不能设置为 `*`，必须是具体的域名。

![Access-Control-Allow-Credentials](https://tva1.sinaimg.cn/large/007S8ZIlgy1gf7atqgwqkj30am05m0sq.jpg)

通过以上的两步，web 浏览器下的 window error 事件里的 error 信息就不会被浏览器拦截成 'script error' 了。

**一个变种场景: jsonp**

这里提一个 script 标签的变种场景：jsonp

jsonp本身解决的问题就是跨域接口请求，因此大部分使用场景自带跨域光环。另外它通过 script 标签运行，和 js 脚本的性质一样。

因此当它出现了异常，也会存在 script error 的情况。

jsonp 出现异常的常见的场景就是：callback 未定义

![callback 未定义](https://tva1.sinaimg.cn/large/007S8ZIlgy1gf7b4srscdj30cf00x3yc.jpg)

虽然可以在 console 看到具体错误信息，但是在被捕获的 error 中，因为 url 跨域，没额外设置信任，只能抓到 Script error。

解决方式就是按照「常规解决方案」里列举的两个步骤去"解构" Script error。

这里有个问题在于大部分接口依赖用户信息，前端需要使用 `crossorigin='use-credentials'` 方式将请求带上 cookie 信息。因此后台在 Access-Control-Allow-Origin 中要返回具体的域名，以及将 Access-Control-Allow-Credentials 设置为 true 以让浏览器通过验证。

![Access-Control-Allow-Credentials](https://tva1.sinaimg.cn/large/007S8ZIlgy1gf7atqgwqkj30am05m0sq.jpg)

![use-credentials](https://tva1.sinaimg.cn/large/007S8ZIlgy1gf5xlzui4fj30le00xwee.jpg)

额外提一点，jsonp 产生的脚本绝大部分是非异步代码。跨域脚本异步代码有一些坑，后面会介绍。

**非常规解决方案**

上面的是常规解决方案。还有一种非常规的方法，就是使用 `try...catch`。

举个简单的栗子：

```js
// https://xxx.badjs.js
window.fireBadjs = function () {
    ops
}
```

```html
<script src="xxx.badjs.js" ></script>
<script>
    window.addEventListener('error', function (errorEvent) {
        const { message, filename, lineno, colno, error } = errorEvent
        debugger
    })
    fireBadjs()
    try {
        fireBadjs()
    } catch (error) {
        debugger
    }
</script>
```

badjs.js 里面往 window 下塞了一个名为 `fireBadjs` 的函数，里面运行了一行会出异常的代码。

第一个 `fireBadjs()` 运行后，`error` 事件触发，内容如下：

![script error](https://tva1.sinaimg.cn/large/007S8ZIlgy1gf60960oitj30dn04mwef.jpg)

情理之中，error 事件只能抓到 Script error。

注释掉第一个 `fireBadjs()`，让在 `try...catch` 中的 `fireBadjs()` 执行，badjs 被 catch 捕获。打在 console 里，它的内容如下：

![错误信息](https://tva1.sinaimg.cn/large/007S8ZIlgy1gf60c39srxj30l40ai3yo.jpg)

神奇的是，原本是 Script error 的内容被挖了出来。

但这种方法终归不优雅，对代码侵入性较强。而且还有个小缺陷，就是不会触发 window 下的 error 事件。

接下来我们谈谈怎么改善。

**try catch 增强**

`try...catch` 是一个让人又爱又恨的工具。很多时候我们为了让代码有好的健壮性，会用它包裹起来。

但是一旦在 `try` 中发生了错误，浏览器不会把错误打在 console 里，也不会触发 error 事件。

来看一段简单的代码：

```js
try {
    JSON.stringify(apiData)
} catch (e) {}
```

如果 JSON.stringify(apiData) 发生了错误，代码可能是运行正常，但是最后表现没有达到预期，我们也不知道这里有问题。最后可能绕了一个大圈子才回到这里。

这里的问题出在异常被捕获以后，没有给予我们提示。那解决方案也很简单，手动补上提示就好。

这时可以在 `catch` 里，把错误打在 `console.error` 里面，并手动包装 `ErrorEvent`，丢给 window 下的 error 事件捕获。

```js
try {
    JSON.stringify(apiData)
} catch (error) {
    console.error(error)
    if (ErrorEvent) {
        window.dispatchEvent(new ErrorEvent('error', { error, message: error.message })) // 这里也会触发window.onerror
    } else {
        window.onerror && window.onerror(null, null, null, null, error)
    }
}
```

需要额外注意的是，`try...catch` 有个缺点，不能捕获异步代码的异常。比如 `setTimeout`、`promise`、事件等。因为这个问题，我们无法在代码里从头到尾简单的包裹一层 `try...catch` 解决所有问题。

**hybrid-webview**

很多 app 里面都会内嵌 webview 运行 h5 页面。如果 h5 在 webview 里面发生了 badjs，各位猜一猜，webview 的表现是什么样的？

这里分两个环境 iOS 和 Android。

1.iOS系统 (todo:ios12/13之外的版本呢？)

在 iOS 中的 webview，跨域脚本的异步代码如果发生了badjs(注意是异步代码)，不管有没有按照常规方案去设置跨域头和 crossOrigin，在 window 下的 error 事件里都只能抓到 'script error'。

你没看错，只要是 iOS 里的 webview，不论是你业务下的 app，还是微信甚至是浏览器。

举个栗子：在iPhone的微信里打开了某个 h5 页面，h5 页面引用了某 (跨域)cdn 的 js 文件。当这个 js 文件内部某个事件产生 badjs 并上报了，我们在日志系统只能看到 'Script error'。

2.Android系统

在 Android 中的 webview，和web端一样的表现。设置好了跨域头和 crossOrigin，如果跨域脚本发生 badjs，不论异步同步代码，都能在 window 下的 error 事件里抓到错误详情。

拿一个笔者负责业务的线上的数据，简单验证下这个现象：

一个绝大部分流量跑在 微信 和 手机QQ 里的业务，在 Android 环境下，有 8043 个 script error

![](https://tva1.sinaimg.cn/large/007S8ZIlgy1gf4svdxr4vj31mi070aab.jpg)

在 iOS 环境下有 25685 个 script error

![](https://tva1.sinaimg.cn/large/007S8ZIlgy1gf4sx081lvj31fc06u0sx.jpg)

顺便一提，iOS 下的非 script error 只有 393 个。

解决方法，主要是针对 iOS 环境。因为 Android 表现和 web 端一致。

1.跨域脚本改为同域脚本。

同域脚本报错不会显示 Script error。

2.try...catch 包裹

把跨域的脚本里的异步方法用 try...catch 包裹一下，在 catch 中手动触发事件。手动包裹比较麻烦，可以考虑用工具打包的时候自动包裹一下。

3.用 Android 的 badjs 数据参考 iOS

因为两个环境出异常的概率近似相等，在做好跨域工作以后，专注解决 Android 下的 badjs，就可以覆盖绝大部分 iOS 的异常了。

**hybrid-代码注入**

这里和上面一节的区别是，上一节是 h5 在 app 里"自由落体"，这一节是由 app 主动干涉。

app native 往 webview 注入代码有两种方式。第一种是 native 代码转换成 js 代码，再发送到 webview 通过 js 引擎执行。第二种是用 c++ 代码，直接写在引擎的底层，成为 native code。

第一种方式实际上还是通过 webview 环境执行，和 js 代码无异。常见于在 JSSDK 里的一些 `callback` 参数。

![](https://tva1.sinaimg.cn/large/007S8ZIlgy1gf50yfxpecj30pw06wweh.jpg)

app 一般会为了避免逻辑过于复杂，通常就是往 webview 里直接执行 `window.callback`，如果 callback 不存在，出了异常，Android 中和 web 端表现一致。

但是 iOS 里只有 Script error。

如果你在日志里看到处于 anonymous 下第一行第一列的报错，并且 ua 是 app 环境。那很有可能是由 app 主动的调用产生的 badjs：

![app 主动调用](https://tva1.sinaimg.cn/large/007S8ZIlgy1gf5vtiq0k0j30wm03v74p.jpg)

Android 没什么好说的，iOS 在 content 这里则会显示 Script error。由于代码是 app 生成发送到 webview 当中的，因此前端没办法做什么去解构这个 Script error，只能跟 app 的同学商量做一层保护，比如调用前先检查函数是否定义过了。

第二种用 c++ 代码，直接写在引擎的底层的方式，常见于 native 生成一段供 h5 使用原生 app 能力的接口，比如 jssdk。这种方式生成的代码会成为 native code。

native code 的示例：console.log

![native](https://tva1.sinaimg.cn/large/007S8ZIlgy1gf50qc86ebj305200qmwx.jpg)

由于运行在引擎当中，一旦出错，webview 可能整个都挂了。前端 badjs 事件对于这种情况是无能力为的，可以忽略。

**iframe**

iframe里面的代码badjs，根页面是抓不到的。这个策略很好理解，毕竟无法确认 iframe 里的内容。

解决方式很简单，在子 iframe 里面同样绑定它的 window 下的 error 事件即可。

### 后台建设

大部分同行使用 badjs 日志系统最大的阻碍来自于后台的建设。根据 logstash 业界标准日志采集流程，前端只能完成第一步：采集。对于储存和展示是无能为力的，需要后台同学的介入。

这里介绍一下笔者所在的京东社交电商部有关的一些背景，以供参考。

**数据量级**

我们的日志数据除了 badjs，还有一些前端主动上报的业务日志信息。因为部门里的业务非常多，上报流程也比较规范，上报数据随着不断增多的业务节节攀升。现在每天的数据大概24亿条(2.5T)。筛掉业务上报，纯 badjs 大概只占了 0.67%。

由于我们有大数据机器的资源，因此这些信息跑在我们自己的数据机器上，没有额外开销。

**性能优化**

虽然机器资源比较足，但是也撑不住无节制的使用。我们有一些性能优化方案，保证日志服务稳定。

- 数据每5天清除一次
  - 数据定期清除，保证储存空间良性循环
  - 过期的日志不再提供查询能力
- 上报降级方案
  - 前端有接入配置，当数据井喷，自动调整采样率，放弃一定比例的上报
  - 后台也有类似配置，数据量过高，自动放弃一些数据

**第一版数据收集系统：webmonitor**

我们最初的日志系统，文件存在 hdfs 当中。使用 url 作为关键字进行查询 badjs。

![简单的报表系统](https://tva1.sinaimg.cn/large/007S8ZIlgy1gez1jthn4tj32940qi75u.jpg)

当发起查询时，使用 impala 暴力扫描 hdfs。效率比较低，查一次大概半分钟。

这个系统没有更多的筛选可以选择，展示方式也十分粗糙。

**第二版数据收集系统：ElasticSearch+Kibana**

第二版的日志系统，大数据同学接入了 ElasticSearch，建立了日志文件的索引，然后通过 Kibana 和它无缝对接。因为有了索引，查询速度快了；通过 Kibana，查询门槛低了，也有更多维度数据的分析了。

![kibana系统](https://tva1.sinaimg.cn/large/007S8ZIlgy1geyztuwuiij31fz0u0wna.jpg)

查询速度的提高，对快速定位和回应上级的效率有了质的飞跃。

我们可以根据关键词进行查询，筛掉无用的，或者选择感兴趣的信息。

也可以方便的选择某个数据集中的区间，很快的分析问题和提出解决方案。

从未感觉作为开发人员的幸福感可以如此之高。

## 小结

### 收集 badjs 的原理

- 通过 `errorEvent` 或者 `window.onerror`

### badjs 上报

- 在 url 上拼接 error.stack 的信息
- 用 img 等方式发送数据

### 数据分析

- 通过栈信息，人工回溯代码
- 通过 ua 简单判断环境和用户身份
- 对于 script error 信息，得不到太多有用的价值，分析的时候可以跳过。但是要想办法在上报的时候把它"解构"

### 常见引起 script error 的场景

- 未通过 crossorigin 跨域脚本内的 badjs
- 未通过 crossorigin 的 jsonp 通过 script 生成的 js 代码里的 badjs
- iOS 下的跨域脚本异步代码内的 badjs
- iOS 下 native 主动执行 js 代码的 badjs

### 关于异步代码需要注意的点

- ios 下的跨域异步代码无法解析 script error
- try catch 无法抓到异步代码的错误
- jsonp 大多是同步代码

## 必要性

最后回过头聊一聊有关这个系统的必要性。

对于笔者的业务，这么一套系统是必须的。因为安全非常重要，我们根本无法承担较长时间，线上出问题后的责任。

可能有的同学就要问了：“我们业务没有这个东西，况且又依赖那么多资源，我根本没法拿主意啊”。

下面稍微分析一下它的优势和缺陷。

### 优势

不确定自己的代码有没有问题，是一件非常不安的事情。要是有客户用户投诉过来，那真是一个头两个大。

绝大部分前端的业务代码都是经过测试把关的。如果通过了充分测试，线上代码业务逻辑出问题的概率比较低，所以关注点集中在有没有代码报错。

这么一个系统的出现，很大程度上增加了前端同学的自信心。发版之后，出了问题，能够第一时间回退代码，定位问题。最重要的是给了老板们**安全感**。

### 缺陷

然而这样一个系统不是万金油，还有很多其他维度的问题无法用它衡量。如果太过于依赖，没有充分测试业务，上线的时候就看了一眼 badjs 日志就算完事了，那线上可能会有很多头大的问题等着你发现。

另外这个资源的开销也是一个不得不提的问题。如果公司没有足够的资源，就需要额外的开销在租借服务器和存储设备上。

### 怎么选择

很多在处于成长期的业务可能根本没精力去做这些基础建设，但我相信在未来的某个时间，你会迫切的需要这些数据。

等到需要的时候，不妨回过头来看看这篇文章。

如果出现问题带来的损失大于建设成本和维护成本，我相信你一定能说服你所在的团队和你的老板。

## 参考资料

- [前端越管越宽，腾讯Now直播如何把监控体系做到极致？](https://mp.weixin.qq.com/s/aqO55IyVCZzh9yhKuOKSCQ)
- [Script Error规范](https://github.com/BetterJS/badjs-report/issues/3)
