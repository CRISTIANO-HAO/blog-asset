# umi源码解析

本文阅读的版本是`"version": "3.2.22"`。umi项目是采用lerna管理的多模块架构，所有包都在packages目录下。

## 入口

umi提供了`umi`命令，入口是`packages/umi/bin/umi.js`：

```typescript
#!/usr/bin/env node

const resolveCwd = require('resolve-cwd');

const { name, bin } = require('../package.json');
const localCLI = resolveCwd.silent(`${name}/${bin['umi']}`);
if (!process.env.USE_GLOBAL_UMI && localCLI && localCLI !== __filename) {
  const debug = require('@umijs/utils').createDebug('umi:cli');
  debug('Using local install of umi');
  require(localCLI);
} else {
  require('../lib/cli');
}
```

该文件又加载了`packages/umi/src/cli.ts`文件：

```javascript
try {
    switch (args._[0]) {
      // umi dev 情况
      case 'dev':
        const child = fork({
          scriptPath: require.resolve('./forkedDev'),
        });
        // ...
        break;
      default:
        const name = args._[0];
        if (name === 'build') {
          process.env.NODE_ENV = 'production';
        }
        await new Service({
          cwd: getCwd(),
          pkg: getPkg(process.cwd()),
        }).run({
          name,
          args,
        });
        break;
    }
  } catch (e) {
  		//...
  }
```

跳到`forkedDev`去看，发现所有命令实质上都是要执行 `new Service().run()`，由此可知Service是核心类。



## service核心类

`packages/umi/src/ServiceWithBuiltIn.ts` 继承了service类，用于实例化时添加内置presets跟plugins：

```javascript
class Service extends CoreService {
  constructor(opts: IServiceOpts) {
    process.env.UMI_VERSION = require('../package').version;
    process.env.UMI_DIR = dirname(require.resolve('../package'));

    super({
      ...opts,
      presets: [
        require.resolve('@umijs/preset-built-in'),
        ...(opts.presets || []),
      ],
      plugins: [require.resolve('./plugins/umiAlias'), ...(opts.plugins || [])],
    });
  }
}
```

`require.resolve('@umijs/preset-built-in')`返回的是该模块文件绝对路径。也就是说presets跟plugins初始化是都是相关模块路径组成的数组。

### 实例化过程

下面看service的实例化过程：

```typescript
constructor(opts: IServiceOpts) {
    super();
		// 项目根路径
    this.cwd = opts.cwd || process.cwd();
    // package.json
    this.pkg = opts.pkg || this.resolvePackage();
  	// 环境变量
    this.env = opts.env || process.env.NODE_ENV;
    // register babel before config parsing
    this.babelRegister = new BabelRegister();
    // load .env or .local.env
    this.loadEnv();
    // get user config without validation
    this.configInstance = new Config({
      cwd: this.cwd,
      service: this,
      localConfig: this.env === 'development',
    });
  	// 从以下几个文件中加载配置信息，优先级是'.umirc.ts','.umirc.js','config/config.ts','config/config.js'
    // 支持多环境多份配置，以'.umirc.ts'为例，配置process.env.UMI_ENV=xxx时，deepmerge文件'.umirc.ts'跟'.umirc.xxx.ts'的配置；默认合并'.umirc.local.ts'；
    this.userConfig = this.configInstance.getUserConfig();
    // get paths
    this.paths = getPaths({
      cwd: this.cwd,
      config: this.userConfig!,
      env: this.env,
    });
    // setup initial presets and plugins
    const baseOpts = {
      pkg: this.pkg,
      cwd: this.cwd,
    };
    // 通过pathToObj返回合并配置后各个preset模块路径的封装对象
    this.initialPresets = resolvePresets({
      ...baseOpts,
      presets: opts.presets || [],
      userConfigPresets: this.userConfig.presets || [],
    });
    // 通过pathToObj返回合并配置后各个plugin模块路径的封装对象
    this.initialPlugins = resolvePlugins({
      ...baseOpts,
      plugins: opts.plugins || [],
      userConfigPlugins: this.userConfig.plugins || [],
    });
    this.babelRegister.setOnlyMap({
      key: 'initialPlugins',
      value: lodash.uniq([
        ...this.initialPresets.map(({ path }) => path),
        ...this.initialPlugins.map(({ path }) => path),
      ]),
    });
  }
```

#### getPluginsOrPresets

看看`resolvePlugins`跟`resolvePlugins`中获取plugin跟preset的方法：

```typescript
const RE = {
  [PluginType.plugin]: /^(@umijs\/|umi-)plugin-/,
  [PluginType.preset]: /^(@umijs\/|umi-)preset-/,
};

function getPluginsOrPresets(type: PluginType, opts: IOpts): string[] {
  const upperCaseType = type.toUpperCase();
  return [
    // opts，内置的
    ...((opts[type === PluginType.preset ? 'presets' : 'plugins'] as any) ||
      []),
    // env，通过环境变量传入
    ...(process.env[`UMI_${upperCaseType}S`] || '').split(',').filter(Boolean),
    // dependencies，package.json读取
    ...Object.keys(opts.pkg.devDependencies || {})
      .concat(Object.keys(opts.pkg.dependencies || {}))
      .filter(isPluginOrPreset.bind(null, type)),
    // user config； 配置文件传入
    ...((opts[
      type === PluginType.preset ? 'userConfigPresets' : 'userConfigPlugins'
    ] as any) || []),
  ].map((path) => {
    // 。。。
  });
}
```

这里重点关注几点：

- 配置文件加载：从以下几个文件中加载配置信息，优先级是'.umirc.ts','.umirc.js','config/config.ts','config/config.js'；支持多环境多份配置，以'.umirc.ts'为例，配置process.env.UMI_ENV=xxx时，深度合并（deepmerge）文件'.umirc.ts'跟'.umirc.xxx.ts'的配置；默认合并'.umirc.local.ts'；
- `resolvePlugins`跟`resolvePlugins`中获取plugin跟preset的来源：

  - 内置的plugin跟preset；
- 配置文件的config跟preset；
  - 读取`package.json`文件`dependencies`跟`devDependencies`中的相关plugin跟preset；
  - 环境变量`process.env[UMI_${upperCaseType}S]`注入；

####  pathToObj

封装preset跟plugin的路径为统一的对象：

```typescript
export function pathToObj({
  type,
  path,
  cwd,
}: {
  type: PluginType;
  path: string;
  cwd: string;
}) {
  // 通过path获取id跟key并做一些处理，略
  // ...
  
  // 生成规范ID
  id = id.replace('@umijs/preset-built-in/lib/plugins', '@@');
  id = id.replace(/\.js$/, '');
	// 生成规范key
  const key = isPkgPlugin
    ? pkgNameToKey(pkg!.name, type)
    : nameToKey(basename(path, extname(path)));
  // 封装成统一对象，方便后续处理
  return {
    id,
    key,
    path: winPath(path),
    apply() {
      // 到时候就是调用apply方法加载执行plugin跟preset
      try {
        const ret = require(path);
        // 导出es模块
        return compatESModuleRequire(ret);
      } catch (e) {
        throw new Error(`Register ${type} ${path} failed, since ${e.message}`);
      }
    },
    defaultConfig: null,
  };
}
```

`apply` 方法返回的是 `require(path)` 模块对象，并且做了es跟cjs模块兼容处理。

### run方法

Service实例化之后紧接着就是调用`run`执行完整程序：

```typescript
async run({ name, args = {} }: { name: string; args?: any }) {
    args._ = args._ || [];
    // shift the command itself
    if (args._[0] === name) args._.shift();
    this.args = args;
    // 主要是调用initPresetsAndPlugins方法初始化preset跟plugin，跟调用钩子方法
    await this.init();
    this.setStage(ServiceStage.run);
    // 调用钩子方法，后面重点关注这个方法
    await this.applyPlugins({
      key: 'onStart',
      type: ApplyPluginsType.event,
      args: {
        args,
      },
    });
    // 真正执行dev, build等命令
    return this.runCommand({ name, args });
  }
```

`run` 方法的核心流程是初始化各种preset跟plugin，并依次调用 `onPluginReady` 、`modifyDefaultConfig` 、 `modifyConfig` 、`modifyPaths`、`onStart` 等钩子。



### initPresetsAndPlugins

现在来详细看preset跟plugin的注册过程。

```typescript
async initPresetsAndPlugins() {
    this.setStage(ServiceStage.initPresets);
    this._extraPlugins = [];
    while (this.initialPresets.length) {
      await this.initPreset(this.initialPresets.shift()!);
    }

    this.setStage(ServiceStage.initPlugins);
    this._extraPlugins.push(...this.initialPlugins);
    while (this._extraPlugins.length) {
      await this.initPlugin(this._extraPlugins.shift()!);
    }
  }
```

非常简单，就是遍历数组逐一注册。下面看各自注册过程：

#### initPreset

```typescript
async initPreset(preset: IPreset) {
    const { id, key, apply } = preset;
    preset.isPreset = true;
    // 封装一个统一的PluginAPI接口，后面细看；需要注意的是公用一个service实例；
    const api = this.getPluginAPI({ id, key, service: this });

    // 注册插件
    this.registerPlugin(preset);
    // 调用前面说的apply方法，执行plugin的功能逻辑；
    // preset可以返回presets跟plugins，用递归注册；初始化@umijs/preset-built-in的时候用到plugins的实现处理；
    const { presets, plugins, ...defaultConfigs } = await this.applyAPI({
      api,
      apply,
    });
		
    // 提供了让preset跟plugin注册额外preset跟plugin的入口
    // register extra presets and plugins
    if (presets) {
      // 插到最前面，下个 while 循环优先执行
      this._extraPresets.splice(
        0,
        0,
        ...presets.map((path: string) => {
          return pathToObj({
            type: PluginType.preset,
            path,
            cwd: this.cwd,
          });
        }),
      );
    }

    // 深度优先
    const extraPresets = lodash.clone(this._extraPresets);
    // 清空，等待下个preset要注册的preset；
    this._extraPresets = [];
    while (extraPresets.length) {
      // 递归注册
      await this.initPreset(extraPresets.shift()!);
    }

    if (plugins) {
      // plugins则留给下一步initPlugin处理
      this._extraPlugins.push(
        ...plugins.map((path: string) => {
          return pathToObj({
            type: PluginType.plugin,
            path,
            cwd: this.cwd,
          });
        }),
      );
    }
  }
```



#### initPlugin

```typescript
// 跟initPreset同理
async initPlugin(plugin: IPlugin) {
    const { id, key, apply } = plugin;

    const api = this.getPluginAPI({ id, key, service: this });

    // register before apply
    this.registerPlugin(plugin);
    await this.applyAPI({ api, apply });
  }
```

可以看出initPreset跟initPlugin非常相似，它们有什么不同呢？

- 从语义上看，preset是前置的意思，顾名思义，即提供前置功能，有点功能集的意思；plugin则是插件的意思，作用是提供某特定功能。

- 从功能上看，preset可以用来注册preset跟plugin，而plugin则不行，只能提供某一特定功能；

> 之所以这样区分有利于各种plugin的组合跟管理；同时兼顾了plugin使用上的灵活性跟preset聚合使用的便利性。



#### getPluginAPI

每个preset跟plugin注入时创建一个独立的PluginAPI对象中，并且通过Proxy共享注册在service实例上的`pluginMethods`对象上面的方法。

当preset或者plugin中调用 `api.props()` 时，实际上调用的是service实例上`pluginMethods`对象上的方法`this.pluginMethods[prop]`

```typescript
getPluginAPI(opts: any) {
    const pluginAPI = new PluginAPI(opts);
    // register built-in methods
    [
      'onPluginReady',
      'modifyPaths',
      'onStart',
      'modifyDefaultConfig',
      'modifyConfig',
    ].forEach((name) => {
      pluginAPI.registerMethod({ name, exitsError: false });
    });

    return new Proxy(pluginAPI, {
      get: (target, prop: string) => {
        // 由于 pluginMethods 需要在 register 阶段可用
        // 必须通过 proxy 的方式动态获取最新，以实现边注册边使用的效果
        if (this.pluginMethods[prop]) return this.pluginMethods[prop];
        if (
          [
            'applyPlugins',
            'ApplyPluginsType',
            'EnableBy',
            'ConfigChangeType',
            'babelRegister',
            'stage',
            'ServiceStage',
            'paths',
            'cwd',
            'pkg',
            'userConfig',
            'config',
            'env',
            'args',
            'hasPlugins',
            'hasPresets',
          ].includes(prop)
        ) {
          return typeof this[prop] === 'function'
            ? this[prop].bind(this)
            : this[prop];
        }
        return target[prop];
      },
    });
  }
```

pluginMethods上面的方法是如何注册上去的呢？这一段比较绕，放到下一段集中讲。



#### applyAPI

`PluginAPI` 实例化之后，作为参数传入到具体的preset或者plugin函数中，通过`opts.apply()(opts.api)` 初始化preset跟plugin。

```typescript
async applyAPI(opts: { apply: Function; api: PluginAPI }) {
    // 每个preset跟plugin都会接收到一个PluginAPI实例参数，从中可以拿到service实例等信息
    let ret = opts.apply()(opts.api);
    if (isPromise(ret)) {
      ret = await ret;
    }
    return ret || {};
  }
```



### PluginAPI

#### 钩子函数机制

umi功能可看作是各类钩子函数的集合，所以梳理钩子函数的注册与使用流程非常关键。这小结先看个总览，后续通过源码详细展开。

> 这一段要注意钩子跟钩子函数的区别，钩子指的是注册钩子函数的函数，一个钩子对应多个钩子函数。

##### 注册钩子

所有的钩子都是挂载到service实例上的 `pluginMethods` 对象上。

```javascript
// 通过pluginApi实例注册钩子函数到pluginMethods对象上，如果已经存在，则不会重复注册
pluginApi.registerMethod(name) ==> this.service.pluginMethods[name]
```

##### 注册钩子函数

```javascript
// 一个钩子可以注册多个钩子函数
pluginApi.name(fn1) ==> proxy ==> this.service.pluginMethods[name](fn1)
pluginApi.name(fn2) ==> proxy ==> this.service.pluginMethods[name](fn2)
```

`pluginApi` 对象被代理，非过滤属性会调用注册在 `pluginMethods` 对象上的钩子，从而注册钩子函数到service实例上。

这样，不同的preset跟plugin针对某个钩子可以注册不同的钩子函数，并按照注册顺序逐一调用。

##### 触发钩子函数

```javascript
// pluginApi.applyPlugins === service.applyPlugins
this.applyPlugins({key, type, initialValue}) {
  const hooks = this.hooks[key] || [];
  switch(type) {
      // ...
  }
}
```

通过key找到对应的钩子函数数组，根据type逐一执行。

#### registerMethod

这一段是umi整个源码的精华所在，所谓的preset跟plugin都是通过在 `pluginMethods` 对象上面注册各种各样类型的钩子函数，然后通过钩子函数掌控整个umi的功能流程实现。

下面看钩子函数的注册过程，方法来自`packages/core/src/Service/PluginAPI.ts`：

```typescript
registerMethod({
    name,
    fn,
    exitsError = true,
  }) {
    // ...
    this.service.pluginMethods[name] =
      fn ||
      // 这里不能用 arrow function，this 需指向执行此方法的 PluginAPI
      // 否则 pluginId 会不会，导致不能正确 skip plugin
      function (fn: Function) {
        // 包装成一个hook对象
        const hook = {
          key: name,
          ...(utils.lodash.isPlainObject(fn) ? fn : { fn }),
        };
        // 注册到service实例上的hooksByPluginId对象上，最终又拷贝到hooks对象上
        this.register(hook);
      };
  }
```

> 这里值得注意一下，如果传参数fn，是直接注册方法，如果没有参数fn，则是注册钩子。

调用registerMethod方法来注册钩子。最终的钩子都会挂载为 `this.service.pluginMethods[name]`。此时钩子函数可以接受多个函数，然后封装为hook，注册到`this.service.hooksByPluginId`上。

再看register方法：

```typescript
register(hook: IHook) {
    this.service.hooksByPluginId[this.id] = (
      this.service.hooksByPluginId[this.id] || []
    ).concat(hook);
}
```

比较简单，需要注意的是每一个钩子可以注册多个钩子函数fn，并通过数组存储起来。

```javascript
// hooksByPluginId -> hooks
// hooks is mapped with hook key, prepared for applyPlugins()
Object.keys(this.hooksByPluginId).forEach((id) => {
  const hooks = this.hooksByPluginId[id];
  hooks.forEach((hook) => {
    const { key } = hook;
    hook.pluginId = id;
    this.hooks[key] = (this.hooks[key] || []).concat(hook);
  });
});
```

`hooksByPluginId` 上所有的hook最终拷贝到`hooks`对象上，所以`applyPlugins` 时候从hooks对象上取对应的hook数组即可。



#### 内置的钩子

内置的钩子是何时内置进去的呢？主要有两个地方：

- getPluginAPI方法

  ```typescript
  getPluginAPI(opts: any) {
      const pluginAPI = new PluginAPI(opts);
      // register built-in methods
      [
        'onPluginReady',
        'modifyPaths',
        'onStart',
        'modifyDefaultConfig',
        'modifyConfig',
      ].forEach((name) => {
        pluginAPI.registerMethod({ name, exitsError: false });
      });
     // 。。。
   }
  ```

- 通过内置preset `@umijs/preset-built-in`注入

  该文件路径是`packages/preset-built-in/src/index.ts` 。

  ```typescript
  export default function () {
    return {
      plugins: [
        // register methods
        require.resolve('./plugins/registerMethods'),
        // misc
        require.resolve('./plugins/routes'),
        // generate files
        require.resolve('./plugins/generateFiles/core/history'),
        // ...
        // bundle configs
        require.resolve('./plugins/features/alias'),
        // ...
        // html
        require.resolve('./plugins/features/html/favicon'),
        //...
        // commands
        require.resolve('./plugins/commands/build/build'),
        //...
      ],
    };
  }
  
  ```

  这里就是initPresetsAndPlugins小节提到的通过preset批量注册plugin。

  其中首先注册的是`require.resolve('./plugins/registerMethods')`:

  ```typescript
  export default function (api: IApi) {
    [
      'onGenerateFiles',
      'onBuildComplete',
      'onExit',
      'onPatchRoute',
      'onPatchRouteBefore',
      'onPatchRoutes',
      'onPatchRoutesBefore',
      'onDevCompileDone',
      'addBeforeMiddewares',
      'addDepInfo',
      'addDevScripts',
      'addMiddewares',
      'addRuntimePlugin',
      'addRuntimePluginKey',
      'addUmiExports',
      'addProjectFirstLibraries',
      'addPolyfillImports',
      'addEntryImportsAhead',
      'addEntryImports',
      'addEntryCodeAhead',
      'addEntryCode',
      'addHTMLMetas',
      'addHTMLLinks',
      'addHTMLStyles',
      'addHTMLHeadScripts',
      'addHTMLScripts',
      'addTmpGenerateWatcherPaths',
      'chainWebpack',
      'modifyHTML',
      'modifyBundler',
      'modifyBundleConfigOpts',
      'modifyBundleConfig',
      'modifyBundleConfigs',
      'modifyBabelOpts',
      'modifyBabelPresetOpts',
      'modifyBundleImplementor',
      'modifyHTMLChunks',
      'modifyDevHTMLContent',
      'modifyExportRouteMap',
      'modifyProdHTMLContent',
      'modifyPublicPathStr',
      'modifyRendererPath',
      'modifyRoutes',
    ].forEach((name) => {
      // 注册钩子
      api.registerMethod({ name });
    });
    
    // 注册方法
    api.registerMethod({
      name: 'writeTmpFile',
      fn({
        path,
        content,
        skipTSCheck = true,
      }) {
        // ...
      },
    });
  }
  
  ```

  ###### 功能函数挂载

  钩子注册后，就是各种功能钩子函数的挂载了，下面随便挑两个来看看。

  ```typescript
  // packages/preset-built-in/src/plugins/generateFiles/umi.ts
  api.onGenerateFiles(async (args) => {
      const umiTpl = readFileSync(join(__dirname, 'umi.tpl'), 'utf-8');
      const rendererPath = await api.applyPlugins({
        key: 'modifyRendererPath',
        type: api.ApplyPluginsType.modify,
        initialValue: renderReactPath,
      });
      // ...
    });
  ```

  ```typescript
  // packages/preset-built-in/src/plugins/generateFiles/core/polyfill.ts
  api.onGenerateFiles(() => {
      const polyfill = api.config.polyfill;
      api.writeTmpFile({
        // ...
      });
    });
  ```

  以上就是在不同plugin中通过onGenerateFiles钩子把不同功能钩子函数挂载到hooks上。



#### applyPlugins

钩子函数挂之后如何使用呢？在run方法小节时我们提到这个方法，下面来看具体实现：

```typescript
async applyPlugins(opts: {
    key: string;
    type: ApplyPluginsType;
    initialValue?: any;
    args?: any;
  }) {
    const hooks = this.hooks[opts.key] || [];
    switch (opts.type) {
      case ApplyPluginsType.add:
        const tAdd = new AsyncSeriesWaterfallHook(['memo']);
        for (const hook of hooks) {
          if (!this.isPluginEnable(hook.pluginId!)) {
            continue;
          }
          tAdd.tapPromise(
            {
              name: hook.pluginId!,
              stage: hook.stage || 0,
              before: hook.before,
            },
            async (memo: any[]) => {
              const items = await hook.fn(opts.args);
              return memo.concat(items);
            },
          );
        }
        return await tAdd.promise(opts.initialValue || []);
      case ApplyPluginsType.modify:
        const tModify = new AsyncSeriesWaterfallHook(['memo']);
        // ...
        return await tModify.promise(opts.initialValue);
      case ApplyPluginsType.event:
        const tEvent = new AsyncSeriesWaterfallHook(['_']);
        // ...
        return await tEvent.promise();
      default:
        throw new Error(
          `applyPlugin failed, type is not defined or is not matched, got ${opts.type}.`,
        );
    }
  }
```

`AsyncSeriesWaterfallHook`出自tapable包，tapable 是一个类似于nodejs 的EventEmitter 的库, 主要是控制钩子函数的发布与订阅。顾名思义，`AsyncSeriesWaterfallHook`是一个异步顺序瀑布流执行的钩子函数实现，即可以控制顺序执行一系列异步函数，并且上一个函数的返回值会传给下一个函数。

理解这个之后，挂载在各个钩子上的功能函数的执行流程就不言而喻啦。



到这里，umi的所有preset跟plugin都已经初始化配置完成，各类功能函数也已经通过钩子函数挂载成功，并且理解了功能函数的执行机制，接下来就是各类功能的组合使用过程了。



## umi dev

到这其实我们已经跨过了umi源码最困难的部分。是不是感觉也不过如此？😄

下面我们就通过一个具体的命令来看看umi是如何实现整个开发过程的封装的。😁



### registerCommand

追根溯源，dev，build，generate等命令是怎么注册到umi上去的呢？

在上面讲内置钩子函数小节的时候，细心的小伙伴可能注意到了初始化`@umijs/preset-built-in`内置preset的时候加载了一系列的command插件模块。没错，正是初始化这些子plugin的时候注册上去的，下面看`packages/preset-built-in/src/plugins/commands/dev/dev.ts`:

```typescript
export default (api: IApi) => {
  // ...
  api.registerCommand({
    name: 'dev',
    description: 'start a dev server for development',
    fn: async function ({ args }) {
      // 后面看具体实现
      // ...
    },
  });
};

```

再看registerCommand的实现，非常简单：

```typescript
registerCommand(command: ICommand) {
    const { name, alias } = command;
    this.service.commands[name] = command;
    if (alias) {
      this.service.commands[alias] = name;
    }
  }
```

所有的命令函数都挂载在service实例的commands对象上，当service.run()的时候执行的就是对应的命令函数。



### dev命令函数

接下来就把上面省略的命令函数展开来看有没有什么神秘魔法：

```typescript
api.registerCommand({
    name: 'dev',
    description: 'start a dev server for development',
    fn: async function ({ args }) {
      // 通过环境变量、service.config获取配置，初始化各种变量参数
      const defaultPort =
        process.env.PORT || args?.port || api.config.devServer?.port;
      port = await portfinder.getPortPromise({
        port: defaultPort ? parseInt(String(defaultPort), 10) : 8000,
      });
      hostname = process.env.HOST || api.config.devServer?.host || '0.0.0.0';
      process.send?.({ type: 'UPDATE_PORT', port });

      // enable https, HTTP/2 by default when using --https
      const isHTTPS = process.env.HTTPS || args?.https;

      cleanTmpPathExceptCache({
        absTmpPath: paths.absTmpPath!,
      });
      const watch = process.env.WATCH !== 'none';

      // 重点关注，生成umi临时文件，最大魔法之一
      const unwatchGenerateFiles = await generateFiles({ api, watch });
      if (unwatchGenerateFiles) unwatchs.push(unwatchGenerateFiles);

      // 热更新逻辑，有兴趣自行深入
      if (watch) {
        // watch pkg changes
        // ...

        // 监听配置文件变化
        const unwatchConfig = api.service.configInstance.watch({
          userConfig: api.service.userConfig,
          onChange: async ({ pluginChanged, userConfig, valueChanged }) => {
            // plugin变化，则重启服务
            if (pluginChanged.length) {
              // restartServer是通过registerMethod注册进来的功能函数
              api.restartServer();
            }
            if (valueChanged.length) {
              //...
              if (reload) {
                api.restartServer();
              } else {
                api.service.userConfig = api.service.configInstance.getUserConfig();
								
                // 触发各种相关钩子函数逻辑
                //...

                if (regenerateTmpFiles) {
                  await generateFiles({ api });
                } else {
                  fns.forEach((fn) => fn());
                }
              }
            }
          },
        });
        unwatchs.push(unwatchConfig);
      }

      // delay dev server 启动，避免重复 compile
      // https://github.com/webpack/watchpack/issues/25
      // https://github.com/yessky/webpack-mild-compile
      await delay(500);

      // 获取webpack函数跟相关配置
      const {
        bundler,
        bundleConfigs,
        bundleImplementor,
      } = await getBundleAndConfigs({ api, port });
      // 通过webpackDevMiddleware处理
      const opts: IServerOpts = bundler.setupDevServerOpts({
        bundleConfigs: bundleConfigs,
        bundleImplementor,
      });

      //触发各种相关钩子函数逻辑
			// ...
      
			// 部署服务
      server = new Server({
        ...opts,
        compress: true,
        https: !!isHTTPS,
        headers: {
          'access-control-allow-origin': '*',
        },
        proxy: api.config.proxy,
        beforeMiddlewares,
        afterMiddlewares: [
          ...middlewares,
          createRouteMiddleware({ api, sharedMap }),
        ],
        ...(api.config.devServer || {}),
      });
			// 监听端口
      const listenRet = await server.listen({
        port,
        hostname,
      });
      return {
        ...listenRet,
        destroy,
      };
    },
  });
```

上面大略过了一下整个启动流程，下面我们不妨仔细逐一探究一下各个细节😄



### generateFiles

umi为了降低开发的门槛，实现了一定程度上的高度封装，我们不需要像常规的项目一样从零开始搭建一个项目，只需要按照约定配置就能用很小的代码量起一个react项目。源头就是因为umi把这些模版代码通过很多临时文件帮我们生成了。

下面看看`packages/preset-built-in/src/plugins/commands/generateFiles.ts`函数：

```typescript
export default async ({ api, watch }: { api: IApi; watch?: boolean }) => {
  const { paths } = api;
  async function generate() {
    // 关键就是触发onGenerateFiles钩子函数
    await api.applyPlugins({
      key: 'onGenerateFiles',
      type: api.ApplyPluginsType.event,
    });
  }
  const watchers: chokidar.FSWatcher[] = [];
  await generate();
  if (watch) {
    // ...
  }
  function unwatch() {
    watchers.forEach((watcher) => {
      watcher.close();
    });
  }
  // ...
  return unwatch;
};
```

前面讲内置钩子函数的时候我们恰巧也是用`onGenerateFiles`钩子函数举的例子，有兴趣的小伙伴可以回头看看。umi通过`onGenerateFiles`钩子函数注册了一系列的功能函数来生成临时文件，下面我们挑入口临时文件的生成来看看`packages/preset-built-in/src/plugins/generateFiles/umi.ts`:

```typescript
api.onGenerateFiles(async (args) => {
    // 读取模版文件
    const umiTpl = readFileSync(join(__dirname, 'umi.tpl'), 'utf-8');
    // 获取临时文件输出路径
    const rendererPath = await api.applyPlugins({
      key: 'modifyRendererPath',
      type: api.ApplyPluginsType.modify,
      initialValue: renderReactPath,
    });
    // 调用注册在service实例上的writeTmpFile方法
    api.writeTmpFile({
      path: 'umi.ts',
      // 通过Mustache.render生成模版内容
      // 以下调用api.applyPlugins都是获取相关参数值，从而替换模版中的变量
      content: Mustache.render(umiTpl, {
        enableTitle: api.config.title !== false,
        defaultTitle: api.config.title || '',
        rendererPath: winPath(rendererPath),
        runtimePath,
        rootElement: api.config.mountElementId,
        enableSSR: !!api.config.ssr,
        dynamicImport: !!api.config.dynamicImport,
        entryCodeAhead: (
          await api.applyPlugins({
            key: 'addEntryCodeAhead',
            type: api.ApplyPluginsType.add,
            initialValue: [],
          })
        ).join('\r\n'),
        polyfillImports: importsToStr(
          await api.applyPlugins({
            key: 'addPolyfillImports',
            type: api.ApplyPluginsType.add,
            initialValue: [],
          }),
        ).join('\r\n'),
        // ...
      }),
    });
  });
```



### getBundleAndConfigs

umi设置了许多默认的webpack配置，并且提供了`chainWebpack`配置文件入口跟相关钩子函数来修改其配置，下面看其实现：

```typescript
export async function getBundleAndConfigs({
  api,
  port,
}: {
  api: IApi;
  port?: number;
}) {
  // 调用钩子函数获取默认值
  //...

  // get config
  async function getConfig({ type }: { type: IBundlerConfigType }) {
    const env: Env = api.env === 'production' ? 'production' : 'development';
    // 提供各种钩子函数修改配置的入口
    const getConfigOpts = await api.applyPlugins({
      type: api.ApplyPluginsType.modify,
      key: 'modifyBundleConfigOpts',
      initialValue: {
        env,
        type,
        port,
        hot: type === BundlerConfigType.csr && process.env.HMR !== 'none',
        entry: {
          umi: join(api.paths.absTmpPath!, 'umi.ts'),
        },
        //...
        // 提供chainWebpack钩子修改webpack配置
        async chainWebpack(webpackConfig: any, opts: any) {
          return await api.applyPlugins({
            type: api.ApplyPluginsType.modify,
            key: 'chainWebpack',
            initialValue: webpackConfig,
            args: {
              ...opts,
            },
          });
        },
      },
      args: {
        ...bundlerArgs,
        type,
      },
    });
    return await api.applyPlugins({
      type: api.ApplyPluginsType.modify,
      key: 'modifyBundleConfig',
      initialValue: await bundler.getConfig(getConfigOpts),
      args: {
        ...bundlerArgs,
        type,
      },
    });
  }

  const bundler: DefaultBundler = new Bundler({
    cwd: api.cwd,
    config: api.config,
  });
  //...
  const bundleConfigs = await api.applyPlugins({
    type: api.ApplyPluginsType.modify,
    key: 'modifyBundleConfigs',
    initialValue: [await getConfig({ type: BundlerConfigType.csr })].filter(
      Boolean,
    ),
    args: {
      ...bundlerArgs,
      getConfig,
    },
  });

  return {
    bundleImplementor,
    bundler,
    bundleConfigs,
  };
}
```

从上可知，最终调用的是`bundler.getConfig(getConfigOpts)`获取配置，然后getConfig最终调用的是`packages/bundler-webpack/src/getConfig/getConfig.ts`文件的函数，这个函数非常长，通过`webpack-chain`的方式里面设置了很多默认的配置，继续看相关代码：

```typescript
export default async function getConfig(
  opts: IOpts,
): Promise<defaultWebpack.Configuration> {
  const {
    cwd,
    config,
    type,
    env,
    entry,
    hot,
    port,
    bundleImplementor = defaultWebpack,
    modifyBabelOpts,
    modifyBabelPresetOpts,
    miniCSSExtractPluginPath,
    miniCSSExtractPluginLoaderPath,
  } = opts;
  let webpackConfig = new Config();

  webpackConfig.mode(env);

  //...

  // entry
  if (entry) {
    // ...
  }

  // devtool
  const devtool = config.devtool as Config.DevTool;
  webpackConfig.devtool(
    // ...
  );

  // resolve
  // prettier-ignore
  webpackConfig.resolve
    // 不能设为 false，因为 tnpm 是通过 link 处理依赖，设为 false tnpm 下会有大量的冗余模块
    .set('symlinks', true)
    .modules
      .add('node_modules')
      .add(join(__dirname, '../../node_modules'))
      // TODO: 处理 yarn 全局安装时的 resolve 问题
      .end()
    .extensions.merge([
      '.web.js',
      '.wasm',
      '.mjs',
      '.js',
      '.web.jsx',
      '.jsx',
      '.web.ts',
      '.ts',
      '.web.tsx',
      '.tsx',
      '.json',
    ]);
	
  // 一堆配置，有兴趣自行深入
  // ...
  
  // 触发chainWebpack钩子函数修改配置
  if (opts.chainWebpack) {
    webpackConfig = await opts.chainWebpack(webpackConfig, {
      type,
      webpack: bundleImplementor,
      createCSSRule: createCSSRuleFn,
    });
  }
  // 用户配置的 chainWebpack 优先级最高
  if (config.chainWebpack) {
    await config.chainWebpack(webpackConfig, {
      type,
      env,
      webpack: bundleImplementor,
      createCSSRule: createCSSRuleFn,
    });
  }
  let ret = webpackConfig.toConfig() as defaultWebpack.Configuration;

  //...
  return ret;
}
```

到这里就很明显啦，webpack的优先级分别是 `用户配置的 chainWebpack`  >  `chainWbpack钩子函数修改的配置`  > `默认配置`；



## 插件机制

umi的插件机制是建立在前面所提到的钩子函数机制基础上。通过各种钩子，提供注册各种插件的入口。

### 插件收集

在umi项目src目录下，我们可以看到 `.umi` 文件夹，里面有umi生成的临时文件。这些文件是由内置的plugin生成的。功能入口文件： `packages/preset-built-in/src/index.ts`：

```javascript
export default function () {
  return {
    plugins: [
      // register methods
      require.resolve('./plugins/registerMethods'),

      // misc
      require.resolve('./plugins/routes'),

      // generate files
      require.resolve('./plugins/generateFiles/core/history'),
      require.resolve('./plugins/generateFiles/core/plugin'),
      require.resolve('./plugins/generateFiles/core/polyfill'),
      require.resolve('./plugins/generateFiles/core/routes'),
      require.resolve('./plugins/generateFiles/core/umiExports'),
      require.resolve('./plugins/generateFiles/core/configTypes'),
      require.resolve('./plugins/generateFiles/umi'),
      
      // ...
    ]
  }
}
```

下面具体看plugin文件的实现，入口`packages/preset-built-in/src/plugins/generateFiles/core/plugin.ts` :

- addRuntimePluginKey：用于收集plugin暴露的方法名，可以多不能少；
- addRuntimePlugin：收集注册插件入口文件路径；

```javascript
export default function (api: IApi) {
  api.onGenerateFiles(async (args) => {
    // 插件的功能的key，一个plugin可以对应多个，需要设置，不然无法通过校验，也就无法调同名key的函数
    const validKeys = await api.applyPlugins({
      key: 'addRuntimePluginKey',
      type: api.ApplyPluginsType.add,
      initialValue: [
        'modifyClientRenderOpts',
        'patchRoutes',
        'rootContainer',
        'render',
        'onRouteChange',
      ],
    });
    // 插件入口文件的路径
    const plugins = await api.applyPlugins({
      key: 'addRuntimePlugin',
      type: api.ApplyPluginsType.add,
      initialValue: [
        getFile({
          base: paths.absSrcPath!,
          fileNameWithoutExt: 'app',
          type: 'javascript',
        })?.path,
      ].filter(Boolean),
    });
    // 生成plugin.ts临时文件，validKeys注入其中用于校验
    api.writeTmpFile({
      path: 'core/plugin.ts',
      content: Mustache.render(
        readFileSync(join(__dirname, 'plugin.tpl'), 'utf-8'),
        {
          validKeys,
          runtimePath,
        },
      ),
    });
    // 生成pluginRegister.ts临时文件
    api.writeTmpFile({
      path: 'core/pluginRegister.ts',
      content: Mustache.render(
        readFileSync(join(__dirname, 'pluginRegister.tpl'), 'utf-8'),
        {
          plugins: plugins.map((plugin: string, index: number) => {
            return {
              index,
              path: winPath(plugin),
            };
          }),
        },
      ),
    });
  });

  api.addUmiExports(() => {
    return {
      specifiers: ['plugin'],
      source: `./plugin`,
    };
  });
}
```

此plugin用于生成`pluginRegister.ts`  跟 `plugin.ts` 文件。前者搜集所有注册的插件的入口文件路径并注册，后者则是注册运行时插件的功能函数名称。

`src/.umi/core/pluginRegister.ts` :

```javascript
import { plugin } from './plugin';
import * as Plugin_0 from '/Users/haotengfei/Desktop/work/project/admin-frontend/src/app.tsx';
import * as Plugin_1 from '/Users/haotengfei/Desktop/work/project/admin-frontend/node_modules/umi-plugin-antd-icon-config/lib/app.js';
import * as Plugin_2 from '/Users/haotengfei/Desktop/work/project/admin-frontend/src/.umi/plugin-access/rootContainer.ts';
import * as Plugin_3 from '/Users/haotengfei/Desktop/work/project/admin-frontend/src/.umi/plugin-dva/runtime.tsx';

plugin.register({
  apply: Plugin_0,
  path: '/Users/haotengfei/Desktop/work/project/admin-frontend/src/app.tsx',
});
plugin.register({
  apply: Plugin_1,
  path: '/Users/haotengfei/Desktop/work/project/admin-frontend/node_modules/umi-plugin-antd-icon-config/lib/app.js',
});
plugin.register({
  apply: Plugin_2,
  path: '/Users/haotengfei/Desktop/work/project/admin-frontend/src/.umi/plugin-access/rootContainer.ts',
});
// ...
```

`src/.umi/core/plugin.ts` :

```javascript
import { Plugin } from '/Users/haotengfei/Desktop/work/source/admin-pages/examples/node_modules/@umijs/runtime';

const plugin = new Plugin({
  validKeys: ['modifyClientRenderOpts','patchRoutes','rootContainer','render','onRouteChange','dva','getInitialState','initialStateConfig','locale','locale','request',],
});

export { plugin };
```



### Plugin

此 `Plugin` 类区别于前面提到的 `PluginAPI` 类，前者是运行时(runtime)使用，后者是编译时使用。

其中`register(plugin: IPlugin)`方法用于注册插件，apply指向plugin模块对象，模块对象导出的功能函数通过key分类存储在hooks对象上。需要注意的是，模块对象暴露的方法需要在validKeys数组里提前声明。

`applyPlugins` 方法根据key与type执行相应的钩子函数数组。

```javascript
export default class Plugin {
  validKeys: string[];
  hooks: {
    [key: string]: any;
  } = {};

  constructor(opts?: IOpts) {
    this.validKeys = opts?.validKeys || [];
  }
  
  // 注册插件，apply指向plugin模块对象，模块对象暴露的方法需要在validKeys数组里提前声明。
  register(plugin: IPlugin) {
    assert(!!plugin.apply, `register failed, plugin.apply must supplied`);
    assert(!!plugin.path, `register failed, plugin.path must supplied`);
    Object.keys(plugin.apply).forEach((key) => {
      assert(
        this.validKeys.indexOf(key) > -1,
        `register failed, invalid key ${key} from plugin ${plugin.path}.`,
      );
      if (!this.hooks[key]) this.hooks[key] = [];
      this.hooks[key] = this.hooks[key].concat(plugin.apply[key]);
    });
  }
  
	// 根据key选择hook数组
  getHooks(keyWithDot: string) {
    const [key, ...memberKeys] = keyWithDot.split('.');
    let hooks = this.hooks[key] || [];
    // ...
    return hooks;
  }
	
	// 根据key与type执行相应的钩子函数数组
  applyPlugins({
    key,
    type,
    initialValue,
    args,
    async,
  }: {
    key: string;
    type: ApplyPluginsType;
    initialValue?: any;
    args?: object;
    async?: boolean;
  }) {
    const hooks = this.getHooks(key) || [];

    if (args) {
      assert(
        typeof args === 'object',
        `applyPlugins failed, args must be plain object.`,
      );
    }

    switch (type) {
      case ApplyPluginsType.modify:
        if (async) {
          return hooks.reduce(
            async (memo: any, hook: Function | Promise<any> | object) => {
              assert(
                typeof hook === 'function' ||
                  typeof hook === 'object' ||
                  isPromiseLike(hook),
                `applyPlugins failed, all hooks for key ${key} must be function, plain object or Promise.`,
              );
              if (isPromiseLike(memo)) {
                memo = await memo;
              }
              if (typeof hook === 'function') {
                const ret = hook(memo, args);
                if (isPromiseLike(ret)) {
                  return await ret;
                } else {
                  return ret;
                }
              } else {
                if (isPromiseLike(hook)) {
                  hook = await hook;
                }
                return { ...memo, ...hook };
              }
            },
            isPromiseLike(initialValue)
              ? initialValue
              : Promise.resolve(initialValue),
          );
        } else {
          return hooks.reduce((memo: any, hook: Function | object) => {
            assert(
              typeof hook === 'function' || typeof hook === 'object',
              `applyPlugins failed, all hooks for key ${key} must be function or plain object.`,
            );
            if (typeof hook === 'function') {
              return hook(memo, args);
            } else {
              // TODO: deepmerge?
              return { ...memo, ...hook };
            }
          }, initialValue);
        }

      case ApplyPluginsType.event:
        return hooks.forEach((hook: Function) => {
          assert(
            typeof hook === 'function',
            `applyPlugins failed, all hooks for key ${key} must be function.`,
          );
          hook(args);
        });

      case ApplyPluginsType.compose:
        return () => {
          return _compose({
            fns: hooks.concat(initialValue),
            args,
          })();
        };
    }
  }
}
```



### 注册

前面已经说到plugin的收集，下面以`plugin-antd`看看plugin是如何注册的：

```javascript
export default (api: IApi) => {  
	api.describe({
    config: {
      schema(joi) {
        return joi.object({
          dark: joi.boolean(),
          compact: joi.boolean(),
          config: joi.object(),
        });
      },
    },
  });

  // ...

  const opts: IAntdOpts = api.userConfig.antd || {};

  if (opts?.dark || opts?.compact) {
    // support dark mode, user use antd 4 by default
    const { getThemeVariables } = require('antd/dist/theme');
    api.modifyDefaultConfig(config => {
      config.theme = {
        ...getThemeVariables(opts),
        ...config.theme,
      };
      return config;
    });
  }

  // ...

  if (opts?.config) {
    api.onGenerateFiles({
      fn() {
        // runtime.tsx
        const runtimeTpl = readFileSync(
          join(__dirname, 'runtime.tpl'),
          'utf-8',
        );
        api.writeTmpFile({
          path: 'plugin-antd/runtime.tsx',
          content: Mustache.render(runtimeTpl, {
            config: JSON.stringify(opts?.config),
          }),
        });
      },
    });
    // Runtime Plugin
    api.addRuntimePlugin(() => [
      join(api.paths.absTmpPath!, 'plugin-antd/runtime.tsx'),
    ]);
    api.addRuntimePluginKey(() => ['antd']);
  }
};
```

首先是动态生成入口文件 `plugin-antd/runtime.tsx` , 然后通过钩子 `addRuntimePlugin` 添加钩子函数，钩子函数返回插件入口文件的路径。`addRuntimePluginKey` 钩子则是用来显式声明入口文件暴露出来的方法，如果缺少则会报错。默认有`modifyClientRenderOpts` ，`patchRoutes`， `rootContainer`， `render`，  `onRouteChange` 等方法，可以不声明。



### 配置校验

有的插件是可以跟umi配置文件配合使用的，在校验配置文件时候，会校验对应的插件配置项。

插件通过describe方法定义配置：

```javascript
api.describe({
  config: {
    schema(joi) {
      return joi.object({
        dark: joi.boolean(),
        compact: joi.boolean(),
        config: joi.object(),
      });
    },
  },
});
```

`describe` 方法没有被代理，而是直接调用了api实例上的方法，packages/core/src/Service/PluginAPI.ts` ：

```javascript
describe({
  id,
  key,
  config,
  enableBy,
}: {
         id?: string;
         key?: string;
         config?: IPluginConfig;
         enableBy?: EnableBy | (() => boolean);
} = {}) {
  const { plugins } = this.service;
  // this.id and this.key is generated automatically
  // so we need to diff first
  if (id && this.id !== id) {
    if (plugins[id]) {
      const name = plugins[id].isPreset ? 'preset' : 'plugin';
      throw new Error(
        `api.describe() failed, ${name} ${id} is already registered by ${plugins[id].path}.`,
      );
    }
    plugins[id] = plugins[this.id];
    plugins[id].id = id;
    delete plugins[this.id];
    this.id = id;
  }
  if (key && this.key !== key) {
    this.key = key;
    plugins[this.id].key = key;
  }

  if (config) {
    plugins[this.id].config = config;
  }

  plugins[this.id].enableBy = enableBy || EnableBy.register;
}
```

插件相关信息都会通过 `registerPlugin` 注册到 `service.plugins` 对象上。

在启动加载配置阶段进行校验，`packages/core/src/Config/Config.ts`

```javascript
getConfig({ defaultConfig }: { defaultConfig: object }) {
    const userConfig = this.getUserConfig();
    // 用于提示用户哪些 key 是未定义的
    // TODO: 考虑不排除 false 的 key
    const userConfigKeys = Object.keys(userConfig).filter((key) => {
      return userConfig[key] !== false;
    });

    // get config
    const pluginIds = Object.keys(this.service.plugins);
    pluginIds.forEach((pluginId) => {
      const { key, config = {} } = this.service.plugins[pluginId];
      // recognize as key if have schema config
      if (!config.schema) return;

      const value = getUserConfigWithKey({ key, userConfig });
      // 不校验 false 的值，此时已禁用插件
      if (value === false) return;

      // do validate
      const schema = config.schema(joi);
      assert(
        joi.isSchema(schema),
        `schema return from plugin ${pluginId} is not valid schema.`,
      );
      const { error } = schema.validate(value);
    })
  // ...
}
```



### 接口导出

umi 生成一个 `src/.umi/core/umiExports.ts` 临时文件，用于导出各种方法与接口。`packages/preset-built-in/src/plugins/generateFiles/core/umiExports.ts` :

```javascript
export default function (api: IApi) {
  api.onGenerateFiles(async () => {
    const umiExports = await api.applyPlugins({
      key: 'addUmiExports',
      type: api.ApplyPluginsType.add,
      initialValue: [],
    });

    let umiExportsHook = {}; // repeated definition
    api.writeTmpFile({
      path: 'core/umiExports.ts',
      content:
        umiExports
          .map((item: IUmiExport) => {
            return generateExports({
              item,
              umiExportsHook,
            });
          })
          .join('\n') + `\n`,
    });
  });
}
```

提供了 `addUmiExports` 钩子，用于收集导出信息，下面以plugin-dva插件为例看其实现：

```javascript
 api.addUmiExports(() =>
    hasModels
      ? [
          {
            exportAll: true,
            source: '../plugin-dva/exports',
          },
          {
            exportAll: true,
            source: '../plugin-dva/connect',
          },
        ]
      : [],
  );
```



### 使用案例

现在通过梳理各种临时文件的生成来了解umi是如何通过插件的形式嵌入各种功能的。

```javascript
// misc
require.resolve('./plugins/routes'),

// generate files
require.resolve('./plugins/generateFiles/core/history'),
require.resolve('./plugins/generateFiles/core/plugin'),
require.resolve('./plugins/generateFiles/core/polyfill'),
require.resolve('./plugins/generateFiles/core/routes'),
require.resolve('./plugins/generateFiles/core/umiExports'),
require.resolve('./plugins/generateFiles/core/configTypes'),
require.resolve('./plugins/generateFiles/umi'),
```

前面提起过这些内置插件，在构建的时候会触发 `onGenerateFiles` 钩子，从而逐一生成 `history`、`plugin`、`polyfill`、`routes`、`umiExports`、`umi` 临时文件。

应用的入口是 `umi` 文件：

```javascript
const getClientRender = (args: { hot?: boolean; routes?: any[] } = {}) => plugin.applyPlugins({
  key: 'render',
  type: ApplyPluginsType.compose,
  initialValue: () => {
    const opts = plugin.applyPlugins({
      key: 'modifyClientRenderOpts',
      type: ApplyPluginsType.modify,
      initialValue: {
        routes: args.routes || getRoutes(),
        plugin,
        history: createHistory(args.hot),
        isServer: process.env.__IS_SERVER,
        dynamicImport: true,
        rootElement: 'root',
      },
    });
    return renderClient(opts);
  },
  args,
});

const clientRender = getClientRender();
export default clientRender();
```

此处的 `plugin` 为运行时的 `packages/runtime/src/Plugin/Plugin.ts` 实例。



#### renderClient

`renderClient` 是可以动态切换的，默认的是 `packages/renderer-react/src/renderClient/renderClient.tsx` :

```javascript
export default function renderClient(opts: IOpts) {
  const rootContainer = opts.plugin.applyPlugins({
    type: ApplyPluginsType.modify,
    key: 'rootContainer',
    initialValue: (
      <RouterComponent
        history={opts.history}
        routes={opts.routes}
        plugin={opts.plugin}
        ssrProps={opts.ssrProps}
        defaultTitle={opts.defaultTitle}
      />
    ),
    args: {
      history: opts.history,
      routes: opts.routes,
      plugin: opts.plugin,
    },
  });

  if (opts.rootElement) {
    const rootElement =
      typeof opts.rootElement === 'string'
        ? document.getElementById(opts.rootElement)
        : opts.rootElement;
    const callback = opts.callback || (() => {});
    
    // ...
      ReactDOM.render(rootContainer, rootElement, callback);
    
  } else {
    return rootContainer;
  }
}
```

此时触发的是 `rootContainer` 钩子，逐一调用各个插件暴露出的 `rootContainer` 方法，返回最终的 `rootContainer` 组件。

像 `plugin-antd`、`plugin-dva` 等插件都是通过此钩子包裹 `rootContainer` 组件，从而达到功能注入的目的。



#### render

Umi 默认把 `app.ts` 文件作为一个内置的插件，所以可以通过在此文件暴露一些 `validKeys` 中的方法，例如 `render`，`patchRoutes` 等默认的，有 `plugin-request` 插件时，则可以暴露 `request` 对象来修改默认配置。

`getClientRender` 方法触发 `render` 钩子，`() => rootContainer` 组件作为`initialValue`传入，通过`compose` 方式组织`render`函数数组，`initialValue` 最后执行，渲染 `rootContainer` 组件。

```javascript
case ApplyPluginsType.compose:
        return () => {
          return _compose({
            fns: hooks.concat(initialValue),
            args,
          })();
        };

function _compose({ fns, args }: { fns: (Function | any)[]; args?: object }) {
  if (fns.length === 1) {
    return fns[0];
  }
  const last = fns.pop();
  return fns.reduce((a, b) => () => b(a, args), last);
}
```



#### 路由生成流程

在umi的路由配置中，我们可以使用layout component，wrappers等配置，这又是如何实现的呢，下面就通过看源码走一遍全流程。

##### getRoutes

不难猜出 `packages/preset-built-in/src/plugins/generateFiles/core/routes.ts` 文件是入口文件：

```typescript
api.onGenerateFiles(async (args) => {
    const routesTpl = readFileSync(join(__dirname, 'routes.tpl'), 'utf-8');
    const routes = await api.getRoutes();
    api.writeTmpFile({
      path: 'core/routes.ts',
      content: Mustache.render(routesTpl, {
        routes: new Route().getJSON({ routes, config: api.config, cwd }),
        runtimePath,
        config: api.config,
        loadingComponent: api.config.dynamicImport?.loading,
      }),
    });
  });
```

既然调用了 `api.getRoutes();` ，通过前面 `registerMethod` 的原理自然知道肯定有地方在service实例上注册了 `getRoutes` 方法，不出所料，在 `packages/preset-built-in/src/plugins/routes.ts` 找到：

```typescript
api.registerMethod({
    name: 'getRoutes',
    async fn() {
      const route = new Route({
        async onPatchRoutesBefore(args: object) {
          await api.applyPlugins({
            key: 'onPatchRoutesBefore',
            type: api.ApplyPluginsType.event,
            args,
          });
        },
        async onPatchRoutes(args: object) {
          await api.applyPlugins({
            key: 'onPatchRoutes',
            type: api.ApplyPluginsType.event,
            args,
          });
        },
        async onPatchRouteBefore(args: object) {
          await api.applyPlugins({
            key: 'onPatchRouteBefore',
            type: api.ApplyPluginsType.event,
            args,
          });
        },
        async onPatchRoute(args: object) {
          await api.applyPlugins({
            key: 'onPatchRoute',
            type: api.ApplyPluginsType.event,
            args,
          });
        },
      });
      return await api.applyPlugins({
        key: 'modifyRoutes',
        type: api.ApplyPluginsType.modify,
        initialValue: await route.getRoutes({
          config: api.config,
          root: api.paths.absPagesPath!,
        }),
      });
    },
  });
```

从上可知，该方法提供了 `onPatchRoutesBefore`, `onPatchRoutes` 等等可以修改路由的钩子，也就是给其他插件提供了修改路由的接口。



##### getJSON

拿到 `routes` 数据后，umi 还给我们提供了按需加载的处理。下面看 `new Route().getJSON({ routes, config: api.config, cwd })` 实现，代码位置 `packages/core/src/Route/routesToJSON.ts` ：

```typescript
function patchRoute(route: IRoute) {
  	// route.component是字符串时执行，
    if (route.component && !isFunctionComponent(route.component)) {
      // transform route component into webpack chunkName
      const webpackChunkName = routeToChunkName({
        route,
        cwd,
      });
      // 解决 SSR 开启动态加载后，页面闪烁问题
      if (config?.ssr && config?.dynamicImport) {
        route._chunkName = webpackChunkName;
      }
      route.component = [
        route.component,
        webpackChunkName,
        route.path || EMPTY_PATH,
      ].join(SEPARATOR);
    }
    if (route.routes) {
      patchRoutes(route.routes);
    }
  }
```

在前面规范名称的基础上，通过正则替换生成按需加载的代码：

```typescript
JSON.stringify(clonedRoutes, replacer, 2)
    .replace(/\"component\": (\"(.+?)\")/g, (global, m1, m2) => {
      return `"component": ${m2.replace(/\^/g, '"')}`;
    })
    .replace(/\"wrappers\": (\"(.+?)\")/g, (global, m1, m2) => {
      return `"wrappers": ${m2.replace(/\^/g, '"')}`;
    })
    .replace(/\\r\\n/g, '\r\n')
    .replace(/\\n/g, '\r\n');
```

replacer就是替换的实现细节：

```typescript
function replacer(key: string, value: any) {
    switch (key) {
      case 'component':
        if (isFunctionComponent(value)) return value;
        if (config.dynamicImport) {
          const [component, webpackChunkName] = value.split(SEPARATOR);
          let loading = '';
          if (config.dynamicImport.loading) {
            loading = `, loading: LoadingComponent`;
          }
          return `dynamic({ loader: () => import(/* webpackChunkName: '${webpackChunkName}' */'${component}')${loading}})`;
        } else {
          return `require('${value}').default`;
        }
      case 'wrappers':
        // ...
      default:
        return value;
    }
  }
```

这里我们可以看到配置文件loading组件的身影 `config.dynamicImport.loading` 。到这里，我们看到的就是 `.umi/core/routes.ts` 文件中路由文件最终的样子。但是我们还是没看到 layout component 跟 wrapper组件的身影，这就到了runtime阶段了，下面继续看。



##### renderRoutes

从`.umi/umi.ts` 入口文件中我们看到了路由入口：

```typescript
const getClientRender = (args: { hot?: boolean; routes?: any[] } = {}) => plugin.applyPlugins({
  key: 'render',
  type: ApplyPluginsType.compose,
  initialValue: () => {
    const opts = plugin.applyPlugins({
      key: 'modifyClientRenderOpts',
      type: ApplyPluginsType.modify,
      initialValue: {
        routes: args.routes || getRoutes(),
        plugin,
        history: createHistory(args.hot),
        isServer: process.env.__IS_SERVER,
        dynamicImport: true,
        rootElement: 'root',
      },
    });
    return renderClient(opts);
  },
  args,
});
```

不难猜出路由的实现在renderClient方法中：

```tsx
function RouterComponent(props: IRouterComponentProps) {
  const { history, ...renderRoutesProps } = props;

  useEffect(() => {
    // ...
    function routeChangeHandler(location: any, action?: string) {
      // ...
      props.plugin.applyPlugins({
        key: 'onRouteChange',
        type: ApplyPluginsType.event,
        args: {
          routes: props.routes,
          matchedRoutes,
          location,
          action,
        },
      });
    }
    routeChangeHandler(history.location, 'POP');
    return history.listen(routeChangeHandler);
  }, [history]);

  return <Router history={history}>{renderRoutes(renderRoutesProps)}</Router>;
}
```

在这里注册了路由变化事件，我们可以通过 `onRouteChange` 钩子监听。

下面看`renderRoutes` 具体实现，`packages/renderer-react/src/renderRoutes/renderRoutes.tsx`

```tsx
function render({
  route,
  opts,
  props,
}: {
  route: IRoute;
  opts: IOpts;
  props: object;
}) {
  // 递归渲染子路由
  const routes = renderRoutes({
    ...opts,
    routes: route.routes || [],
    rootRoutes: opts.rootRoutes,
  });
  let { component: Component, wrappers } = route;
  if (Component) {
    const defaultPageInitialProps = opts.isServer
      ? {}
      : (window as any).g_initialProps;
    const newProps = {
      ...props,
      ...opts.extraProps,
      ...(opts.pageInitialProps || defaultPageInitialProps),
      route,
      routes: opts.rootRoutes,
    };
    // @ts-ignore
    let ret = <Component {...newProps}>{routes}</Component>;
    // route.wrappers
    if (wrappers) {
      let len = wrappers.length - 1;
      while (len >= 0) {
        ret = React.createElement(wrappers[len], newProps, ret);
        len -= 1;
      }
    }
    return ret;
  } else {
    return routes;
  }
}
```

通过renderRoutes递归渲染子路由；当 component （即layout component）有值，则包裹相应的子路由；当

wrappers有值，则从右到左层层包裹component。

到这里，整个路由的渲染流程就结束。