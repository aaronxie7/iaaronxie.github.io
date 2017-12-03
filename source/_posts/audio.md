---
title: Web 语音识别
date: 2017-12-03 21:33:30
tags:
---

这篇文章不会考虑 IE 浏览器，原因在于浏览器的语音识别依赖于两个核心 API，分别为：`navigator.getUserMedia` 和 `AudioContext`。IE 并不支持这两个 API，如果想要兼容 IE，可以参考下[这篇文章](http://varyu.com/notes/293.html)。

在 Web 中，语音识别的实现方式基本上大同小异，基本上都会经历以下 4 个步骤：
- 获取浏览器麦克风权限
- 将用户使用麦克风说出的语音转换成实际的音频文件
- 将音频文件发送给服务器
- 服务器使用开源/自有语音识别库将音频文件解析成文字，并将结果返回给用户

## 获取浏览器麦克风权限
在浏览器中，我们使用 `getUserMedia` 方法来获取麦克风或摄像头权限，由于这个方法各浏览器实现可能会带前缀，并且这个方法原本是作为 `navigator` 的方法实现，后来又迁移到 `MediaDevices` 下，所以我们需要对这个方法做一些浏览器支持的校验，可以参考下面的代码：

```
/**
 * 获取用户设备
 * @param config 传递给原生 getUserMedia 的配置
 * @return {Object} 返回 promise
 * @description 先判断是否支持最新API，然后逐步降级去判断是否支持已废弃的API，如果都都不支持
 * 则直接提示
 */
function getUserMedia(config) {
  if (navigator.mediaDevices) {
    return navigator.mediaDevices.getUserMedia(config);
  }
  
  navigator.getUserMedia = navigator.getUserMedia ||
    navigator.webkitGetUserMedia ||
    navigator.mozGetUserMedia;
  
  if (navigator.getUserMedia) {
    return new Promise((resolve, reject) => {
      return navigator.getUserMedia(config, resolve, reject);
    });
  }

  return new Error('Browser not support getUserMedia');
}
```

## 将用户使用麦克风说出的语音转换成实际的音频文件
将语音转换成音频文件我们需要使用 `AudioContext` 获取上一步拿到的用户语音转换成媒体流对象，并使用 [recorder.js](https://github.com/mattdiamond/Recorderjs) 将媒体流对象转换成特定格式的音频文件。

```

// 使用用户语音来初始化 recorder 对象
let recorder = null;
getUserMedia({ audio: true }).then(stream => {
  let audioCtx = new AudioContext();
  recorder = new Recorder(audioCtx.createMediaStreamSource(stream));
}).catch(e => console.log(e));

/**
 * 开始记录
 */
function start() {
  recorder.record();
}

/**
 * 结束记录并导出文件
 */
function stop() {
  recorder.stop();
  recorder.exportWAV();
}
```

## 将音频文件发送给服务器、服务器解析结果并返回给用户
上一步 `recorder.js` 已经将语音转换成可以发送给服务器的格式，接下来只要正常请求将文件发送给服务器，服务器使用微信、百度等开源库将语音解析成实际的文字返回给用户就可以了。

完。