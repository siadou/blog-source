---
title: NUXT.js踩坑记录
date: 2018-08-16 17:27:32
tags: 前端技术
---
此次的项目有SEO的要求，又想利用前端框架方便快速开发，因此选用基于VUE的SSR框架NUXT.js。
NUXT.js除了支持SSR模式之外还支持一种生成静态页面的方式在生产环境中部署，相比于SSR对于后台的要求更低，而且可以利用已有的部署环境进行部署，此次我们就采用了这种方式。


# 遇到的问题

由于项目是渐进式更新，需要将新版的页面发布到旧版的环境里。由于既包含新版页面又包含旧版页面，在测试的过程中出现了一些问题。

### 组件生命周期钩子在特定阶段执行(SSR)

应用在编译或服务器端进行预渲染阶段会执行以下生命周期相关函数：
* beforeCreate
* created
其他生命周期会在客户端执行：
* beforeMount
* mounted
所以应该避免在`beforeCreate`和`created`生命周期时产生全局副作用的代码，例如在其中使用`setInterval`设置 timer。应将其移动到客户端执行的生命周期中。

对于不支持SSR的组件，可以进行以下配置：
```
 plugins: [
 	{ src: "~/plugins/xxxxx", ssr: false }
]
```
或者增加针对环境的判断：
```
process.browser == true
or 
window !== undefined
```
### 点击超链接跳转到未上线的页面 nuxt-link

nuxt-link采用前端路由进行定向，页面不会发生跳转，点击之后不会跳转到旧版页面而是会跳转到新版页面里

### 将生成的静态文件发布到与项目不同的新目录里/v2/xxx，所有的js都没有执行

nuxtjs的页面脚本通过页面路径来判断是否执行。所以发布静态路径的路由一定要和项目的路径保持一致

### axios发送的所有的网络请求都转到了localhost:3000上

第一次在服务器上运行时发现页面的所有请求都被请求到了localhost:3000上。
@nuxtjs/axios会对axios设置一个默认的baseURL地址，具体逻辑如下：

```
  const defaultPort =
    process.env.API_PORT ||
    moduleOptions.port ||
    process.env.PORT ||
    process.env.npm_package_config_nuxt_port ||
    3000

  // Default host
  let defaultHost =
    process.env.API_HOST ||
    moduleOptions.host ||
    process.env.HOST ||
    process.env.npm_package_config_nuxt_host ||
    'localhost'

  /* istanbul ignore if */
  if (defaultHost === '0.0.0.0') {
    defaultHost = 'localhost'
  }
  
  const prefix = process.env.API_PREFIX || moduleOptions.prefix || '/'

  
  baseURL: `http://${defaultHost}:${defaultPort}${prefix}`,
```
`browserBaseURL`默认为`baseURL` 或 `prefix` 当 `options.proxy`为`true`

此时配置`baseURL`或`browserBaseURL`即可解决该问题：

```
    axios: {
        baseURL: "/"
    },
```

### NUXT.js的babel配置

为了处理IE浏览器的兼容问题需要配置babel-polyfill
nuxtjs默认采用 @vue/babel-preset-app 配置babel。
由于使用的webpack4去掉了对vendor的支持，故在入口处添加babel-polyfill的引用。
原默认配置如下：
```
build: {
    babel: {
        presets: ['vue-app']
    }
}
```
引入babel-polyfill需要将相关配置修改如下：

```
build: {
    babel: {
        presets: ['vue-app', {
            useBuiltIns: false,
            targets: { ie: 9, uglify: false }
        }]
    },
    extend(config, { isDev, isClient, isServer }) {
        if (isClient) {
            // 拓展 webpack 配置
            if (typeof config.entry == 'string') {
                config.entry = {
                    app: config.entry,
                    polyfill: ['babel-polyfill']
                }
            } else if(typeof config.entry == 'object'){
                config.entry['polyfill'] = ['babel-polyfill']
            }
        }
    }
}
```
首先我们在client端的编译阶段增加babel-polyfill的入口。
其中babel-preset-env中的`useBuiltIns`的参数配置目的是将babel-polyfill拆解成多个模块，再根据环境分别进行引用。
此处将useBuiltIns修改为`false`会将babel-polyfill整体引用，如果不修改此处配置ie环境下调用仍然会报错。

### 项目里遇到的NUXT.js以外的问题
埋点包含中文信息，通过base64编码会出现加号，经过转码可能会被处理成空格，需要和后台协调转成其他字符进行处理。

