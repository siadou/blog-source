---
title: HTML标签的crossorigin属性
date: 2020-08-14 09:59:09
tags: [前端技术, 浏览器]
---

# 范围
以下几种标签可以设置crossorigin属性：
`<audio>`, `<img>`, `<link>`, `<script>`,`<video>`

crossorigin参数可以控制这些标签的CORS配置，具体配置方法如下所示。

# 可选值

值 | 描述
---|---
“” | 设置空值，如 crossorigin 或 crossorigin=""，效果同 anonymous。
anonymous | 该元素的CORS请求将会有credentials的标识，credentials 模式是'same-origin'
use-credentials | 该元素的CORS请求将会有credentials的标识，credentials 模式是'include'
未设置 | 不启用CORS（区分空值）

默认情况下，“anonymous”关键字意味着在非同源的情况下不会通过cookies进行用户credentials的交换，用户侧的SSL证书或者HTTP认证。


# 配置

1. 引用的标签配置对应的属性：`crossorigin`
2. 同时图片的服务器设置图片返回的HTTP报头配置
`Access-Control-Allow-Origin` 添加对应域名或`*`


# 场景
## 解决canvas使用非同源元素的问题

问题：

不通过 CORS 就可以在 `<canvas>`中使用其他来源的图片，但是这会污染画布，可能在 `<canvas>` 检索数据过程中引发异常。

如果从外部引入的 HTML `<img>` 或 SVG `<svg>`，并且图像源不符合规则，将会被阻止从 `<canvas>`中读取数据。（`<audio>`,`<video>`同理）。

在"被污染"的画布中调用以下方法将会抛出安全错误：

在 `<canvas>` 的上下文上调用 `getImageData()`
在 `<canvas>` 上调用 `toBlob()`
在 `<canvas>` 上调用 `toDataURL()`


```
function startDownload() {
  let imageURL = "https://cdn.glitch.com/4c9ebeb9-8b9a-4adc-ad0a-238d9ae00bb5%2Fmdn_logo-only_color.svg?1535749917189";
 
  downloadedImg = new Image;
  downloadedImg.crossOrigin = "Anonymous";
  downloadedImg.addEventListener("load", imageReceived, false);
  downloadedImg.src = imageURL;
}
```


## script标签的crossorigin
那些没有通过标准CORS检查的正常`script`元素传递最少的信息到 `window.onerror`。可以使用本属性来使那些将静态资源放在另外一个域名的站点打印错误信息。

```
<script src="" crossorigin="anonymous"></script>
```




# 参考文献
1. [HTML attribute: crossorigin](https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes/crossorigin)
2. [MDN - 启用了 CORS 的图片](https://developer.mozilla.org/zh-CN/docs/Web/HTML/CORS_enabled_image)
3. [CORS protocol and credentials](https://fetch.spec.whatwg.org/#concept-request-credentials-mode)
4. [MDN - \<img\>](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/img#attr-crossorigin)
5. [MDN - \<script\>](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/script)
