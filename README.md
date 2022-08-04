# 跨窗口通信

## 同源

> 同源策略会限制 窗口(window) 和 frame 之间的通信，因此首先要知道同源策略。
>
> 同源策略目的是保护用户信息免遭信息盗窃：加入小王有两个打开的页面：一个是 shop.com，一个是 email.com。小王不希望 shop.com 的脚本可以读取 mail 的邮件，这时同源策略就起作用了。

如果两个 URL 具有相同的协议，域，和端口，则称它们是`同源`的。

以下几个URL是同源的：

* http://site.com
* http://site.com/
* http://site.com/a/index.html

以下是不同源的：

* **https**://site.com
* http://**bbs**.site.com
* http://site.com:**8080**
* http://site.**org**

同源策略规定：

 * 如果我们有对另一个窗口的引用（window.open || iframe），并且该窗口是同源的，那么我们就具有对该窗口的全部访问权限。<font size=2>*见代码：2-1 、2-2*</font>
 * 如果不是同源的，我们就不能访问窗口中的内容：变量，文档，任何东西。唯一例外是location：我们可以修改它，使用它进行重定向。但是我们无法读取 location 。因此，我们无法看到用户当前所处的位置，也就不会泄露任何信息。

## iframe

> iframe 标签承载了一个单独的嵌入的窗口，它有自己的 **document** 和 **window**
>
> iframe.contentWindow 来获取 <iframe> 中的 window
>
> iframe.contentDocument 来获取 <iframe> 中的 document ， 是 iframe.contentWindow.document 的简写。
>
> 当我们访问嵌入的窗口中的东西时，浏览器会检查 iframe 是否具有相同的源。如果不是，则会拒绝访问（对 `location` 进行写入是一个例外，它是会被允许的）。



**代码 2-1** : （在 <u>**同源**</u> 情况下）

```html
<!-- 1.html 内容 -->
<!-- http://127.0.0.1:8000/1.html -->
<body>
    我是 1.html, 下面嵌套 2.html
    <iframe src="http://127.0.0.1:8000/2.html" ></iframe>
    <script>
        function hello () { console.log('this is 1.html') }
        
        var iframe = document.getElementsByTagName('iframe')[0];
        console.log('contentWindow 🥝', iframe.contentWindow); // 能访问
        console.log('contentDocument 🥝', iframe.contentDocument); // 能访问
        
        // 注意访问方式, 需要在 onload 后才能取到值
        console.log( iframe.contentWindow.hello() ) // Uncaught TypeError: iframe.contentWindow.hello is not a function
        
        iframe.onload = function(){
      		console.log( iframe.contentWindow.hello() ) // this is 2.html
            
            //  输出 Location 对象， 依然要在 iframe.onload 中访问
            console.log('contentWindow.location 🥝', iframe.contentWindow.location)
            //  iframe.contentWindow.location.host : 127.0.0.1:8000
            //  xxx.hash:     ""
            //  xxx.host:     "127.0.0.1:8000"
            //  xxx.hostname: "127.0.0.1"
            //  xxx.href:     "http://127.0.0.1:8000/2.html"
            //  xxx.origin:   "http://127.0.0.1:8000"
            //  xxx.pathname: "/2.html"
            //  xxx.port:     "8000"	
            //  xxx.protocol: "http:"
            //  ...
            
            // 有相同的源 我们可以进行任何操作
            iframe.contentDocument.body.innerHTML('<p>hi, i am ur father !</p>');
            iframe.contentDocument.getElementsByTagName('p');
	    })
        
        iframe.contentWindow.location = 'http://www.360doc.com'; // 可以直接修改 iframe 地址, 不受同源策略的限制。 有的网站不支持被iframe引用, 所以会报错。 注意区分错误信息。
    </script>
</body>


<!-- 2.html 内容 -->
<!-- http://127.0.0.1:8000/2.html -->
<body>
    我是 2.html
    
    <script>
        function hello () { console.log('this is 2.html') }
    </script>
</body>
```



**`iframe.onload` vs `iframe.contentWindow.onload`**

`iframe.onload` 事件（在 `<iframe>` 标签上）与 `iframe.contentWindow.onload`（在嵌入的 window 对象上）基本相同。当嵌入的窗口的所有资源都完全加载完毕时触发。

……但是，我们无法使用 `iframe.contentWindow.onload` 访问不同源的 iframe。因此，请使用 `iframe.onload`。

###  window：document.domain

但是，如果窗口的二级域相同，例如 `bbs.site.com`，`nav.site.com` 和 `site.com`（它们共同的二级域是 `site.com`），我们可以使浏览器忽略该差异，使得它们可以被作为“同源”的来对待，以便进行跨窗口通信。

为了做到这一点，每个这样的窗口都应该执行下面这行代码：

```javascript
document.domain = 'site.com';
```

这样就可以了。现在它们可以无限制地进行交互了。但是再强调一遍，这仅适用于具有相同二级域的页面。

**已弃用，但仍有效**

`document.domain` 属性正在被从 [规范](https://html.spec.whatwg.org/multipage/origin.html#relaxing-the-same-origin-restriction) 中删除。跨窗口通信（下面将很快解释到）是建议的替代方案。

也就是说，到目前为止，所有浏览器都支持它。并且未来也将继续支持它，而不会导致使用了 `document.domain` 的旧代码出现问题。



**代码 2-1** : （在 <u>**不同源**</u> 情况下）注意端口

```html
<!-- 1.html 内容 -->
<!-- http://127.0.0.1:8000/1.html -->
<body>
    我是 1.html, 下面嵌套 2.html
    <!-- 端口不同, 不同源 -->
    <iframe src="http://127.0.0.1:8001/2.html" ></iframe>
    <script>
        function hello () { console.log('this is 1.html') }
        
        var iframe = document.getElementsByTagName('iframe')[0];
        console.log('contentWindow 🥝', iframe.contentWindow); // 可以获取对内部 window 的引用
        console.log('contentDocument 🥝', iframe.contentDocument); // 空的 document 对象
        
        iframe.onload = function(){
      		console.log( iframe.contentWindow.hello() ) // Uncaught DOMException: Blocked a frame with origin "http://127.0.0.1:8000" from accessing a cross-origin frame.
            
            
 			// 无法读取 iframe 中页面的 URL
            console.log( iframe.contentWindow.location )
            // Location {then: undefined, Symbol(Symbol.toStringTag): undefined, Symbol(Symbol.hasInstance): undefined, Symbol(Symbol.isConcatSpreadable): undefined, replace: ƒ}
            
            
            console.log(iframe.contentWindow.location.href) // Uncaught DOMException: Blocked a frame with origin "http://127.0.0.1:8000" from accessing a cross-origin frame. 
	    })
        
        
        iframe.contentWindow.location = 'http://www.360doc.com'; // 可以直接修改 iframe 地址, 不受同源策略的限制，不受同源策略的限制，不受同源策略的限制。 
    </script>
</body>


<!-- 2.html 内容 -->
<!-- http://127.0.0.1:8001/2.html -->
<body>
    我是 2.html
    
    <script>
        function hello () { console.log('this is 2.html') }
    </script>
</body>
```



### iframe：错误文档陷阱

当一个 iframe 来自同一个源时，我们可能会访问其 `document`，但是这里有一个陷阱。它与跨源无关，但你一定要知道。<u>在创建 iframe 后，iframe 会立即就拥有了一个文档。但是该文档不同于加载到其中的文档</u>！

因此，如果我们要立即对文档进行操作，就可能出问题，因为那是**错误的文档**。

**正确的文档在 `iframe.onload` 触发时肯定就位了。但是，只有在整个 iframe 和它所有资源都加载完成时，`iframe.onload` 才会触发。**

看一下下面这段代码：

```javascript
let oldDoc = iframe.contentDocument;

iframe.onload = function() {
    let newDoc = iframe.contentDocument;
    // 加载的文档与初始的文档不同！
    alert(oldDoc == newDoc); // false
};
```



### window.frames

获取 `<iframe>` 的 window 对象的另一个方式是从命名集合 `window.frames` 中获取：

- 通过索引获取：`window.frames[0]` —— 文档中的第一个 iframe 的 window 对象。
- 通过名称获取：`window.frames.iframeName` —— 获取 `name="iframeName"` 的 iframe 的 window 对象。

例如：

```javascript
// <iframe src="/" style="height:80px" name="win" id="iframe"></iframe>

alert(iframe.contentWindow == frames[0]); // true
alert(iframe.contentWindow == frames.win); // true
```

一个 iframe 内可能嵌套了其他的 iframe。相应的 `window` 对象会形成一个层次结构（hierarchy）。

可以通过以下方式获取：

- `window.frames` —— “子”窗口的集合（用于嵌套的 iframe）。
- `window.parent` —— 对“父”（外部）窗口的引用。
- `window.top` —— 对最顶级父窗口的引用。

例如：

```javascript
window.frames[0].parent === window; // true
```

我们可以使用 `top` 属性来检查当前的文档是否是在 iframe 内打开的：

```javascript
if (window === window.top) { 
  alert('不是在 iframe 中打开的');
} else {
  alert('在 iframe 中打开的');
}
```

**值得一提**
`window.parent` 可以调用父窗口的环境的方法， 这个特性在 小程序宣讲会直播页面使用到了

```js
// 小程序使用web-view
// <web-view src="www.laoyaoba.com/workplace" />

// h5 www.laoyaoba.com/workplace 内容如下
// <iframe src="zb.laoyaoba.com/watch/10594885?inset" />
window.hello = function () {console.log('this is www.laoyaoba.com/workplace')}

// ------
// zb.laoyaoba.com 中又嵌入了 <iframe src="www.laoyaoba.com/workplaceposition/1"> 页面
// www.laoyaoba.com/workplaceposition/1 调用 www.laoyaoba.com/workplace 方式如下
window.parent.parent.hello()
```

还使用到了`cookie`的跨窗口获取。

``` html
<!-- 1.html 内容 -->
<!-- http://127.0.0.1:8000/1.html -->
<body>
    我是 1.html, 下面嵌套 2.html
    <!-- 端口不同, 不同源 -->
    <iframe src="http://127.0.0.1:8001/2.html" ></iframe>
    <script>
        if (!document) {
        	document.cookie = 'name=1';
	        document.cookie = 'old=10';    
        }

        console.log('1.html', document.cookie)
    </script>
</body>


<!-- 2.html 内容 -->
<!-- http://127.0.0.1:8000/2.html -->
<body>
    我是 2.html
    
    <script>
        document.cookie = 'name=2';
        document.cookie = 'year=2020';

        console.log('2.html', document.cookie)
    </script>
</body>
```

第一次渲染输出：`1.html name=1; old=10` ，`2.html old=10; name=2; year=2020`

刷新页面输出：`1.html old=10; year=2020; name=2` ， `2.html old=10; year=2020; name=2`

我们可以得出以下结论：

* iframe 嵌套的 2.html 设置的 cookie 我们可以从 1.html 中获取
* iframe 中设置的 cookie 会覆盖 1.html cookie 中 Name相同的值( 不同源也是同样的效果 )

### “sandbox” iframe 特性

> `sandbox` 特性（attribute）允许在 `<iframe>` 中禁止某些特定行为，以防止其执行不被信任的代码。它通过将 iframe 视为非同源的，或者应用其他限制来实现 iframe 的“沙盒化”。
>
> 对于 `<iframe sandbox src="...">`，有一个应用于其上的默认的限制集。但是，我们可以通过提供一个以空格分隔的限制列表作为特性的值，来放宽这些限制，该列表中的各项为不应该应用于这个 iframe 的限制，例如：`<iframe sandbox="allow-forms allow-popups">`。
>
> 换句话说，一个空的 `"sandbox"` 特性会施加最严格的限制，但是我们用一个以空格分隔的列表，列出要移除的限制。

以下是限制的列表：

- `allow-same-origin`

  默认情况下，`"sandbox"` 会为 iframe 强制实施“不同来源”的策略。换句话说，它使浏览器将 `iframe` 视为来自另一个源，即使其 `src` 指向的是同一个网站也是如此。具有所有隐含的脚本限制。此选项会移除这些限制。

- `allow-top-navigation`

  允许 `iframe` 更改 `parent.location`。 

- `allow-forms`

  允许在 `iframe` 中提交表单。

- `allow-scripts`

  允许在 `iframe` 中运行脚本。

- `allow-popups`

  允许在 `iframe` 中使用 `window.open` 打开弹窗。

  查看 [官方手册](https://developer.mozilla.org/zh/docs/Web/HTML/Element/iframe) 获取更多内容。

### 跨窗口通信

`postMessage` 接口允许窗口之间相互通信，无论它们来自什么源。

因此，这是解决“同源”策略的方式之一。它允许来自于 `marh.com` 的窗口与来自于 `qq.com` 的窗口进行通信，并交换信息，但前提是它们双方必须均同意并调用相应的 JavaScript 函数。这可以保护用户的安全。

这个接口有两个部分。

#### 1 postMessage

想要发送消息的窗口需要调用接收窗口的 `postMessage` 方法。换句话说，如果我们想把消息发送给 `win`，我们应该调用 `win.postMessage(data, targetOrigin)`。

**参数**

`data`

要发送的数据。可以是任何对象，数据会被通过使用“结构化序列化算法（structured serialization algorithm）”进行克隆。<u>IE 浏览器只支持字符串</u>，因此我们需要对复杂的对象调用 `JSON.stringify` 方法进行处理，以支持该浏览器。

`targetOrigin`

指定目标窗口的源，以便只有来自给定的源的窗口才能获得该消息。

```js
// <iframe src="http://127.0.0.1:8080/2.html" name="example" />  

let win = window.frames.example;    
win.postMessage("message", "http://127.0.0.1:8080"); 
```


#### 2 onmessage

为了接收消息，目标窗口应该在 `message` 事件上有一个处理程序。当 `postMessage` 被调用时触发该事件（并且 `targetOrigin` 检查成功）。

event 对象具有特殊属性：

- `data`

  从 `postMessage` 传递来的数据。

- `origin`

  发送方的源，例如 `http://javascript.info`。

- `source`

  对发送方窗口的引用。如果我们想，我们可以立即 `source.postMessage(...)` 回去。

要为 `message` 事件分配处理程序，我们应该使用 `addEventListener`，简短的语法 `window.onmessage` 不起作用。

```js
window.addEventListener("message", function(event) {
  console.log(event)
  if (event.origin != 'http://http://127.0.0.1:8080') {
    // 来自未知的源的内容，我们忽略它
    return;
  }

  if (window == event.source) {
    // chrome 下, 页面初次加载后会触发一次 message 事件, event.source 是 window 对象, 此时 event.source.postMessage 会形成死循环
    // 跳过第一次的初始化触发的情况
    return
  }
	
  console.log( "received: " + event.data );

  // 可以使用 event.source.postMessage(...) 向回发送消息
  event.source.postMessage(’i am 2.html‘)
}, source);
```

