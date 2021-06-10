# H5与Native通信机制

在`Hybrid`开发模式下，`H5`与`Native`需要进行通信。`JavaScript`运行在单独的`JS Context`中（`e.g.` `Webview`的`Webkit`引擎、`JS Core`）。由于这些`Context`与原生运行环境天然隔离，可以将这种情况与 [RPC](https://searchapparchitecture.techtarget.com/definition/Remote-Procedure-Call-RPC )通信进行类比，将`Native`与`JavaScript`的每次调用看做一次 RPC 调用。

## Native调用JavaScript

不管是`IOS`的`UIWebview`还是`WKWebview`，还是`Android`的`Webview`组件，都以子组件的形式存在于`View/Activity`中，直接调用相应的`API`即可。`Native`调用`js`，实际是执行拼接`js`字符串，从外部调用`js`方法，因此`js`方法必须在全局的`window`上。

| 平台          | 方法                                                         | 用法                                                         | 详细                                                         |
| ------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Android       | `Webview.loadUrl`                                            | `mWebview.loadUrl("javascript: funcName()")`                 | 执行`js`定义好的函数，拿不到返回信息，支持4.4以下，会刷新页面 |
| Android       | `Webview.evaluateJavascript`                                 | `mWebview.evaluateJavascript("javascript: funcName()", new ValueCallback<String>(){`<br />  `@Override`<br />  `public void onReceiveValue(String value){`<br />    `return;`<br />  `}`<br />`})` | 执行完函数后，可以拿到返回的信息，只支持 4.4 以上            |
| IOS UIWebview | `stringByEvaluatingJavascript`                               | [`func stringByEvaluatingJavaScript(from script: String) -> String?`](https://developer.apple.com/documentation/uikit/uiwebview/1617963-stringbyevaluatingjavascript) |                                                              |
| IOS WKWebview | `evaluateJavaScript`<br />`addUserScript` 加载页面的时候执行 | [`func evaluateJavaScript(_ javaScriptString: String,        completionHandler: ((Any?, Error?) -> Void)? = nil)`](https://developer.apple.com/documentation/webkit/wkwebview/1415017-evaluatejavascript)<br />[`func addUserScript(_ userScript: WKUserScript)`](https://developer.apple.com/documentation/webkit/wkusercontentcontroller/1537448-adduserscript) |                                                              |

其中，在IOS中，存在两种`WebView`，其中`UIWebView`自`IOS2`就有，会占用很多内存，加载速度较慢。`WKWebview`是他的全面升级版，不仅提升了内存和加载速度，而且对`HTML5`的特性支持更多，唯一问题是`WKWebView`在`IOS8`以后才支持，这两种`webview`的`API`存在比较大的差异，客户端需要书写两套方案。

## JS调用Native

### 拦截相关 BOM 方法

`js调用alert`、`confirm`、`prompt`、`console.log`方法时，`Native`可以对他们进行拦截。这种拦截的方式可以在`Native`的拦截方法里面设置里面进行调用前端回调，通知数据已成功接收。

- 安卓：拦截方法：`onJSAlert`,`onJSPrompt`,`onJSConfirm`,`onConsoleMessage`

  

  `native`会拦截`H5`发出的`prompt`，当检测到协议为`jsb`而非普通的`http/https/file`等协议时，解析出 `URL`中的`methodName`, `args`, `callbackName`，执行该方法并调用`js`回调

  ```js
  // js
  function callNative(methodName, args, cb) {
    const url = 'jsb://' + method + '?' + JSON.stringfy(args);
    prompt(url);
  }
  ```

  ```java
  // java
  import android.webkit.JsPromptResult;
  import android.webkit.WebChromeClient;
  import android.webkit.Webview;
  
  public class JSBridgeChromeClient extends WebChromeClient {
    @Override
    public boolean onJsPrompt(WebView view, String url, String message, String defaultValue, JsPromptResult result)
      result.confirm(JSBridge.call(view, message));
      return true;
  }
  ```

- IOS

  - `UIWebview`中，暂不支持对上面方法的拦截
  - `WKWebView`中，`runJavaScriptAlertPanelWithMessage`, `runJavaScriptConfirmPanelWithMessage`, `runJavaScriptTextInputPanelWithPrompt`

  IOS中没有原生监听 `console.log`的方法，IOS使用`evaluate`对原生的`console.log`进行改写，每执行一次`console.log`，就发送一次`schema`请求，IOS监听到请求，即可拿到相应参数。

### 拦截URL

通过拦截一个正常的`H5`页面跳转，并解析`URL`中包含的信息，来实现通信。

一个符合`RFC`标准的`URL`通常由以下几部分组成：

```
const URL = [scheme:][host][path][?query][#fragment]
```

各个手机的APP是相互独立的，偶尔需要相互通信，如调端操作。`scheme`可以用来标志某APP，其他APP发送一个`scheme`请求，相应的APP可以对此请求进行拦截，并解析到相应的数据。

#### js发送scheme请求

前端发送请求方式

| 方式                | 问题                                                     |
| ------------------- | -------------------------------------------------------- |
| <a>标签进行跳转     | 需要用户点击才能发送请求                                 |
| 更改`location.href` | 调用会引起页面的跳转                                     |
| `iframe`发送请求    | 连续发送多次请求，后续请求会丢失                         |
| `ajax` 发送请求     | IOS提供了拦截方法`NSURLProtocol`，安卓没有对应的拦截方法 |

`iframe`发送请求丢失的原因：

`iframe`存在一个问题，同一次事件循环中，如果连续发送多次请求，只有第一次的请求能够被`Native`拦截，后面的请求会丢失。`iframe`请求本质上是一种模拟跳转，webview会做出相应的限制，当`h5`连续发送多次跳转时，`webview`会直接过滤掉后发的跳转请求，因此后面的消息收不到。

基本解决方案：

- 前端维护一个队列`sendMessageQueue`，在发送请求的时候，只发送一个特殊的空请求，将真正的请求信息全部放入`sendMessageQueue`队列中
- `Native`接收到这个空请求（或者由客户端决定从队列取数据的时机），执行前端的`fetchQueue`方法，此方法将`sendMessageQueue`队列的信息打包，再传送一整个数据请求给`Native`处理
- `Native`对请求进行解析，并执行相应的方法

```javascript
// 前端
// 前端定义_fetchQueue，客户端调用。取出队列中的消息，并清空接收队列
_fetchQueue() {
  const messageQueueString = JSON.stringify(this.sendMessageQueue);
  if(this.sendMessageQueue.length > 0) {
    this._dispatchUrlMsg(`${this.bridgeSchema}${this.setResultPath}${this.fetchQueue}&${encodeBase64andURI(messageQueueString)}`)
    this.sendMessageQueue = []
    return messageQueueString
  }
}
_call() {
  const msgJSON = {
    JSSDK: sdkVersion,
    func: this.actionMap[func] ? this.actionMap[func]['native'] : func,
    params,
    __msg_type: 'call',
    __callback_id: callbackID
  }
  this.sendMessage.push(msgJSON);
  this._dispatchUrlMsg(this.readyMessageIframeId, `${this.bridgeScheme}${this.dispatchMsgPath}`)
}
/**
* 1. 前端发送 bytedance://dispatch_message 信息给 native
* 2. native 拦截到 scheme后，调用前端 ToutiaoJSBridge._fetchQueue 获得真正的 JSB 信息
* 3. native 处理完数据后，将结果和 callback 信息放入 ToutiaoJSBridge._handleMessageFromToutiao 方法中，即可执行相应的 callback 信息
*/
```

```java
// Native
String p_dispatch = SCHEMA_PREFIX + 'dispatch_message';
String p_result = SCHEMA_PREFIX + 'private/setresult';
try {
  if (url.equals(p_dispatch)) {
    Webview wv = getWebview();
    if(wv != null) {
      LoadUrlUtils.loadUrl(wv, 'javascript: ToutiaoJSBridge._fetchQueue()')
    }
  } else if(url.startsWith(p_result)) {
    if(scene.equals('SCENE_FETCHQUEUE') && msg.length() > 0) {
      // 解析msg，调用逻辑处理
      parseMsgQueue(msg)
    }
  }
} catch (Exception ignore) {
  //
}
```

#### Native拦截scheme请求

- 安卓：`shouldOverrideUrlLoading`
- IOS
  - `UIWebView`：`shouldStartLoadWithRequest`
  - `WKWebView`：`decidePolicyForNavigationAction`

#### 回调

整个过程，`Native`可以从前端获取一个数据，并执行自己的方法，对于需要获取返回值的情况，需要单独处理。

- 前端定义一个`callbackFunction`
- 前端发送请求的时候，将此`callback`的名字一并发送给客户端
- 客户端拦截到请求以后，将返回的数据，塞到`callback`里面，并执行
- 前端获取数据，交互完成

![image](./imgs/消息队列实现.png)

优劣：

- 优点：兼容性好，安卓和IOS的各个版本都支持此功能
- 缺点
  - 调用延时较高，`200ms-400ms`，在`Android`上表现明显；
  - `URL scheme`长度有限，内容过多会丢失字符；
  - 不支持同步返回结果，所有信息传送都需要调用`iframe`请求，使用`callback`得到返回数据

### 注入API

通过`WebView`提供的接口，向`JS`的`Context`(`window`)中注入对象或方法，`JS`调用时，可直接执行相应的`Native`代码逻辑，从而达到`js`调用`Native`的目的。

- 安卓注入

  使用`webView.addJavascriptInterface`方法进行注入。此方法存在漏洞，在`Android 4.2`以上提供`@javascriptInterface`注解来规避该漏洞，但对于4.2以下版本则没有任何方法。使用该方法有一定的风险和兼容问题。

- IOS注入

  - `UIWebview`中，使用`JavascriptCore`方法注入
  - `WKWebview`中，使用`messageHandlers`方法注入

## JSB通讯流程

- 前端：调用一个`method`，不用考虑调用的方式是拦截式还是注入
- `Jssdk`：根据`Webview`的`UI`信息判断使用拦截式/注入式的调用方式
- `Native sdk`：根据`jssdk`规范提供拦截式/注入式的通讯方案，判断`method`是否存在，调用方是否有权限，拼装返回数据
- `IOS/Android`：`method`对应具体的业务逻辑，不用考虑调用的方式是拦截还是注入。





