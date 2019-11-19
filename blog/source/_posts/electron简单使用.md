---
title: Electron简单使用
date: 2019-11-19
---
Electron是由Github开发，用HTML，CSS和JavaScript来构建跨平台桌面应用程序的一个开源库。
***
## Electron简单使用

### 1.简介
 Electron通过将Chromium和Node.js合并到同一个运行时环境中，可以让你使用纯 JavaScript 调用丰富的原生(操作系统) APIs 来创造跨桌面应用。 你可以把它看作一个被 JavaScript 控制的，精简版的 Chromium 浏览器，也可以看做一个 Node. js 的变体，能使用nodejs方法来处理数据，不过专注于桌面应用而不是 Web 服务器端。

### 2.运行环境
nodejs，electron开发环境下依赖Node环境，所以需要先安装nodejs，具体参考https://nodejs.org/

### 3.简单使用
#### 3.1项目结构和启动
一个简单的electron项目是这样的，package.json 是项目配置文件，electron运行时会通过package.json找到入口文件main.js；而index.html则是electron的主窗口文件。
```
app
├── package.json
├── main.js
└── index.html
```
electron安装有全局安装和局部安装两种方式，如果安装过慢，建议使用淘宝镜像代理。全局安装时，项目启动如下，注意点.不好少：
```
electron .
```
局部安装启动方式
```
npm start  //或者node_modules/.bin/electron .
```
#### 3.2项目开发
Electron apps 使用JavaScript开发，其工作原理和方法与Node.js 开发相同。 electron模块包含了Electron提供的所有API和功能，可以通过require引入使用。electron 模块所提供的功能都是通过命名空间暴露出来的。 比如说： electron.app负责管理Electron 应用程序的生命周期， electron.BrowserWindow类负责创建窗口。
###### 主进程
一个 Electron 应用只有 一个主进程，即main文件。运行在主进程中的脚本可以通过创建一个窗口，并传入 URL，让这个窗口加载一个网页来展示图形界面。
与创建 GUI 相关的接口只应该由主进程来调用。
下面是一个main.js文件，编写好index.html文件，即可使用npm start启动项目。
```
/** main.js */
const { app, BrowserWindow } = require('electron');
let mainWindow;
// Electron 会在初始化后并准备
// 创建浏览器窗口时，调用这个函数。
// 部分 API 在 ready 事件触发后才能使用。
app.on('ready', () => {
    mainWindow = new BrowserWindow({
        width: 1000,
        height: 640,
    });
    // mainWindow.setMenu(null); // 隐藏Chromium菜单
    // mainWindow.webContents.openDevTools() // 开启调试模式
    var mySocks = 'socks5://'+'localhost:1080';//设置代理
    mainWindow.webContents.session.setProxy({  // 可以设置翻墙代理
        proxyRules: mySocks,
        proxyBypassRules: `url`//多个用.隔开设置不翻墙ip
    }, function () {
        mainWindow.loadFile('index.html'); // 主窗口文件引入
    });

    mainWindow.on('closed', () => {
        mainWindow = null;
    });

// 忽略证书相关错误
app.commandLine.appendSwitch('ignore-certificate-errors');


app.on('window-all-closed', () => {
    /* 在Mac系统用户通过Cmd+Q显式退出之前，保持应用程序和菜单栏处于激活状态。*/
    if (process.platform !== 'darwin') {
        app.quit();
    }
});

app.on('activate', () => {
    /* 当dock图标被点击并且不会有其它窗口被打开的时候，在Mac系统上重新建立一个应用内的window。*/
    if (mainWindow === null) {
        createWindow();
    }
});

```
###### 渲染进程renderer
在Electron里的每个页面都有它自己的进程，叫作渲染进程。主进程通过实例化 BrowserWindow，每个 BrowserWindow 实例都在它自己的渲染进程内返回一个 web 页面。当 BrowserWindow 实例销毁时，相应的渲染进程也会终止。
渲染进程由主进程进行管理。每个渲染进程都是相互独立的，它们只关心自己所运行的 web 页面。一般在对应的html文件的script引入。可以在渲染进程中和主进程进行通信，根据数据渲染更新视图。

###### 主进程main和渲染进程通讯
主进程和渲染进程通讯有两种方式：
**1.使用 ipcMain 和 ipcRenderer 模块**
参照官方文档例子

```
// 在主进程中.
const { ipcMain } = require('electron')
ipcMain.on('asynchronous-message', (event, arg) => {
  console.log(arg) // prints "ping"
  event.reply('asynchronous-reply', 'pong')
})

ipcMain.on('synchronous-message', (event, arg) => {
  console.log(arg) // prints "ping"
  event.returnValue = 'pong'
})
```

```
//在渲染器进程 (网页) 中。
const { ipcRenderer } = require('electron')
console.log(ipcRenderer.sendSync('synchronous-message', 'ping')) // prints "pong"

ipcRenderer.on('asynchronous-reply', (event, arg) => {
  console.log(arg) // prints "pong"
})
ipcRenderer.send('asynchronous-message', 'ping')
```
渲染进程通过ipcRenderer向主进程sendSync异步消息synchronous-message，和send同步消息asynchronous-message，主进程使用ipcMain.on监听这两种消息。主进程监听的回调函数中，会传递 event 对象及 arg 对象。arg 对象中保存渲染进程传递过来的参数。通过 event.sender 对象，主进程可以向渲染进程发送消息。如果主进程执行的是同步方法，还可以通过设置 event.returnValue 来返回信息。

主进程向渲染进程发送消息时则需要通过BrowserWindow 对象的webContents 属性，webContets 提供了 send 方法来实现向渲染进程发送消息。如下：
```
mainWindow.webContents.on('new-window', function (event, arg) {
    let childWin = new BrowserWindow({
        width: 1000,
        height: 640,
    });
    mainWindow.webContents.send('ping', 'hhhhh');
    event.preventDefault();
})
```
**注： webContents.on只能监听webContents已定义的事件，不能监听自定义事件，监听自定义的事件还是需要ipcMain和ipcRenderer**
ipcRenderer.on监听主进程传来的消息，也可以通过 event.sender 来向主进程发送消息。这个对象只是 ipcRenderer 的引用(event.sender === ipcRenderer)，还是需要ipcMain.on来接收。
**2.使用remote（在渲染进程中使用主进程模块）**
Electron中GUI 相关的模块 (如 dialog、menu 等) 仅在主进程中可用, 在渲染进程中不可用。 为了在渲染进程中使用它们, ipc 模块是向主进程发送进程间消息所必需的。 使用 remote 模块, 你可以调用 main 进程对象的方法, 而不必显式发送进程间消息。
注意事项： 因为安全原因，remote 模块能在以下几种情况下被禁用：
**BrowserWindow - 通过设置 enableRemoteModule 选项为 false。**
**webview - 通过把 enableremotemodule属性设置成 false。**
如下：在渲染进程创建新窗口打开github。
```
const { BrowserWindow } = require('electron').remote
let win = new BrowserWindow({ width: 800, height: 600 })
win.loadURL('https://github.com')
```

###### webview
使用 webview 标签在Electron 应用中嵌入 "外来" 内容 (如 网页)。外来"内容包含在 webview 容器中。 应用中的嵌入页面可以控制外来内容的布局和重绘。
(Electron的 webview 标签基于 Chromium webview ，后者正在经历巨大的架构变化。这将影响 webview 的稳定性，包括呈现、导航和事件路由，Electron >= 5禁用 webview 标签。 目前官方建议不使用 webview 标签，并考虑其他替代方案，如 iframe 、Electron的 BrowserView 或完全避免嵌入内容的体系结构。)

使用webview时候只需要在页面中加入该标签，以下是一个简单的webview标签，webview 标签包括网页的 src、控制 webview 容器外观的 css 样式和运行在外来页面的javascript preload.js:
```
<webview src="https://github.com/" preload="./preload.js" style="display:inline-flex; width:800px; height:640px"></webview>

// preload.js
const ipc = require('electron').ipcRenderer;
window.onload = function(event) {
  ipc.on('answer', (event, arg) => {
        console.log(arg);
    });
	ipc.sendToHost('pageReady', true);

})

```
如果要以任何方式控制外来内容, 则可以写用于侦听 webview 事件的 JavaScript, 并使用 webview 方法响应这些事件。 参照官方示例代码: 一个侦听网页开始加载, 另一个用于网页停止加载, 并在加载时显示 "loading..." 消息。也可以在该js中监听上面preload.js的sendToHost事件，或者通过webview.send向preload发送消息，从而使外来页面和webview可以进行通讯：
```
<script>
  onload = () => {
    const webview = document.querySelector('webview')
    const indicator = document.querySelector('.indicator')

    const loadstart = () => {
      indicator.innerText = 'loading...'
    }

    const loadstop = () => {
      indicator.innerText = ''
    }

    webview.addEventListener('did-start-loading', loadstart)
    webview.addEventListener('did-stop-loading', loadstop)

  // webview监听 preload的sendToHost事件
    webview.addEventListener('ipc-message', (event) => {
        if (event.channel === 'pageReady) { // channel为事件名称
            console.log(event.args[0]); // event.args[0]为事件所传参数
            webview.send('answer', 'ok'); //webview.send向preload发送消息
        }
     });
  }
</script>
```

###### net（使用Chromium的原生网络库发出HTTP / HTTPS请求）
net 模块是一个发送 HTTP(S) 请求的客户端API。 它类似于Node.js的HTTP 和 HTTPS 模块 ，但它使用的是Chromium原生网络库来替代Node.js的实现，提供更好的网络代理支持。
**注：只有在应用程序发出 ready 事件之后, 才能使用 net API。尝试在 ready 事件之前使用该模块将抛出一个错误。**
```
const { app } = require('electron')
app.on('ready', () => {
  const { net } = require('electron')
  const request = net.request('https://github.com')
  request.on('response', (response) => {
    console.log(`STATUS: ${response.statusCode}`)
    console.log(`HEADERS: ${JSON.stringify(response.headers)}`)
    let chunks = [];
	let size = 0;
	response.on('data', function (chunk) {
		chunks.push(chunk);
		size += chunk.length;
	});
	response.on('end', function () {
		let buf = Buffer.concat(chunks, size);
		let str = iconv.decode(buf, 'utf8'); //  iconv转为utf8
		let result = JSON.parse(str);
	});
  })
  request.end()
})
```

### 4.打包
electron有electron-packager及electron-builder两种打包方式。
使用electron-packager来进行打包时，可支持平台Windows (32/64 bit)、OS X (also known as macOS)、Linux (x86/x86_64);
appname为应用名称，
sourcedir为打包目录
--arch决定了使用 x86 还是 x64 还是两个架构都用，其中x86位ia32也可以在64位的电脑使用，如果使用ia64则只能在64位的使用。
--asar可以把源码打包到一个文件里面，进而获得一定的代码加密和整合的效果
ignore ，在将默认打包的范围内，排除掉一些不打包进去的。例如第三方资源文件，是无论如何，都不能打包到asar里面的。打包进去的话，或者没有用，占体积。或者影响程序逻辑实现，不能访问到这些第三方资源。
extra-resource，可以将第三方资源，在打包的时候，复制到app.asar的同级目录。
--icon决定了应用的图标。
--version决定了应用的版本。
```
npm run package

//package.json设置package
"package": "electron-packager . <appname> --asar --out <sourcedir> --arch=ia32 --version 1.4.0  --icon=./icon.ico  --ignore=res/ --extra-resource=res/"
```
electron-builder就是有比electron-packager有更丰富的的功能，支持更多的平台，同时也支持了自动更新。除了这几点之外，由electron-builder打出的包更为轻量，并且可以打包出不暴露源码的setup安装程序。所以推荐使用electron-builder 。
首先，安装依赖。
```
npm i electron-builder --save-dev
```
在package.json中做如下配置
scripts	npm运行的脚本别名
build打包配置
icon应用生成的图标
target打包成何种应用，如windows打包成exe、tar等，Linux打包成AppImage、snap等
category应用分类，固定即可，Linux、Mac上要求的
nsis windows下打包nsis时的特有安装参数，如指定可以更改安装目录等
```
build": {
    "win": {
      "icon": "icon.png",
      "target": [
        "nsis"
      ]
    },
    "nsis": {
      "allowToChangeInstallationDirectory": true,
      "oneClick": false,
      "menuCategory": true,
      "allowElevation": false
    },
    "linux": {
      "icon": "icon.png",
      "category": "Utility",
      "target": [
        "AppImage"
      ]
    },
    "mac": {
      "icon": "icon.png",
      "type": "development",
      "category": "public.app-category.developer-tools",
      "target": [
        "dmg"
      ]
    }
},
"scripts": {
    "build:linux": "node_modules/.bin/electron-builder -l",
    "build:windows": "node_modules/.bin/electron-builder -w",
    "build:mac": "node_modules/.bin/electron-builder -m"
},
```

### 结语
以上是个人简单使用了一段时间electron后了解到的知识，记录下来弄了个简单的使用指南，如有错误欢迎指正。现阶段使用的是electron4.04版本，版本升级后可能有其它方法使用需校正。具体还以官网文档为准。[electron文档](https://electronjs.org/docs "electron文档")