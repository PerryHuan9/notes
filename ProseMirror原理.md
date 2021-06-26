
<div align="center">
    <img width="400" src="https://st0.dancf.com/csc/1681/configs/system/20210512-113204-374b.svg">
</div>

## ProseMirror是什么 ？
ProseMirror提供了一整套搭建富文本编辑器的工具，它的设计哲学是：你能掌握文档的一切。那么它是怎么是实现这一点的呢？它有四个核心库 `prosemirror-state`、`prosemirror-view`、`prosemirror-model`、`prosemirror-transform`。首先，`prosemirror-state`用以实现文档的`state`， `state`对应文档当前的状态，这个状态包含文档中的所有内容，以及光标的选区信息，在编辑器中输入内容，state会改变，通过代码修改文档也是通过变更state，新state在旧state的基础上建立。state本质上就是一个大对象，它下面有三大块内容，`doc`,`selection`和`transaction`
### `doc` 虚拟DOM
`doc`就是文档树，它是编辑器的`虚拟dom`, 在这棵树上有不同类型的`node`(块)。比如有标题块，段落块，图片块，表格块等等。这些块是我们使用`prosemirror-model`这个包定义。每个块在定义的时候需要声明它的内容类型，即它允许装什么内容，另外需要通过`toDom`方法定义它应该渲染成什么样的`html`， 块的渲染是借助`prosemirror-view`, 它负责把`doc`渲染成对应的`HTML`和响应用户的操作。

### `selection` 光标系统
`prosemirror`的`selection`与浏览器的`selection`类似，但实际上它们指示的是不同文档的位置，浏览器的`selection`表示的是在`html`文档上的位置，而`prosemirror`的`selection`则表示的是在`doc虚拟文档`的位置，通过`selection`可以选中某一段文本，可以选中某个块，还可以选中多个块。选中文本或块之后就可以对文档进行增删改查的修改，那么修改又是怎么实现的呢？

### transaction 修改文档
修改文档的功能由`prosemirror-transform`包提供，这个包提供了一个核心类`Transform`， state下的`transaction`是这个类的实例，之所以挂在state下，是为了操作方便。所有的修改操作，添加新的块，更新现有的块，删除块，修改光标都是通过`transaction`，将`transaction` `dispatch`出去就能修改`state`， 让文档过渡到下一个状态。`transform`还有另外一个重要的功能，为协同提供支持，当前用户将它的`transaction`发给其它用户，其它用户将这些`transaction`应用到文档上，就能保持文档状态的一致。

### 总结
所以`state`是`prosemirror`的核心，所有的操作都是围绕着`state`来进行，`prosemirror-model`用于定义`state`的doc, `prosemirror-view`将`state`渲染成`html`， `prosemirror-transform`提供`transaction`用来修改`state`.





