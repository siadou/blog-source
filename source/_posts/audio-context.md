---
title: 【译】 使用Web Audio创建电话音 
date: 2020-07-15 14:57:47
tags: [前端技术]
---

创建电话音对于创建声音来讲很适合入门，相对来讲容易理解，只需要几行代码就能实现。

在本文中，我们将创建以下声音：

* 拨号音 
* DTMF音 
* 回铃音 

不同国家/地区的声音听起来有细微的差异，但本文将重点介绍英国版本。但是，您应该能够使用本文介绍的方法创建任何国家/地区任何电话音。

# 拨号音

拨号音是您拿起座机电话时听到的声音。在英国，它由2个连续的正弦波组成，频率分别为350 Hz和440 Hz。

首先，我们将创建一个音频上下文。

```
var context = new AudioContext();
```
我们将创建一个`Tone`对象。我们将传入音频上下文，将其设置为对象的属性。我们还将有一个名为`status`的属性，如果状态等于1，则表示正在播放该声音；如果该属性为0，则将关闭拨号音。

```
function Tone(context) {
    this.context = context;
    this.status = 0;
}
```

接下来，我们将编写一个`setup`方法，创建两个振荡器节点并将其频率设置为350 Hz和440 Hz。虽然可以将它们直接连接到输出，本文选择通过增益节点将它们连接起来，这样可以更轻松地调节音量。（在后文我们将用增益器处理不连续音频场景）。为了使声音更接近真实，我还将连接一个低通滤波器，该滤波器可以切掉所有高频，就像真实的电话线一样。

考虑以上问题，我将在此方法中创建一个增益节点和滤波器，将两个振荡器都连接到增益器，并将增益器连接到滤波器。

```
Tone.prototype.setup = function(){
    this.osc1 = context.createOscillator();
    this.osc2 = context.createOscillator();
    this.osc1.frequency.value = 350;
    this.osc2.frequency.value = 440;

    this.gainNode = this.context.createGain();
    this.gainNode.gain.value = 0.25;

    this.filter = this.context.createBiquadFilter();
    this.filter.type = "lowpass";
    this.filter.frequency = 8000;

    this.osc1.connect(this.gainNode);
    this.osc2.connect(this.gainNode);

    this.gainNode.connect(this.filter);
    this.filter.connect(context.destination);
}
```
现在我们需要考虑如何启动和停止。我们将创建start()and stop()方法。

```
Tone.prototype.start = function(){
    this.setup();
    this.osc1.start(0);
    this.osc2.start(0);
    this.status = 1;
    }

Tone.prototype.stop = function(){
    this.osc1.stop(0);
    this.osc2.stop(0);
    this.status = 0;
}
```

接下来，我们将创建一个基本的HTML用户界面，该界面由一个按钮组成。我们还需要创建`Tone`对象的实例。接下来，我们需要向按钮添加事件监听器。当按下按钮时，我们的事件监听器代码将确定我们是否需要调用`start`或`stop`方法。

```
<button id='js-toggle-dial-tone'>Toggle Dial Tone</button>

<script>
var context = new AudioContext();
var dialTone = new Tone(context);
$("#js-toggle-dial-tone").click(function(e){
    e.preventDefault();
    if (dialTone.status === 0){
        // The tone is currently off, so we need to turn it on.
        dialTone.start();
    } else {
        // The tone is currently on, so we need to turn it off.
        dialTone.stop();
    }
});
</script>
```
这样就完成了拨号音。


<p class="codepen" data-height="265" data-theme-id="light" data-default-tab="js,result" data-user="edball" data-slug-hash="bVZGMj" style="height: 265px; box-sizing: border-box; display: flex; align-items: center; justify-content: center; border: 2px solid; margin: 1em 0; padding: 1em;" data-pen-title="Dial Tone in Web Audio">
!!!

# DTMF频率

接下来，我们将开始处理DTMF（双音多频）音。这些是您在键盘上按数字时听到的声音，就像拨号音一样，由成对的正弦波组成。下表中显示了每个数字使用的确切频率。


0 | 1209Hz | 1336Hz | 1477Hz
---|---|---|---
697Hz | 1 | 2 | 3
770Hz | 4 | 5 | 6
852Hz | 7 | 8 | 9
941Hz | * | 0 | #


我们在创建拨号音时已经完成了主要工作。我们只需要一个指定2个频率的方法。

为此，我们将修改该Tone对象以接受2个新参数`freq1`和`freq2`。这些将成为对象的属性。

```
function Tone(context, freq1, freq2) {
    this.context = context;
    this.status = 0;
    this.freq1 = freq1;
    this.freq2 = freq2;
}
```

然后，我们将修改该setup方法以使用2个新属性。

```
Tone.prototype.setup = function(){
    this.osc1 = context.createOscillator();
    this.osc2 = context.createOscillator();
    this.osc1.frequency.value = this.freq1;
    this.osc2.frequency.value = this.freq2;

    this.gainNode = this.context.createGain();
    this.gainNode.gain.value = 0.25;

    this.filter = this.context.createBiquadFilter();
    this.filter.type = "lowpass";
    this.filter.frequency = 8000;

    this.osc1.connect(this.gainNode);
    this.osc2.connect(this.gainNode);

    this.gainNode.connect(this.filter);
    this.filter.connect(context.destination);
}
```
现在，我们需要创建某种HTML用户界面来使用我们新修改的对象。我们将创建12个按钮，每个DTMF音调一个。

```
<ul class='js-dtmf-interface'>
    <li>1</li>
    <li>2</li>
    <li>3</li>
    <li>4</li>
    <li>5</li>
    <li>6</li>
    <li>7</li>
    <li>8</li>
    <li>9</li>
    <li>*</li>
    <li>0</li>
    <li>#</li>
</ul>
```

在功能方面，当我们单击/点击并按住其中一个元素时，我们希望听到提示音，然后在释放鼠标/手指时模仿模拟真实的拨号盘，然后将其关闭。

在我们的JavaScript中，我们将有一个对象，dtmfFrequencies其中包含所有12种可能的频率组合。当我们检测到点击时，可以通过在该对象中查找来确定要播放的频率。

```
var dtmfFrequencies = {
    "1": {f1: 697, f2: 1209},
    "2": {f1: 697, f2: 1336},
    "3": {f1: 697, f2: 1477},
    "4": {f1: 770, f2: 1209},
    "5": {f1: 770, f2: 1336},
    "6": {f1: 770, f2: 1477},
    "7": {f1: 852, f2: 1209},
    "8": {f1: 852, f2: 1336},
    "9": {f1: 852, f2: 1477},
    "*": {f1: 941, f2: 1209},
    "0": {f1: 941, f2: 1336},
    "#": {f1: 941, f2: 1477}
}
```
与其创建12个音调实例，不如创建一个音调实例，并根据需要更改频率。
```
// Create a new Tone instance. (We've initialised it with 
// frequencies of 350 and 440 but it doesn't really matter
// what we choose because we will be changing them in the 
// function below)
var dtmf = new Tone(context, 350, 440);

$(".js-dtmf-interface li").on("mousedown touchstart", function(e){
    e.preventDefault();

    var keyPressed = $(this).html(); // this gets the number/character that was pressed
    var frequencyPair = dtmfFrequencies[keyPressed]; // this looks up which frequency pair we need

    // this sets the freq1 and freq2 properties
    dtmf.freq1 = frequencyPair.f1;
    dtmf.freq2 = frequencyPair.f2;

    if (dtmf.status == 0){
        dtmf.start();
    }
});

// we detect the mouseup event on the window tag as opposed to the li
// tag because otherwise if we release the mouse when not over a button,
// the tone will remain playing
$(window).on("mouseup touchend", function(){
    if (typeof dtmf !== "undefined" && dtmf.status){
        dtmf.stop();
    }
});
```

到此完成了DTMF部分。您可以在下面的Codepen中看到一个有效的版本，在该版本中，我还向界面中添加了一些CSS。


<p class="codepen" data-height="459" data-theme-id="light" data-default-tab="js,result" data-user="edball" data-slug-hash="EVMaVN" style="height: 459px; box-sizing: border-box; display: flex; align-items: center; justify-content: center; border: 2px solid; margin: 1em 0; padding: 1em;" data-pen-title="DTMF Tones in Web Audio">
  <span>See the Pen <a href="https://codepen.io/edball/pen/EVMaVN">
  DTMF Tones in Web Audio</a> by Ed Ball (<a href="https://codepen.io/edball">@edball</a>)
  on <a href="https://codepen.io">CodePen</a>.</span>
</p>
<script async src="https://static.codepen.io/assets/embed/ei.js"></script>


# 回铃音

最后，我们将研究如何创建回铃音。当您正在呼叫的人的电话响起时，这就是您听到的声音。再一次，它是由2个正弦波（在英国为400 Hz和450 Hz）组成的音调。但是，与前面的示例不同，该音调不是连续的。在英国，其模式或“节奏”为0.4s开，0.2s关，0.4s开，2s关。此循环的总长度为3秒。

我们可以通过使振荡器保持运行状态来解决此问题，但是在需要的时间将增益节点设为0。有许多方法可以做到这一点，例如利用某种计时系统，例如Web Audio Api Clock。但是，由于该模式非常简单，因此我想使用低频振荡器（LFO）。

如果模式以相同的时间打开和关闭，例如打开0.5s然后关闭0.5s，依此类推，我们可以简单地使用方波作为频率为1Hz的LFO。本文使用了其他方法实现LFO。

我们将手动创建波形数据，并将其放置在缓冲区（一小段声音）中，然后将其连续循环。缓冲区需要3秒长，并包含以下波形。

<div class="chart-wrapper" style="background-color: #f8f8f8; padding: 20px; border: 1px solid #ccc; border-radius: 3px; margin: 2em 0; width: 600px; height: 279px;"><div class="ct-chart ct-octave"><svg xmlns:ct="http://gionkunz.github.com/chartist-js/ct" width="100%" height="100%" class="ct-chart-line" style="width: 100%; height: 100%;"><text class="ct-axis-title" x="0" y="139.5" transform="rotate(-90, 0, 119.5)" text-anchor="middle">Value</text><text class="ct-axis-title" x="314" y="269" text-anchor="middle">Time (s)</text><g class="ct-grids"><line x1="70" x2="70" y1="20" y2="219" class="ct-grid ct-horizontal"></line><line x1="232.66666666666666" x2="232.66666666666666" y1="20" y2="219" class="ct-grid ct-horizontal"></line><line x1="395.3333333333333" x2="395.3333333333333" y1="20" y2="219" class="ct-grid ct-horizontal"></line><line x1="558" x2="558" y1="20" y2="219" class="ct-grid ct-horizontal"></line><line y1="219" y2="219" x1="70" x2="558" class="ct-grid ct-vertical"></line><line y1="194.125" y2="194.125" x1="70" x2="558" class="ct-grid ct-vertical"></line><line y1="169.25" y2="169.25" x1="70" x2="558" class="ct-grid ct-vertical"></line><line y1="144.375" y2="144.375" x1="70" x2="558" class="ct-grid ct-vertical"></line><line y1="119.5" y2="119.5" x1="70" x2="558" class="ct-grid ct-vertical"></line><line y1="94.625" y2="94.625" x1="70" x2="558" class="ct-grid ct-vertical"></line><line y1="69.75" y2="69.75" x1="70" x2="558" class="ct-grid ct-vertical"></line><line y1="44.875" y2="44.875" x1="70" x2="558" class="ct-grid ct-vertical"></line><line y1="20" y2="20" x1="70" x2="558" class="ct-grid ct-vertical"></line></g><g><g class="ct-series ct-series-a"><path d="M70,219L71.627,169.25L73.253,169.25L74.88,169.25L76.507,169.25L78.133,169.25L79.76,169.25L81.387,169.25L83.013,169.25L84.64,169.25L86.267,169.25L87.893,169.25L89.52,169.25L91.147,169.25L92.773,169.25L94.4,169.25L96.027,169.25L97.653,169.25L99.28,169.25L100.907,169.25L102.533,169.25L104.16,169.25L105.787,169.25L107.413,169.25L109.04,169.25L110.667,169.25L112.293,169.25L113.92,169.25L115.547,169.25L117.173,169.25L118.8,169.25L120.427,169.25L122.053,169.25L123.68,169.25L125.307,169.25L126.933,169.25L128.56,169.25L130.187,169.25L131.813,169.25L133.44,169.25L135.067,219L136.693,219L138.32,219L139.947,219L141.573,219L143.2,219L144.827,219L146.453,219L148.08,219L149.707,219L151.333,219L152.96,219L154.587,219L156.213,219L157.84,219L159.467,219L161.093,219L162.72,219L164.347,219L165.973,219L167.6,169.25L169.227,169.25L170.853,169.25L172.48,169.25L174.107,169.25L175.733,169.25L177.36,169.25L178.987,169.25L180.613,169.25L182.24,169.25L183.867,169.25L185.493,169.25L187.12,169.25L188.747,169.25L190.373,169.25L192,169.25L193.627,169.25L195.253,169.25L196.88,169.25L198.507,169.25L200.133,169.25L201.76,169.25L203.387,169.25L205.013,169.25L206.64,169.25L208.267,169.25L209.893,169.25L211.52,169.25L213.147,169.25L214.773,169.25L216.4,169.25L218.027,169.25L219.653,169.25L221.28,169.25L222.907,169.25L224.533,169.25L226.16,169.25L227.787,169.25L229.413,169.25L231.04,169.25L232.667,219L234.293,219L235.92,219L237.547,219L239.173,219L240.8,219L242.427,219L244.053,219L245.68,219L247.307,219L248.933,219L250.56,219L252.187,219L253.813,219L255.44,219L257.067,219L258.693,219L260.32,219L261.947,219L263.573,219L265.2,219L266.827,219L268.453,219L270.08,219L271.707,219L273.333,219L274.96,219L276.587,219L278.213,219L279.84,219L281.467,219L283.093,219L284.72,219L286.347,219L287.973,219L289.6,219L291.227,219L292.853,219L294.48,219L296.107,219L297.733,219L299.36,219L300.987,219L302.613,219L304.24,219L305.867,219L307.493,219L309.12,219L310.747,219L312.373,219L314,219L315.627,219L317.253,219L318.88,219L320.507,219L322.133,219L323.76,219L325.387,219L327.013,219L328.64,219L330.267,219L331.893,219L333.52,219L335.147,219L336.773,219L338.4,219L340.027,219L341.653,219L343.28,219L344.907,219L346.533,219L348.16,219L349.787,219L351.413,219L353.04,219L354.667,219L356.293,219L357.92,219L359.547,219L361.173,219L362.8,219L364.427,219L366.053,219L367.68,219L369.307,219L370.933,219L372.56,219L374.187,219L375.813,219L377.44,219L379.067,219L380.693,219L382.32,219L383.947,219L385.573,219L387.2,219L388.827,219L390.453,219L392.08,219L393.707,219L395.333,219L396.96,219L398.587,219L400.213,219L401.84,219L403.467,219L405.093,219L406.72,219L408.347,219L409.973,219L411.6,219L413.227,219L414.853,219L416.48,219L418.107,219L419.733,219L421.36,219L422.987,219L424.613,219L426.24,219L427.867,219L429.493,219L431.12,219L432.747,219L434.373,219L436,219L437.627,219L439.253,219L440.88,219L442.507,219L444.133,219L445.76,219L447.387,219L449.013,219L450.64,219L452.267,219L453.893,219L455.52,219L457.147,219L458.773,219L460.4,219L462.027,219L463.653,219L465.28,219L466.907,219L468.533,219L470.16,219L471.787,219L473.413,219L475.04,219L476.667,219L478.293,219L479.92,219L481.547,219L483.173,219L484.8,219L486.427,219L488.053,219L489.68,219L491.307,219L492.933,219L494.56,219L496.187,219L497.813,219L499.44,219L501.067,219L502.693,219L504.32,219L505.947,219L507.573,219L509.2,219L510.827,219L512.453,219L514.08,219L515.707,219L517.333,219L518.96,219L520.587,219L522.213,219L523.84,219L525.467,219L527.093,219L528.72,219L530.347,219L531.973,219L533.6,219L535.227,219L536.853,219L538.48,219L540.107,219L541.733,219L543.36,219L544.987,219L546.613,219L548.24,219L549.867,219L551.493,219L553.12,219L554.747,219L556.373,219L558,219" class="ct-line"></path></g></g><g class="ct-labels"><foreignObject style="overflow: visible;" x="70" y="224" width="162.66666666666666" height="20"><span class="ct-label ct-horizontal ct-end" style="width: 163px; height: 20px" xmlns="http://www.w3.org/1999/xhtml">0</span></foreignObject><foreignObject style="overflow: visible;" x="232.66666666666666" y="224" width="162.66666666666666" height="20"><span class="ct-label ct-horizontal ct-end" style="width: 163px; height: 20px" xmlns="http://www.w3.org/1999/xhtml">1</span></foreignObject><foreignObject style="overflow: visible;" x="395.3333333333333" y="224" width="162.66666666666669" height="20"><span class="ct-label ct-horizontal ct-end" style="width: 163px; height: 20px" xmlns="http://www.w3.org/1999/xhtml">2</span></foreignObject><foreignObject style="overflow: visible;" x="558" y="224" width="30" height="20"><span class="ct-label ct-horizontal ct-end" style="width: 30px; height: 20px" xmlns="http://www.w3.org/1999/xhtml">3</span></foreignObject><foreignObject style="overflow: visible;" y="194.125" x="30" height="24.875" width="30"><span class="ct-label ct-vertical ct-start" style="height: 25px; width: 30px" xmlns="http://www.w3.org/1999/xhtml">0</span></foreignObject><foreignObject style="overflow: visible;" y="169.25" x="30" height="24.875" width="30"><span class="ct-label ct-vertical ct-start" style="height: 25px; width: 30px" xmlns="http://www.w3.org/1999/xhtml">0.125</span></foreignObject><foreignObject style="overflow: visible;" y="144.375" x="30" height="24.875" width="30"><span class="ct-label ct-vertical ct-start" style="height: 25px; width: 30px" xmlns="http://www.w3.org/1999/xhtml">0.25</span></foreignObject><foreignObject style="overflow: visible;" y="119.5" x="30" height="24.875" width="30"><span class="ct-label ct-vertical ct-start" style="height: 25px; width: 30px" xmlns="http://www.w3.org/1999/xhtml">0.375</span></foreignObject><foreignObject style="overflow: visible;" y="94.625" x="30" height="24.875" width="30"><span class="ct-label ct-vertical ct-start" style="height: 25px; width: 30px" xmlns="http://www.w3.org/1999/xhtml">0.5</span></foreignObject><foreignObject style="overflow: visible;" y="69.75" x="30" height="24.875" width="30"><span class="ct-label ct-vertical ct-start" style="height: 25px; width: 30px" xmlns="http://www.w3.org/1999/xhtml">0.625</span></foreignObject><foreignObject style="overflow: visible;" y="44.875" x="30" height="24.875" width="30"><span class="ct-label ct-vertical ct-start" style="height: 25px; width: 30px" xmlns="http://www.w3.org/1999/xhtml">0.75</span></foreignObject><foreignObject style="overflow: visible;" y="20" x="30" height="24.875" width="30"><span class="ct-label ct-vertical ct-start" style="height: 25px; width: 30px" xmlns="http://www.w3.org/1999/xhtml">0.875</span></foreignObject><foreignObject style="overflow: visible;" y="-10" x="30" height="30" width="30"><span class="ct-label ct-vertical ct-start" style="height: 30px; width: 30px" xmlns="http://www.w3.org/1999/xhtml">1</span></foreignObject></g></svg></div></div>


要创建缓冲区，我们需要使用Web Audio的createBuffer方法。我们将在Tone名为的对象的新方法中创建并填充缓冲区createRingerLFO。
```
Tone.prototype.createRingerLFO = function() {
    // Create an empty 3 second mono buffer at the
    // sample rate of the AudioContext
    var channels = 1;
    var sampleRate = this.context.sampleRate;
    var frameCount = sampleRate * 3;
    var arrayBuffer = this.context.createBuffer(channels, frameCount, sampleRate);

    // getChannelData allows us to access and edit the buffer data and change.
    var bufferData = arrayBuffer.getChannelData(0);
    for (var i = 0; i < frameCount; i++) {
        // if the sample lies between 0 and 0.4 seconds, or 0.6 and 1 second, we want it to be on.
        if ((i/sampleRate > 0 && i/sampleRate < 0.4) || (i/sampleRate > 0.6 && i/sampleRate < 1.0)){
            bufferData[i] = 0.25;
        }
    }

    this.ringerLFOBuffer = arrayBuffer;
}
```
现在，我们已经创建了音频数据并将其存储在缓冲区中，可以将其加载到缓冲区源节点中以完成LFO。完成后，我们需要将LFO连接到增益节点的增益参数，然后开始LFO播放以获得所需的效果。

```
Tone.prototype.startRinging = function(){
    this.start();
    // set our gain node to 0, because the LFO is calibrated to this level
    this.gainNode.gain.value = 0;
    this.status = 1;

    this.createRingerLFO();

    this.ringerLFOSource = this.context.createBufferSource();
    this.ringerLFOSource.buffer = this.ringerLFOBuffer;
    this.ringerLFOSource.loop = true;
    // connect the ringerLFOSource to the gain Node audio param
    this.ringerLFOSource.connect(this.gainNode.gain);
    this.ringerLFOSource.start(0);
}
```

我们还将创建一个stopRinging调用该stop方法的方法，并停止播放LFO。

```
Tone.prototype.stopRinging = function(){
    this.stop();
    this.ringerLFOSource.stop(0);
}
```
最后，就像拨号音一样，我们将创建一个切换按钮，一个新对象的实例，并设置一个事件侦听器，该侦听器将使我们可以轻松地开始和停止响铃。
```
<button id='js-toggle-ringback-tone'>Toggle Dial Tone</button>

<script>
    var context = new AudioContext();
    var ringbackTone = new Tone(context, 400, 450);

    $("#js-toggle-ringback-tone").click(function(e){
        e.preventDefault();
        if (ringbackTone.status === 0){
            ringbackTone.startRinging();
        } else {
            ringbackTone.stopRinging();
        }
    });
</script>
```
下面的是该部分的demo实现。


<p class="codepen" data-height="265" data-theme-id="light" data-default-tab="js,result" data-user="edball" data-slug-hash="ojVgGq" style="height: 265px; box-sizing: border-box; display: flex; align-items: center; justify-content: center; border: 2px solid; margin: 1em 0; padding: 1em;" data-pen-title="Ringback tone in Web Audio">
  <span>See the Pen <a href="https://codepen.io/edball/pen/ojVgGq">
  Ringback tone in Web Audio</a> by Ed Ball (<a href="https://codepen.io/edball">@edball</a>)
  on <a href="https://codepen.io">CodePen</a>.</span>
</p>
<script async src="https://static.codepen.io/assets/embed/ei.js"></script>



# 最终实现

根据以上，我实现了以下代码用来完成各个按键的拨号音：

```
const DtmfFrequencies = {
    "1": {f1: 697, f2: 1209},
    "2": {f1: 697, f2: 1336},
    "3": {f1: 697, f2: 1477},
    "4": {f1: 770, f2: 1209},
    "5": {f1: 770, f2: 1336},
    "6": {f1: 770, f2: 1477},
    "7": {f1: 852, f2: 1209},
    "8": {f1: 852, f2: 1336},
    "9": {f1: 852, f2: 1477},
    "*": {f1: 941, f2: 1209},
    "0": {f1: 941, f2: 1336},
    "#": {f1: 941, f2: 1477}
}

export default class Tone {
  constructor () {
    try {
      this.context = new AudioContext()
    } catch (e) {
      console.error(e)
      return
    }
    this.status = 0
    this.osc1 = this.context.createOscillator()
    this.osc2 = this.context.createOscillator()
    // 增益
    this.gainNode = this.context.createGain()
    this.gainNode.gain.setValueAtTime(0.25, this.context.currentTime)
    // 低通滤波
    this.filter = this.context.createBiquadFilter();
    this.filter.type = "lowpass";
    this.filter.frequency.setValueAtTime(8000, this.context.currentTime)
    // 连接
    this.osc1.connect(this.gainNode)
    this.osc2.connect(this.gainNode)
    this.gainNode.connect(this.filter)
    this.osc1.start(0)
    this.osc2.start(0)

    document.addEventListener('mouseup', this.__stop)
  }
  destroy () {
    this.osc1.stop(0)
    this.osc2.stop(0)
    this.__stop()
    document.removeEventListener('mouseup', this.__stop)
  }
  __start () {
    console.log(this.status, 'start')
    if (!this.status) {
      this.filter.connect(this.context.destination)
      this.status = 1
      setTimeout(() => {
        this.__stop()
      }, 200)
    }
  }
  __stop = () => {
    console.log(this.status, 'stop')
    if (this.status) {
      this.filter.disconnect(this.context.destination)
      this.status = 0
    }
  }
  ring (key) {
    let f = DtmfFrequencies[key]
    this.osc1.frequency.setValueAtTime(f.f1, this.context.currentTime)
    this.osc2.frequency.setValueAtTime(f.f2, this.context.currentTime)
    this.__start()
  }
}
```


实现中遇到了以下几个问题：

1. 一个AudioContext不能重复地调用start()，如有需要，可以调用suspend方法和resume方法。或disconnect destination实现。
2. chome禁止用户在未与document交互的时候播放音频。需要把对应的方法放在点击事件里面调用，或者引导用户与document进行交互。

原文链接：

http://outputchannel.com/post/recreating-phone-sounds-web-audio/

<style type="text/css">
.ct-double-octave:after,.ct-major-eleventh:after,.ct-major-second:after,.ct-major-seventh:after,.ct-major-sixth:after,.ct-major-tenth:after,.ct-major-third:after,.ct-major-twelfth:after,.ct-minor-second:after,.ct-minor-seventh:after,.ct-minor-sixth:after,.ct-minor-third:after,.ct-octave:after,.ct-perfect-fifth:after,.ct-perfect-fourth:after,.ct-square:after{content:"";clear:both}.ct-label{fill:rgba(0,0,0,0.4);color:rgba(0,0,0,0.4);font-size:.75rem;line-height:1}.ct-chart-bar .ct-label,.ct-chart-line .ct-label{display:block;display:-webkit-box;display:-moz-box;display:-ms-flexbox;display:-webkit-flex;display:flex}.ct-label.ct-horizontal.ct-start{-webkit-box-align:flex-end;-webkit-align-items:flex-end;-ms-flex-align:flex-end;align-items:flex-end;-webkit-box-pack:flex-start;-webkit-justify-content:flex-start;-ms-flex-pack:flex-start;justify-content:flex-start;text-align:left;text-anchor:start}.ct-label.ct-horizontal.ct-end{-webkit-box-align:flex-start;-webkit-align-items:flex-start;-ms-flex-align:flex-start;align-items:flex-start;-webkit-box-pack:flex-start;-webkit-justify-content:flex-start;-ms-flex-pack:flex-start;justify-content:flex-start;text-align:left;text-anchor:start}.ct-label.ct-vertical.ct-start{-webkit-box-align:flex-end;-webkit-align-items:flex-end;-ms-flex-align:flex-end;align-items:flex-end;-webkit-box-pack:flex-end;-webkit-justify-content:flex-end;-ms-flex-pack:flex-end;justify-content:flex-end;text-align:right;text-anchor:end}.ct-label.ct-vertical.ct-end{-webkit-box-align:flex-end;-webkit-align-items:flex-end;-ms-flex-align:flex-end;align-items:flex-end;-webkit-box-pack:flex-start;-webkit-justify-content:flex-start;-ms-flex-pack:flex-start;justify-content:flex-start;text-align:left;text-anchor:start}.ct-chart-bar .ct-label.ct-horizontal.ct-start{-webkit-box-align:flex-end;-webkit-align-items:flex-end;-ms-flex-align:flex-end;align-items:flex-end;-webkit-box-pack:center;-webkit-justify-content:center;-ms-flex-pack:center;justify-content:center;text-align:center;text-anchor:start}.ct-chart-bar .ct-label.ct-horizontal.ct-end{-webkit-box-align:flex-start;-webkit-align-items:flex-start;-ms-flex-align:flex-start;align-items:flex-start;-webkit-box-pack:center;-webkit-justify-content:center;-ms-flex-pack:center;justify-content:center;text-align:center;text-anchor:start}.ct-chart-bar.ct-horizontal-bars .ct-label.ct-horizontal.ct-start{-webkit-box-align:flex-end;-webkit-align-items:flex-end;-ms-flex-align:flex-end;align-items:flex-end;-webkit-box-pack:flex-start;-webkit-justify-content:flex-start;-ms-flex-pack:flex-start;justify-content:flex-start;text-align:left;text-anchor:start}.ct-chart-bar.ct-horizontal-bars .ct-label.ct-horizontal.ct-end{-webkit-box-align:flex-start;-webkit-align-items:flex-start;-ms-flex-align:flex-start;align-items:flex-start;-webkit-box-pack:flex-start;-webkit-justify-content:flex-start;-ms-flex-pack:flex-start;justify-content:flex-start;text-align:left;text-anchor:start}.ct-chart-bar.ct-horizontal-bars .ct-label.ct-vertical.ct-start{-webkit-box-align:center;-webkit-align-items:center;-ms-flex-align:center;align-items:center;-webkit-box-pack:flex-end;-webkit-justify-content:flex-end;-ms-flex-pack:flex-end;justify-content:flex-end;text-align:right;text-anchor:end}.ct-chart-bar.ct-horizontal-bars .ct-label.ct-vertical.ct-end{-webkit-box-align:center;-webkit-align-items:center;-ms-flex-align:center;align-items:center;-webkit-box-pack:flex-start;-webkit-justify-content:flex-start;-ms-flex-pack:flex-start;justify-content:flex-start;text-align:left;text-anchor:end}.ct-grid{stroke:rgba(0,0,0,0.2);stroke-width:1px;stroke-dasharray:2px}.ct-point{stroke-width:10px;stroke-linecap:round}.ct-line{fill:none;stroke-width:4px}.ct-area{stroke:none;fill-opacity:.1}.ct-bar{fill:none;stroke-width:10px}.ct-slice-donut{fill:none;stroke-width:60px}.ct-series-a .ct-bar,.ct-series-a .ct-line,.ct-series-a .ct-point,.ct-series-a .ct-slice-donut{stroke:#d70206}.ct-series-a .ct-area,.ct-series-a .ct-slice-pie{fill:#d70206}.ct-series-b .ct-bar,.ct-series-b .ct-line,.ct-series-b .ct-point,.ct-series-b .ct-slice-donut{stroke:#f05b4f}.ct-series-b .ct-area,.ct-series-b .ct-slice-pie{fill:#f05b4f}.ct-series-c .ct-bar,.ct-series-c .ct-line,.ct-series-c .ct-point,.ct-series-c .ct-slice-donut{stroke:#f4c63d}.ct-series-c .ct-area,.ct-series-c .ct-slice-pie{fill:#f4c63d}.ct-series-d .ct-bar,.ct-series-d .ct-line,.ct-series-d .ct-point,.ct-series-d .ct-slice-donut{stroke:#d17905}.ct-series-d .ct-area,.ct-series-d .ct-slice-pie{fill:#d17905}.ct-series-e .ct-bar,.ct-series-e .ct-line,.ct-series-e .ct-point,.ct-series-e .ct-slice-donut{stroke:#453d3f}.ct-series-e .ct-area,.ct-series-e .ct-slice-pie{fill:#453d3f}.ct-series-f .ct-bar,.ct-series-f .ct-line,.ct-series-f .ct-point,.ct-series-f .ct-slice-donut{stroke:#59922b}.ct-series-f .ct-area,.ct-series-f .ct-slice-pie{fill:#59922b}.ct-series-g .ct-bar,.ct-series-g .ct-line,.ct-series-g .ct-point,.ct-series-g .ct-slice-donut{stroke:#0544d3}.ct-series-g .ct-area,.ct-series-g .ct-slice-pie{fill:#0544d3}.ct-series-h .ct-bar,.ct-series-h .ct-line,.ct-series-h .ct-point,.ct-series-h .ct-slice-donut{stroke:#6b0392}.ct-series-h .ct-area,.ct-series-h .ct-slice-pie{fill:#6b0392}.ct-series-i .ct-bar,.ct-series-i .ct-line,.ct-series-i .ct-point,.ct-series-i .ct-slice-donut{stroke:#f05b4f}.ct-series-i .ct-area,.ct-series-i .ct-slice-pie{fill:#f05b4f}.ct-series-j .ct-bar,.ct-series-j .ct-line,.ct-series-j .ct-point,.ct-series-j .ct-slice-donut{stroke:#dda458}.ct-series-j .ct-area,.ct-series-j .ct-slice-pie{fill:#dda458}.ct-series-k .ct-bar,.ct-series-k .ct-line,.ct-series-k .ct-point,.ct-series-k .ct-slice-donut{stroke:#eacf7d}.ct-series-k .ct-area,.ct-series-k .ct-slice-pie{fill:#eacf7d}.ct-series-l .ct-bar,.ct-series-l .ct-line,.ct-series-l .ct-point,.ct-series-l .ct-slice-donut{stroke:#86797d}.ct-series-l .ct-area,.ct-series-l .ct-slice-pie{fill:#86797d}.ct-series-m .ct-bar,.ct-series-m .ct-line,.ct-series-m .ct-point,.ct-series-m .ct-slice-donut{stroke:#b2c326}.ct-series-m .ct-area,.ct-series-m .ct-slice-pie{fill:#b2c326}.ct-series-n .ct-bar,.ct-series-n .ct-line,.ct-series-n .ct-point,.ct-series-n .ct-slice-donut{stroke:#6188e2}.ct-series-n .ct-area,.ct-series-n .ct-slice-pie{fill:#6188e2}.ct-series-o .ct-bar,.ct-series-o .ct-line,.ct-series-o .ct-point,.ct-series-o .ct-slice-donut{stroke:#a748ca}.ct-series-o .ct-area,.ct-series-o .ct-slice-pie{fill:#a748ca}.ct-square{display:block;position:relative;width:100%}.ct-square:before{display:block;float:left;content:"";width:0;height:0;padding-bottom:100%}.ct-square:after{display:table}.ct-square>svg{display:block;position:absolute;top:0;left:0}.ct-minor-second{display:block;position:relative;width:100%}.ct-minor-second:before{display:block;float:left;content:"";width:0;height:0;padding-bottom:93.75%}.ct-minor-second:after{display:table}.ct-minor-second>svg{display:block;position:absolute;top:0;left:0}.ct-major-second{display:block;position:relative;width:100%}.ct-major-second:before{display:block;float:left;content:"";width:0;height:0;padding-bottom:88.8888888889%}.ct-major-second:after{display:table}.ct-major-second>svg{display:block;position:absolute;top:0;left:0}.ct-minor-third{display:block;position:relative;width:100%}.ct-minor-third:before{display:block;float:left;content:"";width:0;height:0;padding-bottom:83.3333333333%}.ct-minor-third:after{display:table}.ct-minor-third>svg{display:block;position:absolute;top:0;left:0}.ct-major-third{display:block;position:relative;width:100%}.ct-major-third:before{display:block;float:left;content:"";width:0;height:0;padding-bottom:80%}.ct-major-third:after{display:table}.ct-major-third>svg{display:block;position:absolute;top:0;left:0}.ct-perfect-fourth{display:block;position:relative;width:100%}.ct-perfect-fourth:before{display:block;float:left;content:"";width:0;height:0;padding-bottom:75%}.ct-perfect-fourth:after{display:table}.ct-perfect-fourth>svg{display:block;position:absolute;top:0;left:0}.ct-perfect-fifth{display:block;position:relative;width:100%}.ct-perfect-fifth:before{display:block;float:left;content:"";width:0;height:0;padding-bottom:66.6666666667%}.ct-perfect-fifth:after{display:table}.ct-perfect-fifth>svg{display:block;position:absolute;top:0;left:0}.ct-minor-sixth{display:block;position:relative;width:100%}.ct-minor-sixth:before{display:block;float:left;content:"";width:0;height:0;padding-bottom:62.5%}.ct-minor-sixth:after{display:table}.ct-minor-sixth>svg{display:block;position:absolute;top:0;left:0}.ct-golden-section{display:block;position:relative;width:100%}.ct-golden-section:before{display:block;float:left;content:"";width:0;height:0;padding-bottom:61.804697157%}.ct-golden-section:after{content:"";display:table;clear:both}.ct-golden-section>svg{display:block;position:absolute;top:0;left:0}.ct-major-sixth{display:block;position:relative;width:100%}.ct-major-sixth:before{display:block;float:left;content:"";width:0;height:0;padding-bottom:60%}.ct-major-sixth:after{display:table}.ct-major-sixth>svg{display:block;position:absolute;top:0;left:0}.ct-minor-seventh{display:block;position:relative;width:100%}.ct-minor-seventh:before{display:block;float:left;content:"";width:0;height:0;padding-bottom:56.25%}.ct-minor-seventh:after{display:table}.ct-minor-seventh>svg{display:block;position:absolute;top:0;left:0}.ct-major-seventh{display:block;position:relative;width:100%}.ct-major-seventh:before{display:block;float:left;content:"";width:0;height:0;padding-bottom:53.3333333333%}.ct-major-seventh:after{display:table}.ct-major-seventh>svg{display:block;position:absolute;top:0;left:0}.ct-octave{display:block;position:relative;width:100%}.ct-octave:before{display:block;float:left;content:"";width:0;height:0;padding-bottom:50%}.ct-octave:after{display:table}.ct-octave>svg{display:block;position:absolute;top:0;left:0}.ct-major-tenth{display:block;position:relative;width:100%}.ct-major-tenth:before{display:block;float:left;content:"";width:0;height:0;padding-bottom:40%}.ct-major-tenth:after{display:table}.ct-major-tenth>svg{display:block;position:absolute;top:0;left:0}.ct-major-eleventh{display:block;position:relative;width:100%}.ct-major-eleventh:before{display:block;float:left;content:"";width:0;height:0;padding-bottom:37.5%}.ct-major-eleventh:after{display:table}.ct-major-eleventh>svg{display:block;position:absolute;top:0;left:0}.ct-major-twelfth{display:block;position:relative;width:100%}.ct-major-twelfth:before{display:block;float:left;content:"";width:0;height:0;padding-bottom:33.3333333333%}.ct-major-twelfth:after{display:table}.ct-major-twelfth>svg{display:block;position:absolute;top:0;left:0}.ct-double-octave{display:block;position:relative;width:100%}.ct-double-octave:before{display:block;float:left;content:"";width:0;height:0;padding-bottom:25%}.ct-double-octave:after{display:table}.ct-double-octave>svg{display:block;position:absolute;top:0;left:0}.ct-line{stroke-width:2px}.ct-series-a .ct-line,.ct-series-a .ct-point{stroke:#f60}.ct-series-b .ct-line,.ct-series-b .ct-point{stroke:#55D43F}

.ct-octave:before {
  display: block;
  float: left;
  content: '';
  width: 0;
  height: 0;
  padding-bottom: 50%;
}
.ct-grid {
  stroke: rgba(0, 0, 0, 0.2);
  stroke-width: 1px;
  stroke-dasharray: 2px;
}
.ct-series-a .ct-line, .ct-series-a .ct-point {
  stroke: #f60;
}
.ct-octave:after {
  display: table;
}
.ct-double-octave:after, .ct-major-eleventh:after, .ct-major-second:after, .ct-major-seventh:after, .ct-major-sixth:after, .ct-major-tenth:after, .ct-major-third:after, .ct-major-twelfth:after, .ct-minor-second:after, .ct-minor-seventh:after, .ct-minor-sixth:after, .ct-minor-third:after, .ct-octave:after, .ct-perfect-fifth:after, .ct-perfect-fourth:after, .ct-square:after {
  content: '';
  clear: both;
}
</style>
