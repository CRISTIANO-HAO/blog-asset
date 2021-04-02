# React-redux源码解析

> 本篇基于版本7.2.1展开。此文需要对redux有所了解，不了解redux的请先移步其官方文档。

## 为什么需要react-redux

使用一门技术之前，我们都要进行灵魂三问，“这是什么？为什么需要？解决了什么问题？”。只有知道其来龙去脉，才能有更深入的认识。

下面看官网介绍：

> **React Redux是React的官方Redux UI绑定库**。如果您同时使用Redux和React，则还应该使用React Redux绑定这两个库。

简单的说就是，redux是一个js状态管理库，react是用于构建用户界面的 js 库，react-redux则是**同步两者状态的桥梁**。

## 用法

将Redux与任何UI层一起使用需要相同的一致步骤：

1. 创建一个store
2. 订阅更新
3. 在订阅回调中：
   1. 获取当前store状态
   2. 提取此UI所需的数据
   3. 使用数据更新用户界面
4. 如有必要，以初始状态呈现UI
5. 通过`dispatch(action)`来响应UI输入

看简单示例：

```typescript
import React from 'react'
import ReactDOM from 'react-dom'
import { Provider } from 'react-redux'
import store from './store'
import App from './App'

const rootElement = document.getElementById('root')
// 注册store
ReactDOM.render(
  <Provider store={store}>
    <App />
  </Provider>,
  rootElement
)
```

```typescript
import { connect } from 'react-redux'
import { increment, decrement, reset } from './actionCreators'
import App from './App'

const mapStateToProps = (state /*, ownProps*/) => {
  return {
    counter: state.counter
  }
}
const mapDispatchToProps = { increment, decrement, reset }
// 绑定store与UI
export default connect(
  mapStateToProps,
  mapDispatchToProps
)(App)
```



## 原理

虽然react-redux代码也比较晦涩，但是原理是很简单的，我们通过把握核心原理，然后按图索骥，有利于阅读整体源码。

下面看[核心原理实现](https://gist.github.com/gaearon/1d19088790e70ac32ea636c025ba424e)：

```typescript
// connect() is a function that injects Redux-related props into your component.
// You can inject data and callbacks that change that data by dispatching actions.
function connect(mapStateToProps, mapDispatchToProps) {
  // It lets us inject component as the last step so people can use it as a decorator.
  // Generally you don't need to worry about it.
  return function (WrappedComponent) {
    // It returns a component
    return class extends React.Component {
      render() {
        return (
          // that renders your component
          <WrappedComponent
            {/* with its props  */}
            {...this.props}
            {/* and additional props calculated from Redux store */}
            {...mapStateToProps(store.getState(), this.props)}
            {...mapDispatchToProps(store.dispatch, this.props)}
          />
        )
      }
      
      componentDidMount() {
        // it remembers to subscribe to the store so it doesn't miss updates
        this.unsubscribe = store.subscribe(this.handleChange.bind(this))
      }
      
      componentWillUnmount() {
        // and unsubscribe later
        this.unsubscribe()
      }
    
      handleChange() {
        // and whenever the store state changes, it re-renders.
        this.forceUpdate()
      }
    }
  }
}
```

代码非常简单，就是通过`store.subscribe(this.handleChange.bind(this))`订阅state的更新，每次变化时就强制`forceUpdate`，从而更新整个视图。

这个实现其实就是整个react-redux所要做的事情，是不是非常简单？！

但是作为对易用性跟性能要求非常高的react基础库，这个实现过于简单粗暴了。state变化时，无论组件消费的那一部分state有没有变化，整个组件都要重新render一遍，这是无法接受的。

下面正式进入源码解析环节，让我们带着这个问题来一探究竟。



## 源码解析

### Provider

顾名思义，不难猜出这个函数干的就是Context.Provider要做的事。

```typescript
function Provider({ store, context, children }) {
  // provider提供的值
  // store变化时才变化，一般不变化
  const contextValue = useMemo(() => {
    const subscription = new Subscription(store)
    // state变化时执行onStateChange，也就是触发notifyNestedSubs
    subscription.onStateChange = subscription.notifyNestedSubs
    return {
      store,
      subscription
    }
  }, [store])
  
  // store变化时才变化，一般不变化
  const previousState = useMemo(() => store.getState(), [store])

  useEffect(() => {
    const { subscription } = contextValue
    // 订阅state变化
    subscription.trySubscribe()

    if (previousState !== store.getState()) {
      subscription.notifyNestedSubs()
    }
    return () => {
      subscription.tryUnsubscribe()
      subscription.onStateChange = null
    }
    // store变化时才变化，一般不变化
  }, [contextValue, previousState])

  const Context = context || ReactReduxContext
  // 注册store跟subscription给后面所有子组件共享
  return <Context.Provider value={contextValue}>{children}</Context.Provider>
}
```

从上可知，Provider主要是做了两件事情：

- 注册store跟subscription给后面所有子组件共享
- 订阅state变化

这里有个疑惑的点就是Subscription实例，它是如何订阅state变化的呢？下面继续看。



### Subscription

这是react-redux的一个核心类，它实现了对state的订阅更新逻辑。

```typescript
export default class Subscription {
  constructor(store, parentSub) {
    this.store = store
    this.parentSub = parentSub
    this.unsubscribe = null
    this.listeners = nullListeners
    this.handleChangeWrapper = this.handleChangeWrapper.bind(this)
  }
  // 添加监听者
  addNestedSub(listener) {
    this.trySubscribe()
    return this.listeners.subscribe(listener)
  }
  // 触发监听函数
  notifyNestedSubs() {
    this.listeners.notify()
  }
  // 封装处理变化统一接口，通过修改onStateChange来实现
  handleChangeWrapper() {
    if (this.onStateChange) {
      this.onStateChange()
    }
  }

  isSubscribed() {
    return Boolean(this.unsubscribe)
  }
  // 开始订阅
  trySubscribe() {
    if (!this.unsubscribe) {
      // 有父订阅容器的时候，监听函数添加到父容器
      this.unsubscribe = this.parentSub
        ? this.parentSub.addNestedSub(this.handleChangeWrapper)
        : this.store.subscribe(this.handleChangeWrapper)
      // 创建订阅者容器，下面细看
      this.listeners = createListenerCollection()
    }
  }
  // 取消订阅
  tryUnsubscribe() {
    if (this.unsubscribe) {
      this.unsubscribe()
      this.unsubscribe = null
      this.listeners.clear()
      this.listeners = nullListeners
    }
  }
}
```

这是一个订阅者容器的实现，为什么叫"容器"呢，因为所有需要监听state变化的函数（后面会知道是checkForUpdates函数）都会塞进`listeners`容器中，然后通过handleChangeWrapper监听state的变化，最后统一遍历listeners来更新。

listeners的实现也有一定的讲究，先来看实现：

```typescript
// encapsulates the subscription logic for connecting a component to the redux store, as
// well as nesting subscriptions of descendant components, so that we can ensure the
// ancestor components re-render before descendants
function createListenerCollection() {
  const batch = getBatch()
  let first = null
  let last = null

  return {
    clear() {
      first = null
      last = null
    },
    notify() {
      batch(() => {
        let listener = first
        // 从头到尾，从父到子依次触发调用
        while (listener) {
          listener.callback()
          listener = listener.next
        }
      })
    },
    get() {
      // ...
    },
    subscribe(callback) {
      let isSubscribed = true
      // 完整listener的封装，双向链表节点
      let listener = (last = {
        callback,
        next: null,
        prev: last
      })
      
      if (listener.prev) {
        // 从链表尾部插入节点
        listener.prev.next = listener
      } else {
        first = listener
      }

      return function unsubscribe() {
        if (!isSubscribed || first === null) return
        isSubscribed = false
        // 通过闭包保持对listener的引用
        // 删除双向链表节点
        if (listener.next) {
          listener.next.prev = listener.prev
        } else {
          last = listener.prev
        }
        if (listener.prev) {
          listener.prev.next = listener.next
        } else {
          first = listener.next
        }
      }
    }
  }
}
```

从上可知listeners是一个双向链表的结构，好处有两个：

- 保证顺序性，父组件的`checkForUpdates`要先于子组件的，从而保证父组件先于子组件re-render。
- 组件卸载，store更新，selector等变化时需要卸载listener，链表结构方便删除节点。



### Connect

#### API回顾

这个方法非常的绕，咋一看可以把人看的头晕眼花的，我们需要从一些基础api用法捋一捋，回头再看会轻松很多。

```typescript
function connect(mapStateToProps?, mapDispatchToProps?, mergeProps?, options?)
```

 `mapStateToProps` 跟 `mapDispatchToProps` 分别接收处理store的 `state` 跟 `dispatch` 作为方法的第一个参数传入，并分别返回`stateProps` 跟 `dispatchProps` 对象。

如果`mergeProps`方法有传入，则`stateProps` 跟 `dispatchProps`对象作为函数的第一个跟第二个参数传入。

```typescript
mapStateToProps?: (state, ownProps?) => Object
```

```typescript
mapDispatchToProps?: Object | (dispatch, ownProps?) => Object
```

```typescript
mergeProps?: (stateProps, dispatchProps, ownProps) => Object
```

```typescript
options?: Object
{
  context?: Object,
  pure?: boolean,
  areStatesEqual?: Function,
  areOwnPropsEqual?: Function,
  areStatePropsEqual?: Function,
  areMergedPropsEqual?: Function,
  forwardRef?: boolean,
}
```

#### 柯里化

connect方法大量使用了柯里化的技术，我们先简单认识一下。

看维基百科的解释：**柯里化**（Currying）是把接受多个参数的函数变换成接受一个单一参数（最初函数的第一个参数）的函数，并且返回接受余下的参数而且返回结果的新函数的技术。

文字可能不是很直观，直接看代码表示：

```typescript
// currying
f(a, b, c) => f(a)(b)(c)
```

这样有什么好处呢？

- 延迟计算
- 参数复用
- 动态生成函数

```typescript
const sum = (a,b,c) => a + b + c; 
// sum(1, 2, 3)

// 延迟计算
const delaySum = (a) => (b) => c => a + b + c; 
delaySum(1)(2)(3); // 6

// 参数复用
const sumOnTen = delaySum(10);
sumOnTen(1)(2); // 13
sumOnTen(2)(3); // 15
```

柯里化还有很多高级的用法，具体就不深入了，有兴趣的小伙伴可以自行去了解。



#### createConnect

我们知道connect方法返回值是一个高阶组件，最终返回的是一个绑定部分state数据、可以发起dispatch的函数集合的封装后的组件。

所以不难猜出其核心就是如何过滤跟比较数据，最大限度的避免组件无意义的重复渲染了。下面看筛减后的connect：

```typescript
export function createConnect({
  connectHOC = connectAdvanced,
  mapStateToPropsFactories = defaultMapStateToPropsFactories,
  mapDispatchToPropsFactories = defaultMapDispatchToPropsFactories,
  mergePropsFactories = defaultMergePropsFactories,
  selectorFactory = defaultSelectorFactory
} = {}) {
  return function connect(
    mapStateToProps,
    mapDispatchToProps,
    mergeProps,
    {
      pure = true,
      areStatesEqual = strictEqual,
      areOwnPropsEqual = shallowEqual,
      areStatePropsEqual = shallowEqual,
      areMergedPropsEqual = shallowEqual,
      ...extraOptions
    } = {}
  ) {
    
    // ...

    return connectHOC(selectorFactory, {
      // ...
      // passed through to selectorFactory
      initMapStateToProps,
      initMapDispatchToProps,
      initMergeProps,
      pure,
      areStatesEqual,
      areOwnPropsEqual,
      areStatePropsEqual,
      areMergedPropsEqual,
      // any extra options args can override defaults of connect or connectAdvanced
      ...extraOptions
    })
  }
}
export default /*#__PURE__*/ createConnect();
```

了解柯里化的思想后，我们很容易看懂这段代码了，主要是分段接收大量参数，方便后续参数复用。

继续看connectHOC（connectAdvanced），其返回值才是真正的高阶组件：

```typescript
export default function connectAdvanced(
  selectorFactory,
  {
    // ...
    // additional options are passed through to the selectorFactory
    ...connectOptions
  } = {}
) {
  return function wrapWithConnect(WrappedComponent) {
    // ...
    const selectorFactoryOptions = {
      ...connectOptions,
      // ...
    }

    function createChildSelector(store) {
      return selectorFactory(store.dispatch, selectorFactoryOptions)
    }
    
    // ...

    function ConnectFunction(props) {
      //...
      const childPropsSelector = useMemo(() => {
        return createChildSelector(store)
      }, [store])
      // 获取最终的props
      const actualChildProps = usePureOnlyMemo(() => {
        if (
          childPropsFromStoreUpdate.current &&
          wrapperProps === lastWrapperProps.current
        ) {
          return childPropsFromStoreUpdate.current
        }
        return childPropsSelector(store.getState(), wrapperProps)
      }, [store, previousStateUpdateResult, wrapperProps])

      // ...
      const renderedWrappedComponent = useMemo(
        () => (
          <WrappedComponent
            {...actualChildProps}
            ref={reactReduxForwardedRef}
          />
        ),
        [reactReduxForwardedRef, WrappedComponent, actualChildProps]
      )
			// ...
    }

    // If we're in "pure" mode, ensure our wrapper component only re-renders when incoming props have changed.
    const Connect = pure ? React.memo(ConnectFunction) : ConnectFunction

    // ....

    return hoistStatics(Connect, WrappedComponent)
  }
}
```

从上可以看出，获取最终props的过程：

```typescript
actualChildProps = selectorFactory(store.dispatch, selectorFactoryOptions)(store.getState(), wrapperProps)
```

这又是一个柯里化的写法。不难猜出，所有的秘密都在selectorFactory里面啦。selectorFactory内容比较多，另起一小节。



#### SelectorFactory

```typescript
export default function finalPropsSelectorFactory(
  dispatch,
  { initMapStateToProps, initMapDispatchToProps, initMergeProps, ...options }
) {
  // 生成被代理后的mapStateToProps，mapDispatchToProps，mergeProps
  const mapStateToProps = initMapStateToProps(dispatch, options)
  const mapDispatchToProps = initMapDispatchToProps(dispatch, options)
  const mergeProps = initMergeProps(dispatch, options)
  
  // pure默认是true
  const selectorFactory = options.pure
    ? pureFinalPropsSelectorFactory
    : impureFinalPropsSelectorFactory

  return selectorFactory(
    mapStateToProps,
    mapDispatchToProps,
    mergeProps,
    dispatch,
    options
  )
}
```

先看`impureFinalPropsSelectorFactory`，此时pure为false，比较少用：

```typescript
export function impureFinalPropsSelectorFactory(
  mapStateToProps,
  mapDispatchToProps,
  mergeProps,
  dispatch
) {
  return function impureFinalPropsSelector(state, ownProps) {
    return mergeProps(
      mapStateToProps(state, ownProps),
      mapDispatchToProps(dispatch, ownProps),
      ownProps
    )
  }
}
```

粗暴处理，直接调用mergeProps生成最终的props。

下面看`pureFinalPropsSelectorFactory`:

```typescript
export function pureFinalPropsSelectorFactory(
  mapStateToProps,
  mapDispatchToProps,
  mergeProps,
  dispatch,
   //比较策略
  { areStatesEqual, areOwnPropsEqual, areStatePropsEqual }
) {
  // 通过闭包保存上一次数据
  let hasRunAtLeastOnce = false
  let state
  let ownProps
  let stateProps
  let dispatchProps
  let mergedProps
  
  // 首次调用初始化数据
  function handleFirstCall(firstState, firstOwnProps) {
    state = firstState
    ownProps = firstOwnProps
    stateProps = mapStateToProps(state, ownProps)
    dispatchProps = mapDispatchToProps(dispatch, ownProps)
    mergedProps = mergeProps(stateProps, dispatchProps, ownProps)
    hasRunAtLeastOnce = true
    return mergedProps
  }
  // state跟ownProps同时变化
  function handleNewPropsAndNewState() {
    stateProps = mapStateToProps(state, ownProps)

    if (mapDispatchToProps.dependsOnOwnProps)
      dispatchProps = mapDispatchToProps(dispatch, ownProps)

    mergedProps = mergeProps(stateProps, dispatchProps, ownProps)
    return mergedProps
  }
  // ownProps变化
  function handleNewProps() {
    if (mapStateToProps.dependsOnOwnProps)
      stateProps = mapStateToProps(state, ownProps)

    if (mapDispatchToProps.dependsOnOwnProps)
      dispatchProps = mapDispatchToProps(dispatch, ownProps)

    mergedProps = mergeProps(stateProps, dispatchProps, ownProps)
    return mergedProps
  }
  // state变化
  function handleNewState() {
    const nextStateProps = mapStateToProps(state, ownProps)
    // 如果仅仅是state变化，再比较一次stateProps变化情况
    const statePropsChanged = !areStatePropsEqual(nextStateProps, stateProps)
    stateProps = nextStateProps

    if (statePropsChanged)
      mergedProps = mergeProps(stateProps, dispatchProps, ownProps)

    return mergedProps
  }
  // 处理非首次情况，有变化更新上一次数据，用于下次比较
  function handleSubsequentCalls(nextState, nextOwnProps) {
    const propsChanged = !areOwnPropsEqual(nextOwnProps, ownProps)
    const stateChanged = !areStatesEqual(nextState, state)
    state = nextState
    ownProps = nextOwnProps

    if (propsChanged && stateChanged) return handleNewPropsAndNewState()
    if (propsChanged) return handleNewProps()
    if (stateChanged) return handleNewState()
    // 没有变化，直接返回上一次的值
    return mergedProps
  }

  return function pureFinalPropsSelector(nextState, nextOwnProps) {
    // 区分首次调用
    return hasRunAtLeastOnce
      ? handleSubsequentCalls(nextState, nextOwnProps)
      : handleFirstCall(nextState, nextOwnProps)
  }
}
```



#### Equal策略

针对state跟ownProps的比较策略是不一样的，默认的情况是：

- state，默认areStatesEqual（strictEqual）来判断store.getState() --> 然后用 areStatePropsEqual （shallowEqual）来比较stateProps，经过两次比较，最终判断是否变化；
- ownProps，默认策略是 areOwnPropsEqual（shallowEqual）；

`strictEqual`是严格比较：

```typescript
function strictEqual(a, b) {
  return a === b
}
```

`shallowEqual`是浅度比较，只判断对象的第一层：

```typescript
export default function shallowEqual(objA, objB) {
  // 严格比较，通过就不用再继续了
  if (is(objA, objB)) return true
  // 没有通过严格比较的话，只有对象类型才有可能返回true了
  if (
    typeof objA !== 'object' ||
    objA === null ||
    typeof objB !== 'object' ||
    objB === null
  ) {
    return false
  }

  const keysA = Object.keys(objA)
  const keysB = Object.keys(objB)
  // 对象key长度不一致，返回false
  if (keysA.length !== keysB.length) return false

  for (let i = 0; i < keysA.length; i++) {
    if (
      // key不一致，或者obj[key]不相等，则返回false
      !Object.prototype.hasOwnProperty.call(objB, keysA[i]) ||
      !is(objA[keysA[i]], objB[keysA[i]])
    ) {
      return false
    }
  }
  // 只有对象类型，且第一层引用obj[key]都相等才到这里
  return true
}

function is(x, y) {
  if (x === y) {
    return x !== 0 || y !== 0 || 1 / x === 1 / y
  } else {
    return x !== x && y !== y
  }
}
```



### connectAdvanced

在createConnect小节，其实我们已经看过了此函数一部分代码，了解到`actualChildProps`是由柯里化函数`selectorFactory`获取得来。

现在还有一问题没有解决，就是`selectorFactory`执行的时机是什么时候呢？换句话就是，***被connect包裹到组件何时re-render***?

这个问题要在`connectAdvanced`函数中寻找答案。

#### ownProps变化时

```typescript
function ConnectFunction(props) {
      const [
        propsContext,
        reactReduxForwardedRef,
        wrapperProps
      ] = useMemo(() => {
        const { reactReduxForwardedRef, ...wrapperProps } = props
        return [props.context, reactReduxForwardedRef, wrapperProps]
      }, [props])
      
      // ...
      
      // wrapperProps变化时，刷新actualChildProps
      const actualChildProps = usePureOnlyMemo(() => {if (
          childPropsFromStoreUpdate.current &&
          wrapperProps === lastWrapperProps.current
        ) {
          return childPropsFromStoreUpdate.current
        }
        return childPropsSelector(store.getState(), wrapperProps)
      }, [store, previousStateUpdateResult, wrapperProps])
      
}
```

`wrapperProps` 就是从外部传进来的`ownProps`，当它变化时，`childPropsSelector`执行获取最新的`actualChildProps`。



#### state变化时

当`store.getState()`变化时，显然每个组件也需要判断一下自己消费的state有没有更新，若有更新，需要re-render该组件。

```typescript
function ConnectFunction(props) {
      // ...
      const [subscription, notifyNestedSubs] = useMemo(() => {
        if (!shouldHandleStateChanges) return NO_SUBSCRIPTION_ARRAY

        // 每个组件创建一个Subscription实例，parentSub是contextValue.subscription
        const subscription = new Subscription(
          store,
          didStoreComeFromProps ? null : contextValue.subscription
        )

        // Subscription子实例的监听者出发接口，一般用不到
        const notifyNestedSubs = subscription.notifyNestedSubs.bind(
          subscription
        )
        return [subscription, notifyNestedSubs]
      }, [store, didStoreComeFromProps, contextValue])
      
      // 通过forceComponentUpdateDispatch触发re-render
      const [
        [previousStateUpdateResult],
        forceComponentUpdateDispatch
      ] = useReducer(storeStateUpdatesReducer, EMPTY_ARRAY, initStateUpdates)
      
      // childPropsFromStoreUpdate.current 重置为null
      useIsomorphicLayoutEffectWithArgs(captureWrapperProps, [
        lastWrapperProps,
        lastChildProps,
        renderIsScheduled,
        wrapperProps,
        actualChildProps,
        childPropsFromStoreUpdate,
        notifyNestedSubs
      ])
      
      // 订阅store更新
      useIsomorphicLayoutEffectWithArgs(
        subscribeUpdates,
        [
          shouldHandleStateChanges,
          store,
          subscription,
          childPropsSelector,
          lastWrapperProps,
          lastChildProps,
          renderIsScheduled,
          childPropsFromStoreUpdate,
          notifyNestedSubs,
          forceComponentUpdateDispatch
        ],
        [store, subscription, childPropsSelector]
      )
      
}  

```

由上可知，每个组件都会初始化一个Subscription实例，用于注册订阅者。而且定义了一个`forceComponentUpdateDispatch`，用于触发组件re-render，下面注意细看其触发时机。

接着看`subscribeUpdates`，看看store更新是如何触发组件re-render的：

```typescript
function subscribeUpdates(
  shouldHandleStateChanges,
  store,
  subscription,
  childPropsSelector,
  lastWrapperProps,
  lastChildProps,
  renderIsScheduled,
  childPropsFromStoreUpdate,
  notifyNestedSubs,
  forceComponentUpdateDispatch
) {
  // If we're not subscribed to the store, nothing to do here
  if (!shouldHandleStateChanges) return

  // Capture values for checking if and when this component unmounts
  let didUnsubscribe = false
  let lastThrownError = null

  // We'll run this callback every time a store subscription update propagates to this component
  const checkForUpdates = () => {
    if (didUnsubscribe) {
      return
    }

    const latestStoreState = store.getState()

    let newChildProps, error
    try {
      // 获取最新actualChildProps
      newChildProps = childPropsSelector(
        latestStoreState,
        lastWrapperProps.current
      )
    } catch (e) {
      error = e
      lastThrownError = e
    }

    if (!error) {
      lastThrownError = null
    }

    // props不更新，触发子subscription的订阅者
    if (newChildProps === lastChildProps.current) {
      if (!renderIsScheduled.current) {
        notifyNestedSubs()
      }
    } else {
      // 更新缓存值
      lastChildProps.current = newChildProps
      childPropsFromStoreUpdate.current = newChildProps
      renderIsScheduled.current = true

      // 触发re-render
      forceComponentUpdateDispatch({
        type: 'STORE_UPDATED',
        payload: {
          error
        }
      })
    }
  }

  // 设置订阅者为checkForUpdates函数
  subscription.onStateChange = checkForUpdates
  // 注册订阅者到父subscription，统一触发
  subscription.trySubscribe()

  // 初次渲染
  checkForUpdates()

  const unsubscribeWrapper = () => {
    didUnsubscribe = true
    subscription.tryUnsubscribe()
    subscription.onStateChange = null

    if (lastThrownError) {
      throw lastThrownError
    }
  }

  return unsubscribeWrapper
}
```

这个值得注意的是`subscription.trySubscribe()`，此时因为有parentSub，所以注册到父subscription的listeners容器中。



### Hooks

流行hooks之后，connect方法已经比较少用了，hook用法更受大伙喜欢，下面来看其实现吧。看过前面的小伙伴就很容易了解了，原理都是一样的。

#### useSelector

```typescript
export function createSelectorHook(context = ReactReduxContext) {
  const useReduxContext =
    context === ReactReduxContext
      ? useDefaultReduxContext
      : () => useContext(context)
  return function useSelector(selector, equalityFn = refEquality) {
    const { store, subscription: contextSub } = useReduxContext()
    
    // 返回消费的state
    const selectedState = useSelectorWithStoreAndSubscription(
      selector,
      equalityFn,
      store,
      contextSub
    )

    useDebugValue(selectedState)

    return selectedState
  }
}
```

很显然，核心逻辑在`useSelectorWithStoreAndSubscription`，继续看：

```typescript
function useSelectorWithStoreAndSubscription(
  selector,
  equalityFn,
  store,
  contextSub
) {
  
  // 用来强制re-render
  const [, forceRender] = useReducer(s => s + 1, 0)
  
  // 实例化一个subscription
  const subscription = useMemo(() => new Subscription(store, contextSub), [
    store,
    contextSub
  ])
  
  // 缓存数据
  const latestSubscriptionCallbackError = useRef()
  const latestSelector = useRef()
  const latestStoreState = useRef()
  const latestSelectedState = useRef()

  const storeState = store.getState()
  let selectedState

  try {
    if (
      selector !== latestSelector.current ||
      storeState !== latestStoreState.current ||
      latestSubscriptionCallbackError.current
    ) {
      selectedState = selector(storeState)
    } else {
      selectedState = latestSelectedState.current
    }
  } catch (err) {
    throw err
  }

  useIsomorphicLayoutEffect(() => {
    latestSelector.current = selector
    latestStoreState.current = storeState
    latestSelectedState.current = selectedState
    latestSubscriptionCallbackError.current = undefined
  })

  useIsomorphicLayoutEffect(() => {
    function checkForUpdates() {
      try {
        const newSelectedState = latestSelector.current(store.getState())
        // 如果没变化，返回，一切都没发生
        if (equalityFn(newSelectedState, latestSelectedState.current)) {
          return
        }
        latestSelectedState.current = newSelectedState
      } catch (err) {
        latestSubscriptionCallbackError.current = err
      }
      // 强制re-render
      forceRender()
    }
    // 设置订阅者为checkForUpdates
    subscription.onStateChange = checkForUpdates
    // 注册订阅者到父subscription
    subscription.trySubscribe()

    checkForUpdates()

    return () => subscription.tryUnsubscribe()
  }, [store, subscription])

  return selectedState
}
```

#### useDispatch

其他的hook就很简单啦，不一一看了。

```typescript
export function createDispatchHook(context = ReactReduxContext) {
  const useStore =
    context === ReactReduxContext ? useDefaultStore : createStoreHook(context)

  return function useDispatch() {
    const store = useStore()
    return store.dispatch
  }
}
```



到这，react-redux的实现我们就已经阐述完毕啦！从一个很简单的原理，到最后高可用性，高性能的实现，中间经历很多精巧的设计。从中可见平时开发中玩具级别跟生产级别的代码差距可以有多大！就酱。









