---
title: 如何编写一个ESLint插件
date: 2020-08-31 11:39:08
tags: 前端技术
---

ESLint是一个插件化的javascript代码检查工具。可以通过自定义插件的方法，对代码进行格式检查。本文的应用场景是通过ESLint对Vue元素的嵌套模式进行约束。

# 特点
eslint 只单纯的逐个对单文件进行语法检查，不分析文件和文件间依赖。没有办法分析动态的结构，如远程获取数据之后渲染的结构（想解决此问题建议使用前段自动化测试工具）。自定义规则可用于强制规定项目嵌套规则语法规则等场景。
 

# 步骤

1. 编写eslint规则, 详情可见 https://eslint.bootcss.com/docs/developer-guide/working-with-plugins

    a. 创建项目
    
    b. 创建规则
   
   
`detail-card.js`

```
const utils = require('eslint-plugin-vue/lib/utils')

// ------------------------------------------------------------------------------
// Rule Definition
// ------------------------------------------------------------------------------
const getElementChildren = function (node) {
  let children = node.children ||[]
  return children.filter((item) => {
    return item.type === 'VElement'
  })
};


const getAllElementChildren = function (node) {
  let res = []
  let children = [node]
  while(children.length) {
    let curLevel = []
    children.forEach(child => {
      if (child.children && child.children.length) {
        let elements = child.children.filter((item) => {
          return item.type === 'VElement'
        })
        curLevel = curLevel.concat(elements)
        res = res.concat(elements)
      }
    })
    children = curLevel
  }
  return res
};

module.exports = {
  meta: {
    type: 'suggestion',
    docs: {
      description: 'enforce detail card style',
      url: 'https://eslint.vuejs.org/rules/html-end-tags.html'
    },
    fixable: 'code',
    schema: []
  },
  /** @param {RuleContext} context */
  /**找出不标准的 detail-card, 即排除 el-card里面只有 <div slot="header"><span class="title"><div> 和 <table class="vertical-table"> 的情况 */
  create(context) {
    return utils.defineTemplateBodyVisitor(
      context,
      {
        "VElement[name='el-card'] VElement[name='table']"(node) {
          const name = node.name
          let parentIsCard = node.parent.name === 'el-card'
          let tableClassName = (utils.getAttribute(node, 'class') || {value: {value: ''}}).value.value
          // 筛选出子元素不是table的元素
          let child = parentIsCard && node.parent.children.filter((item) => {
            return item.type === 'VElement' && item.name !== 'table'
          })
          let onlyChild = false
          // 排除 div slot === name
          if (child.length === 1) {
            child = child[0]
            slotName = (utils.getAttribute(child, 'slot') || { value: {value: ''} }).value.value
            let children = getAllElementChildren(child)
            if (slotName === 'header' && children.length === 1) {
              let className = (utils.getAttribute(children[0], 'class') || { value: {value: ''} }).value.value
              if (className === 'title') {
                onlyChild = true
              }
            }
          }
          if ((tableClassName !== 'vertical-table') || !onlyChild) {
            context.report({
              node: node.startTag,
              loc: node.startTag.loc,
              message: "'<{{name}}>' is wanted element.\nparentIsCard:{{parentIsCard}}\ntableClassName:{{tableClassName}}\nonlyChild:{{onlyChild}}\n",
              data: { name, parentIsCard, tableClassName, onlyChild : onlyChild },
              fix: (fixer) => fixer.insertTextAfter(node, `</${name}>`)
            })
          }
        }
      }
    )
  }
}

```

`index.js`
```
'use strict'

module.exports = {
  rules: {
    'valid-detail-card': require('./detail-card') // 在rules里面引入
  }
}
```
> 注：此处为示例，实际建议按照官方文档来。插件需要安装 eslint-plugin-vue 包。但是在项目中引用的plugin是项目自己node_modules目录下的plugin。


2. 安装插件


a. 在项目里安装eslint插件，如`eslint-plugin-local`：
```
  "devDependencies": {
    "eslint-plugin-local": "file:./eslintrules",  // 这一行，此处为直接文件引入，建议git仓库引用
    "@vue/babel-preset-app": "3.0.5",
    "@vue/cli-plugin-babel": "3.0.5",
    "@vue/cli-plugin-eslint": "3.0.5",
```
> 注：eslint不支持非npm包以外类型的插件。如直接引入项目里的文件等等等。且包名开头必须是eslint-plugin-xxx。不支持除此以外的命名方式。

b. 在.eslintrc.js里面配置插件，如：

如果插件名叫eslint-plugin-xxxx，则plugins里面引用插件引用eslint-plugin-xxxx或xxxx，rules里引用插件引用：xxxx/规则名，如下：

```
module.exports = {
  root: true,
  env: {
    node: true
  },
  'extends': [
    //'plugin:vue/essential',
    'plugin:vue/base',  // 必须引用vue的扩展，vue/base或vue/essential
  ],
  plugins: [
    'eslint-plugin-local'  // 这一行
  ],
  globals: {
    $: true,
    _: true
  },
  rules: {
    'local/valid-detail-card': 'error' // 这一行
  },
  parserOptions: {
    parser: 'babel-eslint'
  }
}
```

3.  指令调用eslint

在文件夹下运行以下指令：

```
$ ./node_modules/.bin/eslint --quiet  path/xxx.vue  //检查单个vue文件
$ ./node_modules/.bin/eslint --quiet  **/*.vue //检查全部vue文件
```
或
```
$ npx eslint --quiet  path/xxx.vue  //检查单个vue文件
$ npx eslint --quiet  **/*.vue   //检查全部vue文件
```

# vue-eslint-parser 解析规则

详情可见 eslint-plugin-vue 自带的rules。
https://github.com/vuejs/eslint-plugin-vue

项目自带方法可见项目目录 https://github.com/vuejs/eslint-plugin-vue/tree/master/lib/utils


vue-eslint-parser解析Vue文件获得的AST树结构可参考：
https://github.com/mysticatea/vue-eslint-parser/blob/master/docs/ast.md


解析JS文件获得的AST树结构可参考：
https://github.com/estree/estree

