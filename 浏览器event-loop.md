## 什么是Event Loop？

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/660f46bfb6384364a25f20282f3c6321~tplv-k3u1fbpfcp-watermark.image)

event loop 核心的组成是执行栈和任务队列，执行栈是一个储存函数调用的栈结构，js代码会被压栈执行，执行完之后弹栈，任务队列储存着已经准备好的异步任务，分为宏任务队列和微任务队列，script、setTimeout和setIntervial回调，I/O操作等会加入宏任务队列，而Promise.then等会被加入微任务队列。宏任务队列一次执行一个宏任务，微任务队列则是整队执行。
1. 一开始执行栈，微任务队列都是空的，只有宏任务队列存在script脚本（整体代码）一个宏任务，
2. script脚本入栈，开始执行同步代码（在执行过程中碰到异步操作时，会把异步操作交给对应的线程执行，执行完成后会由事件触发线程加入任务队列）会产生新的宏任务和微任务，在执行完成之后，script脚本出栈，
3. 接下来会执行微任务队列，微任务队列中一个个微任务入栈并被执行，直到微任务队列被清空
4. 接下来执行渲染操作，更新界面
5. 检查是否有webworker任务，如果有进行处理
6. 以上步骤循环往复，直到两个任务队列都被清空