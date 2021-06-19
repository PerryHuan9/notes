<div align="center" style="margin-top: 40px;">
    <img width="500" src="https://nodejs.dev/static/nodejs-logo-light-mode-e8344f71081da53be8ee1098584a0ab6.svg">
</div>

## node 是什么？
node是一个JavaScript运行时环境。它使用Chrome的V8引擎解释执行JS， 并使用libuv封装不同平台的异步I/O。

node.js使用事件驱动机制： 它有一个事件循环线程负责任务的编排，和一个专门处理繁杂任务的工作线程池。

node.js 擅长于I/O密集型任务，拥有非常高的吞吐量，但是对于复杂的计算，它可能不是一种好的选择。



## node Event loop
node Event loop 是一种任务编排的机制， 主要是处理非阻塞I/O任务，node.js在事件循环线程即主线程中完成Event loop。

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0093ebf60c79468c9bfbca615cff5f84~tplv-k3u1fbpfcp-watermark.image)

有6个阶段：
* 定时器： 执行`setTimeout`和`setInterval`的回调
* 待定回调： 执行延迟到下个事件循环的I/O回调。
* idle,prepare: 仅系统内部使用。
* 轮询： 检索I/O事件，执行与I/O相关的回调，几乎所有的情况的回调都在这里执行，除了定时器
、`setImmediate`以及关闭的回调函数，其它情况node将在适当时候在这里阻塞。
* 检测： 执行`setImmediate`的回调.
* 关闭的回调函数： 一些关闭的回调函数，，如：`socket.on('close', () => {})`。

## 工作线程池

尽管node.js是单线程执行的，但是当有可能的时候，它会把部分操作转移到其它内核。nodejs提供了具有`k`个线程的工作线程池，工作线程池是通过libuv实现的。它主要用来执行“高成本”的任务，这包括一些没有提供非阻塞版本的I/O操作，和一些CPU密集型（简单理解是有很高的计算成本，很耗CPU的）的任务。

Node模块中有如下API用到了工作线程池：

* I/O 密集型任务：
    1. DNS：`dns.lookup()`，`dns.lookupService()`。
    2. 文件系统：所有的文件系统 API。除 `fs.FSWatcher()` 和那些显式同步调用的 API 之外，都使用 libuv 的线程池。

* CPU 密集型任务：

    1. Crypto：`crypto.pbkdf2()`、`crypto.scrypt()`、`crypto.randomBytes()`、`crypto.randomFill()`、`crypto.generateKeyPair()`。
    2. Zlib：所有 Zlib 相关函数，除那些显式同步调用的 API 之外，都适用 libuv 的线程池。

## 阻塞与非阻塞


