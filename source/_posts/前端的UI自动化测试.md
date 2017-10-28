---
title: 前端的UI自动化测试
date: 2017-10-28 15:14:09
tags:
---

## 0. 背景
- 单元测试：对构成程序的每个单元进行测试。工程上的一个共识是，如果程序的每个模块都是正确的、模块与模块的连接是正确的、那么程序基本上就会是正确的。以上是抄的，我也不懂。我的理解分两块，什么是单元？可以是函数、接口、组件、事务（what？）。什么是测试？就是验证功能，给予相同的输入（可以是数据、行为），会有相同的输出。
- TDD（Test-Driven Development）：测试驱动开发。现写测试，后写业务，或者并行。对写测试的开发要求较高。
- vue、react等组件化框架天生易于被测试，如[element对alert组件测试的例子](https://github.com/ElemeFE/element/blob/dev/test/unit/specs/alert.spec.js)。理论上项目中的vue、react等组件都可以写测试，但前提是组件封装优良，模块间松耦合，对代码编写规范有要求。
- 下面要吹的是一套针对与UI测试的套件。

## 1. 相关工具
- 测试框架: Mocha（可以参考[阮一峰的文章](http://www.ruanyifeng.com/blog/2015/12/a-mocha-tutorial-of-examples.html)）
- 断言库: Chai [详情看文章介绍](http://www.jianshu.com/p/f200a75a15d2)
- 测试工具: nightmare(git:https://github.com/segmentio/nightmare)。这是一个基于electron的自动化框架，相比起PhantomJS的语法更加简单。

## 2. 使用
- 全局安装mocha：npm install -g mocha
```

# Linux & Mac
$ env ELECTRON_MIRROR=https://npm.taobao.org/mirrors/electron/ 
$ npm install

# Windows
$ set ELECTRON_MIRROR=https://npm.taobao.org/mirrors/electron/
$ npm install
```
- 运行一个测试
```
mocha ./testDemo/demo3.useChai.fn1.test.js
```
- 运行一系列测试
```
mocha ./testDemo
```

## 3. nightmare的使用
```javascript
const Nightmare = require('nightmare');
const nightmare = Nightmare({
  	show: true, //是否显示浏览器窗口
  	width: 1920, //浏览器窗口宽度
	height: 1080 //浏览器窗口高度
});
nightmare
	.goto('http://www.linghit.com/') //打开一个url
	.wait('#generalize_content') //等待某个元素出现在dom
	.wait(2000) //等待2000ms
	.click('#closed') //点击某个dom
	.evaluate(function () { //在浏览器环境下的操作
		return window.location.href;
	})
	.end() //结束一个nightmare队列
	.then((res) => { //获取到evaluate的return值
		console.log(res);
	})
	.catch((err) => { //捕捉错误
		console.log(err);
	})
```

## 4. mocha结合nightmare进行测试
```javascript
'use strict';
const expect = require('chai').expect;
const Nightmare = require('nightmare');
const nightmare = Nightmare({
  	show: true,
  	width: 1920,
	height: 1080
});

describe('灵机官网', function() {
	// 用例超时时长
    this.timeout(5 * 1000);

	it('页面是否能正常打开', (done) => {
		nightmare
			.goto('http://www.linghit.com/')
			.wait('#generalize_content')
			.wait(1000)
			.click('#closed')
			.evaluate(function () {
				var bannerLen = $('.banner').length;
				return {
					bannerLen: bannerLen
				};
			})
			.end()
			.then((res) => {
				expect(res.bannerLen).to.above(0);
				done();
			})
			.catch((err) => {
				done(err);
			})
	});
});
```
## 5. 使用mochawesome生成测试报告
```javascript
mocha .\testDemo\demo7.linghit.report.test.js --reporter mochawesome
```

## 6. 其他东西
- nightmare的截屏
```javascript
.screenshot([path][, clip])
```
- mocha的钩子函数
```javascript
before(function() {
    // 在本区块的所有测试用例之前执行
});

after(function() {
    // 在本区块的所有测试用例之后执行
});

beforeEach(function() {
    // 在本区块的每个测试用例之前执行
});

afterEach(function() {
    // 在本区块的每个测试用例之后执行
});
```
- 通过node脚本定时巡航页面，并上报测试结果
```javascript
<!-- index.js -->
setInterval(() => {
    // 当然定时任务不会用setInterval。。。
	shell.exec('mocha ./testDemo/demo8.other.test.js --reporter mochawesome', function(err) {
		if( err ) {
			throw err;
		} else 
		}
	});
}, 10 * 1000);

<!-- 测试脚本 -->
after(async function() {
	// 统一上报测试结果
	var options = {
		method: 'POST',
    	url: 'http://localhost:3000/api/test/report',
    	body: reportObj, //reportObj对象收集图片url，错误信息等等
	    json: true
    };
 
    var res = await request(options);
  });
```
