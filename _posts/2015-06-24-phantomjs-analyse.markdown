---
published: true
title: PhantomJS原理分析
layout: post
tags: [phantomjs, qt, 源码分析]
categories: [JavaScript, C/C++]
---

项目中用到了和phantomjs类似的技术，基于的是phantomjs中的headless webkit，phantomjs源码中带的qt编译后没有图形库依赖，很适合用在服务器上，这里只是分析外层wrapper的原理，不对其JS模块和解释器做说明。

实际上，看完phantomjs的文档，对phantom对象实现方式的第一反应就应该是addToJavaScriptWindowObject。这个API由QWebFrame类提供，如果js引擎是JSC的话，最终会调用到WebCore的bridge中针对qt的createRuntimeObject实现。函数的作用是将指定的C++对象绑定到js引擎中的Window上，实现类似javascirpt代码中document/navigator对象的效果。

按照这个思路，在phantomjs的源码目录中grep一下，果然找到代码如下：

```
work@yuanvi:~/workspace/phantomjs-master/src$ grep addToJavaScriptWindowObject *.cpp
config.cpp:    webPage.mainFrame()->addToJavaScriptWindowObject("config", this);
phantom.cpp:    m_page->mainFrame()->addToJavaScriptWindowObject("phantom", this);
repl.cpp:    m_webframe->addToJavaScriptWindowObject("_repl", this);
webpage.cpp:        frame->addToJavaScriptWindowObject(CALLBACKS_OBJECT_NAME, callbacksObject, QWebFrame::QtOwnership);
```

其中_repl对象实现了命令行交互的效果(Read Execute Print Loop)，使用了 [CoffeeScript](http://coffee-script.org/) 来将命令行输入编译成js代码并注入执行，具体流程不做详细分析。

来看phantom对象的实现，从官方的文档上看，phantom对象的作用主要有两个，注入执行JS代码和管理cookies。注入执行JS代码调用了 [QWebFrame](http://doc.qt.io/qt-4.8/qwebframe.html) 的evaluateJavascript函数；对cookies的管理代码则是对 [QNetworkCookieJar](http://doc.qt.io/qt-4.8/qnetworkcookiejar.html) 增删查改的封装。

PhantomJS中经常操作对象还有webpage，webpage不仅提供了对当前访问页面的各种操作，还提供了webkit从网络请求到dom节点操作的各类回调事件，通常用phantomjs来做爬虫或页面操作用得最多的也是这个对象。

虽然对象名是webpage，但实际上webpage操作的是一个页面中的frame，提供了switchToChildFrame、switchToMainFrame等几个函数来在可操作的frames之间转换。再看上面grep出的代码，从webpage.cpp中找到具体实现：

```c++
#define CALLBACKS_OBJECT_NAME           "_phantom"
#define INPAGE_CALL_NAME                "window.callPhantom"
#define CALLBACKS_OBJECT_INJECTION      INPAGE_CALL_NAME" = function() { return window."CALLBACKS_OBJECT_NAME".call.call(_phantom, Array.prototype.splice.call(arguments
#define CALLBACKS_OBJECT_PRESENT        "typeof(window."CALLBACKS_OBJECT_NAME") !== \"undefined\";"

static void injectCallbacksObjIntoFrame(QWebFrame *frame, WebpageCallbacks *callbacksObject)
{
    // 判断是否已经存在_pantom对象
    if (frame->evaluateJavaScript(CALLBACKS_OBJECT_PRESENT).toBool() == false) {
        // 把callbackObject绑定到_phantom对象
        frame->addToJavaScriptWindowObject(CALLBACKS_OBJECT_NAME, callbacksObject, QScriptEngine::QtOwnership);
        // 把window.callPhantom设置为一个调用_phantom对象中函数的函数（无误）
        // window.callPhantom = function () {
        //     return window._phantom.call.call(_phantom. Array.prototype.splice.call(arguments);
        // }
        frame->evaluateJavaScript(CALLBACKS_OBJECT_INJECTION);
    }    
}
```

上面的代码把callbacksObject绑定为了window._phantom，callbackObjects类WebpageCallbacks实现非常简单，存储了四个callback函数，供注入的js发生回调事件时调用，其中大部分的回调都是通过genericCallback来传递。

现在已经有了绑定到页面中的对象，并且可以注入代码执行，还可以接收回调，已经可以实现基本需求，那么接下来就是js部分的实现。

phantomjs实现的js模块在modules文件夹中，这里不分析具体实现，基本模式都是通过注入对象来调用对应的C++代码完成特定功能。目前分析的1.9.7版本中包括了fs，webpage，webserver，child_process模块，很显然，这些模块让phantomjs不只可以做一个headless browser，还可以实现很多其他功能。

JavaScript Engine提供的这种和C++的交互执行的能力让JS不再局限在浏览器中，而是可以通过外部的交互模块做到c/c++能做到的所有操作，这让很多前端程序员不用再去学习新的语言就可以处理后端业务，这也是JavaScript现在被称为全栈式语言的原因，一些跨平台APP开发框架也同样是基于这种原理。
