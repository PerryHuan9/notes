# CORS 原理
CORS全称Cross-Origin Resource Sharing, 即跨域资源共享，是一个由一系列HTTP头组成的系统，这些HTTP头决定浏览器是否阻止前端`javascript`代码获取跨域请求的响应。为什么需要CORS ？ 这是因为浏览器存在同源安全策略，当我们在当前域请求另外一个域的资源时，浏览器默认会阻止脚本读取它的响应，这时CORS就有了用武之地。

## 同源策略
同源策略是浏览器的一个重要的安全策略，它用于限制一个源的文档或其加载的脚本如何与另外一个源进行交互，它能够隔绝恶意文档，减少被攻击的媒介。

### 同源的定义

如果两个URL的协议、主机名和端口号都是相同的，那么这两个URL就是同源的，

下表给出了与 URL http://store.company.com/dir/page.html 的源进行对比的示例:
<table>
 <thead>
    <tr>
        <th>URL</th>
        <th>结果</th>
        <th>原因</th>
    </tr>
 </thead>
 <tbody>
 <tr>
    <td>http://store.company.com/dir2/other.html</td>
    <td>同源</td>
    <td>只有路径不同</td>
 </tr>
 <tr>
    <td>https://store.company.com/secure.html</td>
    <td>非同源</td>
    <td>协议不同</td>
 </tr>
 <tr>
    <td>http://store.company.com:81/dir/etc.html</td>
    <td>非同源</td>
    <td>端口号不同</td>
 </tr>
 <tr>
    <td>http://news.company.com/dir/other.html</td>
    <td>非同源</td>
    <td>主机名不同</td>
 </tr>
 <tbody>
</table>

也就是说当在`http://store.company.com/dir/page.html`这个网站中向`https://store.company.com`、`http://store.company.com:81`和`http://news.company.com`三个地址发起`AXJX`请求都会失败并且会报跨域的错误。这就是浏览器的同源策略，只能访问同源的数据。

### 源的修改
实际上，在一个源下可以修改当前源，比如在`http://hello.word.com/index.html`下，有一个脚本执行了以下语句：
```js
document.domain = 'word.com'
```
那么当前网站能与`http://word.com`通过同源检测，即允许访问`http://word.com`的数据。<b>需要注意的是`document.domain`只能设置为当前域的父域, 并且在设置之后，端口号会变成null</b>，例如还是在`http://hello.word.com/index.html`下执行以下语句会失败：
```js
document.domain = 'hello2.word.com'
```

### 跨源网络访问
事实上，不是所有跨源操作都被禁止。跨源操作主要分为三大类，<b>跨源写操作</b>、<b>跨源资源嵌入</b>和<b>跨源读操作</b>。
* <b>跨源写操作</b>一般是被允许的。如超链接、重定向、表单提交。
* <b>跨源资源嵌入</b>一般也是被允许的。如嵌入`<img />`、`<video />`、`<audio />`等。
* <b>跨源读操作</b>不被允许，要实现跨源读操作，可以使用<b>JSONP</b>或<b>CORS</b>。


## 跨源访问
跨域（跨源）的解决方案主要有两种，`JSONP`和`CORS`, 当然还可以是代理或者反代理的手段。

### JSONP
`JSONP`是一种`hack`，并不是一种官方解决跨域的手段，JSONP只能进行`GET`请求。它主要是利用`<script>`标签来发起请求，来达到突破浏览器同源策略的目的。

```html
<html>
  <head>
    <title>JSONP</title>
  </head>
  <body>
    <script>
      function myCallback(res) {
        console.log("result:", res);
      }
    </script>
    <!-- 发起请求 -->
    <script src="http://localhost:8080/getData?callback=myCallback"></script>
  </body>
</html>
```

后端服务实现：
```js
const http = require('http');

var data = {
    name: 'BruceLee',
    password: '123456'
};

const server = http.createServer((request, response) => {
    console.log('url:', request.url)
    if (request.url.startsWith('/getData')) {
       // 取到callback参数
        const callbackFn = request.url.split('=')[1];
        response.writeHead(200, {
            'Content-Type': 'application/json;charset=utf-8'
        });

        // 返回一段 JavaScript 代码
        response.end(`${callbackFn}(${JSON.stringify(data)})`);
    }
});

server.listen(8080, () => {
    console.log('The server is running at http://localhost:8080');
});

```
其原理非常简单：
1. 前端使用`<script>`向`http://localhost:8080/getData?callback=myCallback`（这里的url携带callback参数是为了告诉后端，前端回调函数的名字是myCallback）请求一个js脚本，其实就是一段字符串。
2. 后端取到它的`callback`参数，然后返回 `myCallback({name: 'BruceLee',password: '123456'})`。
3. 前端`script`标签在拿到脚本后自然会执行脚本，也就是执行`myCallback({name: 'BruceLee',password: '123456'})`,所以会调用`myCallback`函数。

### CORS
前文已指出：CORS是一套跨域资源共享机制，它基于HTTP头实现。那么什么场景需要用到CORS? 它具体是怎样工作？又使用了哪些HTTP头？

#### 用到CORS的场景
有四类场景需要用到CORS，
1. 使用`XMLHttpRequest` 或 `Fetch` 发起的跨源 HTTP 请求。
2. Web 字体 (CSS 中通过 `@font-face `使用跨源字体资源)。
3. `WebGL` 贴图。
4. 使用 `drawImage` 将 Images/video 画面绘制到 canvas。

#### CORS工作过程
CORS对那些可能对服务器数据产生副作用的请求需要在请求之前发送一个`OPTIONS`预检请求，从而获知服务器是否允许该跨源请求。我们这里把那些 '不需要发预检请求' 的请求称为简单请求，满足以下条件的请求被称为简单请求。
1. 使用`GET`、`HEAD`、`POST`。
2. 请求头仅包含允许人为设置的在Fetch规范定义的对CORS安全的字段集合。该集合为：
   * `Accept`
   * `Accept-Language`
   * `Content-Language`
   * `Content-Type` （值仅限于`text/plain`, `multipart/form-data`,`application/x-www-form-urlencoded`）
3. 请求中的任意`XMLHttpRequestUpload` 对象均没有注册任何事件监听器；
4. 请求中没有使用 ReadableStream 对象。

简单请求会直接发起请求，请求头会携带`origin`表明当前源，响应头需要返回`Access-Control-Allow-Origin: *`, `*`通配符表示允许任何源请求数据，可以使用具体的值来限制访问数据的源，比如当为`Access-Control-Allow-Origin: http://foo.example`,则表示只允许`http://foo.example`这个域访问数据。必须注意：<b>当xhr请求设置`withCredentials`为true，即请求携带`cookies`时，`Access-Control-Allow-Origin`的值不能为`*`，必须为具体的某个域, 而且响应头还必须存在`Access-Control-Allow-Credentials: true`，否则请求会失败。</b>

与简单请求不同，“需预检的请求”要求必须首先使用 `OPTIONS` 方法发起一个预检请求到服务器,以获知浏览器是否允许该请求。

首先在预检请求头中使用
```http
Access-Control-Request-Method: POST
Access-Control-Request-Headers: X-PINGOTHER, Content-Type
```
* `Access-Control-Request-Method` 告知服务器，实际请求将使用 POST 方法。
* `Access-Control-Request-Headers` 告知服务器，实际请求将携带两个自定义请求首部字段：`X-PINGOTHER` 与 `Content-Type`。服务器据此决定，该实际请求是否被允许。

预检请求响应头会包含:
```http
Access-Control-Allow-Origin: http://foo.example
Access-Control-Allow-Methods: POST, GET, OPTIONS
Access-Control-Allow-Headers: X-PINGOTHER, Content-Type
Access-Control-Max-Age: 86400
```
* `Access-Control-Allow-Methods`表明服务器允许客户端使用 POST, GET 和 OPTIONS 方法发起请求。
* `Access-Control-Allow-Headers` 表明服务器允许请求中携带字段 `X-PINGOTHER` 与 `Content-Type`。
* `Access-Control-Max-Age` 表明该响应的有效时间为 86400 秒，也就是 24 小时。在有效时间内，浏览器无须为同一请求再次发起预检请求。请注意，浏览器自身维护了一个最大有效时间，如果该首部字段的值超过了最大有效时间，将不会生效。

#### CORS HTTP头
上面已经介绍了一些用于CORS的HTTP头，下面再简单归纳一下：

CORS HTTP 请求头字段

* `Origin`字段表明预检请求或实际请求的源站。
* `Access-Control-Request-Method`用于预检请求。其作用是，将实际请求所使用的 HTTP 方法告诉服务器。
* `Access-Control-Request-Headers` 用于预检请求。其作用是，将实际请求所携带的首部字段告诉服务器。


CORS HTTP 响应头字段

* `Access-Control-Allow-Origin` 值指定了允许访问该资源的外域 URI。
* `Access-Control-Expose-Headers`在跨源访问时，`XMLHttpRequest.getResponseHeader()`方法只能拿到一些最基本的响应头，`Cache-Control`、`Content-Language`、`Content-Type`、`Expires`、`Last-Modified`、`Pragma`，如果要访问其他头，则需要服务器设置本响应头。
* `Access-Control-Max-Age`头指定了预检请求的结果能够被缓存多久。
* `Access-Control-Allow-Credentials`,当请求的`credentials`设置为true时是否允许浏览器读取response的内容。当用在对preflight预检测请求的响应中时，它指定了实际的请求是否可以使用credentials。
* `Access-Control-Allow-Methods`字段用于预检请求的响应。其指明了实际请求所允许使用的 HTTP 方法。
* `Access-Control-Allow-Headers`字段用于预检请求的响应。其指明了实际请求中允许携带的首部字段。


## 参考文档

[跨资源共享](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/CORS)















