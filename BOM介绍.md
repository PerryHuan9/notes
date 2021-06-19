## BOM是什么？
  BOM为浏览器对象模型，是与浏览器就行交互的所有接口的统称，主要有window、screen、navigator、history、location对象和localstorage与sessionstorage存储对象。
* Window是全局对象，所有的其它浏览器对象（如navigator，screen，history）都会作为属性挂在该对象下，该对象还有一些全局属性如outerHight。
* screen包含了客户端屏幕的信息，如width属性为屏幕宽度
* navigator则包含的用户浏览器信息，如发送给服务器的userAgent信息
* history包含了用户在浏览器中访问的URL，length属性为历史列表中的网址数，
    另有go(), back(),forward()，以及h5新增的pushState和replaceState方法。
* location对象包含当前URL的信息，如hash，host，pathname等属性。
* localStorage用于长久保存网站的数据，保存没有过期时间，直到手动删除。
* sessionStorage用于临时保存同一窗口或标签页的数据，在关闭窗口或标签页之后将会删除这些数据。
