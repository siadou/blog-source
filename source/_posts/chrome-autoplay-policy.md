---
title: Chrome 自动播放策略
date: 2020-08-04 15:05:59
tags:
---

最近实现了一个在网页上特定网络事件触发播放音频功能，一定情况下会失效控制台会报错，发现是由于Chrome不支持无互动播放有声音频导致的。考虑到斗鱼等网站没有互动自动播放视频，我查看了相关的功能说明并对此进行了一点点研究。以下是我的记录。

2018年4月，Chrome修改了浏览器自动播放媒体文件的策略。对有声视频的播放进行了限制。

# 新特性

Chrome新的自动播放政策如下：

- 始终允许静音媒体自动播放。
- 顶层框架可以将自动播放权限委派给其iframe，以允许自动播放声音。
- 在以下情况下，允许自动播放声音：
1. 用户已与域进行了交互（单击，点击等）。
2. 在本台机器上，被访问的站点已经超过了用户的“媒体参与度索引（MEI）”阈值，这意味着该用户以前曾播放有声视频。
3. 用户已将网站添加到移动设备上的主屏幕上，或在桌面上安装了PWA。


## 媒体参与度指数（MEI）

MEI衡量个人在站点上播放媒体的倾向。Chrome 当前的方法是计算每个来源的访问次数与重点媒体播放事件的比率：

- 媒体（音频/视频）的浏览时间必须大于7秒。
- 音频必须展示且不能静音。
- 带有视频的标签处于活动状态。
- 视频大小（以px为单位）必须大于200x140。

由此，Chrome计算出的媒体参与度得分在定期播放媒体的网站上最高。足够高时，则允许在浏览器上自动播放有声媒体。

注意：媒体参与度是按照客户端计算的，不是按照账号计算的。隐身模式不受影响。

可以在浏览器的 chrome://media-engagement 页面上找到用户的MEI：如下图

![](/images/chrome-autoplay-policy/media-engagement.png)

## 开发人员开关

实测本开关`chrome://flags/#autoplay-policy`已经在新版的Chrome里面移除了。不再做多余叙述了。（不理解为什么移除）。

## iframe委托

开发人员可以选择性地启用和禁用的各种浏览器的功能和API。父框架获得自动播放权限后，将该权限委派给跨域的iframe。（默认在此情况下，同源的iframe与父框架一致允许自动播放。）

```
<!-- Autoplay is allowed. -->
<iframe src="https://cross-origin.com/myvideo.html" allow="autoplay">

<!-- Autoplay and Fullscreen are allowed. -->
<iframe src="https://cross-origin.com/myvideo.html" allow="autoplay; fullscreen">
```
如果没有开启autoplay开关，没有用户手势的play()调用会抛出异常。元素的autoplay也会被忽略。

```
Warning: Older articles incorrectly recommend using the attribute gesture=media which is not supported.
```

# Web开发人员的最佳实践

## 音频/视频元素
记住：永远不要假设视频会播放，并且在视频未实际播放时也不要显示暂停按钮。

通过调用Play得到Promise的返回结果可以知道播放是否被rejected:

```
var promise = document.querySelector('video').play();

if (promise !== undefined) {
  promise.then(_ => {
    // Autoplay started!
  }).catch(error => {
    // Autoplay was prevented.
    // Show a "Play" button so that user can start playback.
  });
}
```

另外一种方法是通过诱导用户与页面发生交互的方式来取消静音。包括Facebook，Instagram，Twitter和YouTube都是这样做的。

```
<video id="video" muted autoplay>
<button id="unmuteButton"></button>

<script>
  unmuteButton.addEventListener('click', function() {
    video.muted = false;
  });
</script>
```

## Web Audio

Web Audio API将包含在M71的Chrome自动播放策略中（2018年12月）。

为了让用户意识到正在发生的事情，Chrome倾向在开始播放音频之前等待用户交互。例如，在界面上添加“播放”按钮或“开/关”开关。或者根据交互场景添加“取消静音”按钮。

如果在页面加载的时候创建AudioContext，您必须在用户与页面进行交互后的某个时间调用resume() （如，用户单击按钮）。或者， 如果AudioContext在任何连接的节点上调用start()，则将在用户交互后播放音频（resume）。

```
// Existing code unchanged.
window.onload = function() {
  var context = new AudioContext();
  // Setup all nodes
  ...
}

// One-liner to resume playback when user interacted with the page.
document.querySelector('button').addEventListener('click', function() {
  context.resume().then(() => {
    console.log('Playback resumed successfully');
  });
});
```

您还可以在用户与页面互动时创建AudioContext。

```
document.querySelector('button').addEventListener('click', function() {
  var context = new AudioContext();
  // Setup all nodes
  ...
});
```

为了检测浏览器是否需要用户交互才可以播放音频，你可以检查AudioContext的state。如果允许播放，则状态会立即切换到running。否则将是suspended。如果您监听statechange事件，则可以异步检测更改。

# 其他网站实践

实际上我们访问斗鱼、爱奇艺等首页，在MEI值为0值且用户与页面无交互时仍然能够触发自动播放首页的视频。尝试了各种hack方法效果不佳。在github的一个用户讨论下面有相关开发人员称并没有什么特殊技巧，只是单纯的chrome有白名单而已。如确实是用了什么好的办法的话，希望读到这篇文章的朋友们可以推荐给我。

# 参考文献
[Autoplay Policy Changes](https://developers.google.com/web/updates/2017/09/autoplay-policy-changes#chrome_enterprise_policies)

[Policy Controlled Features](https://github.com/w3c/webappsec-permissions-policy/blob/master/features.md)

[爱奇艺解决办法](https://github.com/gnipbao/iblog/issues/25#issuecomment-643894062)