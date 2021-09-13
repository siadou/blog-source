---
title: Canvas绘制中的内外阴影还原问题
date: 2021-09-13 17:14:51
tags: [前端技术, Canvas]
---
## canvas阴影相关的属性

canvas一般通过以下三个属性控制阴影：

属性 | 描述
---|---
shadowColor | 设置或返回用于阴影的颜色
shadowBlur | 设置或返回用于阴影的模糊级别
shadowOffsetX | 设置或返回阴影与形状的水平距离
shadowOffsetY | 设置或返回阴影与形状的垂直距离

相比于CSS，canvas的阴影少了两个参数，一个spread参数用于控制阴影的扩散大小，一个inset参数用于控制阴影是内阴影还是外阴影。



CSS阴影的整体覆盖区域的大小相当于原始元素的borrder-box 大小并增加了spread距离 + blur距离 / 2。参考文献[CSS Backgrounds and Borders Module Level 3]

Canvas的阴影整体的覆盖区域只增加了blur距离/2。没有spread距离。（除此以外，阴影算法指定了2D Gaussian Blur，shadowBlur / 2作为标准差。而HTML规范里未指定算法）。

除了参数以外，由于canvas不区分内阴影和外阴影，canvas阴影效果会受到fillColor透明度的影响，但是CSS的阴影不会受到background-color的影响。原因在于canvas在绘制阴影的时候会考虑到绘制图形已经完成部分的alpha通道，但是css阴影的box-shadow不会考虑填充颜色。（补充photoshop：photoshop的阴影区分外阴影和内阴影。设置透明填充后不会因为设置外阴影影响已绘制图形内部表现。然而如果通过设置图层透明度实现透明填充会影响阴影透明度。这和css采用opacity控制透明度表现一致。）

实际应用时，参考文献3的作者发现Canvas的阴影受到绘图时原图像大小的影响。原图像很小的时候阴影会变浅。这点CSS和canvas表现一直。这是由于对原图形的大小小于高斯模糊的范围，不同像素叠加在一起会使阴影变透明。理论上来讲，只要绘制的图形大小远大于blur距离，绘制的阴影就不会受到图形的影响变浅。


如下代码所示，实际画图中忽略spread，在未设置透明度的时候利用canvas画出的阴影和css画出的阴影效果是一样的。


<p class="codepen" data-height="300" data-default-tab="html,result" data-slug-hash="mdwWYER" data-user="siadou" style="height: 300px; box-sizing: border-box; display: flex; align-items: center; justify-content: center; border: 2px solid; margin: 1em 0; padding: 1em;">
  <span>See the Pen <a href="https://codepen.io/siadou/pen/mdwWYER">
  Canvas shadow vs CSS shadow</a> by siadou (<a href="https://codepen.io/siadou">@siadou</a>)
  on <a href="https://codepen.io">CodePen</a>.</span>
</p>
<script async src="https://cpwebassets.codepen.io/assets/embed/ei.js"></script>

## canvas实现css的阴影spread

由上可知，CSS的阴影的spread是在原图形的外侧补一圈扩散半径。可以在绘制规则图形时增加半径，通过设置偏移量将box移出画布，仅保留阴影。再将阴影绘制到需要的位置即可。


参考figma工程师发表的[Behind the feature: shedding light on shadow spread]这篇文章，这篇文章在采用何种方法进行模拟边界扩展的时候提出了两种方案，比如是简单的对原图形进行拉伸还是生成一个更大的图形。（作者测试各种不同的box形状后得出结论：有一些实现缺乏标准，还有一些浏览器对于box-shadow一些实现的方案也不符合W3C标准，捂脸）参考现有CSS的实现我们可以得到一些图形的简单模拟：

矩形：圆角矩形
椭圆形：在原来椭圆形的基础上增加长轴短轴
圆形：一个更大的圆形


> 考虑到canvas阴影受到原图alpha通道的影响，此方法仅对内部透明一致的图形适用。

## canvas内阴影的实现

以下是一种内阴影的实现办法，首先需要画一个很粗的矩形边框加阴影，然后通过pathClip切除不需要的部分实现。

```
function rectInnerShadow (ctx, x, y, w, h, shadowColor, shadowBlur, lineWidth) {
  shadowColor = shadowColor || '#00f' // 阴影颜色
  lineWidth = lineWidth || 20 // 边框越大，阴影越清晰
  shadowBlur = shadowBlur || 30 // 模糊级别，越大越模糊，阴影范围也越大，注意shadowBlur越大，lineWidth也要相应增大。

  ctx.save()
  ctx.beginPath()

  // 裁剪区(只保留内部阴影部分)
  ctx.rect(x, y, w, h)
  ctx.clip()

  // 边框+阴影
  ctx.beginPath()
  ctx.lineWidth = lineWidth
  ctx.shadowColor = shadowColor
  ctx.shadowBlur = shadowBlur
  // 因线是由坐标位置向两则画的，所以要移动起点坐标位置，和加大矩形。
  // ctx.strokeRect(x - lineWidth / 2, y - lineWidth / 2, w + lineWidth, h + lineWidth)
  ctx.strokeRect(x - lineWidth / 2, y - lineWidth / 2, +w + lineWidth, +h + lineWidth)

  // 取消阴影
  ctx.shadowBlur = 0

  ctx.restore()
}
```



另一种通过混合模式实现。首先新建一个画布，用黑色像素在'source-out'模式下完成内外阴影的绘制，之后通过'source-in'混合模式叠加阴影颜色完成阴影的上色。最后叠加到已有的画布上即可。（如果想实现内外阴影，将'source-out'混合模式改成'xor'即可。）

绘制带阴影的填充/描边实际相当于2个步骤，第一个步骤先根据图形计算阴影绘制阴影层，之后再绘制图形层。所以对填充之前设置混合模式会影响阴影绘制的结果。

> 利用混合模式实现内阴影必须保证阴影在新的画布上绘制，再将阴影重新绘制到之前的图形画布上，否则会影响之前的画布上已有的图形。

```
const width = 100 * devicePixelRatio;
const height = 100 * devicePixelRatio;
// original canvas
const c = document.getElementById('canvas');
c.width = 300 * devicePixelRatio;/* w ww. d e  m  o 2s  .  c o m*/
c.height = 300 * devicePixelRatio;
c.style.width = '300px';
c.style.height = '300px';
const cctx = c.getContext('2d');
cctx.fillStyle = 'rgb(20,205,75)';
cctx.arc(150 * devicePixelRatio, 150 * devicePixelRatio, 50 * devicePixelRatio, 0, Math.PI * 2);
cctx.fill();
// temporary canvas
const canvas = document.createElement('canvas');
canvas.width = width;
canvas.height = height;
canvas.style.width = `${width / devicePixelRatio}px`;
canvas.style.height = `${height / devicePixelRatio}px`;
document.body.appendChild(canvas);
var ctx = canvas.getContext('2d');
// original object on temporary canvas
/*
ctx.arc(50 * devicePixelRatio, 50 * devicePixelRatio, 50 * devicePixelRatio, 0, Math.PI * 2);
ctx.fill();

*/
ctx.globalCompositeOperation = 'source-out';
/*
ctx.arc(50 * devicePixelRatio, 50 * devicePixelRatio, 50 * devicePixelRatio, 0, Math.PI * 2);
ctx.fill();
*/

// shadow props
ctx.shadowBlur = 20;
ctx.shadowOffsetX = 0;
ctx.shadowOffsetY = 0;
ctx.shadowColor = '#000';
ctx.arc(50 * devicePixelRatio, 50 * devicePixelRatio, 50 * devicePixelRatio, 0, Math.PI * 2);
ctx.fill();
// shadow cutting


// shadow color
ctx.globalCompositeOperation = 'source-in';
ctx.fillStyle = 'blue';
ctx.fillRect(0, 0, canvas.width, canvas.height);

// object cutting
/*
ctx.globalCompositeOperation = 'destination-in';
ctx.arc(50 * devicePixelRatio, 50 * devicePixelRatio, 50 * devicePixelRatio, 0, Math.PI * 2);
ctx.fill();
*/
// shadow opacity
cctx.globalAlpha = .4;
// inserting shadow into original canvas
cctx.drawImage(canvas, 100* devicePixelRatio, 100* devicePixelRatio);
```


## 参考文献
1. [CSS Backgrounds and Borders Module Level 3](https://www.w3.org/TR/css-backgrounds-3/#box-shadow)

2. [HTML Standard (HTML)# dom-context-2d-shadowblur-dev](https://html.spec.whatwg.org/multipage/canvas.html#shadows)

3. [Is there something of CANVAS shadow properties like the “spread” property of CSS box-shadow?](https://stackoverflow.com/questions/60656468/is-there-something-of-canvas-shadow-properties-like-the-spread-property-of-css)

4. [Behind the feature: shedding light on shadow spread](https://www.figma.com/blog/behind-the-feature-shadow-spread/)

5. [canvas中常见问题的解决方法及分析，踩坑填坑经历](http://www.yanghuiqing.com/web/352)

6. [Javascript Canvas Inset-shadow on HTML5 canvas image](https://www.demo2s.com/javascript/javascript-canvas-inset-shadow-on-html5-canvas-image.html)

7. [how to make a shadow in HTML canvas](https://stackoverflow.com/questions/29393591/how-to-make-a-shadow-in-html-canvas)

8. [画布的排版效果](https://www.html5rocks.com/zh/tutorials/canvas/texteffects/#toc-spaceage)

9. [CanvasRenderingContext2D.globalCompositeOperation](https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D/globalCompositeOperation)
