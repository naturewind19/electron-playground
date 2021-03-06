# 协议自动唤起应用与自定义协议

## 1. 协议: 从网页端唤起Electron应用

### 1.1 协议唤起示例:

![](wakeUp.jpg?height=200)


### 1.2 什么是协议

electron注册的协议, electron会将协议注册到系统的协议列表中，它是系统层级的API，只能在当前系统下使用, 其他未注册协议的电脑不能识别。

Electron的app模块提供了一些处理协议的方法, 这些方法允许您设置协议和取消协议, 来让你的应用成为默认的应用程序。

### 1.3 协议的作用

注册一个协议到系统协议中, 当通过其他应用/浏览器网页端**打开新协议的链接时，浏览器会检测该协议有没有在系统协议中, 如果该协议注册过，然后唤起协议的默认处理程序(我们的应用)**。

### 1.4 注册协议: app.setAsDefaultProtocolClient

协议需要在ready事件后注册，具体代码如下.

```js
// @@code-renderer: runner
// @@code-props: { height: '400px' }
// 注册自定义协议
const { app } = require('electron')
const path = require('path')

// 注册自定义协议
function setDefaultProtocol() {
  const agreement = 'electron-playground-code' // 自定义协议名
  let isSet = false // 是否注册成功

  app.removeAsDefaultProtocolClient(agreement) // 每次运行都删除自定义协议 然后再重新注册
  // 开发模式下在window运行需要做兼容
  if (process.env.NODE_ENV === 'development' && process.platform === 'win32') {
    // 设置electron.exe 和 app的路径
    isSet = app.setAsDefaultProtocolClient(agreement, process.execPath, [
      path.resolve(process.argv[1]),
    ])
  } else {
    isSet = app.setAsDefaultProtocolClient(agreement)
  }
  console.log('是否注册成功', isSet)
}

setDefaultProtocol()
```

### 1.5 使用协议

使用方式: 在浏览器地址栏输入注册好的协议，即可唤起应用。 

协议唤起的链接格式: 自协议名称://参数

比如上文注册: `electron-playground-code`协议,触发时会默认带上`://`。

在使用的时候, 需要在浏览器地址栏输入: 

```js
electron-playground-code://1234 // 1234是参数 可根据业务自行修改
```

#### 1.6 注册协议并通过浏览器唤起后台应用的gif示例:


![](./setProtocol.gif?height=400)

### 1.7 监听应用程序被唤起

应用程序唤起，mac系统会触发open-url事件，window系统会触发second-instance事件。

```js
// @@code-renderer: runner
// @@code-props: { height: '450px' }
// 注册自定义协议
const { app, dialog } = require('electron')
const agreement = 'electron-playground-code' // 自定义协议名
// 验证是否为自定义协议的链接
const AGREEMENT_REGEXP = new RegExp(`^${agreement}://`)

// 监听自定义协议唤起
function watchProtocol() {
  // mac唤醒应用 会激活open-url事件 在open-url中判断是否为自定义协议打开事件
  app.on('open-url', (event, url) => {
    const isProtocol = AGREEMENT_REGEXP.test(url)
    if (isProtocol) {
      console.log('获取协议链接, 根据参数做各种事情')
      dialog.showMessageBox({
        type: 'info',
        message: 'Mac protocol 自定义协议打开',
        detail: `自定义协议链接:${url}`,
      })
    }
  })
  // window系统下唤醒应用会激活second-instance事件 它在ready执行之后才能被监听
  app.on('second-instance', (event, commandLine) => {
    // commandLine 是一个数组， 唤醒的链接作为数组的一个元素放在这里面
    commandLine.forEach(str => {
      if (AGREEMENT_REGEXP.test(str)) {
        console.log('获取协议链接, 根据参数做各种事情')
        dialog.showMessageBox({
          type: 'info',
          message: 'window protocol 自定义协议打开',
          detail: `自定义协议链接:${str}`,
        })
      }
    })
  })
}

// 在ready事件回调中监听自定义协议唤起
watchProtocol()
console.log('监听成功')
```

### 1.8 唤起应用执行回调示例

![](./protocolWatch.gif?height=400)


### 1.9 应用场景

1. 单纯唤醒应用

    只需注册协议，系统会自动打开应用。
    表现：如果应用未打开将打开应用，如果应用已经打开应用将会激活应用窗口。

2. 根据协议链接的参数进行各种操作

   如上面的弹窗演示, **在监听协议链接打开的时候，可以获取完整的协议链接**。

   我们可以根据协议链接来进行各种业务操作。

   比如跳转指定链接地址，比如判断是否登录再进行跳转，比如下载指定文件等。

### 1.10 一些其他API

**app.removeAsDefaultProtocolClient(protocol)** 删除注册的协议, 返回是否成功删除的Boolean

**Mac: app.isDefaultProtocolClient(protocol)** 当前程序是否为协议的处理程序。

**app.getApplicationNameForProtocol(url)** 获取该协议链接的应用处理程序


参数说明: 

protocol 不包含:// 注册的协议名。

url 包含://


```js
// @@code-renderer: runner
// @@code-props: { height: '200px' }
// 自定义协议的其他相关API
const { app } = require('electron')
const agreement = 'electron-playground-code' // 自定义协议名

console.log('自行注释，自由尝试')
const isApp = app.isDefaultProtocolClient(agreement)
console.log('当前程序是否为自定义协议的处理程序: ', isApp)

const AgreementAppName = app.getApplicationNameForProtocol(`${agreement}://`)
console.log('获取该自定义协议链接的应用处理程序的名字', AgreementAppName)

const isDelete = app.removeAsDefaultProtocolClient(agreement)
console.log('删除自定义协议', isDelete)
```

## 2. 自定义协议

注册自定义协议，拦截基于现有协议的请求，根据注册的自定义协议类型返回对应类型的数据。

在该项目中的代码地址: electron-playground/app/protocol, 可以运行项目调试一下看看效果。

### 2.1 protocol.registerSchemesAsPrivileged

将协议注册成标准的scheme, 方便后续调用。

**注意**： 它必须在ready事件加载之前调用，并且只能调用一次。

```js
protocol.registerSchemesAsPrivileged([
  { scheme: 'myscheme', privileges: { bypassCSP: true } },
])
```

### 2.2 protocol.registerFileProtocol

拦截自定义协议的请求回调，重新处理后再请求路径。

**示例**

```js
protocol.registerFileProtocol(
  'myscheme',
  (request, callback) => {
    // 拼接绝对路径的url
    const resolvePath = path.resolve(__dirname, '../../playground')
    let url = request.url.replace( `${myScheme}://`, '' )
    url = `${resolvePath}/${url}`
    return callback({ path: decodeURIComponent(url) })
  },
  error => {
    if (error) console.error('Failed to register protocol')
  },
)
```

PS: 在[文档](https://www.electronjs.org/docs/api/protocol)上提供了不同种类的加载API，这里只演示其中的一种。


### 2.3 使用方式

在html中使用自定义协议请求文件，即可自动拦截。

项目中的使用地址: electron-playground/playground/page/protocol/protocol.md

```html
 <img src={"myscheme://page/protocol/wakeUp.jpg"} alt="wakeUp"/>
```

### 2.4 protocol其他API

```js
protocol.unregisterProtocol(scheme) // 取消对自定义scheme的注册
protocol.isProtocolRegistered(scheme) // 自定义协议是否已经拦截
protocol.uninterceptProtocol(scheme) // 移除自定义协议的拦截器
// 各种用新的拦截器替换原有的拦截器
protocol.interceptFileProtocol(scheme, handler)
protocol.interceptStringProtocol(scheme, handler)
protocol.interceptBufferProtocol(scheme, handler)
protocol.interceptHttpProtocol(scheme, handler)
protocol.interceptStreamProtocol(scheme, handler)
```
