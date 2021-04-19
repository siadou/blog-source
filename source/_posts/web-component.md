---
title: Web Component笔记
date: 2021-01-19 12:34:28
tags: 前端技术
---
Web Component是HTML标准里原生的一种组件的实现方式。类似于Vue React的组件。允许用户创建特定的组件并在HTML Doc中使用。


# 定义组件

有以下几种方式创建组件

1. 创建一个独立的自定义组件类
```
class PopUpInfo extends HTMLElement {
  constructor() {
    // 必须首先调用 super方法 
    super(); 

    // 其它构造写在这里
  }
}
```
在doc文档里的使用方法：
```
<popup-info>
```
JS的引用方法：
```
<script>
document.createElement('popup-info')
</script>
```


2. 基于已有的标准html元素继承

```
class WordCount extends HTMLParagraphElement {
  constructor() {
    // 必须首先调用 super方法 
    super(); 

    // 其它构造写在这里
  }
}
```
在doc文档里的使用方法：
```

<ul is="expanding-list">
</ul>
```
JS的引用方法：
```
<script>
document.createElement('ul', {is: 'expanding-list'})
</script>
```
# shadow DOM
WebComponent通过Shadow DOM。构建组件的内部细节，
它可以将一个隐藏的、独立的 DOM 附加到一个元素上。而在组件外无法对组件内的DOM进行访问。
一个典型的应用是`<video>`标签，内部的按钮控制器都是通过Shadow DOM实现的。

可以使用 Element.attachShadow() 方法来将一个 shadow root 附加到任何一个元素上。它接受一个配置对象作为参数，该对象有一个 mode 属性，值可以是 open 或者 closed：
```
let shadow = elementRef.attachShadow({mode: 'open'});
let shadow = elementRef.attachShadow({mode: 'closed'});

// open表示可以通过页面内的Javascript方法来获取Shadow DOM,例如：
// let myShadowDom = myCustomElem.shadowRoot
```
如果你将一个 Shadow root 附加到一个 Custom element 上，并且将 mode 设置为 closed，那么就不可以从外部获取 Shadow DOM 了——myCustomElem.shadowRoot 将会返回 null。浏览器中的某些内置元素就是如此，例如`<video>`，包含了不可访问的 Shadow DOM。

将ShadowDOM添加到元素上后，就可以使用 DOM APIs对其进行操作，和操作DOM相同。

下面是一个复杂的Component定义：

```
// Create a class for the element
class PopUpInfo extends HTMLElement {
  constructor() {
    // Always call super first in constructor
    super();

    // Create a shadow root
    const shadow = this.attachShadow({mode: 'open'});

    // Create spans
    const wrapper = document.createElement('span');
    wrapper.setAttribute('class', 'wrapper');

    const icon = document.createElement('span');
    icon.setAttribute('class', 'icon');
    icon.setAttribute('tabindex', 0);

    const info = document.createElement('span');
    info.setAttribute('class', 'info');

    // Take attribute content and put it inside the info span
    const text = this.getAttribute('data-text');
    info.textContent = text;

    // Insert icon
    let imgUrl;
    if(this.hasAttribute('img')) {
      imgUrl = this.getAttribute('img');
    } else {
      imgUrl = 'img/default.png';
    }

    const img = document.createElement('img');
    img.src = imgUrl;
    icon.appendChild(img);

    // Create some CSS to apply to the shadow dom
    const style = document.createElement('style');
    console.log(style.isConnected);

    style.textContent = `
      .wrapper {
        position: relative;
      }
      .info {
        font-size: 0.8rem;
        width: 200px;
        display: inline-block;
        border: 1px solid black;
        padding: 10px;
        background: white;
        border-radius: 10px;
        opacity: 0;
        transition: 0.6s all;
        position: absolute;
        bottom: 20px;
        left: 10px;
        z-index: 3;
      }
      img {
        width: 1.2rem;
      }
      .icon:hover + .info, .icon:focus + .info {
        opacity: 1;
      }
    `;

    // Attach the created elements to the shadow dom
    shadow.appendChild(style);
    console.log(style.isConnected);
    shadow.appendChild(wrapper);
    wrapper.appendChild(icon);
    wrapper.appendChild(info);
  }
}

// Define the new element
customElements.define('popup-info', PopUpInfo);

```

# template

让我们看一个简单的示例：
```
<template id="my-paragraph">
  <p>My paragraph</p>
</template>
```
上面的代码不会展示在你的页面中，直到你用 JavaScript 获取它的引用，然后添加到DOM中， 如下面的代码：
```
let template = document.getElementById('my-paragraph');
let templateContent = template.content;
document.body.appendChild(templateContent);
```
我们可以在WebComponent中利用这种方法添加为我们自定义的WebComponent添加Shadow DOM。

```
<template id="my-paragraph">
  <style>
    p {
      color: white;
      background-color: #666;
      padding: 5px;
    }
  </style>
  <p>My paragraph</p>
</template>
```

```
customElements.define('my-paragraph',
  class extends HTMLElement {
    constructor() {
      super();
      let template = document.getElementById('my-paragraph');
      let templateContent = template.content;

      const shadowRoot = this.attachShadow({mode: 'open'})
        .appendChild(templateContent.cloneNode(true));
  }
})
```
这里通过cloneNode添加，防止对一个Component 的DOM结构进行修改引起其他Component不应该有的变化。

# slot

插槽的意义同和Vue的插槽意义相同。允许在Component内部插入一个元素

如果你想添加一个槽到我们的这个例子，我们会将模板的 p 标签改成下面这样:
```
<p><slot name="my-text">My default text</slot></p>
```
如果引用组件时未定义插槽内容，或浏览器不支持slot属性，则<my-paragraph>仅包含默认值"My default text"。

要定义插槽内容，我们在<my-paragraph>元素内包括一个HTML结构，该结构具有slot属性，其值等于我们要填充的<slot>的name属性的值。 和以前一样，它可以是您喜欢的任何东西，例如：

```
<my-paragraph>
  <span slot="my-text">Let's have some different text!</span>
</my-paragraph>
```

# 生命周期

* connectedCallback: 当自定义元素第一次被连接到文档DOM时被调用。
* disconnectedCallback: 当自定义元素与文档DOM断开连接时被调用。
* adoptedCallback: 当自定义元素被移动到新文档时被调用。
* attributeChangedCallback: 当自定义元素的一个属性被增加、移除或更改时被调用。


# 参考文献
1. [Web Components](https://developer.mozilla.org/zh-CN/docs/Web/Web_Components)
2. [使用 templates and slots](https://developer.mozilla.org/zh-CN/docs/Web/Web_Components/Using_templates_and_slots)
3. [web-components-examples - github](https://github.com/mdn/web-components-examples)
