<div align="center"> 
<img src="https://www.vitejs.net/logo.svg" width = 500 height = 300 />
</div>

## `Vite` 介绍

`Vite`是`Vue`作者尤雨溪又一个备受关注的开源项目，它是一个前端开发工具，可类比`Webpack`,号称是下一代前端开发和构建工具。它主要解决了两个问题：
1. `bundle-base`(如webpack)打包器启动速度慢，而且其启动时间是跟应用规模成正比的。
2. 在更新时，打包器即便使用了`HMR`，但是其热更新的时间仍是会随着应用规模的增长而直线下降。

它解决的是开发的时候的效率问题，对于生产环境则是交给了`Rollup`。除此之外，它还有以下优点：

* 对`ts`、`jsx`、`css`等开箱即用，无需配置。
* 对于库开发者也是可以通过简单的配置即可打包输出多种格式的包。
* 开发和生产共享了`rollup`的插件接口，大部门的`rollup`插件可以在`vite`上使用。
* 类型化配置，配置文件可以使用`ts`，具有配置类型提示。

## Get Started
### 创建项目模板
`Vite`提供了一个快捷创建各类型项目模板的包`@vitejs/create-app`的包，对外暴露了可执行的`bin`文件：

```
yarn create @vitejs/app my-vue-app --template vue
```
顺便提一下， `yarn create`是`yarn`提供的一个聚合两个命令的语法糖，上面命令等价：
```
yarn global add @vitejs/create-app
@vite/create-app my-vue-app --template vue
```
这里是创建`vue3`模板。除此之外，`vite`还支持
* `vanilla`
* `vue-ts`
* `react`
* `react-ts`
* `preact`
* `preact-ts`
* `lit-element`
* `lit-element-ts`

### 启动vite

开发环境，启动服务器

```
vite
```

生产环境，打包应用或包

```
vite build
```

这里使用的都是vite的核心包 `vite`，所有的优化都集中在这个包中，另外`vite`还提供了`vite.config.ts`的配置文件，允许针对整个构建过程做出一些配置。具体请移步[vite官网](https://www.vitejs.net/guide/)。

上面的`vite`和`vite build`是做了什么呢？
![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/30864cb0239148cf9729cb324ae818c6~tplv-k3u1fbpfcp-watermark.image)

`vite`命令会执行`vite.js`这个脚本，而在这个脚本中会执行`start`函数，最终引入了`node`目录下的`cli`脚本，接着看看`cli`脚本中干了什么？

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a92b9ef738b048c4911fc443f9df0380~tplv-k3u1fbpfcp-watermark.image)

`cli`脚本中主要监听了`vite`和`vite build`两个命令，这便是`vite`开发和生产的入口了。

## `Vite` 原理
前文提到了`vite`主要解决了开发效率问题，那么`vite`是怎样解决这两个问题呢？
`vite`主要通过 `esbuild`预构建依赖和让浏览器接管部分打包程序两种手段，下面从上文中`vite`明命令开始，按图索骥。

### `esbuild`预构建依赖
`vite`将代码分为源码和依赖两部分并分别处理，所谓依赖便是应用使用的第三方包，一般存在于`node_modules`目录中，一个较大项目的依赖及其依赖的依赖，加起来可能达到上千个包，这些代码可能比我们源码数量要大，这些依赖通常是不会改变的（除非你要进行本地依赖调试），所以无论是`webpack`或者`vite`在启动时都会编译后将其缓存下来。区别的是，`vite`会使用`esbuild`进行依赖编译和转换（commonjs包转为esm），而`webpack`则是使用`acorn`或者`tsc`进行编译，而`esbuild`是使用`Go`语言写的，其速度比使用`js`编写的`acorn`速度要快得多。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f6a9cc0325be4cd2ba8639150243c4a3~tplv-k3u1fbpfcp-watermark.image)

`esbuild`官方做了一个测试，打包生产环境的`three.js`包十次，上图是各大工具的打包时长。`esbuild`在打包速度上比现在前端打包工具快10-100倍。

而且`vite`在打包之后，还会对这些依赖包的请求设置`cache-control: max-age=31536000,immutable;`,即设置了强缓存，之后针对依赖的请求将不会到达服务器。如果要进行依赖调试，可以在启动服务器时使用 `--force` 标志，它会重新打包依赖。下面看看源码中是怎样实现这些的？

`node/cli.ts`
```ts
// dev
cli
  .command('[root]') // default command
  .alias('serve')
  .option('--host [host]', `[string] specify hostname`)
  .option('--port <port>', `[number] specify port`)
  .option('--https', `[boolean] use TLS + HTTP/2`)
  .option('--open [path]', `[boolean | string] open browser on startup`)
  .option('--cors', `[boolean] enable CORS`)
  .option('--strictPort', `[boolean] exit if specified port is already in use`)
  .option('-m, --mode <mode>', `[string] set env mode`)
  .option(
    '--force',
    `[boolean] force the optimizer to ignore the cache and re-bundle`
  )
  .action(async (root: string, options: ServerOptions & GlobalCLIOptions) => {
//   核心代码在server脚本中
    const { createServer } = await import('./server')
    try {
    //   创建服务器
      const server = await createServer({
        root,
        base: options.base,
        mode: options.mode,
        configFile: options.config,
        logLevel: options.logLevel,
        clearScreen: options.clearScreen,
        server: cleanOptions(options) as ServerOptions
      })
    //   监听端口
      await server.listen()
    } catch (e) {
      createLogger(options.logLevel).error(
        chalk.red(`error when starting dev server:\n${e.stack}`)
      )
      process.exit(1)
    }
  })
```

`node/server/index.ts`文件中：
```ts
import { DepOptimizationMetadata, optimizeDeps } from '../optimizer'

export async function createServer(
  inlineConfig: InlineConfig = {}
): Promise<ViteDevServer> {
   // 省略无关代码

  // 优化
  const runOptimize = async () => {
    //   cacheDir为缓存目录，一般为node_modules/.vite 目录
    if (config.cacheDir) {
      server._isRunningOptimizer = true
      try {
        // 依赖预构建
        server._optimizeDepsMetadata = await optimizeDeps(config)
      } finally {
        server._isRunningOptimizer = false
      }
      server._registerMissingImport = createMissingImporterRegisterFn(server)
    }
  }

  if (!middlewareMode && httpServer) {
    // overwrite listen to run optimizer before server start
    const listen = httpServer.listen.bind(httpServer)
    httpServer.listen = (async (port: number, ...args: any[]) => {
      try {
        // 执行所有插件的 buildStart方法
        await container.buildStart({})
        // 依赖预构建优化
        await runOptimize()
      } catch (e) {
        httpServer.emit('error', e)
        return
      }
      return listen(port, ...args)
    }) as any
  }
}
```

`node/optimizer/index.ts`:

```ts
import { build, BuildOptions as EsbuildBuildOptions } from 'esbuild'

export async function optimizeDeps(
  config: ResolvedConfig,
  force = config.server.force,
  asCommand = false,
  newDeps?: Record<string, string> // missing imports encountered after server has started
): Promise<DepOptimizationMetadata | null> {
  
 // 缓存目录存在一个保存预构建信息的配置 
  const dataPath = path.join(cacheDir, '_metadata.json')
  const mainHash = getDepHash(root, config)
  const data: DepOptimizationMetadata = {
    hash: mainHash,
    browserHash: mainHash,
    optimized: {}
  }
  // 强制重新预构建
  if (!force) {
    let prevData
    try {
      prevData = JSON.parse(fs.readFileSync(dataPath, 'utf-8'))
    } catch (e) {}
    // hash is consistent, no need to re-bundle
    if (prevData && prevData.hash === data.hash) {
      log('Hash is consistent. Skipping. Use --force to override.')
      return prevData
    }
  }

  // 依赖列表
  let deps: Record<string, string>, missing: Record<string, string>
  if (!newDeps) {
    //  依赖列表是通过扫描import语句进行收集的，所以只有在代码中import进来的包才会进行预构建
    ;({ deps, missing } = await scanImports(config))
  } else {
    deps = newDeps
    missing = {}
  }

  const define: Record<string, string> = {
    'process.env.NODE_ENV': JSON.stringify(config.mode)
  }
  // 在vite.config.ts配置的define变量也会注入到包里面
  for (const key in config.define) {
    const value = config.define[key]
    define[key] = typeof value === 'string' ? value : JSON.stringify(value)
  }

  const start = Date.now()

  const { plugins = [], ...esbuildOptions } =
    config.optimizeDeps?.esbuildOptions ?? {}
  // 使用esbuild对所有的包进行转换
  const result = await build({
    entryPoints: Object.keys(flatIdDeps),
    bundle: true,
    format: 'esm',
    external: config.optimizeDeps?.exclude,
    logLevel: 'error',
    splitting: true,
    sourcemap: true,
    outdir: cacheDir,
    treeShaking: 'ignore-annotations',
    metafile: true,
    define,
    plugins: [
      ...plugins,
      esbuildDepPlugin(flatIdDeps, flatIdToExports, config)
    ],
    ...esbuildOptions
  })

  const meta = result.metafile!

  // the paths in `meta.outputs` are relative to `process.cwd()`
  const cacheDirOutputPath = path.relative(process.cwd(), cacheDir)

  for (const id in deps) {
    const entry = deps[id]
    data.optimized[id] = {
      file: normalizePath(path.resolve(cacheDir, flattenId(id) + '.js')),
      src: entry,
      needsInterop: needsInterop(
        id,
        idToExports[id],
        meta.outputs,
        cacheDirOutputPath
      )
    }
  }
 // 将打包好的代码写到 node_modules/.vite目录下
  writeFile(dataPath, JSON.stringify(data, null, 2))
  return data
}
```
`_metadata.json`文件

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/78af4e35ab8f48eea337d2ab327f4a72~tplv-k3u1fbpfcp-watermark.image)

`esbuild`打包输出的代码格式是esm

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6e9760c9961840b2ba529ffb810b477e~tplv-k3u1fbpfcp-watermark.image)


### 让浏览器接管部分打包
Bundle based dev server在启动时时会把全部的源码都编译，当一个项目有很多路由页面时，它也会按照每一个路由入口查找编译所有模块，但实际上我们是否需要在启动的时候就打包所有的模块源码？

![Bundle based](https://www.vitejs.net/assets/bundler.37740380.png)

在启动的时候，`vite`并不会打包源码，而是在浏览器请求路由时才会进行打包，而且也仅仅打包当前路由的源码，这相当于让浏览器掌握了打包的控制权。从而将Bundle based dev server一次性打包全部源码的操作改为了多次，启动速度无疑会快非常多，并且在访问时转换的速度也不会慢下来，因为每次转换的源码只有当前路由下的；并且源码模块还会设置协商缓存，当模块没有改变时，浏览器的请求会返回`304 Not Modified`。

![Bundle based](https://www.vitejs.net/assets/esm.3070012d.png)

这一切的前提是基于原生的`ES Module`,浏览器在处理`ES6 Module`时，该模块中所有`import`进来的`module`都会通过`http`请求抓取，并且其请求是精确有序的。`ESM`还对`vite`的`HMR`起了非常大的作用。当源码文件修改时，因为源码采取的是`ESM`,`vite`只需要精确地使当前修改模块与其最近的`HMR`边界失效，大多数情况只需要替换当前修改的模块，这让`vite`的`HMR`直接与当前应用的大小没有关系，无论应用多大时，都能保持快速的更新速度。

`main.ts`
```ts
import { createApp } from 'vue'
import App from './App.vue'
import {get} from 'lodash'


export const app = createApp(App).mount('#app')

console.log(get(app, 'name'))
```

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8c5872a925e44dd0a3aeb1608739c375~tplv-k3u1fbpfcp-watermark.image)

从上图可以看出浏览器在处理`ESM`执行的请求 以及 `vite`对源码和依赖分别执行的强缓存和协商缓存。下面从`vite`源码分析是怎样实现这个功能的。

入口还是在`node/server/index.ts`的`createServer`函数：

```ts
import connect from 'connect'
import chokidar from 'chokidar'
import { transformMiddleware } from './middlewares/transform'

export async function createServer(
  inlineConfig: InlineConfig = {}
): Promise<ViteDevServer> {
    // 省略无关代码
    
  // 创建connect中间件，用于有序地服务请求，
  // 具体可查看 https://github.com/senchalabs/connect
  const middlewares = connect() as Connect.Server
  const httpServer = middlewareMode
    ? null
    : await resolveHttpServer(serverConfig, middlewares, httpsOptions)
  const ws = createWebSocketServer(httpServer, config, httpsOptions)

  const { ignored = [], ...watchOptions } = serverConfig.watch || {}
  // 用于监测文件修改或新增文件
  const watcher = chokidar.watch(path.resolve(root), {
    ignored: ['**/node_modules/**', '**/.git/**', ...ignored],
    ignoreInitial: true,
    ignorePermissionErrors: true,
    disableGlobbing: true,
    ...watchOptions
  }) as FSWatcher

  const plugins = config.plugins
 // 插件容器，vite的所有插件，包括vite.config.js都会被收集到container中
  const container = await createPluginContainer(config, watcher)
  // 建立模块图，请求模块是通过moduleGraph查找并转换模块的
  const moduleGraph = new ModuleGraph(container)

  const server: ViteDevServer = {
    config: config,
    middlewares,
    httpServer,
    watcher,
    pluginContainer: container,
    ws,
    moduleGraph,
    transformWithEsbuild,
    transformRequest(url, options) {
      return transformRequest(url, server, options)
    },
    listen(port?: number, isRestart?: boolean) {
      return startServer(server, port, isRestart)
    },
  }
  // 用于解析入口 index.html文件
  server.transformIndexHtml = createDevHtmlTransformFn(server)

  watcher.on('change', async (file) => {
    file = normalizePath(file)
    // 源码修改可需要通知moduleGraph更新缓存中的模块
    moduleGraph.onFileChange(file)
    if (serverConfig.hmr !== false) {
      try {
        await handleHMRUpdate(file, server)
      } catch (err) {
        ws.send({
          type: 'error',
          err: prepareError(err)
        })
      }
    }
  })
  // 新增文件
  watcher.on('add', (file) => {
    handleFileAddUnlink(normalizePath(file), server)
  })
  // 删除文件
  watcher.on('unlink', (file) => {
    handleFileAddUnlink(normalizePath(file), server, true)
  })
  
  // 计算请求的时长
  if (process.env.DEBUG) {
    middlewares.use(timeMiddleware(root))
  }

  // 处理CORS
  const { cors } = serverConfig
  if (cors !== false) {
    middlewares.use(corsMiddleware(typeof cors === 'boolean' ? {} : cors))
  }

  // 代理接口
  const { proxy } = serverConfig
  if (proxy) {
    middlewares.use(proxyMiddleware(httpServer, config))
  }

  // 当base不等于'/'
  if (config.base !== '/') {
    middlewares.use(baseMiddleware(server))
  }

  
  // 处理/public目录下文件的请求，核心
  if (config.publicDir) {
    middlewares.use(servePublicMiddleware(config.publicDir))
  }

  // 处理当前路由下的代码转换
  middlewares.use(transformMiddleware(server))

  // 其它静态文件处理
  middlewares.use(serveRawFsMiddleware(config))
  middlewares.use(serveStaticMiddleware(root, config))
  return server
}
```
源码只要是通过`transformMiddleware`中间件，下面看看在这个中间件中是如何转换源码的？

`/node/server/middlewares/transform.ts`
```ts
import { transformRequest } from '../transformRequest'

export function transformMiddleware(
  server: ViteDevServer
): Connect.NextHandleFunction {
  const {
    config: { root, logger, cacheDir },
    moduleGraph
  } = server
  return async function viteTransformMiddleware(req, res, next) {
   
    let url
    try {
      url = removeTimestampQuery(req.url!).replace(NULL_BYTE_PLACEHOLDER, '\0')
    } catch (err) {
    }

    const withoutQuery = cleanUrl(url)

    try {
      if (
        isJSRequest(url) ||
        isImportRequest(url) ||
        isCSSRequest(url) ||
        isHTMLProxy(url)
      ) {
        
        const ifNoneMatch = req.headers['if-none-match']
        // 当当前模块没有改变时，直接返回304
        if (
          ifNoneMatch &&
          (await moduleGraph.getModuleByUrl(url))?.transformResult?.etag ===
            ifNoneMatch
        ) {
          isDebug && debugCache(`[304] ${prettifyUrl(url, root)}`)
          res.statusCode = 304
          return res.end()
        }

        // 转换当前路由下的所有源码，并拿到转换结果
        const result = await transformRequest(url, server, {
          html: req.headers.accept?.includes('text/html')
        })
        if (result) {
         // 将转换好的源码返回
          return send(
            req,
            res,
            result.code,
            type,
            result.etag,
            // allow browser to cache npm deps!
            isDep ? 'max-age=31536000,immutable' : 'no-cache',
            result.map
          )
        }
      }
    } catch (e) {
      return next(e)
    }
    next()
  }
}
```
上文中时通过`transformRequest`函数转换源码，下面看看这个函数做了什么？

`/node/server/transformRequest.ts`
```ts
export async function transformRequest(
  url: string,
  { config, pluginContainer, moduleGraph, watcher }: ViteDevServer,
  options: TransformOptions = {}
): Promise<TransformResult | null> {
  url = removeTimestampQuery(url)
  const { root, logger } = config
  const prettyUrl = isDebug ? prettifyUrl(url, root) : ''
  const ssr = !!options.ssr

  //通过moduleGraph检查是否已经存在该模块，存在则直接返回
  const module = await moduleGraph.getModuleByUrl(url)
  const cached =
    module && (ssr ? module.ssrTransformResult : module.transformResult)
  if (cached) {
    isDebug && debugCache(`[memory] ${prettyUrl}`)
    return cached
  }

  // 准备一个mod，以将转换完成的模块保存到moduleGraph中
  const mod = await moduleGraph.ensureEntryFromUrl(url)

  // 通过所有的vite插件转换源码
  const transformResult = await pluginContainer.transform(code, id, map, ssr)
  code = transformResult.code!
  map = transformResult.map
  // 将源码保存到mod中并返回
  return (mod.transformResult = {
    code,
    map,
    etag: getEtag(code, { weak: true })
  } as TransformResult)
}
```
通过以上源码，可以发现整个流程: 
1. 总的入口是在`createServer`函数中，在这个函数中，`vite`创建一个`http`或`https`的服务器。
2. 使用[connect](https://github.com/senchalabs/connect)中间件来服务请求，有处理`CORS`,`Proxy`等的中间件，其中`transformMiddleware`中间件用于转换代码。
3. 在`transformMiddleware`中间件中，会先查询`moduleGraph`中是否存在当前模块，存在则返回，不存在则会调用`transformRequest`转换代码。
4. 最终在`transformResult`中使用`vite plugin`完成了对源码的转换，转换后会缓存到`moduleGraph`中，方便下次直接使用。

值得一提的是，`vite`内部内置了需要`plugin`,这些插件会被默认添加以支持某些开箱即用的功能，另外有一个`esbuildPlgin`的插件用于转换代码，所以即便是源码也还是会先经过`esbuild`的编译，这样可以获得速度上的提升。


## `Rollup`打包