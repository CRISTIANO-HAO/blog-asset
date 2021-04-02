# Redux-saga源码解析

> 本篇是基于版本 "1.1.3" 展开，由于redux-saga是redux的中间件，所以最好是小伙伴们对redux的使用有一定了解。

## 为什么不是redux-thunk

在介绍redux-saga之前，我们先看另一个处理异步逻辑的redux中间件`redux-thunk`。看看在redux-saga之前我们是如何处理异步action逻辑的。

其源码非常简单：

```typescript
function createThunkMiddleware(extraArgument) {
  return ({ dispatch, getState }) => (next) => (action) => {
    if (typeof action === 'function') {
      return action(dispatch, getState, extraArgument);
    }
    return next(action);
  };
}
const thunk = createThunkMiddleware();
thunk.withExtraArgument = createThunkMiddleware;
export default thunk;
```

短短十行代码。前面是redux中间件的写法，不了解的小伙伴可以先去了解一下。核心逻辑则是判断传进来的action是否是函数，是函数则把dispatch等方法传进去，不是则走常规的更新逻辑。

看一个具体的例子：

```typescript
const INCREMENT_COUNTER = 'INCREMENT_COUNTER';
function increment() {
  return {
    type: INCREMENT_COUNTER,
  };
}
function incrementAsync() {
  return (dispatch) => {
    new Promise((resolve) => {
      setTimeout(() => {
        // Yay! Can invoke sync or async actions with `dispatch`
        dispatch(increment());
        resolve();
      }, 1000);
    });
  };
}
```

redux-thunk的好处是我们写action creator的时候不仅可以返回一个action对象，而且还能够返回一个函数，从而可以在里面封装异步的逻辑。

看起来挺完美，有没有什么弊端呢？当然有，不然也就不会有redux-saga的出现了。大神们追求完美的脚步是永不停止的。

弊端主要有两个：

- 既然是封装异步逻辑，则避免不了各种地狱回调，导致同步dispatch逻辑散落在各个回调函数之间，可读性较差。
- 异步逻辑难以测试。

如果使用redux-saga来写是什么样子呢，让我们先一睹为快：

```typescript
import { call, put, take } from 'redux-saga/effects'
function asyncDoSomething() {
  return new Promise(resolve => {
    setTimeout(() => {
      resolve()
    }, 1000)
  })
}
function* sagaFn() {
  yield take('incrementAsync');
  yield call(asyncDoSomething);
  yield put(increment())
}
```

虽然语法似乎有些晦涩难懂，但是一眼看过去逻辑是不是贼清晰：先监听（take）一个incrementAsync的action，然后发起（call）一个异步操作，最后更新（put）数据。完全避免了redux-thunk的第一个地狱回调的弊端，而且核心逻辑显得更加清晰。

至于测试方面，由于take, call, put等方法返回的是一个固定格式Effect对象：

```typescript
{
  [IO]: true,
  combinator: false,
  type,
  payload,
}
```

而且saga又是一个generator函数，则可以一步一步测试 saga generator 函数的返回值，从而解决了测试的难题。



## 什么是redux-saga？

我们先来看官网介绍：`redux-saga` 是一个用于管理应用程序 Side Effect（副作用，例如异步获取数据，访问浏览器缓存等）的 library，它的目标是让副作用管理更容易，执行更高效，测试更简单，在处理故障时更容易。

虽然有上面的小例子打底，但是这样子看还是太抽象了，让我们先来看两个关键概念，这对我们后面的阅读理解至关重要。

### Saga

**什么是Saga?**

> 一个 saga 就像是应用程序中一个单独的线程，它独自负责处理副作用。

这是官网中的一小段话。

在 `redux-saga` 的世界里，Saga 都用 Generator 函数实现。我们从 Generator 里 yield 纯 JavaScript 对象以表达 Saga 逻辑。 

以上面例子为例，sagaFn生成器函数就是一个Saga，它代表着一段处理副作用的逻辑。

我们应用中有无数这样的逻辑，也意味着有无数的saga函数要定义，多个saga函数要怎么管理呢？

最简单的方法当然是一个个注册执行了：

```typescript
const sagaMiddleware = createSagaMiddleware()
const store = createStore(reducer, applyMiddleware(sagaMiddleware))
// 通过run方法执行saga函数
sagaMiddleware.run(saga1);
sagaMiddleware.run(saga2);
```

**<u>不过这个方式是有bug的，在下面scheduler小节会提到。</u>**

另一个方法则是通过`fork`函数组合saga：

```typescript
export default function* rootSaga() {
  yield fork(saga1)
  yield fork(saga2)
  // ...
}
const sagaMiddleware = createSagaMiddleware()
const store = createStore(reducer, applyMiddleware(sagaMiddleware))
// 通过run方法执行rootSaga函数
sagaMiddleware.run(rootSaga);
```

`fork`函数也是redux-saga中的一个副作用实现，它的特点是不会被阻塞，也就是说执行rootSaga的时候，saga1跟saga2都会顺序被调用执行自己的逻辑，saga2无需等待saga1整体逻辑执行完成。

### Effect副作用

首先我们要明白的是，无论是saga还是effect，它们都是`redux-saga`世界里的概念，脱离了这个前提去理解，就像想脱离海贼王的世界去理解海盗一样，总会有这样那样的偏差。

在`redux-saga`世界里，从 Saga 函数里 yield 纯 JavaScript 对象以表达 Saga 逻辑。 我们称呼那些对象为 `Effect`。

Effect 是一个简单的对象，这个对象包含了一些给 middleware 解释执行的信息。 你可以把 Effect 看作是发送给 middleware 的指令以执行某些操作（调用某些异步函数，发起一个 action 到 store，等等）。

同样以上面例子为例，take, call, put等方法返回的是一个固定格式Effect对象：

```typescript
{
  [IO]: true,
  combinator: false,
  type, //take, call, put。。。
  payload,
}
```

有没有感觉很熟悉的样子？

是的，相信聪明的你已经想到了，它就像一个`redux`中的action呀！不过它是`redux-saga`世界里的action罢了，而且还换了个名字，改叫`Effect`了！😓

> 在`redux-saga`的世界里，effect就是信息传递的载体，每yield一个effect之时，都会针对该effect进行对应的逻辑，当该逻辑执行完成之后，才会调用saga生成器的next方法，从而yield下一段逻辑，以此循环反复，直到生成器执行结束，从而实现`redux-saga`世界的魔幻效果。



## 源码解析

为什么此篇源码解析在进入正题之前要写如此长的一段导读呢？因为根据个人学习saga的经验，理解其中的思想理念要比源码重要的多。

在理解其思想理念之前，其源码显得异常的晦涩难懂，但是理解之后，就柳暗花明，畅通无阻了。现在正式开始吧。

### middleware

`redux-saga`的入口文件是`packages/core/src/internal/middleware.js`:

```typescript
export default function sagaMiddlewareFactory({ context = {}, channel = stdChannel(), sagaMonitor, ...options } = {}) {
  let boundRunSaga
  // 返回中间件
  function sagaMiddleware({ getState, dispatch }) {
    boundRunSaga = runSaga.bind(null, {
      ...options,
      context,
      channel,
      dispatch,
      getState,
      sagaMonitor,
    })
    return next => action => {
      // ...
      // 执行dispatch逻辑
      const result = next(action) // hit reducers
      // 关键，执行通过yield effect注册进去的回调函数，驱动saga生成器执行下一步
      channel.put(action)
      return result
    }
  }
  // 静态方法，执行saga函数
  sagaMiddleware.run = (...args) => {
    return boundRunSaga(...args)
  }
  return sagaMiddleware
}
```

入口文件很简单，就是一个工厂方法，返回一个redux中间件。

`run`静态方法执行saga的过程实际上执行的是`runSaga`方法，并且传进去一堆参数。

中间件增加的逻辑也非常简单，就是执行了一句`channel.put(action)`。整个源码这么多的代码就为了加上这一句！

不难猜测，所有的准备工作跟实现逻辑都在runSaga方法里了，我们回头再看`channel.put(action)`是如何享受胜利的果实的。



### runSaga

```typescript
export function runSaga(
  { channel = stdChannel(), dispatch, getState, context = {}, sagaMonitor, effectMiddlewares, onError = logError },
  saga,
  ...args
) {
  // 执行saga生成器，返回一个迭代器对象
  const iterator = saga(...args)
  // 一堆校验
  // 。。。
  let finalizeRunEffect
  // 中间件，增强runEffect方法的，可以修改effect，可以类比redux的dispatch跟action
  if (effectMiddlewares) {
    const middleware = compose(...effectMiddlewares)
    finalizeRunEffect = runEffect => {
      return (effect, effectId, currCb) => {
        const plainRunEffect = eff => runEffect(eff, effectId, currCb)
        return middleware(plainRunEffect)(effect)
      }
    }
  } else {
    finalizeRunEffect = identity
  }
  // 把传进来的参数封装起来，统一管理
  const env = {
    channel,
    // 增强dispatch，redux-saga内部发起的action加上action['SAGA_ACTION']= true来标识
    dispatch: wrapSagaDispatch(dispatch),
    getState,
    sagaMonitor,
    onError,
    finalizeRunEffect,
  }

  return immediately(() => {
    // 核心方法
    // 类似一个后台进程，消费一个迭代器，如果迭代器过程中产生额外的迭代器，则另起一个进程消费新的迭代器
    const task = proc(env, iterator, context, effectId, getMetaInfo(saga), /* isRoot */ true, undefined)
    // ...
    return task
  })
}
```

从上可知，run一个saga生成器函数的结果是执行该生成器生成一个迭代器，并通过`proc`方法消费该迭代器，并最终返回一个`Task`实例对象。



### scheduler

在runSaga小节，我们有一个辅助函数`immediately`没有提及，这是干嘛用的呢？

顾名思义，这是一个经典调度器的实现，目的就是解决不同优先级任务执行顺序的问题。简而言之，就是让优先级高的先执行，优先级低的后执行。

让我们看一个场景：

```typescript
function* rootSaga() {
  yield fork(genA) // LINE-1
  yield fork(genB) // LINE-2
}
function* genA() {
  yield put({ type: 'A' })
  yield take('B')
}
function* genB() {
  yield take('A')
  yield put({ type: 'B' })
}
```

由于代码是顺序执行的，如果没有特殊处理，genA会执行到`yield take('B')`被挂起，然后开始执行genB被fork的逻辑，然后在`yield take('A')`处被挂起。但是这时候`yield put({ type: 'A' })`已经被执行，也就是永远也take不到来自genA的action了。

这时候就要区分任务的优先级了，显而易见，run方法跟fork方法的优先级最高，put方法的优先级较低，可以等前者执行后再执行，这样就不会丢失任何一个action了。

下面看源码，源码注释足够详细，就不重复解释了。

```typescript
/**
  Variable to hold a counting semaphore
  - Incrementing adds a lock and puts the scheduler in a `suspended` state (if it's not
    already suspended)
  - Decrementing releases a lock. Zero locks puts the scheduler in a `released` state. This
    triggers flushing the queued tasks.
**/
let semaphore = 0

/**
  Executes a task 'atomically'. Tasks scheduled during this execution will be queued
  and flushed after this task has finished (assuming the scheduler endup in a released
  state).
**/
function exec(task) {
  try {
    suspend()
    task()
  } finally {
    release()
  }
}

/**
  Executes or queues a task depending on the state of the scheduler (`suspended` or `released`)
**/
export function asap(task) {
  queue.push(task)

  if (!semaphore) {
    suspend()
    flush()
  }
}

/**
 * Puts the scheduler in a `suspended` state and executes a task immediately.
 */
export function immediately(task) {
  try {
    suspend()
    return task()
  } finally {
    flush()
  }
}

/**
  Puts the scheduler in a `suspended` state. Scheduled tasks will be queued until the
  scheduler is released.
**/
function suspend() {
  semaphore++
}

/**
  Puts the scheduler in a `released` state.
**/
function release() {
  semaphore--
}

/**
  Releases the current lock. Executes all queued tasks if the scheduler is in the released state.
**/
function flush() {
  release()
  let task
  while (!semaphore && (task = queue.shift()) !== undefined) {
    exec(task)
  }
}
```

源码暴露出两个方法，分别是`immediately`跟`asap`:

- `immediately`：立即执行，并且执行的task如果也包含`immediately`的task，则该task也会立即执行
- `asap`：asap 是 as soon as possible 的缩写，意思是尽快执行，也就是当semaphore===0，`immediately`类的任务都执行结束时，紧跟着执行。

前面我们提到逐一执行saga的时候会有bug，原因就是在这里了，因为run方法是被`immediately`包裹的，也就是里面所有的fork都属于`immediately`方法里面的`immediately`，从而保证了所有的`immediately`都能在第一时间执行。

但是分开执行saga之后，两个`immediately`方法就互不干扰了，后一个`immediately`也要等前一个的asap执行后才执行，从而会导致可能丢失action。

有兴趣的小伙伴可以拿上面的例子测试一下。



### proc

接下来就是核心方法啦。此方法主要是处理迭代器的迭代逻辑。

```typescript
export default function proc(env, iterator, parentContext, parentEffectId, meta, isRoot, cont) {

  const finalRunEffect = env.finalizeRunEffect(runEffect)
  // ...
  const task = newTask(env, mainTask, parentContext, parentEffectId, meta, isRoot, cont)
  
  // ...

  // 调用迭代器执行器
  next()

  // 返回封装的task实例对象
  return task

  /**
   * 迭代器的执行器
   * 异步递归调用，直到迭代器终止
   */
  function next(arg, isErr) {
    try {
      // 执行迭代器
      let result
      if (isErr) {
        result = iterator.throw(arg)
      } else if (shouldCancel(arg)) {
        // ...
        next.cancel()
      } else if (shouldTerminate(arg)) {
        result = is.func(iterator.return) ? iterator.return() : { done: true }
      } else {
        // 执行迭代器
        result = iterator.next(arg)
      }
      if (!result.done) {
        // 消费effect对象，callback传入执行器next，用于异步递归调用
        digestEffect(result.value, parentEffectId, next)
      } else {
        // ...
        // 当前mainTask结束，如果此mainTask是被proc的，callback执行父迭代器
        mainTask.cont(result.value)
      }
    } catch (error) {
      // ...
    }
  }
  
  // 最终消费effect对象
  function runEffect(effect, effectId, currCb) {
    if (is.promise(effect)) {
      resolvePromise(effect, currCb)
    } else if (is.iterator(effect)) {
      // 如果effect是一个迭代器，另开一个类进程处理
      // 传入currCb，等待此迭代器结束才会callback
      proc(env, effect, task.context, effectId, meta, /* isRoot */ false, currCb)
    } else if (effect && effect[IO]) {
      // effect是redux-saga内置的，则针对性处理
      const effectRunner = effectRunnerMap[effect.type]
      effectRunner(env, effect.payload, currCb, executingContext)
    } else {
      // 不是上述类型，直接返回yield 表达式的值
      currCb(effect)
    }
  }

  function digestEffect(effect, parentEffectId, cb, label = '') {
    // Completion callback passed to the appropriate effect runner
    function currCb(res, isErr) {
      if (effectSettled) {
        return
      }
      effectSettled = true
      // ...
      cb(res, isErr)
    }
     // 每迭代一次都安装一次cancel函数
    cb.cancel = () => {
      // prevents cancelling an already completed effect
      if (effectSettled) {
        return
      }
      effectSettled = true
      // ...
    }
    finalRunEffect(effect, effectId, currCb)
  }
}
```

经过删繁就简，我们梳理出了proc方法的主要逻辑其实非常简单，就是每一个类进程proc过程，都会构造一个执行器`next`，不断异步递归调用，从而执行迭代器`iterator`直到终止或者被cancel。

#### task

每一个proc过程都新建一个newTask实例对象，里面封装了当前函数的上下文环境。

```typescript
  // 保存上下文环境
  const mainTask = { meta, cancel: cancelMain, status: RUNNING }

  /**
   Creates a new task descriptor for this generator.
   A task is the aggregation of it's mainTask and all it's forked tasks.
   **/
  const task = newTask(env, mainTask, parentContext, parentEffectId, meta, isRoot, cont)

```

值得注意的是`cont`变量，由上面可知，当effect是一个iterator的时候，会另起一个子类进程逻辑proc该iterator，然后把当前上下文的next当作cont传进子类进程，等该iterator结束时调用，从而切换回当前上下文，继续执行父类进程的逻辑。

#### cancel

`redux-saga`的每一个task都是可cancel的。

```typescript
// 保存上下文环境
  const mainTask = { meta, cancel: cancelMain, status: RUNNING }

  function cancelMain() {
    if (mainTask.status === RUNNING) {
      mainTask.status = CANCELLED
      // 调用next执行器cancel当前迭代器
      next(TASK_CANCEL)
    }
  }
	function digestEffect(effect, parentEffectId, cb, label = '') {
     // 每迭代一次都安装一次cancel函数
    cb.cancel = () => {
      // prevents cancelling an already completed effect
      if (effectSettled) {
        return
      }
      effectSettled = true
      // ...
    }
  }
```

可以看出cancel方法也是挂载在task对象里面暴露出来，具体的cancel逻辑则是挂载在next执行器静态方法上，并且执行器next每递归一次都会重新挂载该静态方法。最终也是通过`next(TASK_CANCEL)`来cancel执行器。

#### runEffect

针对不同的effect，iterator采用不同的迭代逻辑：

```typescript
function runEffect(effect, effectId, currCb) {
    if (is.promise(effect)) {
      resolvePromise(effect, currCb)
    } else if (is.iterator(effect)) {
      // resolve iterator
      proc(env, effect, task.context, effectId, meta, /* isRoot */ false, currCb)
    } else if (effect && effect[IO]) {
      const effectRunner = effectRunnerMap[effect.type]
      effectRunner(env, effect.payload, currCb, executingContext)
    } else {
      // anything else returned as is
      currCb(effect)
    }
  }
```

- 如果是promise对象:

  ```typescript
  promise.then(cb, error => {
      cb(error, true)
  })
  ```

- 如果是iterator，则另起一proc子类进程，并且等iterator结束后callback。

- 如果是内置的Effect对象，则选择相应的effectRunner执行。后面会重点介绍常用effectRunner。

- 其他情况则直接callback，返回effect。



### effectRunner

从上一小节我们知道，当effect是一个内置Effect对象时，会有不同的effectRunner处理，从而实现不同的功能，下面就看看几个常用effectRunner实现。

先看effect构造方法：

```typescript
const makeEffect = (type, payload) => ({
  [IO]: true,
  combinator: false,
  type,
  payload,
})
```

前面已经提过了，不再赘述。

#### put

```typescript
export function put(channel, action) {
  if (is.undef(action)) {
    action = channel
    channel = undefined
  }
  return makeEffect(effectTypes.PUT, { channel, action })
}
```

put方法可以接收两个参数。channel放后面介绍。

```typescript
function runPutEffect(env, { channel, action, resolve }, cb) {
  asap(() => {
    let result
    try {
      result = (channel ? channel.put : env.dispatch)(action)
    } catch (error) {
      cb(error, true)
      return
    }

    if (resolve && is.promise(result)) {
      resolvePromise(result, cb)
    } else {
      cb(result)
    }
  })
}
```

可以看出put方法是被asap包裹的，优先级比`immediately`低。

代码还是很简单的，就是发起一个action更新逻辑，然后callback。

#### call

```typescript
export function call(fnDescriptor, ...args) {
  return makeEffect(effectTypes.CALL, getFnCallDescriptor(fnDescriptor, args))
}
```

很简单，看看payload是什么：

```typescript
function getFnCallDescriptor(fnDescriptor, args) {
  let context = null
  let fn

  if (is.func(fnDescriptor)) {
    fn = fnDescriptor
  } else {
    if (is.array(fnDescriptor)) {
      ;[context, fn] = fnDescriptor
    } else {
      ;({ context, fn } = fnDescriptor)
    }

    if (context && is.string(fn) && is.func(context[fn])) {
      fn = context[fn]
    }
  }
  return { context, fn, args }
}
```

可以看出是返回一个封装fn相关信息的对象。

```typescript
function runCallEffect(env, { context, fn, args }, cb, { task }) {
  try {
    const result = fn.apply(context, args)
    if (is.promise(result)) {
      resolvePromise(result, cb)
      return
    }
    if (is.iterator(result)) {
      // resolve iterator
      proc(env, result, task.context, currentEffectId, getMetaInfo(fn), /* isRoot */ false, cb)
      return
    }
    cb(result)
  } catch (error) {
    cb(error, true)
  }
}
```

可以看出，call方法是用来执行一个函数。

- 如果函数返回值是promise，则等promise状态转移后callback；
- 如果返回值是iterator，则另起一个proc子类进程，等该iterator结束后再callback；
- 其他情况直接callback。

#### take

```typescript
export function take(patternOrChannel = '*', multicastPattern) {
  if (is.pattern(patternOrChannel)) {
    return makeEffect(effectTypes.TAKE, { pattern: patternOrChannel })
  }
  if (is.multicast(patternOrChannel) && is.notUndef(multicastPattern) && is.pattern(multicastPattern)) {
    return makeEffect(effectTypes.TAKE, { channel: patternOrChannel, pattern: multicastPattern })
  }
  if (is.channel(patternOrChannel)) {
    return makeEffect(effectTypes.TAKE, { channel: patternOrChannel })
  }
}
```

`is.pattern`允许的场景非常丰富：

```typescript
export const pattern = pat => pat && (string(pat) || symbol(pat) || func(pat) || (array(pat) && pat.every(pattern)))
```

可以是字符串，symbol，函数，数组。所以不出意外都是走第一种情况。

```typescript
function runTakeEffect(env, { channel = env.channel, pattern, maybe }, cb) {
  // 封装成一个回调，在里面callback
  const takeCb = input => {
    if (input instanceof Error) {
      cb(input, true)
      return
    }
    if (isEnd(input) && !maybe) {
      cb(TERMINATE)
      return
    }
    cb(input)
  }
  try {
    // 塞进监听队列
    channel.take(takeCb, is.notUndef(pattern) ? matcher(pattern) : null)
  } catch (err) {
    cb(err, true)
    return
  }
  cb.cancel = takeCb.cancel
}
```

我们知道take方法是会阻塞iterator迭代的，它是如何实现监听指定action的呢？这时候就不得不介绍channel的实现啦。



### channel

channel 是用于在任务间发送和接收消息的对象。Channel 接口定义了 3 个方法：`take`，`put` 和 `close`。

`redux-saga`实现了几个类型的channel，默认使用的是`stdChannel`，其又是基于`multicastChannel`实现。现在删繁就简，看其核心实现：

```typescript
export function multicastChannel() {
  let closed = false
  let currentTakers = []
  let nextTakers = currentTakers
  // 在take跟put两个操作之间维持两个任务队列，防止在遍历taker时，taker发生变化。
  const ensureCanMutateNextTakers = () => {
    if (nextTakers !== currentTakers) {
      return
    }
    nextTakers = currentTakers.slice()
  }
  return {
    [MULTICAST]: true,
    put(input) {
      // ...
      const takers = (currentTakers = nextTakers)
      for (let i = 0, len = takers.length; i < len; i++) {
        const taker = takers[i]
        // 匹配通过
        if (taker[MATCH](input)) {
          taker.cancel()
          taker(input)
        }
      }
    },
    take(cb, matcher = matchers.wildcard) {
      // ...
      // 挂载匹配函数
      cb[MATCH] = matcher
      ensureCanMutateNextTakers()
      nextTakers.push(cb)
      cb.cancel = once(() => {
        ensureCanMutateNextTakers()
        remove(nextTakers, cb)
      })
    },
  }
}
```

代码思想其实很简单，就是一个任务管理模型，每触发一个任务就从当前任务队列里移除。

到这里，结合take方法跟channel的实现，我们就能看懂前面middleware小节的那一段代码了。

```
// take方法
channel.take(takeCb, is.notUndef(pattern) ? matcher(pattern) : null)
// middleware
channel.put(action)
```

完美，坑终于填上啦！

### takeEvery

到这里，`redux-saga`核心实现已经阐述完毕了，接下来看其一些高级的实现。

```typescript
export default function takeEvery(patternOrChannel, worker, ...args) {
  // 监听
  const yTake = { done: false, value: take(patternOrChannel) }
  // 执行监听器
  const yFork = ac => ({ done: false, value: fork(worker, ...args, ac) })

  let action,
    setAction = ac => (action = ac)

  return fsmIterator(
    {
      q1() {
        return { nextState: 'q2', effect: yTake, stateUpdater: setAction }
      },
      q2() {
        return { nextState: 'q1', effect: yFork(action) }
      },
    },
    'q1',
    `takeEvery(${safeName(patternOrChannel)}, ${worker.name})`,
  )
}
```

不用猜，fsmIterator返回的是一个iterator。

前面我们提到过，如果effect是一个iterator，则另起一proc子类进程，并且等iterator结束后callback。

但是这个iterator有些不一样，它是无限迭代的，在q1跟q2状态之间不断切换，永远不会callback，从而实现了takeEvery无限监听的功能。

```typescript
export default function fsmIterator(fsm, startState, name) {
  let stateUpdater,
    errorState,
    effect,
    nextState = startState
  // iterator的next方法，返回effect
  function next(arg, error) {
    if (nextState === qEnd) {
      return done(arg)
    }
    if (error && !errorState) {
      nextState = qEnd
      throw error
    } else {
      stateUpdater && stateUpdater(arg)
      const currentState = error ? fsm[errorState](error) : fsm[nextState]()
      ;
      // 状态转换 q1 -> q2 -> q1 -> ....
      ({ nextState, effect, stateUpdater, errorState } = currentState)
      // 返回effect
      return nextState === qEnd ? done(arg) : effect
    }
  }

  return makeIterator(next, error => next(null, error), name)
}

export function makeIterator(next, thro = kThrow, name = 'iterator') {
  const iterator = { meta: { name }, next, throw: thro, return: kReturn, isSagaIterator: true }

  if (typeof Symbol !== 'undefined') {
    iterator[Symbol.iterator] = () => iterator
  }
  return iterator
}
```

这段代码的设计非常精妙，值得细细体会。至于takeLatest，throttle等高级实现就不一一过了，所谓的高级实现也是在take, put, call等基础实现的基础上组合而来，有兴趣的小伙伴可以去研究。



