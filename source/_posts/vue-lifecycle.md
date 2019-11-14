---
title: vue生命周期踩坑记录
date: 2019-11-13 10:41:38
tags: [前端技术, vue]
---

![](/images/vue-lifecycle/vue-lifecycle.png)

 # 概述
 
 Vue组件的生命周期主要为8个: beforeCreate、created、beforeMount、mounted、beforeUpdate、updated、beforeDestroy、destroyed。

- 创建阶段（注册实例与挂载）： beforeCreate、created、beforeMount、mounted
- 运行阶段：beforeUpdate、updated
- 注销阶段：beforeDestroy、destroyed

# 测试
以下代码测试生命周期的执行顺序：

## 代码

https://codesandbox.io/embed/vue-template-g7w0m?fontsize=14

基本结构：
```
<father>
    <child :msg="msg" :key="key" />
</father>
```

父组件更新msg触发子组件更新，更新key触发子组件重新构造。

## 打印结果

控制台打印结果：（p为父组件，c为子组件）：

```
------创建阶段-------
p beforeCreated 
p created 
p beforemount 
c beforeCreated 
c created 
c beforemount 
c mount
p mount 
```

父组件先创建，在beforeMount查找子组件，发现有没创建的子组件，重复构建子组件，等子组件mount完成后执行mount

```
-----更新阶段（子组件更新）-------
p beforeupdate 
c beforeupdate 
c updated 
p updated 
```

父组件向里触发子组件更新，再冒泡回来触发父组件更新完成。

```
------更新阶段（子组件重建）------
p beforeupdate 
c beforeCreated 
c created 
c beforemount 
c beforeDestroy
c destroyed 
c mount
p updated
```

首先创建新的子组件，在新的子组件mount之前销毁旧的子组件，再挂载新的子组件。

# 遇到的问题

## 前提

父子组件通过eventBus通信，父组件在created里面注册了事件绑定的方法，在destroyed里面释放方法。子组件通过emit事件触发父组件调用方法。

## 现象
更新子组件后emit事件，没有调用父组件的获取数据的方法

## debug过程

1. 到哪一步没有被调用

- 子组件emit事件后，没有转到对应的父组件里面

2. 父子组件调用的是否是同一个eventBus

- console出来是同一个

3. 是否是父组件在eventBus上绑定事件没有触发

- 绑定事件触发

4. 解绑事件是否被提前触发

- 解绑事件被提前触发，在打开页面不久就被触发了。原因为了解决子组件更新问题，在子组件上bind了动态的key。key更新触发元素destroy。

5. 解绑事件在destroyed里面，为何被提前触发

- 观察绑定事件触发了两次，解绑事件触发了一次，顺序是
```
on block created
on block created
off block destroy
```
由于绑定事件在off前进行了两次不会叠加，解绑事件把之前绑定的两次都移除掉了，确认是生命周期执行顺序问题。

## 修复
1. 修正了原有组件在启动的时候key值变化的问题。
2. 修改了vue组件事件绑定的位置，在mount上绑定，在beforeDestroyed上解绑。

## 改进意见

避免在created/destroyed上绑定事件/解绑事件，首先是可能会出现以上问题，而且不利于迁移服务端渲染。

# 参考文献

https://www.cnblogs.com/jaykoo/p/10529518.html