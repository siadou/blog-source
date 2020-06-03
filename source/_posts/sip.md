---
title: SIP.js实现Web电话
date: 2020-06-03 15:11:49
tags:
---

最近在做Web电话相关的开发。Web端软电与后台的FreeSwitch通信实现了软电话的所有功能，通过websocket交换SIP报文，建立WebRTC流媒体连接，实现点对点的语音、视频和文字通信。
目前实现SIP Phone的工具包括Sip.js和JSSip两个库。对比了文档，本次使用的库是Sip.js。

# 调研问题

1 使用范围：由于现在所有的实现都是基于WEBRTC，所以只有支持WEBRTC功能的浏览器才能使用。适用范围在CAN I USE上查找如下：

![can i use](/images/sip/caniuse.png)

2 一个浏览器窗口就是一台分机，如果打开多个窗口自动登录多个分机，体验不好。实际实现的时候只保证一台分机在线，检测当前活动窗口，注销其他窗口。窗口和窗口之间通过localStorage通信。

# 简介

Sip.js提供了三种开发的接口：
1. SimpleUser 类

基于API框架封装的简单类，只提供基本功能。

2. API框架

对CoreLibrary进行的封装。为大多数开发者提供的调用API框架。相比于SimpleUSer提供更多的配置项。支持音频、视频、文本多媒体实时传输。（作者在做相关功能的时候此接口尚在实验阶段，在0.16.0版本里已正式发布。）

相关代码详见SIP.js文档：https://github.com/onsip/SIP.js/blob/master/docs/api.md

3. Core Library

提供了最底层的单元接口。

SimpleUSer的配置可能不够用，CoreLibrary需要用户掌握更底层的协议相关知识。一般开发建议使用API框架层级接口完成。


# 遇到的问题：

1. 听不到对面的振铃音：

理论上来讲电话未拨通的时候，应该能听到对面的振铃音(early media),其他情况包括电话未接通的unavailable提示音（对不起，你拨打的用户XXXX）。但是最开始测试的时候都听不到。

后来查了下官方文档一个隐蔽的角落，SIP.js只支持带SDP的INVITE请求携带early media。free switch默认的配置是183 session progress报文携带SDP，这种情况SIP.js是处理不了的。所以服务器和客户端必须同时开启100 rel配置才可以。代价是为了听到回铃音，free switch必须配置提前应答，提供回铃音。对于客户端来讲通话建立时间和实际通话建立时间不一样。计算时间的时候和预期不符合。（但是又改不了）

官方还提供了一种inviteWithoutSdp的配置，由于服务端改动较大，没被采纳。后来只用了100 rel的这种方法。

2. add track的时候chrome报错

> Uncaught (in promise) DOMException: The play() request was interrupted by a call to pause().

报错的是以下这段代码：


``` 
        this.remoteVideo.pause()
        this.remoteVideo.srcObject = remoteStream;
        this.remoteVideo.load()
        this.remoteVideo.play()
```

查询了下video.play()是一个异步操作，返回一个promise。在播放未结束的时候promise没有被resolve，此时调用pause就会报错。

后来修改成了这个样子：

```
      if (this.lastVideo) {
        this.lastVideo.then(() => {
          this.remoteVideo.srcObject = remoteStream;
          this.remoteVideo.load()
          this.lastVideo = this.remoteVideo.play();
        })
      } else {
        this.remoteVideo.srcObject = remoteStream;
        this.remoteVideo.load()
        this.lastVideo = this.remoteVideo.play();
      }
```

 
3. 注销登录没有收到服务器的回复

注销登录的时候，客户端向free switch服务器发送一个register报文。然而并没有收到free switch的回复。也没有触发user agent的unregiter事件。

发送的报文里将注册用户的expire过期时间置零。（如图）。实际测试的时候客户端确实已经登出（收不到电话）。此次收不到回复报文没有触发user agent的unregister事件不影响功能。

![unregister](/images/sip/unregister.png)

4. 建立通话前卡顿数秒

最开始默认配置实验的时候实验A与B进行通信。在B接起电话之后，两边会等待若干秒才建立连接。同时控制台log输出：ICEChecking TIMEOUT after 5000ms。这个5000ms大概就是两边等待的时间。情况时好时坏。

我后来查询了ICE媒体连接建立的原理，虽然本应用场景实际是client浏览器free switch server建立连接，但WebRTC更多的场景是点对点连接，需要通过STUN协议穿透NAT来和通信的客户端建立连接。在建立连接前对通过访问一个公网的STUN服务器来获取本机地址。SIP.js默认的STUN服务器是谷歌的服务器。由于DNS解析出来的地址不固定，有的能访问通畅，有的不可以，所以连接时好时坏。

后来经历了一番波折（google直接查询到的是以前版本的API），通过将STUN服务器(iceServers)置空解决了问题。由于主要的使用场景是内网中可直连的设备，所以不需要获取NAT网络以外的地址进行连接。这样连接速度大大的提升了。

UA初始化的option:

```
      let option = {
        rel100: SIP.C.supported.SUPPORTED,
        transportOptions: {
          wsServers: [ config.server ],
        },
        uri, 
        authorizationUser,
        displayName,
        contactName: displayName,
        password: config.password,
        autostart:true,
        register: true,
        dtmfType: SIP.C.dtmfType.RTP,
        sessionDescriptionHandlerFactoryOptions: {
          constraints: {
            audio: true,
            video: false
          },
          peerConnectionOptions: {
            rtcConfiguration: {
              iceServers: []
            },
            iceCheckingTimeout: 5000
          }
        },
        log: {level: 0}
    }
```

最终电话类的实现（Sip.js 版本:0.15.11，不包括UI）

```
<template>
  <div>
    <audio id="remoteVideo" autoplay controls hidden preload="auto">不支持audio</audio>
    <audio id="localVideo" autoplay controls hidden preload="auto">不支持audio</audio>
  </div>
</template>
<script>
const SIP = require('sip.js')

export default {
  name: 'WebPhone',
  props: {
    config: {
      default: () => {
        return {
          server: '',
          username: '',
          password: ''
        }
      },
      type: Object
    },
    enableVolume: {
      default: true,
      type: Boolean
    }
  },
  data() {
    return {
      isRegister: 0, // 1 : success -1: failed -3 logout
      mediaSupport: true,

      callSuffix: '',
      phone: null,
      session: null,
      remoteVideo: null,
      localVideo: null,

      loginPromise: {
        resolve: () => {},
        reject: () => {}
      },
      logoutPromise: {
        resolve: () => {},
        reject: () => {}
      }
    }
  },
  mounted () {
    this.remoteVideo = document.getElementById('remoteVideo')
    this.localVideo = document.getElementById('localVideo')
  },
  methods: {
    checkMedia () {
      navigator.getUserMedia  = navigator.getUserMedia ||
                          navigator.webkitGetUserMedia ||
                          navigator.mozGetUserMedia ||
                          navigator.msGetUserMedia;
      if (navigator.getUserMedia) {
        this.mediaSupport = true
    // 支持
      } else {
    // 不支持
        this.mediaSupport = false
      }
      return this.mediaSupport
    },
    getOptions() {
      return {}
      /*
      let options = {
        // inviteWithoutSdp: true,
        // rel100: SIP.C.supported.SUPPORTED,
        sessionDescriptionHandlerOptions: {
          constraints: {
            audio: true,
            video: false
          }
        }
      }
      return options ;
      */
    },
    register (cfg) {
      const config = cfg || this.config
      let serverPort = config.server.split('//')[1]
      this.callSuffix = `@${serverPort}`
      // let [ server, port ] = serverPort.split(':')
      let server = serverPort.split(':')[0]
      let notHasHost = config.username.indexOf('@') === -1
      let uri, authorizationUser, displayName
      if (notHasHost) {
        uri = `${config.username}@${server}`
        authorizationUser = config.username
        displayName = config.username
      } else {
        uri = config.username 
        authorizationUser = config.username.split('@')[0]
        displayName = authorizationUser
      }
      let option = {
        rel100: SIP.C.supported.SUPPORTED,
        transportOptions: {
          wsServers: [ config.server ],
        },
        stunServers: [{ url: 'stun:stun.services.mozilla.com'}],
        uri: uri, 
        authorizationUser,
        displayName,
        contactName: displayName,
        password: config.password,
        autostart:true,
        register: true,
        dtmfType: SIP.C.dtmfType.RTP,
        sessionDescriptionHandlerFactoryOptions: {
          constraints: {
            audio: true,
            video: false
          },
          peerConnectionOptions: {
            rtcConfiguration: {
              iceServers: []
            },
            iceCheckingTimeout: 5000
          }
        },
        // viaHost: '10.123.192.105',
        // realm: '10.123.192.105',
        // contactName: '1001',
        log: {level: 0}
      }
      try {
        this.phone = new SIP.UA(option)
        // this.phone = UserAgent(new UserAgentOptions(option))
        this.bind()
        return new Promise((resolve, reject) => {
          this.loginPromise.resolve = resolve
          this.loginPromise.reject = reject
        })
      } catch (e) {
        this.isRegister = -1
        this.$emit('login-failed', 'config')
        console.warn('CONFIG ERROR', e)
        return Promise.reject('config', e)
      }
    },
    logout () {
      let options = {
        'all': true,
      };
      if (this.phone) {
        this.phone.stop()
        this.phone.unregister(options)
        return new Promise((resolve) => {
          this.$emit('unregistered')
          this.logoutPromise.resolve()
          this._resetPromise(this.logoutPromise)
          console.warn('unregistered')
          resolve()
        })
      } else {
        return Promise.resolve()
      }
    },
    hangup () {
      let reason = 'bye'
      if (this.session) {
        if (this.session.hasAnswer){
          this.session.bye()
        } else if (this.session.isCanceled == false){
          reason = 'cancel'
          // outbound connection is not built
          this.session.cancel()
        } else {
          reason = 'reject'
          // inbound connection is not built
          this.session.reject()
        }
      }
      this.$emit('hangup', reason)
      // this.$emit('post-message', 'busy-off')
      // this.statusHangup()
    },
    sendDtmf (number) {
      this.session.dtmf(number)
    },
    call (number) {
      // 呼叫
      try {
        this.session = this.phone.invite(number + this.callSuffix, this.getOptions())
        // this.statusDialing()
        // this.startRingBack()
        this.bindSessionEvent() //拨打电话
        this.$emit('call')
      } catch (e) {
        console.log(e)
      }
    },
    answer () {
      // 接听
      this.session.accept(this.getOptions())
      this.$emit('answer')
    },
    _resetPromise (prms) {
      prms.resolve = () => {}
      prms.reject = () => {}
    },
    bind () {
      this.phone.transport.on('transportError', (e) => {
        this.isRegister = -1
        this.$emit('login-failed', 'network')
        console.warn('TRANSPORT ERROR', e); // gets called 2x per try (with e === undefined)
        this.loginPromise.reject('newtork', e)
        this._resetPromise(this.loginPromise)
      });
      this.phone.transport.on('disconnected', (e) => {
        this.$emit('login-failed', 'network')
        console.log('DISCONNECTED', e);  // gets called 1x per try with e of {code: 1006, reason: ""}
        this.loginPromise.reject('newtork', e)
        this._resetPromise(this.loginPromise)
      });

      this.phone.on('invite', (session) => {
        this.session = session
        this.$emit('ring', this.session.request.from.uri.user)
        this.bindSessionEvent() //接听电话
      })
      this.phone.on('registered', (e) => {
        this.$emit('registered')
        console.warn('registered', e)
        this.isRegister = 1
        this.loginPromise.resolve()
        this._resetPromise(this.loginPromise)
      })
      this.phone.on('unregistered', (e) => {
        this.$emit('unregistered')
        this.logoutPromise.resolve()
        this._resetPromise(this.logoutPromise)
        console.warn('unregistered', e)
      })
      this.phone.on('registrationFailed', (e) => {
        this.$emit('login-failed', 'register')
        console.warn('login-failed', e)
        this.isRegister = -1
        this.loginPromise.reject('register', e)
        this._resetPromise(this.loginPromise)
      })
    },
    bindSessionEvent() {
      this.session.on("progress" , (response) => {
        console.warn(response)
        this._appendMedia()
      })
      this.session.on("accepted" , (data) => {
        this._appendMedia()
        this.$emit('answer-receive', data)
      })
      this.session.on("rejected" , () => {
        this.remoteVideo.srcObject = null
        this.$emit('hangup-receive', 'rejected')
      })
      this.session.on("failed" , () => {
        this.remoteVideo.srcObject = null
        this.$emit('hangup', 'failed')
      })
      this.session.on("terminated" , () => {
        this.remoteVideo.srcObject = null
        this.$emit('hangup', 'terminated')
      })
      this.session.on("cancel" , () => {
        this.$emit('hangup', 'cancel')
      })

      this.session.on("bye" , () => {
        this.remoteVideo.srcObject = null
        this.$emit('hangup', 'bye')
      })
      this.session.on('dtmf', () => {
        this.$emit('dtmf')
      })

      // not in use
      this.session.on('trackAdded', () => {
        // We need to check the peer connection to determine which track was added
        console.warn('track-add')
        // this._appendMedia()
      });
      this.session.on('referRequested', (referServerContext) => {
        console.warn("refer-requested")
        referServerContext.accept();
      })
    },
    _appendMedia () {
      let pc = this.session.sessionDescriptionHandler.peerConnection;
      // Gets remote tracks
      let remoteStream = new MediaStream();
      pc.getReceivers().forEach(function(receiver) {
        remoteStream.addTrack(receiver.track);
      });
      console.log(remoteStream)
      if (this.lastVideo) {
        this.lastVideo.then(() => {
          this.remoteVideo.srcObject = remoteStream;
          this.remoteVideo.load()
          this.lastVideo = this.remoteVideo.play();
        })
      } else {
        this.remoteVideo.srcObject = remoteStream;
        this.remoteVideo.load()
        this.lastVideo = this.remoteVideo.play();
      }
    },
    _cleanMedia () {
      this.remoteVideo.srcObject = null
      this.remoteVideo.pause()
    }
  },
  destroyed() {
    this.remoteVideo = null
    this.localStream = null
    console.log("Destroyed")
    this.logout()
  }
}
</script>

```


参考文献：

1. [SIP.js](https://sipjs.com/)