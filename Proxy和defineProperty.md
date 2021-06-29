## defineProperty的缺陷
1. defineProperty只能劫持对象的属性，如果对象的属性是对象，那么还需要做更深一层的遍历。具有一定的监控数组下标变化的能力，但是其性能比较差。
2. proxy是对整个对象的代理，还可以代理动态新增的属性，并且会返回一个新的对象



## websocket如何建立连接？
1. 客户端发起http get请求，请求协议升级为websocket
2. 服务端响应101， 自此建立websocket连接






