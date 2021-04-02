# redux源码解析

> 本篇基于4.0.4版本展开。

`redux` 是react技术生态中至关重要的一环，许许多多的应用都是基于此构建而成。为什么它如此重要且流行呢？让我们通过源码的阅读来了解其中精巧的设计与深邃的思想吧。

## 为什么需要redux

万物皆有因，redux的诞生也不例外。

React作者、 Redux 的创建者之一 Dan Abramov 说过:

> 我想修正一个观点：当你在使用 React 遇到问题时，才使用 Redux。

随着单页应用开发日趋复杂，**JavaScript 需要管理比任何时候都要多的 state （状态）**。 这些 state 可能包括服务器响应、缓存数据、本地生成尚未持久化到服务器的数据，也包括 UI 状态，如激活的路由，被选中的标签，是否显示加载动效或者分页器等等。

管理不断变化的 state 非常困难。特别是当一个model会引起另一个model变化时，或者state的变化隐藏在各种异步流程中时，**state 在什么时候，由于什么原因，如何变化已然不受控制。**

这时候`redux`应运而生。

先来看官网对redux的定位：

> Redux 是 JavaScript 状态容器，提供可预测化的状态管理。

显而易见。Redux的目的就是让state的变化变得可预测。



## 三大原则

如何让state的变化变得可预测呢？人类社会是需要法律规范来维持秩序的，redux的世界里同样需要规范，简称三大原则。

- 单一数据源

  整个应用的 state 被储存在一棵 object tree 中，并且这个 object tree 只存在于唯一一个 store 中。

- State是只读的

  唯一改变 state 的方法就是触发 action，action 是一个用于描述已发生事件的普通对象。

  有的小伙伴说我直接赋值修改store行不行，当然可以，但是年轻人不能不讲武德啊，这样修改是无法触发store的listener执行的，视图也就不会即时刷新🙄️。

- 使用纯函数执行修改

  如果把action看做是命令的话，纯函数reducer就是执行命令的士兵了。reducer接收先前的 state 和 action，并返回新的 state。

  

## redux使用示例

在分析源码之前，让我们先看一个简单的示例热热身，温习一下redux的使用。

**reducer**

```typescript
export default (state = 0, action) => {
  switch (action.type) {
    case 'INCREMENT':
      return state + 1
    case 'DECREMENT':
      return state - 1
    default:
      return state
  }
}
```

**middleware**

```typescript
export const logger = store => next => action => {
  console.info('dispatching', action)
  let result = next(action)
  console.log('next state', store.getState())
  return result
}
```

**store**

```typescript
import { createStore, applyMiddleware } from 'redux'
import { logger } './middleware'
import reducer from './reducer'

const middleware = [ logger ];
const initState = {}

const store = createStore(
  reducer,
  initState,
  applyMiddleware(...middleware)
)
```

以上就是创建一个store的核心流程。



## 源码解析

`redux`的源码相对简单，闲话少叙，直接看代码。

### createStore

```typescript
export default function createStore (reducer, preloadedState, enhancer) {
  // 如果有中间件注入，则传入createStore，由enhancer去创建store
  if (typeof enhancer !== 'undefined') {
    return enhancer(createStore)(
      reducer,
      preloadedState
    )
  }
  
  let currentReducer = reducer
  let currentState = preloadedState as S
  let currentListeners: (() => void)[] | null = []
  let nextListeners = currentListeners
  let isDispatching = false
  
  function ensureCanMutateNextListeners() {
    // ...
  }
  
  function getState(): S {
    // ...
  }
  
  function subscribe(listener: () => void) {
    // ...
  }
  
  function dispatch(action: A) {
    // ...
  }
  
  function replaceReducer(nextReducer: Reducer<NewState, NewActions>){
    // ...
  }
  
  function observable() {
    // ...
  }
  
  // 初始化，生成state tree
  dispatch({ type: ActionTypes.INIT } as A)

  const store = ({
    dispatch: dispatch as Dispatch<A>,
    subscribe,
    getState,
    replaceReducer,
    [$$observable]: observable
  })
  return store
}
```

可以看出createStore方法没有什么复杂逻辑，主要是声明了一堆函数，然后返回包含这些方法的store对象，属于闭包的典型应用。下面逐个看各个方法。

#### subscribe

此方法用来订阅dispatch方法的回调函数，每次dispatch后都会执行一遍listeners。

```typescript
function subscribe(listener: () => void) {
    if (typeof listener !== 'function') {
      throw new Error('Expected the listener to be a function.')
    }
    // 不能在dispatch过程中subscribe
    if (isDispatching) {
      throw new Error()
    }

    let isSubscribed = true
    // 生成快照队列
    ensureCanMutateNextListeners()
    // push进快照队列
    nextListeners.push(listener)

    return function unsubscribe() {
      if (!isSubscribed) {
        return
      }
      isSubscribed = false
      // 如果已经dispatch，则生成新的快照队列，并从新的快照队列中移除，等下一次dispatch才会起效果
      ensureCanMutateNextListeners()
      const index = nextListeners.indexOf(listener)
      nextListeners.splice(index, 1)
      currentListeners = null
    }
  }
```

难点在于理解ensureCanMutateNextListeners函数的作用，下面会详细解析。

#### dispatch

```typescript
function dispatch(action: A) {
    // ...
    try {
      isDispatching = true
      // 执行reducer，返回最新state
      currentState = currentReducer(currentState, action)
    } finally {
      isDispatching = false
    }
	  // 同步快照队列到执行队列currentListeners，执行队列不可变
    const listeners = (currentListeners = nextListeners)
    for (let i = 0; i < listeners.length; i++) {
      const listener = listeners[i]
      listener()
    }

    return action
  }
```

#### ensureCanMutateNextListeners

```typescript
function ensureCanMutateNextListeners() {
  if (nextListeners === currentListeners) {
    // 浅拷贝一份作为快照队列
    nextListeners = currentListeners.slice()
  }
}
```

这个方法目的就是维持listeners队列的可靠性。

我们知道，在dispatch一个action后，自然会执行一系列的listeners。假设一个场景，在执行某个listener的过程中，又往listeners队列里面push一个新的listener，此时新listener是不应该被这个action所触发的。

所以此时只维护一个队列是不够的，需要维护一个快照队列`nextListeners`，每次subscribe一个listener之时，都先push到快照队列`nextListeners`，等到dispatch之后，取出该快照赋值为`currentListeners`。这样就做到了每个dispatch都对应一份确定的listeners列表。

下面举例解释：

```typescript
let unsubscribe = null;
const listener1 = () => {
  console.log('listener1');
  // subscribe
  store.subscribe(listener2);
}
const listener2 = () => {
  console.log('listener2');
  unsubscribe();
}
unsubscribe = store.subscribe(listener1);
store.dispatch(); // listener1
store.dispatch(); // listener1，listener2
store.dispatch(); // listener2
```



#### observable

```typescript
function observable() {
    const outerSubscribe = subscribe
    return {
      // 迷你版观察订阅模式，传入一个有next方法等对象，用来接收state
      subscribe(observer: unknown) {
				// ...
        function observeState() {
          const observerAsObserver = observer
          if (observerAsObserver.next) {
            // 用next方法接收state
            observerAsObserver.next(getState())
          }
        }

        observeState()
        // 真正订阅
        const unsubscribe = outerSubscribe(observeState)
        return { unsubscribe }
      },

      [$$observable]() {
        return this
      }
    }
  }
```



### combineReducers

从createStore小节可知，该方法只接收一个reducer函数参数。但是实际开发时候，我们的type是多种多样的，如果都通过一个reducer方法来处理，可以想象该方法将无限膨胀，直至无法阅读与维护。

这时候就需要将reducer拆分开来，然后通过combineReducers方法组合成一个reducer函数。先看其用法：

```typescript
import { reducer1, reducer2, reducer3 } = './reducers'

const rootReducer = combineReducers({
  reducer1,
  reducer2,
  reducer3,
})
const store = createStore(
  rootReducer,
)
```

下面看源码：

```typescript
export default function combineReducers(reducers: ReducersMapObject) {
  const reducerKeys = Object.keys(reducers)
  const finalReducers: ReducersMapObject = {}
  // 过滤掉不合法reducer，保证reducer为函数
  for (let i = 0; i < reducerKeys.length; i++) {
    const key = reducerKeys[i]
    if (typeof reducers[key] === 'function') {
      finalReducers[key] = reducers[key]
    }
  }
  const finalReducerKeys = Object.keys(finalReducers)

  // 校验reducer是否合法，initialState不能为undefined
  // ...

  return function combination(
    state: StateFromReducersMapObject<typeof reducers> = {},
    action: AnyAction
  ) {
    // 校验与警告提示
    // ...

    let hasChanged = false
    // 用空对象存值
    const nextState: StateFromReducersMapObject<typeof reducers> = {}
    // 逐一遍历
    for (let i = 0; i < finalReducerKeys.length; i++) {
      const key = finalReducerKeys[i]
      const reducer = finalReducers[key]
      // 前值
      const previousStateForKey = state[key]
      // 后值
      const nextStateForKey = reducer(previousStateForKey, action)
      // 值不能为undefined
      if (typeof nextStateForKey === 'undefined') {
        const errorMessage = getUndefinedStateErrorMessage(key, action)
        throw new Error(errorMessage)
      }
      // 更新值
      nextState[key] = nextStateForKey
      // 判断是否值有变化
      hasChanged = hasChanged || nextStateForKey !== previousStateForKey
    }
    hasChanged =
      hasChanged || finalReducerKeys.length !== Object.keys(state).length
    // 如果有变化，则返回新对象
    return hasChanged ? nextState : state
  }
}
```



### applyMiddleware

中间件的应用是redux源码最精巧的设计，短短几十行代码，给redux功能的拓展提供无限的可能。

```typescript
export default function applyMiddleware(
  ...middlewares: Middleware[]
) {
  return (createStore) => (reducer, preloadedState) => {
    // 这里创建store
    const store = createStore(reducer, preloadedState)
    // 定义初始dispatch，后面被重写
    let dispatch: Dispatch = () => {}

    const middlewareAPI: MiddlewareAPI = {
      getState: store.getState,
      // 传入的dispatch已经被重写
      dispatch: (action, ...args) => dispatch(action, ...args)
    }
    // 中间件初始化，传入middlewareAPI
    const chain = middlewares.map(middleware => middleware(middlewareAPI))
    // 重写dispatch
    dispatch = compose<typeof dispatch>(...chain)(store.dispatch)

    return {
      ...store,
      dispatch
    }
  }
}
```

这里涉及到一个功能强大的函数`compose`:

```typescript
export default function compose(...funcs: Function[]) {
  if (funcs.length === 0) {
    return <T>(arg: T) => arg
  }

  if (funcs.length === 1) {
    return funcs[0]
  }

  return funcs.reduce((a, b) => (...args: any) => a(b(...args)))
}
```

顾名思义，compose是组合的意思，也就是把一系列函数组合成一个函数，统一执行。

看个小例子：

```typescript
const a = (action) => {
  console.log('a')
  console.log('receive: ' + action)
  console.log('======')
  return 'a'
}
const b = (action) => {
  console.log('b')
  console.log('receive: ' + action)
  console.log('======')
  return 'b'
}
const c = (action) => {
  console.log('c')
  console.log('receive: ' + action)
  console.log('======')
  return 'c'
}
const fn = compose([a,b,c]);
fn();
/**
c
receive: start
======
b
receive: c
======
a
receive: b
======
*/
```



## 中间件演变过程

我们知道redux中间件作用的实质是对dispatch方法的增强，在执行真正dispatch方法之前，可以注入各种逻辑，提供了无限拓展的接口。

所以可以换个说法，**中间件实现的过程也就是对一个函数增强的过程**。

现在我们从零开始推导其过程。

假设有个方法dispatch，我们想对其添加一点功能。

```typescript
let dispatch = (action) => console.log(action)
```

最直接的思路就是，重写该方法：

```typescript
const dispatch = (action) => console.log(action)
const extendDispatch = (action) => {
  // 添加逻辑 ...
  dispatch(action)
  // 添加逻辑 ...
}
```

如果只是想增一个功能自然已经满足，但是如果想增强多个功能呢？我们自然不想把所有的功能都写在一个方法里面，而且也不现实，因为我们还想直接使用别人已经实现的功能，不想重写一遍。

自然而然，便想到提供一个方法，通过传入一个函数，返回一个全新的增强后的函数：

```typescript
const extendDispatch = (dispatch) => (action) => {
  // ...
  dispatch(action)
  // ...
}
dispatch = extendDispatch(dispatch)
```

这样有什么好处呢，假设我们实现了多个`extendDispatch`，则只需要迭代执行所有的`extendDispatch`方法，就能得到一个增强后的dispatch。

```typescript
let dispatch = (action) => console.log(action)

let i = 0;
const extendDispatch = (fn) => (arg) => {
  const j = i++;
  console.log('begin ====' + j + '====')
  fn(arg)
  console.log('end ====' + j + '====')
}

for (let i = 0; i < 3; i++) {
   // 迭代增强
   dispatch = extendDispatch(dispatch) 
}
// 最终被增强后的dispatch
dispatch('exec action')

/**
begin ====0====
begin ====1====
begin ====2====
exec action
end ====2====
end ====1====
end ====0====
*/
```

我们可以实现各种各样的`extendDispatch`方法，然后自定义排序组合即可实现dispatch方法的增强。

前面迭代增强的过程也可以看作是compose的过程，

```typescript
dispatch = compose([extendDispatch, extendDispatch, extendDispatch])(dispatch)
dispatch('exec action')
```



















