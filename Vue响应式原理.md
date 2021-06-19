
## 响应式是什么？
所谓响应式就是设置好了一件依赖某个状态的事情，在状态改变的时候，这个事情能自动重新执行一遍，并且执行的结果取决于最新状态。
放到`Vue`中则是，某个`data`变量`dataVar`（视图依赖改变量）改变，视图重新渲染，并且渲染结果取决于`dataVar`。

我们都知道在`Vue`中无论用模板还是使用`jsx`，编译输出的都会是 `render function`，模板或`jsx`中使用的`data`变量也会被放在 `render function`中，也就是说所谓的渲染可以简单理解为执行`render`函数。那么响应式更新的问题可以转换为： 在一个函数中使用某个变量，在该变量改变的时候，这个函数重新执行一遍。



## 实现响应式
上文提到，响应式的实质是： 在一个函数中使用某个变量，在该变量改变的时候，这个函数重新执行一遍。
需求已经很清楚了，那么应该怎么实现这个需求呢？

我们可以接着分解：
1. 要让函数重新执行，我们首先需要知道哪些函数使用了我们这个变量， 也就是要拿到所有使用了这个变量的函数。
2. 要在变量更新的时候执行这个函数，我们需要知道变量什么时候更新，也就是变量更新要通知我。

`JS`恰好给我们提供了访问变量的`get`钩子和修改变量的`set`钩子，其中在`ES5`中可以是`Object.defineProperties(obj, props)`, `ES6`中则提供了功能更为强大的`Proxy`。下面实现一下响应式：

`effect.ts`
```ts
type Dep = Set<ReactiveEffect>;

interface ReactiveEffect {
  (): any;
  deps: Dep[];
}

/**
 * 当前的 reactiveEffect 函数，
 */
let activeEffect: ReactiveEffect | undefined;

/**
 * 保存了所有的 （对象 -> reactiveEffect）映射
 * 比如有一个响应式对象 obj = {a: 1, b: 2}, 那我们需要知道有多少副作用函数使用obj.a,
 * 又有多少副作用函数使用了obj.b, 所以targetMap的键就是obj,
 */
const targetMap: WeakMap<any, KeyToDepMap> = new WeakMap();

/**
 * 对象的（属性 -> 依赖） 映射
 */
type KeyToDepMap = Map<any, Dep>;

/**
 * 追踪一个依赖
 * @param target 一个对象
 * @param key 对象的属性
 */
export function track(target: object, key: unknown) {
  if (!activeEffect) return;
  let depMap = targetMap.get(target);
  if (!depMap) {
    targetMap.set(target, (depMap = new Map()));
  }
  let dep = depMap.get(key);
  if (!dep) {
    depMap.set(key, (dep = new Set()));
  }
  if (!dep.has(activeEffect)) {
    dep.add(activeEffect);
    activeEffect.deps.push(dep);
  }
}

/**
 * 在依赖改变的时候执行该函数以通知观察者执行
 * @param target
 * @param key
 */
export function trigger(target: object, key: unknown) {
  const depMap = targetMap.get(target);
  if (!depMap) return;
  const dep = depMap.get(key);
  dep?.forEach((effect) => effect());
}

/**
 * 使fn具备副作用
 * @param fn
 */
export function effect(fn: () => unknown) {
  const effectFn: ReactiveEffect = () => {
    try {
      activeEffect = effectFn;
      return fn();
    } finally {
      activeEffect = undefined;
    }
  };
  effectFn.deps = [];
  effectFn();
  return effectFn;
}

```
`reactive.ts`

```ts
import { track, trigger } from "./effect";

/**
 * 使一个对象具备响应式
 * 实现过程是使用proxy代理target对象，然后在get钩子中track， 在set钩子中trigger
 * @param target
 */
export function reactive<T extends object>(target: T): T {
  const proxyObj = new Proxy(target, {
    get(target, key, receiver) {
      const res = Reflect.get(target, key, receiver);
      //   收集依赖了proxyObj变量的 reactiveEffect函数
      track(target, key);
      return res;
    },
    set(target, key, value, receiver) {
      const res = Reflect.set(target, key, value, receiver);
      // 触发依赖proxyObj对象的 reactiveEffect函数更新
      trigger(target, key);
      return res;
    },
  });
  return proxyObj;
}
```
`index.ts`
```ts
import { effect } from "./effect";
import { reactive } from "./reactive";

const obj = reactive({ key: "number" });

const handle = () => {
  console.log(obj.key);
}

const effectHandle = effect(handle);

obj.key = "object";

setTimeout(() => {
  obj.key = "boolean";
}, 1000);

// 打印结果
// number
// object
// boolean

```
在以上的测试中：
1. 首先使用`reactive`函数包装普通对象，生成一个响应式变量`obj`。
2. 建一个函数`handle`，函数中存在`console.log(obj.key);`，即依赖了`obj`变量。
3. 使用了一个`effect`函数包裹`handle`函数生成`effectHandle`，以使`handle`函数具备副作用（`obj.key`改变的时候会重新执行）。
4. 最后两次修改`obj.key`的值。

以上测试结果是`handle`函数打印了三次, 在`obj.key`两次改变后都能重新执行`handle`函数，并且打印出了正确的结果。

显然上面的实现还是有很多问题，比如不能实现数组响应式更新，但其中一个比较大的问题是：
```ts
const obj = reactive({ key: "number" });
const obj1 = reactive({ key: "number" });

effect(() => {
  effect(() => {
    const d = obj1.key;
  });
  const b = obj.key;
});
```
当`effect`中再使用`effect`的时候，没有方法拿到正确的结果，这是因为`effect.ts`中的`acctiveEffext`在`effect`内部的`effect`执行的时候被修改了，以至于不能正确收集到依赖，可以通过维护一个栈`effectStack`来解决这个问题，`Vue`内部正是这样处理的。






## Vue响应式










