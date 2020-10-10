---
title: 自定义主题颜色实现方案
date: 2020-10-10 10:51:11
tags: [前端技术, CSS]
---

最近由于需要给各种客户修改主题色，想实现一个通用的用户自定义主题色的功能，看了下相关的案例主要是通过修改变量实现的。以下是几个典型的案例。

# 修改less变量的方法

引入less的样式，通过修改less的变量，实现主题色的修改

样式定义：

```
@primary-color: #1890ff;

/*定义同样的颜色*/
.ant-alert-info .ant-alert-icon {color: @primary-color;}
a {color: @primary-color;background-color: transparent;}
/*定义不同亮度颜色*/
a:hover {color: color(~`colorPalette("@{primary-color}", 5)`);}
a:active {color: color(~`colorPalette("@{primary-color}", 7)`);}

```

colorPalette的函数定义详见： https://github.com/ant-design/ant-design/blob/master/components/style/color/colorPalette.less
实际colorPalette的计算不只和透明度相关，详细可见以下的函数分析。

```
  let lessLoaded = false
  handleColorChange = (color: string) => {
    // set loading start
    const changeColor = () => {
      (window as any).less
        .modifyVars({
          '@primary-color': color,
        })
        .then(() => {
          // set loading end
        });
    };

    // 异步加载less.js，不修改主题不加载
    const lessUrl = 'https://gw.alipayobjects.com/os/lib/less/3.10.3/dist/less.min.js';

    if (lessLoaded) {
      changeColor();
    } else {
      (window as any).less = {
        async: true,
        javascriptEnabled: true,
      };
      loadScript(lessUrl).then(() => {
        lessLoaded = true;
        changeColor();
      });
    }
  };

```

# 修改var变量的方法

通过var定义基础颜色，通过修改变量对应的颜色，实现主题色的修改
原理基本和上面的方法相同，不同之处在于var的浏览器兼容性相比于less的方法差了一些，且颜色调整灵活度不够，LESS能定义函数，而var不可以。仅作为补充方案列出来。由于是浏览器原生接口，免去了编译，性能更优秀。

var函数的兼容性如下：
![](/images/custom-stylesheet/caniuse-var.png)

样式定义：

```
:root {
    --rgb-color: #4e6e53;
    /* #f0f0f0 in decimal RGB */
    --opacity-color: 240, 240, 240;
    --hsla-color: 290,100%;
}

#element0 {
    color: var(--rgb-color);
}
/*定义不同透明度的颜色*/
#element1 {
    background-color: rgba(var(--opacity-color), 0.8);
}
/*利用hsl色彩空间定义不同明度、透明度的颜色*/
#element2 {
    background-color: hsla(var(--red-color),50%,0.3);
}
```
（参考文献1还提供了别的方法，如使用渐变/伪元素叠加带透明度的白色/黑色和基础色等，这里就不多说了。）
可以通过以下方法修改变量的定义：

```
handleChangeColor = (color) => {
  let root = document.querySelector(':root')
  root.style.setProperty('--rgb-color',color)
}
```
通过调整hsl的明度是一种粗糙的方法，相比于这种方法less函数调整出来的结果更细腻灵活。

# 附录：colorPalette函数

```
(function() {
  var hueStep = 2;
  var saturationStep = 0.16;
  var saturationStep2 = 0.05;
  var brightnessStep1 = 0.05;
  var brightnessStep2 = 0.15;
  var lightColorCount = 5;
  var darkColorCount = 4;

  var getHue = function(hsv, i, isLight) {
    var hue;
    if (hsv.h >= 60 && hsv.h <= 240) {
      hue = isLight ? hsv.h - hueStep * i : hsv.h + hueStep * i;
    } else {
      hue = isLight ? hsv.h + hueStep * i : hsv.h - hueStep * i;
    }
    if (hue < 0) {
      hue += 360;
    } else if (hue >= 360) {
      hue -= 360;
    }
    return Math.round(hue);
  };
  var getSaturation = function(hsv, i, isLight) {
    var saturation;
    if (isLight) {
      saturation = hsv.s - saturationStep * i;
    } else if (i === darkColorCount) {
      saturation = hsv.s + saturationStep;
    } else {
      saturation = hsv.s + saturationStep2 * i;
    }
    if (saturation > 1) {
      saturation = 1;
    }
    if (isLight && i === lightColorCount && saturation > 0.1) {
      saturation = 0.1;
    }
    if (saturation < 0.06) {
      saturation = 0.06;
    }
    return Number(saturation.toFixed(2));
  };
  var getValue = function(hsv, i, isLight) {
    var value;
    if (isLight) {
      value = hsv.v + brightnessStep1 * i;
    }else{
      value = hsv.v - brightnessStep2 * i
    }
    if (value > 1) {
      value = 1;
    }
    return Number(value.toFixed(2))
  };

  this.colorPalette = function(color, index) {
    var isLight = index <= 6;
    var hsv = tinycolor(color).toHsv();
    var i = isLight ? lightColorCount + 1 - index : index - lightColorCount - 1;
    return tinycolor({
      h: getHue(hsv, i, isLight),
      s: getSaturation(hsv, i, isLight),
      v: getValue(hsv, i, isLight),
    }).toHexString();
  };
})()
```
函数将原来的颜色映射到hsv域上，定义了1-10共计11个不同的深浅度等级，1-4为深色，6-10为浅色。将落在不同区域的颜色进行不同的调整。


# 参考文献

1. [How do I apply opacity to a CSS color variable?](https://stackoverflow.com/questions/40010597/how-do-i-apply-opacity-to-a-css-color-variable/41265350)
2. [CSS 变量教程](http://www.ruanyifeng.com/blog/2017/05/css-variables.html)
4. [ant-design](https://ant.design/components/overview-cn/)
5. [HSL和HSV色彩空间](https://www.sohu.com/a/245724770_802226)
