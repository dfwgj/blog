
---
title: 若依框架 + Tauri 2.0：一次跨平台桌面应用的踩坑之旅
date: 2025-11-14 00:06:00
cover: /covers/Tauri.svg
categories:
     - 工程化
tags:
    - Tauri
    - 若依
---
# 若依框架 + Tauri 2.0：一次跨平台桌面应用的踩坑之旅

## 起因

课程设计要做个管理系统，用的是若依（RuoYi-Vue3）前后端分离版。能打包成桌面应用交付...这不就是Electron的活吗？

但是。

一搜发现Electron打包后动辄上百MB，我这项目本身也没多大，光个空壳就要占这么多？再看看Tauri，官方说包体积能减少91%，实际打包出来只有几MB。

选Tauri，就是这么朴素的理由。

技术栈：

- 前端：Vue 3.2.45 + Element Plus 2.11.1
- 框架：若依前后端分离版
- 桌面：Tauri 2.x

然后...就开始了一整天的踩坑之旅。

---

## 问题1：Excel上传突然不工作了

这是第一个遇到的问题。浏览器里上传Excel文件好好的，打包成Tauri应用后点击上传按钮，没有任何反应。控制台也不报错，就是静悄悄的...不干活。

排查了半天才发现，Element Plus的 `<el-upload>`组件这样写：

```vue
<el-upload
  :action="uploadUrl"
  :headers="headers"
  :data="extraParams"
>
```

这会触发浏览器原生的 `XMLHttpRequest`，完全绕过了Tauri的HTTP适配器。Tauri环境里所有网络请求必须走自定义adapter，不然Cookie带不上，跨域也处理不了。

改成自定义上传：

```vue
<el-upload
  :http-request="customUploadRequest"
>
```

```javascript
async function customUploadRequest(options) {
  const { file, onSuccess, onError } = options
  const formData = new FormData()
  formData.append('files', file.raw || file)

  try {
    const response = await service.post(uploadUrl, formData, {
      headers: {
        'Content-Type': 'multipart/form-data',
        Authorization: 'Bearer ' + getToken()
      }
    })

    onSuccess(response, file)  // 注意传整个response，不是response.data
  } catch (error) {
    onError(error, file)
  }
}
```

这里有个坑：`onSuccess`必须传完整的 `response`对象，因为响应拦截器需要 `response.code`字段来判断上传成功还是失败。如果只传 `response.data`，拦截器拿不到 `code`，会认为上传失败。

---

## 问题2：登录密码错了，一点提示都没有

输错密码点登录，什么都不显示。浏览器里会弹"用户名或密码错误"，桌面应用里...安静得像什么都没发生。

后端明明返回了HTTP 500，前端就是收不到错误。

原因是Tauri适配器把所有响应都当成功了：

```javascript
async function tauriAdapter(config) {
  const response = await tauriFetch(url, {...})
  return {
    data: await response.json(),
    status: response.status
  }  // 不管status是200还是500，都直接返回
}
```

axios的逻辑是：4xx/5xx要抛异常。这里没抛，错误就被"吞"了。

加上状态码检查：

```javascript
if (!response.ok) {  // HTTP 2xx才算ok
  const axiosError = new Error(response.statusText || 'Request failed')
  axiosError.response = axiosResponse
  axiosError.isAxiosError = true
  throw axiosError  // 这样才会触发错误提示
}
```

---

## 问题3：Linux构建完报"scheme tauri not supported"

Windows打包的应用跑得好好的，传到Linux服务器上一运行...

```
❌ Tauri fetch error: "scheme tauri not supported"
```

所有接口都404。

查了半天发现 `.env.production`里写的是：

```bash
VITE_APP_BASE_API = 'https://sxxt.gdmu-stuorg.com/prod-api'
```

那对单引号...Vite会把它当成值的一部分。Linux构建时这个变量不知道为啥就变 `undefined`了，于是相对路径 `/login`拼出来变成了 `tauri://localhost/login`。

Tauri HTTP插件只认 `http://`和 `https://`，看到 `tauri://`直接拒绝。

解决办法很简单，去掉引号：

```bash
VITE_APP_BASE_API = https://sxxt.gdmu-stuorg.com/prod-api
```

顺便加了个验证逻辑，防止以后再踩坑：

```javascript
if (!url.startsWith('http://') && !url.startsWith('https://')) {
  throw new Error(`Invalid URL: ${url}, check your .env.production`)
}
```

---

## 问题4：Windows能登录，Linux登录了又退出来（最离谱的bug）

这个问题...真的折磨人。

**Windows上**：输入账号密码，登录成功，跳转首页，一切正常。

**Linux上**：输入账号密码，点登录，页面闪了一下...又回到登录页。

加日志一看：

```
✅ setToken: Token保存成功
✅ login: 登录成功，准备跳转
🛡️ route guard: 检查Token...
❌ getToken: Token是undefined
🔄 redirecting to /login
```

???

Token明明刚存进去，怎么立刻就读不到了？

一开始怀疑Cookie跨域问题，配置检查了三遍，没问题。
又怀疑Token格式有问题，打日志看Token本身正常。
再怀疑路由守卫逻辑有bug，但Windows上能跑啊...

**Windows上为什么能跑？**

最后加了时间戳日志才发现：

```
[0ms]   setToken: 开始写Cookie
[0.1ms] setToken: 写入完成
[0.2ms] router.push: 触发路由跳转
[0.3ms] route guard: 开始执行
[0.4ms] getToken: 读到Token ✅
```

Windows的WebView2（Chromium内核）写Cookie快到像是同步操作。

**Linux上的时序：**

```
[0ms]    setToken: 开始写Cookie
[0.5ms]  router.push: 触发路由跳转
[1.0ms]  route guard: 开始执行
[1.5ms]  getToken: undefined ❌  ← Cookie还没写完！
[2.0ms]  Cookie写入完成（晚了）
```

Linux的WebKitGTK写Cookie有1-2毫秒的延迟。这个延迟在Windows上短到可以忽略，Linux上就够触发竞态条件了。

Tauri在不同平台用的WebView完全不同：

- Windows: Edge WebView2 (Chromium) - Cookie写入极快
- Linux: WebKitGTK - Cookie写入有延迟
- macOS: WebKit - 同样有延迟

这就是为什么Windows测试通过，Linux/macOS就翻车。

**解决方案：不用Cookie了，改localStorage**

```javascript
const isTauri = window.__TAURI__ !== undefined

export function setToken(token) {
  return isTauri
    ? localStorage.setItem(TokenKey, token)  // 完全同步
    : Cookies.set(TokenKey, token)  // 浏览器还是用Cookie
}

export function getToken() {
  return isTauri
    ? localStorage.getItem(TokenKey)
    : Cookies.get(TokenKey)
}
```

`localStorage`的读写是完全同步的，不存在时序问题。改完后三个平台都正常了。

这个bug印象最深，因为完全想不到"能跑"和"跑不了"的差别居然是微秒级的时序问题...

---

## 问题5：日期全变成"0-0-0 0:0:0"

Linux和macOS打包后，所有日期显示 `0-0-0 0:0:0`。Windows还是好好的 `2025-11-13 16:23:59`。

后端返回的是标准ISO格式：`"2025-11-13T16:23:59.000Z"`

前端 `parseTime`函数处理时正则写错了，最后字符串里还残留着 `T`和 `Z`：

```
"2025/11/13T16:23:59Z"
```

Chromium能容错解析这种"半标准"格式，WebKit不行，直接返回 `Invalid Date`。

改成标准流程：

```javascript
if (typeof time === 'string') {
  time = time.replace(/\.\d{3}/, '')              // 去毫秒
  time = time.replace(/Z$/i, '')                  // 去时区标识
  time = time.replace('T', ' ')                   // T换成空格
  time = time.replace(/-/g, '/')                  // 横杠换斜杠
}
// 最终格式: "2025/11/13 16:23:59" ✅

date = new Date(time)
if (isNaN(date.getTime())) {
  return null  // 无效日期返回null，别显示0-0-0
}
```

---

## 总结

这次折腾下来最大的感受：**跨平台开发不能只在Windows上测**。

Tauri底层用的WebView完全不一样：

- Windows是Chromium，性能好、容错强
- Linux/macOS是WebKit，更严格、有细微差异

Windows能跑不代表Linux能跑。Cookie时序问题就是血淋淋的例子。

技术要点记录：

- Tauri环境下所有HTTP请求走自定义adapter
- 手动检查 `response.ok`模拟axios错误处理
- Tauri用localStorage存Token，浏览器用Cookie
- 日期统一转 `YYYY/MM/DD HH:mm:ss`格式
- `.env`文件别加引号

项目已经上线了，可以直接体验：**https://sxxt.gdmu-stuorg.com/**

源码在Gitee（https://gitee.com/yi-jingqq/graduationMachingSystem），完整的Tauri配置和若依框架集成细节都在代码里。

如果你也在用Tauri + 若依做项目，希望这篇文章能帮你少踩点坑。
