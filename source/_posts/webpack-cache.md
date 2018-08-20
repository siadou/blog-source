---
title: webpack持久化缓存相关笔记
date: 2018-08-20 10:33:53
tags: [前端技术, webpack]
---

在JS项目中，我们希望能够在代码分割后将一些不经常变动的库打包在一起，设置一个长时间的缓存，减少用户加载代码的时间。

在使用 webpack构建的典型应用中，有三种主要的代码类型：
* 业务直接相关的源码。
* 依赖的任何第三方的 library 或 "vendor" 代码。此部分代码不经常变动，我们希望能够将这部分打包在一起。
* webpack 管理所有模块的交互的 runtime 和 manifest代码。runtime和manifest代码在每次构建时都不一样，不应该注入到缓存相关的文件块中。

代码分割后产生以下几种chunk:

* your entry chunks, which have children created by the split points in your code
* 入口chunk，在分割代码后会产生子chunk
* the chunks created by your split points (i.e. the children of your entry chunks), or
* 入口chunk产生的子chunk（一般是异步加载模块生成）
* the commons chunk(s) you merge into with the CommonsChunkPlugin.
* CommonsChunkPlugin 产生合并的公共chunk

由上我们可知，为了充分利用缓存提高用户加载速度，我们可以将所有的调用的第三方库进行打包，因为此部分代码不经常产生变动。
既包含入口模块的公共部分，又包含异步引用的代码公共部分。

> 注：

> runtime代码：在浏览器运行时，webpack 用来连接模块化的应用程序的所有代码。

> manifest代码：列出所有的引用模块和模块标识符(module identifier)的关系，通过使用 manifest 中的数据，runtime 将能够查询模块标识符，检索出背后对应的模块。

> 之前一直混淆了chunk和模块（Module）的概念，其实很简单，chunk是描述打包后的代码分割结构，模块是描述写代码时的引用结构

# webpack3的处理方法

webpack3 可以采用 CommonsChunkPlugin 插件处理公共模块合并，进行持久化缓存

CommonsChunkPlugins会读取每个文件的引入， 通过将公共模块拆出来，最终合成的文件能够在最开始的时候加载一次，便存到缓存中供后续使用。

CommonsChunkPlugins 通过minChunks设置过滤规则，规定打包在文件中的模块的最小引用次数或路径。 会将所有满足minChunks过滤条件的模块放到新的chunk里。

```javascript
    // 打包公共模块
    // 为了保证缓存，采用[chunkhash]作为JS的文件名标记而不是[hash]
    new webpack.optimize.CommonsChunkPlugin({
      name: 'vendors',    
      minChunks: function(module){
        return module.context && module.context.includes('node_modules')
      }
      // 所有在node_modules里面引用的模块都被打包进来
    }),
    
    // 打包异步加载模块
    new webpack.optimize.CommonsChunkPlugin({
       name: 'app',
       async: 'vendor-async',
       children: true,
       minChunks: 3 
    }),
    // 当children为true时，name必须与entry chunk的名字相同，若忽略为所有entry chunk
    // or
    // names: ['app', 'subPageA']
    // 如async 为 true，异步调用的模块的公共部分会被打包到名为 vendor-async 的模块，当child chunk 加载时自动加载
    // async为字符串表示common chunk的名字
    // 如async 为 false，异步调用的模块的公共部分会被打包到首屏加载的bundle中，增加了首屏时间

    // 打包manifest和runtime
    new webpack.optimize.CommonsChunkPlugin({ 
      name: 'mainifest',
      minChunks: Infinity 
      //Infinity
      // 这个配置保证没其它的模块会打包，包括manifest和runtime
    }),
```

缺陷：
1. 加载的代码很有可能比引用的代码多，因为是把所有的common chunks按照minChunks的规则打包在一起，一起分块、一起引用
1. 异步的时候效率低
1. 由于调用的模块包含manifest和runtime，每次打包都不同，不能包含在要被缓存的文件中，需要在minChunks的过滤函数里面做特殊处理，不支持默认配置，使用起来不方便

# webpack4的处理方法

> There is a paradigm shift here. The CommonsChunkPlugin was like: “Create this chunk and move all modules matching minChunks into the new chunk”. The SplitChunksPlugin is like: “Here are the heuristics, make sure you fullfil them”. (imperative vs declarative)

webpack4废弃了CommonsChunkPlugin，将其拆成了 runtimeChunkPlugin 和 splitChunksPlugin两个插件。runtimeChunkPlugin用于处理生成的动态的manifest和runtime，splitChunksPlugin用于处理其他代码模块，包括代码中的引用模块以及node_modules的公共模块，这样分离时就不需要手动配置过滤规则。

## runtimeChunkPlugin
分离动态生成的mainfest和runtime通过将`runtimeChunk`的值设置为`true`即可达到效果。


## SplitChunksPlugin
SplitChunksPlugin 相比于之前规则更为复杂，它采用一种启发式的方法通过设置缓存组完成分割。

SplitChunksPlugin 的默认规则如下：
* 新 bundle 被两个及以上模块引用，或者来自 node_modules
* 新 bundle 大于 30kb （压缩之前）
* 异步加载并发加载的 bundle 数不能大于 5 个
* 初始加载的 bundle 数不能大于 3 个

### 缓存组（cacheGroups）

SplitChunkPlugin通过缓存组对chunk进行拆分,公共配置和缓存组自身配置合并为其配置策略。（仅有`test`、`piority`、`reuseExistingChunk`只能通过缓存组单独配置。）缓存组里采取优先策略，一个module如果满足多个过滤条件，优先加入priority数值较大优先级较高的组。新增的缓存组默认值为0。

缓存组可以通过设置 `chunks: "async" / "initial" / "all"` 来规定分割模块的影响范围是异步加载chunk(async chunks/on-demand chunks)、初始加载chunk(initial chunks)或者所有chunk。

其默认将node_modules中的模块拆分到名为`vendors`的代码块中，将最少重复引用两次的模块放入`default`中。通过设置`optimization.splitChunks.cacheGroups.default`为`false`禁用默认缓存组。

当某个缓存组的`enforce`参数配置为`true`的时候，
`minSize`和`maxSize`置为0,`minChunks`置为1,`maxAsyncRequests`和`maxInitialRequests`置为`Infinity`。

```javascript
splitChunks: {
    //公共配置
    chunks: "async",
    minSize: 30000,
    minChunks: 1, //在分割之前共享一个module的最小chunk数量
    maxAsyncRequests: 5,
    maxInitialRequests: 3, //一般情况下，入口需要加载一个vendor一个default，再加上其他代码刚好是3，不改变默认值看不出打包效果
    name: true,
    cacheGroups: {
        default: {
            minChunks: 2,
            priority: -20
            reuseExistingChunk: true,
        },
        vendors: {
            test: /[\\/]node_modules[\\/]/,
            priority: -10
        }
    }
}
```



待补充:
* 不同的`cacheGroup`设置不同的`maxInitialRequests`和不同的`maxAsyncRequests`效果如何？
* 用HashedModuleIdsPlugin 解决module ID变动引起的缓存变动(生产环境)
* DLLPlugin配置缓存组





#### 参考文献：

https://webpack.js.org/plugins/commons-chunk-plugin/

https://webpack.js.org/plugins/hashed-module-ids-plugin/

https://webpack.js.org/plugins/split-chunks-plugin/

https://github.com/webpack/webpack/tree/master/examples/common-chunk-and-vendor-chunk

https://medium.com/webpack/webpack-4-code-splitting-chunk-graph-and-the-splitchunks-optimization-be739a861366

https://juejin.im/post/5b304f1f51882574c72f19b0

https://github.com/lancewux/anatomy-of-webpack

http://qiutianaimeili.com/html/page/2018/06/d348hdviz3w.html

https://github.com/Adamwu1992/adamwu1992.github.io/issues/7

https://stackoverflow.com/questions/42559961/what-does-children-refer-to-in-commonschunkplugin-config
