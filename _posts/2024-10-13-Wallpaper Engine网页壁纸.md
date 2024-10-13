---
layout: post
title: "Wallpaper Engine网页壁纸"
subtitle: "Wallpaper Engine网页壁纸基础制作流程"
date: 2024-10-13 19:48:00
author: "Momoka7"
header-style: text
catalog: true
tags:
  - Wallpaper Engine
  - Frontend
---

# 1. 前言

Wallpaper Engine 支持各种各样的壁纸，从静态图片到视频再到有动效可交互的壁纸，但是如何去制作一个自己的有动效可交互的壁纸呢。

首先 Wallpaper Engine 是支持网页形式的壁纸的，即使用网页渲染的结果作为壁纸，这样一来其实能做的事就非常多了。

接下来以一个带有跟踪鼠标、音频监听的网页壁纸实例来讲解一些所使用到的知识。

# 2. Wallpaper Engine 网页壁纸

## 2.1 工程结构

和普通网页一样，主要使用`html、css、javascript`三剑客即可实现一个网页壁纸，在 Wallpaper Engine 中导入时选择`index.html`即可。

此外，和单纯的网页所不同的是，Wallpaper Engine 网页壁纸工程下还可以有一个 project.json 配置文件，用于配置属性、启用 Wallpaper Engine 特性等功能。如下为`project.json`的一个示例：

```json
{
  "file": "index.html",
  "general": {
    "properties": {
      "schemecolor": {
        "order": 0,
        "text": "ui_browse_properties_scheme_color",
        "type": "color",
        "value": "0 0 0"
      },
      "timeformat": {
        "order": 1,
        "text": "时间格式",
        "type": "combo",
        "value": "24",
        "options": [
          {
            "label": "24小时制",
            "value": "24"
          },
          {
            "label": "12小时制",
            "value": "12"
          }
        ]
      },
      "showseconds": {
        "order": 2,
        "text": "显示秒数",
        "type": "bool",
        "value": true
      },
      "showaudiospectrum": {
        "order": 3,
        "text": "显示音频频谱",
        "type": "bool",
        "value": true
      }
    },
    "supportsaudioprocessing": true,
    "mousehighlight": true
  },
  "title": "Gradient Clock Wallpaper",
  "preview": "preview.png",
  "type": "web",
  "audio": true,
  "audioprocessing": true
}
```

## 2.2 Wallpaper Engine 网页 API

### 2.2.1 属性监听

主要用于提供给用户自定义属性配置，具体分为如下几步：

1. 注册属性：如前面`project.json`中的配置所述，要在 Wallpaper Engine 中提供自定义属性，需要在`general.propperties`中对属性进行注册，具体如何定义属性参考上节所提供的实例。
2. 在网页的脚本中定义属性：如是否显示秒
   ```javascript
   let showSeconds = true; //建议设置一个默认值
   //... other codes
   ```
3. 注册属性变化监听事件：
   ```javascript
   // Wallpaper Engine属性监听
   window.wallpaperPropertyListener = {
     applyUserProperties: function (properties) {
       if (properties.showseconds !== undefined) {
         showSeconds = properties.showseconds.value;
       }
     },
   };
   ```
4. 使用属性：由于在第三步中我们监听了对应的属性及其变化，并将其赋值到了对应的变量中，我们就可以直接在其他代码中使用此变量了，如：

   ```javascript
   function updateClock() {
     const now = new Date();
     let hours = now.getHours();
     const minutes = String(now.getMinutes()).padStart(2, "0");
     const seconds = String(now.getSeconds()).padStart(2, "0");

     if (timeFormat === "12") {
       const ampm = hours >= 12 ? "下午" : "上午";
       hours = hours % 12;
       hours = hours ? hours : 12; // 0应显示为12
       hours = String(hours).padStart(2, "0");
       document.getElementById("clock").textContent = `${hours}:${minutes}${
         showSeconds ? ":" + seconds : ""
       } ${ampm}`;
     } else {
       hours = String(hours).padStart(2, "0");
       document.getElementById("clock").textContent = `${hours}:${minutes}${
         showSeconds ? ":" + seconds : ""
       }`;
     }
   }
   ```

   <br/>

### 2.2.2 音频监听器

1. 启用音频监听：需要在`project.json`的`general`中添加属性`"supportsaudioprocessing":true`
2. 注册音频监听事件：这里音频监听器的回调会提供一个保存了音频各个频段功率的数组`audioArray`，长度为 64。这个事件会不断被调用。
   ```javascript
   window.wallpaperRegisterAudioListener((audioArray) => {
     updateAudioSpectrum(audioArray);
     //   updateBackground(); // 每次接收到新的音频数据时更新背景
   });
   ```
3. 使用`audioArray`：可以通过这些音频数据来可视化等操作。

### 2.2.3 其他技巧

- 定时更新：善用`setInteval`，如`setInterval(updateClock, 1000);`
- 鼠标交互：监听鼠标事件即可，如：
  ```javascript
  // 修改 window.onmousemove 事件
  window.onmousemove = function (e) {
    const follower = document.getElementById("mouse-follower");
    if (!follower) {
      createMouseFollower(); //创建光标dom元素
    }
    updateMouseFollowerPosition(e);
    createTrailDot(e);
  };
  ```
- 日志信息：在用到 Wallpaper Engine 的 API 时无法直接通过网页来查看其效果，而有时需要使用日志信息对脚本进行调试，如下是实现在 Wallpaper Engine 中 Debug 的过程：
  1. 开启 CEF：在 Wallpaper Engine 的`设置->常规->开发人员`配置中，配置日志级别和 CEF 开发工具端口即可；
  2. 进入 CEF：访问上面配置的本地端口即可，例如`localhost:8080`，可以通过控制台查看日志信息以到达调试的目的。
