---
title: vue项目的错误捕获方式
date: 2017-11-01 12:16:28
tags:
---
## 为什么要捕获错误？
- 为什么要花那么多的时间成本和精力去做什么捕获错误的事情？为什么要提高代码质量？为什么要让项目更健壮？这个问题我也不懂回答，不过这就好比"有得吃就行了，为什么要吃好，为什么要花大钱去吃什么米其林三星。"
- 目前为止我接触到的需要捕获错误的地方或目的有2：
1. 做错误兼容，如：
```
var obj = {};
try {
    obj = JSON.parse(json);
} catch(e) {}
```
2. 做错误日志收集，帮助发现与定位问题。
- 已知的错误都在开发时处理掉了。需要捕获的是未知的错误：
1. 未知的接口返回内容
2. 未知的资源加载情况，如图片或其他媒体资源。
3. 改了一处代码，没发现另一处被影响到的逻辑
4. 较深层级的交互逻辑，没有在测试阶段被发现的问题

## 在MVVM时代，用window.onerror捕获错误已经不适用了
- 通常MVVM项目会有一个（或多个）入口html or js，假如我在入口处如此监听全局错误：
```
  <body>
    <div id="container">
      <div id="app">
        <router-view></router-view>
      </div>
    </div>
    
    <script type="text/javascript">
        window.onerror = function (msg, script, line, columns, error) {
            console.log(msg);
            console.log(script);
            console.log(line);
            console.log(columns);
        }
    </script>
    <script type="text/javascript" src="http://XXX.XXX.XXX/static/js/app.js"></script></body>
  </body>
```
- 当入口js逻辑出现问题时,控制台出现报错信息
![image](http://7xpsli.com1.z0.glb.clouddn.com/vue_catch_error1.png)
- 然而在onerror的错误捕获代码却输出
```
console.log(msg); //Script error.
console.log(script); //''
console.log(line); //0
console.log(columns); //0
```
- 这是什么鬼？以下是网络上的一些搜索结果
> 因为同源策略，Firefox, Chrome, Safari 等浏览器， 页面引用的非同域的外部脚本中抛出了异常，本页面无权限获得这个异常详情， 所以就成了 Script error.。

> 解决办法有：1.静态文件服务器设置 Access-Control-Allow-Origin 头信息。2.script 标签添加 crossorigin 属性。
    
- 似乎一切就明朗了吗？就能早点回家吃饭了吗？
    图样图森破啊~
    就算最后能按方法解决掉script跨域脚本的问题。也无法确实可行地捕获到vue代码里的错误。webpack合并压缩混淆过的代码，输出的报错信息可读性也是有限的。另外，在vue组件内的错误其实是已经被捕获过，不会再抛给全局的onerror。
    vue是个先进的框架，它自己有便捷健全的错误监听机制。

## 捕获vue项目内全局错误
- errorHandLer： [先读文档](https://cn.vuejs.org/v2/api/#errorHandler)。具体用法：
```
Vue.config.errorHandler = function (err, vm, info) {
    let { message, name, script, line, column, stack } = err;
    // 在vue提供的error对象中，script、line、column目前是空的。但这些信息其实在错误栈信息里可以看到。
    script = !_.isUndefined(script) ? script : '';
    line = !_.isUndefined(line) ? line : 0;
    column = !_.isUndefined(column) ? line : 0;
    // 解析错误栈信息
    let stackStr = stack ? stack.toString() : `${name}:${message}`;
    
    console.log(stackStr); 
    /*
    ReferenceError: bbb is not defined
    at a.created (shortLink.vue:361)
    at It (vue.esm.js:2701)
    at a.t._init (vue.esm.js:4293)
    at new a (vue.esm.js:4463)
    at ee (vue.esm.js:3740)
    at init (vue.esm.js:3557)
    at u (vue.esm.js:5212)
    at l (vue.esm.js:5155)
    at a.t.nodeOps [as __patch__] (vue.esm.js:5697)
    at a.t._update (vue.esm.js:2460)
    */
    
    // report code
}
```
- 错误信息已经很足够了。如果有错误上报系统，开发者根据上报的错误信息回归到具体的案发现场，配合webpack的sourceMap也能快速定位问题。
    
## 捕获vue组件错误
- 最近vue2.5发布，新增了一个errorCaptured钩子: [先看文档](https://cn.vuejs.org/v2/api/#errorCaptured)
- errorCaptured所为一个vue组件的钩子函数，能捕获到子孙组件的错误。在错误信息的显示上面与errorHandler是一致的，在做错误收集时无必要重复。但errorCaptured的目的我估计更加集中于去做即时性的兼容处理，这与try catch的性质是相似的，但vue提供了一个组件层面可用的try catch。
    
## 特殊位置埋点捕获
- 比如说在收发请求时，有些问题并不会造成一个js的报错，然而前后端的交互也经常会因为协议上面的疏漏而造成问题。在项目中我用到了vue-resource这个http请求库。本身支持Promise API。那我可以这样做：
```
Vue.http.get(url).then((resp) => {
    if (返回数据不符合正确的格式) {
        throw `格式不正确`;
    }
    if (typeof successFn === 'function') {
        successFn(resp.body); //在处理回调函数内部出错时，也会被捕获到错误
    }
}, resp => {
    throw `网络问题的错误`;
}).catch((err) => {
    // 捕获到的错误信息会被包含在err对象里
    // report code
    /*
        在收集错误信息时，可以选择性地把调用的url、method、query、response数据一并上报，方便还原案发现场。
    */
});
```
- 是否便于埋点捕获还与项目的构成有关系，假如说项目里的请求都各走各路，那么处理逻辑就会很分散，开发者也会烦于维护。
- 出问题是无可避免的，问题的解决手段才是更重要，在问题大范围扩散前能发现问题，至少在代码上线后能淡定一点。
