

# dva源码解析

> 本篇是基于版本2.0.3进行展开。由于此次主要是解析dva，所以不了解redux跟redux-saga的小伙伴可以先去熟悉一下再过来。

## 为什么是dva

在了解一门技术之前，我们要先思考一个问题，为什么是它？

在回答这问题之前，我们先了解什么是dva。

看官网解释：

> dva 首先是一个基于 [redux](https://github.com/reduxjs/redux) 和 [redux-saga](https://github.com/redux-saga/redux-saga) 的数据流方案，然后为了简化开发体验，dva 还额外内置了 [react-router](https://github.com/ReactTraining/react-router) 和 [fetch](https://github.com/github/fetch)，所以也可以理解为一个轻量级的应用框架。

为什么要有dva呢，难道redux跟redux-saga还不够好吗？

熟悉这两者的小伙伴都知道，确实还不够好：

- 编辑成本高，需要在 reducer, saga, action 之间来回切换
- 不便于组织业务模型 (或者叫 domain model) 。比如我们写了一个 userlist 之后，要写一个 productlist，需要复制很多文件。

还有一些其他的：

- saga 书写太复杂，每监听一个 action 都需要走 fork -> watcher -> worker 的流程
- entry 书写麻烦
- ...

而 dva 正是用于解决这些问题。



## dva的样子

既然前任不够好，那现任又长什么样呢？好在哪里？

```typescript
app.model({
  namespace: 'products',
  state: {
    list: [],
    loading: false,
  },
  subscriptions: [
    function(dispatch) {
      dispatch({type: 'products/query'});
    },
  ],
  effects: {
    ['products/query']: function*() {
      yield call(delay(800));
      yield put({
        type: 'products/query/success',
        payload: ['ant-tool', 'roof'],
      });
    },
  },
  reducers: {
    ['products/query'](state) {
      return { ...state, loading: true, };
    },
    ['products/query/success'](state, { payload }) {
      return { ...state, loading: false, list: payload };
    },
  },
});
```

在有 dva 之前，我们通常会创建 `sagas/products.js`, `reducers/products.js` 和 `actions/products.js`，然后在这些文件之间来回切换。

而现在，通过 `app.model` 方法，把 reducer, initialState, action, saga 封装到一起了。

> 简单一句话，dva不是新鲜事物，它是为提升开发体验而生。



## dva源码

dva源码相对其他库来说简单很多，GitHub仓库下主要有四个包，dva、dva-core、dva-loading、dva-immer。

其中dva-core是核心包，其余三个衍生产品，dva包是dva与react-redux、react-router-dom的集成；dva-loading、dva-immer则是挂载在dva上的plugin实现。

下面先从dva-core开始分析。

### 简单示例

读源码当然不是一头扎进去猛啃，通过一个简单的使用实例，找到其中关键的实现代码深入分析，可能会效果会更明显。

```typescript
import React from 'react';
import dva, { connect } from 'dva';
import counter from './models/counter';

// 1. Initialize
const app = dva({
  onReducer: reducer => (state, action) => {
    const newState = undoable(reducer, {})(state, action);
    return { ...newState };
  },
});

// 2. Model
app.model(counter);

// 3. View
const App = connect(({ present: { counter } }) => ({
  counter,
}))(props => {
  return (
    <div style={{ textAlignLast: 'center' }}>
      <h2>Count: {props.counter}</h2>
      <button
        onClick={() => {
          props.dispatch({ type: 'counter/minus' });
        }}
      >
    </div>
  );
});

// 4. Router
app.router(() => <App />);

// 5. Start
app.start('#root');
```

根据多年阅码经验，可以大胆猜测1-4步骤都是准备阶段，最后一步才是真正的启动步骤；通过代码梳理，这五个步骤分别对应的是 create、model、connect、router、start方法，其中connect、router不属于dva的内容，也就是说我们只需要了解三个方法就行啦！

### create

```typescript
export function create(hooksAndOpts = {}, createOpts = {}) {
  // initialReducer，普通reducer
  // setupApp，给app对象start后安装的入口
  const { initialReducer, setupApp = noop } = createOpts;
  
  // 挂载plugin，并只能挂载指定hooks
  const plugin = new Plugin();
  plugin.use(filterHooks(hooksAndOpts));
  
  // 返回一个对象，暴露出model，start方法
  const app = {
    _models: [prefixNamespace({ ...dvaModel })],
    _store: null,
    _plugin: plugin,
    use: plugin.use.bind(plugin),
    model,
    start,
  };
  return app;
  
  // ...
}
```

#### prefixNamespace

无论是reduces还是effects，最终还是要分别转换为一个个reducer跟saga。所以`type`值的约定处理就显得很有必要啦。

```typescript
function prefix(obj, namespace, type) {
  return Object.keys(obj).reduce((memo, key) => {
    // 最终的type
    const newKey = `${namespace}${NAMESPACE_SEP}${key}`;
    memo[newKey] = obj[key];
    return memo;
  }, {});
}
export default function prefixNamespace(model) {
  const { namespace, reducers, effects } = model;
  if (reducers) {
    if (isArray(reducers)) {
      model.reducers[0] = prefix(reducers[0], namespace, 'reducer');
    } else {
      model.reducers = prefix(reducers, namespace, 'reducer');
    }
  }
  if (effects) {
    model.effects = prefix(effects, namespace, 'effect');
  }
  return model;
}
```

显而易见，最终reduces跟effects的key都会被处理成`“namespace/key”`。



#### plugin.use

dva通过一个`Plugin`实例对象进行各类hooks的挂载。各个hooks分别在不同节点提供功能嵌入的接口。

```typescript
// 内置的hooks接口
const hooks = [
  'onError',
  'onStateChange',
  'onAction',
  'onHmr',
  'onReducer',
  'onEffect',
  'extraReducers',
  'extraEnhancers',
  '_handleActions',
];

export default class Plugin {
  constructor() {
    // 只能挂载一个
    this._handleActions = null;
    // 其他hook可以挂载多个，且只能挂载指定的hook
    this.hooks = hooks.reduce((memo, key) => {
      memo[key] = [];
      return memo;
    }, {});
  }

  use(plugin) {
    const { hooks } = this;
    for (const key in plugin) {
      if (Object.prototype.hasOwnProperty.call(plugin, key)) {
        // _handleActions只能挂载一个
        if (key === '_handleActions') {
          this._handleActions = plugin[key];
        } else if (key === 'extraEnhancers') {
          // extraEnhancers也是挂载一个
          hooks[key] = plugin[key];
        } else {
          hooks[key].push(plugin[key]);
        }
      }
    }
  }
  // 执行指定hook
  apply(key, defaultHandler) {
    const { hooks } = this;
    // 此方法只用于'onError', 'onHmr'
    const validApplyHooks = ['onError', 'onHmr'];
    invariant(validApplyHooks.indexOf(key) > -1, `plugin.apply: hook ${key} cannot be applied`);
    const fns = hooks[key];
    return (...args) => {
      if (fns.length) {
        for (const fn of fns) {
          fn(...args);
        }
      } else if (defaultHandler) {
        defaultHandler(...args);
      }
    };
  }
  // 获取hook
  get(key) {
    // 。。。
  }
}
```

从上可知，dva上plugin对象挂载的hook是有限制的，只能挂载内置的几类hook。这几类的hook的使用在下面适当的时间会说到。

plugin对象实例最终会挂载到app对象上面，通过app.use()挂载需要的hook：

```typescript
app.use({
	_handleActions: () => {},
  onError: () => {}
})
app.use({
  onEffect: () => {}
})
```



### model

model的注册时机有两点，一个上启动前，一个上启动后，注册之后还能卸载。

#### model

```typescript
/**
 * Register model before app is started.
 */
function model(m) {
    if (process.env.NODE_ENV !== 'production') {
      checkModel(m, app._models);
    }
    const prefixedModel = prefixNamespace({ ...m });
    // 挂载到_models
    app._models.push(prefixedModel);
    return prefixedModel;
  }
```

#### injectModel

```typescript
/**
 * Inject model after app is started.
 */
function injectModel(createReducer, onError, unlisteners, m) {
    m = model(m);
    const store = app._store;
    // 挂载到asyncReducers
    store.asyncReducers[m.namespace] = getReducer(m.reducers, m.state, plugin._handleActions);
    // 全量替换reducer
    store.replaceReducer(createReducer());
    if (m.effects) {
      store.runSaga(app._getSaga(m.effects, m, onError, plugin.get('onEffect'), hooksAndOpts));
    }
    // 执行订阅逻辑
    if (m.subscriptions) {
      // 收集unlistener，卸载时执行
      unlisteners[m.namespace] = runSubscription(m.subscriptions, m, app, onError);
    }
  }
```

#### unmodel

```typescript
/**
   * Unregister model.
   */
  function unmodel(createReducer, reducers, unlisteners, namespace) {
    const store = app._store;
    // Delete reducers
    delete store.asyncReducers[namespace];
    delete reducers[namespace];
    store.replaceReducer(createReducer());
    store.dispatch({ type: '@@dva/UPDATE' });

    // Cancel effects
    store.dispatch({ type: `${namespace}/@@CANCEL_EFFECTS` });

    // Unlisten subscrioptions
    unlistenSubscription(unlisteners, namespace);

    // Delete model from app._models
    app._models = app._models.filter(model => model.namespace !== namespace);
  }
```



### start

接下来就是最关键的start方法了。start 方法是 dva-core 的核心，在 start 方法里，dva 完成了 store 初始化 以及 redux-saga 的调用。

```typescript
function start() {
    // 捕获全局错误，可以通过hook注入
    const onError = (err, extension) => {
      // ...
    };
    
    // saga中间件
    const sagaMiddleware = createSagaMiddleware();
    // 处理model.effects[key]的action的中间件
    const promiseMiddleware = createPromiseMiddleware(app);
    app._getSaga = getSaga.bind(null);

    const sagas = [];
    const reducers = { ...initialReducer };
    for (const m of app._models) {
      // 收集reducer
      reducers[m.namespace] = getReducer(m.reducers, m.state, plugin._handleActions);
      if (m.effects) {
        // 收集saga
        sagas.push(app._getSaga(m.effects, m, onError, plugin.get('onEffect'), hooksAndOpts));
      }
    }
    // hook
    const reducerEnhancer = plugin.get('onReducer');
    const extraReducers = plugin.get('extraReducers');

    // 创建store
    app._store = createStore({
      reducers: createReducer(),
      initialState: hooksAndOpts.initialState || {},
      plugin,
      createOpts,
      sagaMiddleware,
      promiseMiddleware,
    });
    // ...
    
    // store收集观察者
    // Execute listeners when state is changed
    const listeners = plugin.get('onStateChange');
    for (const listener of listeners) {
      store.subscribe(() => {
        listener(store.getState());
      });
    }

    // 注册saga
    sagas.forEach(sagaMiddleware.run);

    // Setup app
    setupApp(app);
    
    // 收集model的subscriptions
    // Run subscriptions
    const unlisteners = {};
    for (const model of this._models) {
      if (model.subscriptions) {
        unlisteners[model.namespace] = runSubscription(model.subscriptions, model, app, onError);
      }
    }

    // Setup app.model and app.unmodel
    // ...

    // Create global reducer for redux.
    function createReducer() {
      return reducerEnhancer(
        combineReducers({
          ...reducers,
          ...extraReducers,
          ...(app._store ? app._store.asyncReducers : {}),
        }),
      );
    }
  }
```

Start 方法包含了dva的核心实现逻辑，下面拆解来看看。

#### onError

```typescript
const onError = (err, extension) => {
      if (err) {
        if (typeof err === 'string') err = new Error(err);
        err.preventDefault = () => {
          err._dontReject = true;
        };
        // 调用通过onError hook注入的函数，没有则执行第二个传入的函数参数
        plugin.apply('onError', err => {
          throw new Error(err.stack || err);
        })(err, app._store.dispatch, extension);
      }
    };
```

`onError`传入到了getSaga、injectModel、replaceModel等方法中，统一监听各个环节的错误。



#### createPromiseMiddleware

该函数目的是创建一个redux的中间件：

```typescript
export default function createPromiseMiddleware(app) {
  return () => next => action => {
    const { type } = action;
    if (isEffect(type)) {
      return new Promise((resolve, reject) => {
        next({
          __dva_resolve: resolve,
          __dva_reject: reject,
          ...action,
        });
      });
    } else {
      return next(action);
    }
  };
  // 判断该action是不是调用model.effects
  function isEffect(type) {
    if (!type || typeof type !== 'string') return false;
    const [namespace] = type.split(NAMESPACE_SEP);
    const model = app._models.filter(m => m.namespace === namespace)[0];
    if (model) {
      if (model.effects && model.effects[type]) {
        return true;
      }
    }
    return false;
  }
}
```

> const middleware = ({dispatch}) => next => (action) => {... return next(action)} 基本上是一个标准的中间件写法。在 return next(action) 之前可以对 action 做各种各样的操作。

redux中间件的目的是在不影响dispatch函数的基础上进行层层包装（也就是所谓的洋葱模型），在执行最终的dispatch函数之前，可以做各种各样的操作，甚至修改action。

本中间件目的是拦截指向 effects 的 action，如果是则dispatch方法返回一个promise对象，并给通过action传进resolve、reject方法。resole跟reject则会在下面的`sagaWithCatch`函数里面调用，返回effect的返回值。

所以可以有以下的用法：

```typescript
// 任意组合两个promise
const promise1 = dispatch({
  type: effects.key1
});
const promise2 = dispatch({
  type: effects.key2
});
```



#### getReducer

本方法返回一个reducer，该reducer由一个model的所有reducer组合而成。

```typescript
export default function getReducer(reducers, state, handleActions) {
  // Support reducer enhancer
  // e.g. reducers: [realReducers, enhancer]
  if (Array.isArray(reducers)) {
    return reducers[1]((handleActions || defaultHandleActions)(reducers[0], state));
  } else {
    return (handleActions || defaultHandleActions)(reducers || {}, state);
  }
}
```

`handleActions`可以由`_handleActions`钩子注入，或者使用默认defaultHandleActions，下面看默认实现：

```typescript
// 把model.reducers[key]包装成一个规范reducer
function handleAction(actionType, reducer = identify) {
  return (state, action) => {
    const { type } = action;
    if (actionType === type) {
      return reducer(state, action);
    }
    return state;
  };
}
// 返回一个reducer，该reducer会把model.reducers所有reducer都执行一遍
function reduceReducers(...reducers) {
  return (previous, current) => reducers.reduce((p, r) => r(p, current), previous);
}
// defaultHandleActions
function handleActions(handlers, defaultState) {
  // 返回规范reducer的map集合
  const reducers = Object.keys(handlers).map(type => handleAction(type, handlers[type]));
  // map封装成一个reducer
  const reducer = reduceReducers(...reducers);
  // 返回一个层层封装的reducer
  return (state = defaultState, action) => reducer(state, action);
}
```



#### getSaga

有的小伙伴可能对什么是saga认识比较模糊，官网上有一段比较通熟易懂的描述：

> 在 `redux-saga` 的世界里，Sagas 都用 Generator 函数实现。我们从 Generator 里 yield 纯 JavaScript 对象以表达 Saga 逻辑。 我们称呼那些对象为 *Effect*。Effect 是一个简单的对象，这个对象包含了一些给 middleware 解释执行的信息。 

简单的说，saga就是一个Generator函数，通过yield一个个effect，控制一段复杂的流程逻辑。

看到effect小伙伴可能有些疑惑，model.effects[key]不也是Generator函数嘛，为什么叫effect呢？

这里需要说明一下，saga跟effect更多是概念上的意思，包含一段流程逻辑的Generator函数可以做saga函数，如果这个Generator函数是被yield的，我们也可以称之为effect，因为被yield的是流程的一部分。

> 一个 Saga 所做的实际上是组合那些所有的 Effect，共同实现所需的控制流。



闲话少叙，继续看源码，本方法返回一个包装后的saga。

```typescript
export default function getSaga(effects, model, onError, onEffect, opts = {}) {
  return function*() {
    for (const key in effects) {
      if (Object.prototype.hasOwnProperty.call(effects, key)) {
        // 把effect包装成一个watcher
        const watcher = getWatcher(key, effects[key], model, onError, onEffect, opts);
        // fork一个watcher线程
        const task = yield sagaEffects.fork(watcher);
        // fork一个辅助线程处理cancel逻辑
        yield sagaEffects.fork(function*() {
          yield sagaEffects.take(`${model.namespace}/@@CANCEL_EFFECTS`);
          yield sagaEffects.cancel(task);
        });
      }
    }
  };
}
```

显而易见，getSaga返回一个Generator函数，等到后续saga执行的时候，给每个model.effects[key]都fork一个watcher线程跟一个辅助线程。细看watcher线程生成过程之前先了解一下相关概念：

##### Watcher/Worker

指的是一种使用两个单独的 Saga 来组织控制流的方式。

- Watcher: 监听发起的 action 并在每次接收到 action 时 `fork` 一个 worker。
- Worker: 处理 action 并结束它。

```javascript
function* watcher() {
  while(true) {
    const action = yield take(ACTION)
    yield fork(worker, action.payload)
  }
}
function* worker(payload) {
  // ... do some stuff
}
```

##### getWatcher

```typescript
function getWatcher(key, _effect, model, onError, onEffect, opts) {
  let effect = _effect;
  let type = 'takeEvery';
  let ms;
  let delayMs;

  if (Array.isArray(_effect)) {
    // 通过数组的形式，传入effect的type
    [effect] = _effect;
    const opts = _effect[1];
    if (opts && opts.type) {
      ({ type } = opts);
    }
  }

  function* sagaWithCatch(...args) {
    // 封装报错处理逻辑
    // ....
  }
  // 提供onEffect钩子处理接口，可以对effect进行二次封装
  const sagaWithOnEffect = applyOnEffect(onEffect, sagaWithCatch, model, key);
  
  // 返回各种type的watcher
  switch (type) {
    case 'watcher':
      return sagaWithCatch;
    case 'takeLatest':
      return function*() {
        yield sagaEffects.takeLatest(key, sagaWithOnEffect);
      };
    case 'throttle':
      return function*() {
        yield sagaEffects.throttle(ms, key, sagaWithOnEffect);
      };
    case 'poll':
      return function*() {
        function delay(timeout) {
          return new Promise(resolve => setTimeout(resolve, timeout));
        }
        function* pollSagaWorker(sagaEffects, action) {
          const { call } = sagaEffects;
          while (true) {
            yield call(sagaWithOnEffect, action);
            yield call(delay, delayMs);
          }
        }
        const { call, take, race } = sagaEffects;
        while (true) {
          const action = yield take(`${key}-start`);
          yield race([call(pollSagaWorker, sagaEffects, action), take(`${key}-stop`)]);
        }
      };
    default:
      return function*() {
        yield sagaEffects.takeEvery(key, sagaWithOnEffect);
      };
  }
}
```

##### sagaWithCatch

此方法主要是对effect做了错误处理的封装，并通过action传进来的resolve方法返回effect的返回值。

```typescript
  function* sagaWithCatch(...args) {
    // 接收promiseMiddleware传进来的resolve跟reject
    const { __dva_resolve: resolve = noop, __dva_reject: reject = noop } =
      args.length > 0 ? args[0] : {};
    try {
      yield sagaEffects.put({ type: `${key}${NAMESPACE_SEP}@@start` });
      // 执行最终的effect
      // createEffects包装了put,take方法
      const ret = yield effect(...args.concat(createEffects(model, opts)));
      yield sagaEffects.put({ type: `${key}${NAMESPACE_SEP}@@end` });
      resolve(ret);
    } catch (e) {
      onError(e, {
        key,
        effectArgs: args,
      });
      if (!e._dontReject) {
        reject(e);
      }
    }
  }
```

##### applyOnEffect

通过onEffect钩子进行二次封装，返回最终的effect：

```typescript
function applyOnEffect(fns, effect, model, key) {
  for (const fn of fns) {
    // 遍历执行所有onEffect钩子
    effect = fn(effect, sagaEffects, model, key);
  }
  return effect;
}
```

##### createEffects

在effects里面执行take或者put的时候，不需要加上namespace作type，是因为两个方法被重写了：

```typescript
function createEffects(model, opts) {
  function put(action) {
    const { type } = action;
    // 重写
    return sagaEffects.put({ ...action, type: prefixType(type, model) });
  }
  function putResolve(action) {
    const { type } = action;
    // 重写
    return sagaEffects.put.resolve({
      ...action,
      type: prefixType(type, model),
    });
  }
  put.resolve = putResolve;
  function take(type) {
    if (typeof type === 'string') {
      // 重写
      return sagaEffects.take(prefixType(type, model));
    } else if (Array.isArray(type)) {
      return sagaEffects.take(
        type.map(t => {
          if (typeof t === 'string') {
            return prefixType(t, model);
          }
          return t;
        }),
      );
    } else {
      return sagaEffects.take(type);
    }
  }
  return { ...sagaEffects, put, take };
}
```

#### createStore

此方法很简单，就是调用redux的createStore方法创建store：

```typescript
export default function({
  reducers,
  initialState,
  plugin,
  sagaMiddleware,
  promiseMiddleware,
  createOpts: { setupMiddlewares = returnSelf },
}) {
  // hook注入函数
  const extraEnhancers = plugin.get('extraEnhancers');
  const extraMiddlewares = plugin.get('onAction');
  // 组合middleware
  const middlewares = setupMiddlewares([
    promiseMiddleware,
    sagaMiddleware,
    ...flatten(extraMiddlewares),
  ]);

  const composeEnhancers =
    process.env.NODE_ENV !== 'production' && win.__REDUX_DEVTOOLS_EXTENSION_COMPOSE__
      ? win.__REDUX_DEVTOOLS_EXTENSION_COMPOSE__({ trace: true, maxAge: 30 })
      : compose;
  // applyMiddleware(...middlewares)第一位compose后最后执行
  const enhancers = [applyMiddleware(...middlewares), ...extraEnhancers];
  // 创建store
  return createStore(reducers, initialState, composeEnhancers(...enhancers));
}
```



#### runSubscription

Subscription 语义是订阅，用于订阅一个数据源，然后根据条件 dispatch 需要的 action。数据源可以是当前的时间、服务器的 websocket 连接、keyboard 输入、geolocation 变化、history 路由变化等等。

start方法执行时收集所有listener：

```typescript
// Run subscriptions
const unlisteners = {};
for (const model of this._models) {
  if (model.subscriptions) {
     unlisteners[model.namespace] = runSubscription(model.subscriptions, model, app, onError);
  }
}
```

下面看runSubscrptions函数：

```typescript
export function run(subs, model, app, onError) {
  const funcs = [];
  const nonFuncs = [];
  for (const key in subs) {
    if (Object.prototype.hasOwnProperty.call(subs, key)) {
      const sub = subs[key];
      // 直接执行订阅逻辑，返回值为unlistener,卸载时执行
      const unlistener = sub(
        {
          dispatch: prefixedDispatch(app._store.dispatch, model),
          history: app._history,
        },
        onError,
      );
      if (isFunction(unlistener)) {
        funcs.push(unlistener);
      } else {
        nonFuncs.push(key);
      }
    }
  }
  // 返回值按照函数非函数收集
  return { funcs, nonFuncs };
}
```

有runSubscrptions当然就有unRunSubscrptions：

```typescript
export function unlisten(unlisteners, namespace) {
  if (!unlisteners[namespace]) return;
  const { funcs, nonFuncs } = unlisteners[namespace];
  // 执行unlistener
  for (const unlistener of funcs) {
    unlistener();
  }
  delete unlisteners[namespace];
}
```



## Dva-loading

源码解读完毕，我们通过分析dva-loading插件来验证一下学习成果吧！

先看插件整体源码：

```typescript
function createLoading(opts = {}) {
  const namespace = opts.namespace || NAMESPACE;

  const { only = [], except = [] } = opts;
  if (only.length > 0 && except.length > 0) {
    throw Error('It is ambiguous to configurate `only` and `except` items at the same time.');
  }

  const initialState = {
    global: false,
    models: {},
    effects: {},
  };

  const extraReducers = {
    [namespace](state = initialState, { type, payload }) {
      const { namespace, actionType } = payload || {};
      // ...
    },
  };

  function onEffect(effect, { put }, model, actionType) {
    //...
  }

  return {
    extraReducers,
    onEffect,
  };
}
```

该插件返回了extraReducers、onEffect两个钩子函数。extraReducers顾名思义，就是添加额外的reducer。

### onEffect

onEffect的作用我们在applyOnEffect小节时介绍过，作用就是对effect进行二次封装。

```typescript
function onEffect(effect, { put }, model, actionType) {
    const { namespace } = model;
    if (
      (only.length === 0 && except.length === 0) ||
      (only.length > 0 && only.indexOf(actionType) !== -1) ||
      (except.length > 0 && except.indexOf(actionType) === -1)
    ) {
      // 二次封装，在effect执行之前分别更新store
      return function*(...args) {
        yield put({ type: SHOW, payload: { namespace, actionType } });
        yield effect(...args);
        yield put({ type: HIDE, payload: { namespace, actionType } });
      };
    } else {
      return effect;
    }
  }
```

很简单，就是在effect执行前后分别更新store，把effect作为一个最小的loading流程。

### extraReducers

```typescript
const extraReducers = {
    [namespace](state = initialState, { type, payload }) {
      const { namespace, actionType } = payload || {};
      let ret;
      switch (type) {
        case SHOW:
          ret = {
            ...state,
            global: true,
            models: { ...state.models, [namespace]: true },
            effects: { ...state.effects, [actionType]: true },
          };
          break;
        case HIDE: {
          const effects = { ...state.effects, [actionType]: false };
          const models = {
            ...state.models,
            [namespace]: Object.keys(effects).some(actionType => {
              const _namespace = actionType.split('/')[0];
              if (_namespace !== namespace) return false;
              return effects[actionType];
            }),
          };
          const global = Object.keys(models).some(namespace => {
            return models[namespace];
          });
          ret = {
            ...state,
            global,
            models,
            effects,
          };
          break;
        }
        default:
          ret = state;
          break;
      }
      return ret;
    },
  };
```

此reducer用来接收show跟hide的action。上面代码中的actionType是包含namespace的key。也就是models对象保存namespace的loading状态；effects对象分别保存各个effect的loading状态。

