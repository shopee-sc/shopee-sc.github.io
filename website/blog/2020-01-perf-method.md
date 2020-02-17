---
title: 前端性能优化之自定义性能指标及上报方法详解
author: Winty zhou
authorURL: https://github.com/LuckyWinty
---
性能优化是所有前端人的追求，在这条路上，方法多种多样。这篇文章，说一下利用浏览器的一些API，可以怎样进行自定义性能指标及上报。
<!--truncate-->

### 自定义性能指标介绍
自定义性能指标这里，主要要介绍的是 `Performance 接口`,这个接口可以获取到当前页面中与性能相关的信息。主要包含了Performance Timeline API、Navigation Timing API、 User Timing API 和 Resource Timing API。

`Performance`类型的对象可以通过调用只读属性 Window.performance 来获得,截止目前，其支持度已经很高了，支持性如下：

![GitHub](https://raw.githubusercontent.com/LuckyWinty/blog/master/images/perf/WechatIMG21363.png)

### performance.now() 
在chrome浏览器中返回的时间是以毫秒为单位的，更精确。

performance.now() 与 Date.now() 不同的是，返回了以微秒（百万分之一秒）为单位的时间，更加精准。

并且与 Date.now() 会受系统程序执行阻塞的影响不同，performance.now() 的时间是以恒定速率递增的，不受系统时间的影响（系统时间可被人为或软件调整）。

这里主要是一些需要入侵业务代码打点的时候，可以使用这个 API 来获取时间戳

注意：
Date.now() 输出的是 UNIX 时间，即距离 1970 的时间，而 performance.now() 输出的是相对于 performance.timing.navigationStart(页面初始化) 的时间。

使用 Date.now() 的差值并非绝对精确，因为计算时间时受系统限制（可能阻塞）。但使用 performance.now() 的差值，并不影响我们计算程序执行的精确时间。

### window.performance.navigation
`window.performance.navigation` 对象提供了在指定的时间段里发生的操作相关信息，包括页面是加载还是刷新、发生了多少次重定向等。我们可以看看：

|    属性    | 含义 |
| ---------- | --- |
| type |  表示是如何导航到这个页面的 |
| redirectCount        |  表示在到达这个页面之前重定向了多少次 |

其中，type 的取值及含义如下表：

|    type的值    | 含义 |
| ---------- | --- |
| 0 |  当前页面是通过点击链接，书签和表单提交，或者脚本操作，或者在url中直接输入地址 |
| 1 |  点击刷新页面按钮或者通过Location.reload()方法显示的页面 |
| 2 |  页面通过历史记录和前进后退访问时 |
| 255 |  任何其他方式 |
具体数据示例：
![GitHub](https://raw.githubusercontent.com/LuckyWinty/blog/master/images/perf/1578912852644.jpg)
这个数据，主要是帮助我们看看页面重定向次数是否过多(能否减少重定向)，页面访问的方式主要是怎样的，针对访问方式较多的场景我们能否做些优化。

这种情况下，我们只需要把数据直接上报，然后自己查看数据的时候，再跟具体含义结合起来理解即可。

### window.performance.timing
`window.performance.timing`里面有很多的性能相关的时间戳记录，我们来看一些常用的：

|    属性    | 含义 |
| ---------- | --- |
| navigationStart |  准备加载页面的起始时间 |
| domainLookupStart       |  开始进行dns查询的时间 |
| domainLookupEnd | dns查询结束的时间|
| connectStart | TCP连接开始 |
| connectEnd | TCP连接完成 | 
| domInteractive | 解析dom树开始 |
| domComplete | 解析dom树结束 |
| loadEventEnd | onload事件结束的时间 |
| fetchStart | 开始检查缓存或开始获取资源的时间 |
| domLoading | loading的时间 (这个时候还木有开始解析文档)|

更多查看：https://developer.mozilla.org/en-US/docs/Web/API/PerformanceTiming

#### 关键指标
这样，我们就可以定出一些关键步骤耗时：
```doc
DNS查询耗时 = domainLookupEnd - domainLookupStart
TCP链接耗时 = connectEnd - connectStart
request请求耗时 = responseEnd - responseStart
解析dom树耗时 = domComplete - domInteractive
白屏时间 = domloading - fetchStart
domready时间 = domContentLoadedEventEnd - fetchStart
onload时间 = loadEventEnd - fetchStart
```
这个数据，上报到数据平台系统。就可以看到页面的性能情况如何，然后进行对应的优化了。不过，实际情况中，前端更关注的性能指标在首屏，比如：
+ HTML 加载完成时间
+ 首屏图片加载完成时间
+ 首屏接口完成加载完成时间

#### 代码实现
这个时候，我们可以自己手动加上一些时间点(这里的手动添加的点都推荐使用performance.now来实现)，结合一起上报。代码示例如下：

```js
//window.loadHtmlTime 在html中的</body>标签前面用打个时间戳即可
HTMLComplete = window.loadHtmlTime - window.performance.timing.navigationStart

//window.lastImgLoadTime 在首屏中的每张图onload之后都更新一次这个时间戳
firstScreenImgFinished = window.lastImgLoadTime - window.performance.timing.navigationStart

//Report.SPEED.MAINCGI 在首屏中的每个接口调用成功后更新时间戳
firstScreenApiFinished = Report.SPEED.MAINCGI - window.performance.timing.navigationStart

//在所有接口打时间点
apiFinishes = Report.SPEED.LASTCGI - window.performance.timing.navigationStart);
```
注意：

我们在做性能埋点的时候，最好不要入侵业务代码。这里我的想法是，每个api调用的方法，我们都返回一个Promise，这样，我们再另外封装一个sdk去找到这些方法，然后分别注册then方法来计时即可。

### window.performance.getEntries
`window.performance.getEntries` 是一个方法，方法调用后可以获取一个包含了页面中所有的 HTTP 请求的时间数据的数组.这个数组是一个按startTime排序的对象数组，数组成员除了会自动根据所请求资源的变化而改变以外，还可以用mark(),measure()方法自定义添加。

其与 performance.timing 对比的差别就是没有与 DOM 相关的属性。而要注意的是， HTTP 请求有可能命中本地缓存，这种情况下请求响应的间隔将非常短，数据可能不准确。 

我们来看看它都包含来了哪些时间，如下一个例子图：
![GitHub](https://raw.githubusercontent.com/LuckyWinty/blog/master/images/perf/1578925952330.jpg)
由图可以看出，每个对象的属性中除了包含资源加载过程各个阶段的时间外，还有以下五个属性：
+ name：资源名称，是资源的绝对路径或调用mark方法自定义的名称
+ startTime:开始时间
+ duration：加载时间
+ entryType：资源类型，entryType类型不同数组中的对象结构也不同
+ initiatorType：发起的请求者

其中，常用entryType的值含义如下：
|    属性    | 含义 |
| ---------- | --- |
| mark |  通过`performance.mark()`方法添加到数组中的对象 |
| measure  |  通过`performance.measure()`方法添加到数组中的对象 |
| resource | 资源类型，其加载时间用处最多 |
| navigation | 现除chrome和Opera外均不支持，导航相关信息 | 
| paint | 获取绘制相关的时间，主要是`first-paint` 和 `first-contentful-paint` | 
| longtask | 任何在浏览器中执行超过 50 ms 的任务，都是 long task (目前处于草案阶段)|

其中，常见initiatorType的值含义如下：
|    属性    | 含义 |
| ---------- | --- |
| link/script/img/iframe等 |  通过标签形式加载的资源，值是该节点名的小写形式 |
| css  |  通过css样式加载的资源，比如background的url方式加载资源 |
| xmlhttprequest | 通过xhr加载的资源|
| navigation | 当对象是PerformanceNavigationTiming时返回 | 

#### 关键指标
由以上，我们可以得出一些，我们比较关心的性能指标如下：
+ 首屏图片完成时间
+ 各资源耗时(主要统计css/js资源耗时)
+ FP(首次绘制时间)
+ FCP(首次内容渲染时间)

#### 代码实现
当我们调用这个方法的时候，我们得到的是调用方法前的所有资源的数据，一些资源可能有延时，或者在一些特殊的逻辑下才加载，这种情况下，我们就需要轮询上报了。

但是，浏览器考虑到这些复杂的情况，它为了我们提供了一个 `PerformanceObserver`,用于监测性能度量事件，在浏览器的性能时间轴记录下一个新的 performance entries  的时候将会被通知.

简单来说，我们可以利用 PerformanceObserver 做到当有性能数据产生时，主动通知你(观察者模式)，所以我们监听自己需要的资源类型，当有这个资源的时候进行上报即可。如下代码：

```js
function perf_observer(list, observer) { 
   // Process the "measure" and "resource" event
    list
    .getEntries()
    .map(({ name, entryType, startTime, duration }) => {
      const perfObj = {
        "Duration": duration,
        "Entry Type": entryType,
        "Name": name,
        "Start Time": startTime,
      };
      return JSON.stringify(obj, null, 2);
    })
    .forEach(console.log); // 可以加上报逻辑
} 
var observer2 = new PerformanceObserver(perf_observer); 
observer2.observe({entryTypes: ["paint","resource"]});
```

这里，关于首屏图片的获取和统计逻辑，我这里贴一个我使用的通用的逻辑，大家可以参考一下：
```js
// 可以在
const getFirstScreenImageLoadTime = () => {
    // 获取所有的 img dom 节点
    const images = document.getElementsByTagName('img');
    const imageEntries = performance.getEntries().filter(function (entry) {
        return entry.initiatorType === 'img'
    });

    // 获取在首屏内的 img dom 节点
    const firstScreenEntry = [];
    for (let i = 0; i < images.length; i++) {
        const image = images[i];
        const ret = image.getBoundingClientRect();
        if (ret.top < (window.innerHeight - 2) && ret.right > 0 && ret.left < (window.innerWidth - 2)) {
            // 如果在首屏内
            const imageEntry = imageEntries.filter(function (entry) {
                return entry.name === image.src;
            })[0];
            imageEntry && firstScreenEntry.push(imageEntry);
        }
    }

    // 获取最晚加载完成的一张
    let maxEntry;
    if (firstScreenEntry.length >= 1) {
        maxEntry = firstScreenEntry.reduce(function (prev, curr) {
            if (curr.responseEnd > prev.responseEnd) {
                return curr;
            } else {
                return prev
            }
        });
    }

    return maxEntry && maxEntry.responseEnd || null;
}
```

注意：

该方法有个草案，可以直接传入过滤参数，然后得到想要的结果，如`window.performance.getEntries(PerformanceEntryFilterOptions)`,但是我试了了一下，目前 chrome 都还不支持，可以参考这里：https://developer.mozilla.org/zh-CN/docs/Web/API/Performance/getEntries。

### 总结
最后总结一下，我们自定义的前端比较关心的性能指标大概有：
+ 白屏时间
+ HTML 加载完成时间
+ 首屏图片加载完成时间
+ 首屏接口完成加载完成时间
+ 各资源耗时(主要统计css/js资源耗时)
+ FP(首次绘制时间)
+ FCP(首次内容渲染时间)
+ onload时间

而这些指标的统计方法，本文都详细写了。可以参考一下实现方案～