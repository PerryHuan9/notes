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

而且`vite`在打包之后，还会对这些依赖包的请求设置`cache-control: max-age=31536000,immutable;`,即设置了强缓存，之后针对依赖的请求将不会到达服务器。如果要进行依赖调试，可以在启动服务器时使用 --force 标志，它会重新打包依赖。下面看看源码中是怎样实现这些的？

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

`server/index.ts`文件中：
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

`optimizer/index.ts`:

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




## `Rollup`打包