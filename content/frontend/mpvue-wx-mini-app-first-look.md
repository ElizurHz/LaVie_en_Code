---
title: mpvue 小程序开发踩坑记
date: 2019-03-24
categories: ["微信小程序", "Vue", "Javascript"]
---

# mpvue 小程序开发踩坑记

之前公司决定开发一款小程序。

我接触微信小程序也就一两个月的时间，没使用过小程序官方的开发语言，而是直接使用 mpvue 上手了，自己开发了一个因为内容的原因无法上架的 DEMO。正好也被推上了前端部门的 leader，所以也就接下了这个任务。

使用 mpvue 的因为本以为由 Vue 转小程序用 mpvue 会方便，不过没想到坑挺多的，一般的 Web 开发者如果以 Vue 的思维入手会遇到很多麻烦，因为小程序的运行环境和浏览器完全不同。

另外京东凹凸实验室的 Taro 也是一个非常好的框架，不过我没有去深入研究它。考虑到公司技术栈转 Vue 的要求，我们团队选择了 mpvue，然而我个人认为 mpvue 还是个不太成熟的框架，文档不完善，在开发编译时也容易出 bug，另外就是分包加载的支持比较慢，好在最近出了新版本的 mpvue-loader。但是新版本的 mpvue-loader 完全更改了 app.json/main.json 文件的写法，文档中也没有明确提到，只能通过新的 quickstart 来了解新的写法。

## 我对小程序和 mpvue 的理解

### mpvue

- mpvue 是美团开发的，允许开发者使用 Vue 语法来编写小程序的框架。
- 小程序和 Web 应用的不同之处是它依靠微信 App 内的 JavaScript 引擎和 WXML/WXSS 引擎进行渲染，所以与 Web 端相比，小程序里不存在 DOM，无法使用原生 DOM 的操作，也不存在浏览器的各种 API。
- 在使用 mpvue 时，实际上可以看作是用 Vue 构建多页 Web App。
- 在 Webpack 打包时，Vue Web App 是最终打包成 index.html 以及 css 和 js 文件各 1 个（如果你不开 source map 的话）。而 mpvue-loader 是将 Vue 语法转换成小程序原生语法的 Webpack loader，打包后的文件实际上就是原生小程序的语法（js 除外），这样就可以在小程序开发工具中预览和 debug。

### weui

weui 是腾讯开发的 css 框架，可以用于编写腾讯规范的小程序控件。

### 小程序开发工具/微信公众平台 - 小程序

- 小程序开发工具是用于本地 debug 的工具，使用了 Chrome Devtools，并可以进行真机调试以及发布体验版
  - 真机调试：使用小程序上的“远程调试”按钮即可调试，在开发者的微信中的小程序列表中显示为“开发版”
  - 体验版：使用小程序上的“上传”按钮即可发布体验版，开发者和体验者扫码后可以访问，在开发者和体验者的微信中的小程序列表中显示为“体验版”
- 微信公众平台 - 小程序 是小程序项目的管理后台
  - 小程序和公众号是两个独立的账号，可以相互关联
  - 如果要团队协作开发，必须要在微信公众平台上添加开发者和体验者
  - 若需要使用网络请求，必须在微信公众平台上添加 https 的服务器
  - https 服务器的 URL 可以是二级、三级...域名，但不能带相对路径，例如可以是 api.test.example.cn 或者 api.example.cn，但不能使用 api.example.com/v1

## 微信支付

微信支付需要小程序上线之后在微信公众平台中申请开通。

[【微信支付】微信小程序支付开发者文档](https://pay.weixin.qq.com/wiki/doc/api/wxa/wxa_api.php?chapter=7_7&index=3)

1. 获取 openid
2. 请求支付接口获取 `wx.requestPayment()` 所需参数
3. `wx.requestPayment()` 发起支付请求，参数见[微信支付 · 小程序](https://developers.weixin.qq.com/miniprogram/dev/api/api-pay.html#wxrequestpaymentobject)
4. 如果在手机上操作会直接弹出微信支付相关弹窗，在开发者工具中操作则会生成二维码，扫描即可；在手机上可以直接弹出微信支付的弹窗。
5. 支付完成后使用 `wx.reLaunch` 转跳到结果页面，但是使用 `wx.reLaunch` 时，在 Android 上会提示 reLaunch:fail can not invoke reLaunch in background，此时只需加一个 setTimeout 即可。因为调用微信支付的窗口时，实际上会先把当前的 page 隐藏，如果在体验版里打开调试模式可以看到页面先 onHide 再 onShow，而我们需要在支付成功后转跳，所以 `wx.reLaunch` 是放在 `wx.requestPayment()` 的回调中执行的，此时页面仍没有 onShow，而 `wx.reLaunch` 需要在 onShow 之后才能调用。

## mpvue 小程序开发中需要注意的点

### mpvue 的项目结构

我们可以在官方的 quickstart 模板的基础上进行开发【[Quickstart - mpvue-docs](http://mpvue.com/mpvue/quickstart/)】。

- static - 静态文件
- src - pages - A - 页面 A - A.vue - .vue 页面 - main.js - Vue 的 js 文件，用于挂载页面 - main.json - 小程序的页面配置文件 - components - 公用组件 - utils - 工具 - app.json - 小程序的 app.json 主文件 - App.vue - 小程序主入口 - main.js - 类似 Vue 的 main.js，加载全局内容，挂载 Vue 实例

### 小程序有包大小的限制

主包最大为 2MB，各个分包大小最大为 8MB。在较大型的小程序开发时，最好是将各个模块独立出来，尽可能只保留入口页面和 Tab 标签页对应的页面在主包中。

分包加载只有在 mpvue-loader@1.1.x 版本才支持。

会打包到主包的内容（包括但不限于如下内容）：

1. 放在 app.json 中的 pages 中的页面
2. 静态文件，如图片
3. 公用组件
4. vuex
5. utils
6. 安装的各种有被引用的 package

注意：千万不要因为某个功能用某个 package 做很方便，就安装并引入它，因为这些 package 全部会打包到主包的 vender.js 中！这样很容易造成主包大小过大！

如果一个 package 是你不得不引入的核心 package（例如 vue, vuex, node-sass, sass-loader 等），那么就将它引入。

如果一个 package 你只使用到它的 1% 的功能（或者有非常多的功能用不上），请不要引入！例如安装了 qs, moment 和 lodash 之后，在我们公司的项目中，打包后的 vender.js 最大可以达到 1.6MB，而在移除它们所有的引用之后只有 462 KB。

JS 的大型工具库（如 lodash）千万不要引入 mpvue，因为它们大部分都可以简单地用原生 JS 实现。

占用空间的静态图片尽可能使用在线资源。导航栏图标和各种 icon，为了用户体验，建议放在本地。

### 小程序有页面栈的限制

由于性能的限制，小程序页面栈中最多只能存在 5 个页面，也就是最多只能转跳 5 层。 此时就需要善用清空页面栈的方法，例如转跳的时候使用 redirectTo / relaunch 等方法。但是在页面栈被清空之后，小程序右上角的返回按钮就会消失，所以在产品设计时处理好转跳逻辑是很关键的。详情见小程序文档：[导航 · 小程序 API](https://developers.weixin.qq.com/miniprogram/dev/api/ui-navigate.html)，[导航 · 小程序组件](https://developers.weixin.qq.com/miniprogram/dev/component/navigator.html)

### 小程序页面间传值

小程序页面间的传值靠的是路由的 params，类似于 vue-router 中的 params。在新页面中，若要接收值，则需要在 onLoad 生命周期中接收，onLoad 方法的第一个参数即为传入的 params 对象。

注意：params 传值默认的数据类型是 String。

### 返回时清空数据

小程序在点击左上角返回的时候不能自动清空当前页面的数据，如果需要清空数据，我们需要手动在 onUnload 的生命周期钩子里将 Vue 的 data 中的所有数据还原成初始状态。

### 返回上一页（上 X 页）时携带参数

我们在返回时会使用 `wx.navigateBack()`，这个方法不可携带参数。所以我们需要用小程序内置的 Storage 或者 Vuex 进行数据存取的操作。

不建议/不应该使用其他可以带参数的导航方式，因为它们会破坏现有的页面栈或者导致页面栈混乱，除非有特殊的需求。

### 小程序的生命周期

小程序有自己一套生命周期，和 Vue 的生命周期不冲突，但是行为有所区别。

举例来说，一个页面加载时，这些生命周期钩子会按如下顺序被调用：onLoad, onShow, mounted。页面存在页面栈中时，如果返回该页面，不会触发 onLoad 和 mounted，但是会触发 onShow，因为这个页面已经被挂载了。
更多小程序生命周期的详情请看小程序的官方开发文档。

### 网络请求

小程序中发起网络请求可以使用 [wx.request()](https://developers.weixin.qq.com/miniprogram/dev/api/network-request.html#wxrequestobject)，也可以使用 [fly.io](https://www.npmjs.com/package/flyio)，一个类似 axios 的开源库。

### 小程序的数据存储

在 Web 中我们有 session 和 WebStorage 可以存储一些公用数据如 token。而小程序有自己的本地缓存：[数据缓存 · 小程序](https://developers.weixin.qq.com/miniprogram/dev/api/data.html)，用法和 WebStorage 相似，但是有同步和异步方法的区别。

在 mpvue 中我们同样可以使用 vuex，但是需要注意的是我们无法像 Web 开发时一样在 `new Vue()` 的时候传入 vuex 作为参数，而是需要手动挂载到 Vue.prototype 上。

### 登录与授权

微信的授权弹窗在目前版本中无法主动触发，需要用 [button 的 open-type 属性](https://developers.weixin.qq.com/miniprogram/dev/component/button.html) 获取授权。授权弹窗只能获取微信的用户信息。授权弹窗中有 encryptedData 和 iv，这两个数据很关键，可以让服务端解码出 unionId。unionId 是用户唯一标识，在第三方登录授权中需要使用到。

如果需要微信开发者平台的第三方登录，需要使用 [wx.login()](https://developers.weixin.qq.com/miniprogram/dev/api/api-login.html) 的方法获取 code，并发送给服务端处理，通过 code 可以获取 sessionKey 和 openid。

`wx.login()` 一定要在授权弹窗之前调用，因为解密 encryptedData 和 iv 需要 sessionKey，sessionKey 是从 code 获得的。在 `wx.login()` 得到的 code（以及 code 获取的 sessionKey） 之后获取的 encryptedData 和 iv 是一一对应的，若颠倒顺序，则 encryptedData 和 iv 无法和 sessionKey 对应上。

### 标签

mpvue 中可以使用 Web 的标签，最后会被编译成对应的小程序标签；也可以使用小程序的原生标签。小程序的原生标签可以帮助我们做很多 Web 标签无法实现的工作，例如 swiper 和 swiper-item 可以实现左滑右滑切换 Tab 页；scroll-view 实际上是一个 `overflow-(x/y): auto;` 的块级元素，并且本身不带滚动条，但是注意 scroll-view 并不会触发 onReachBottom 的钩子。

### 分页加载

分页加载有两种方式: onReachBottom 和 scroll-view

- 使用 onReachBottom 时不能使用 scroll-view，只能使用普通的 view / div。scroll-view 使用 @scrolltolower (原生小程序是 bindscrolltolower) 的 event。
- 包裹着列表组件的容器必须保证能滚动到屏幕底部，也就是说 CSS 里对高度的设置需要注意
- 页面*组件中需要有变量记录当前页码或者下一页的页码，在触发 onReachBottom*@scrolltolower 时使用这个页码去获取新的分页数据
- 分页加载完成后，数据需要加到原列表数组的最后
- 页面*组件中需要有变量记录总页数，在 onReachBottom*@scrolltolower 时，如果已经到最后一页则不继续加载或者弹出提示
- 在进行一些复杂操作的时候记得清空当前页码和总页数，防止数据混乱

### 图片

小程序中的 image 如果不手动设置大小，会默认被设置为 320\*240 (rpx)。小程序中 image 的自适应和 Web 开发并不一样，使用的是 [image 的 mode 属性](https://developers.weixin.qq.com/miniprogram/dev/component/image.html)。

小程序的 image 不支持 svg 的图片，需要用 css 的 background-image。

app.json 中，TabBar 的 icon 不能用 svg。

在 mpvue 中可以使用 iconfont。

### 关于边框

mpvue-weui 和微信小程序的某写原生组件如 button 会有自带的边框，它们都是写在 `::after` 伪元素中的，如果你使用 border-radius 的话，会导致某些边框消失。此时建议将其隐藏并重新用下面的 rpx 的方式来写边框。

### 长度单位与 1 像素边框

按照从移动 H5 转过来的思维，1 像素边框一般需要用伪元素来写。小程序中同样可以使用 ::after 伪元素写 1 像素的边框，但是实际的显示出来的位置都和 Web 有区别。可以参考 [mpvue-weui](http://kuangpf.com/mpvue-weui/#/README) 的 weui.css 中的方式进行开发。

但事实上，微信小程序的 rpx 就是自适应单位，相当于移动 H5 做自适应常用的 rem。1rpx = 0.5px = 1 物理像素。所以我们实际上可以使用 rpx 来写 1 像素边框。当然微信小程序也支持 rem，规定屏幕宽度为 20rem，所以 1rem = (屏幕宽度 / 20)rpx。

button 在有 border-radius 的时候可能边框显示不全，此时建议用 div 包裹 button 并在 div 上写 1px 的 border，隐藏 button 自带的 border。

### position: fixed

小程序的 position: fixed 和 Web 上的表现有所不同。如果 mpvue 小程序的页面中有其他 Vue 组件，那么在组件中设定的 `position: fixed` 只会相对于这个组件是 fixed，并不会在页面中 fixed。如果设定 `bottom: 0`，那么它只会固定在 Vue 组件的底部，而不是整个页面的底部。也就是说，mpvue 小程序中的 Vue 组件实际上是一个 BFC。

### 事件

mpvue 中同样有事件，但是调取参数 e 的值的方式和 Web 不同，需要使用 `e.mp.detail`

### 点击导航条转跳到相应位置（类似通讯录）

在项目中，服务端只会给出所有的城市列表，之后我们需要在本地对其进行分类。按字母分类使用了 [GitHub - hotoo/pinyin: 汉字拼音 ➜ hàn zì pīn yīn](https://github.com/hotoo/pinyin) 。

页面分为侧边字母导航条和城市列表。项目的需求是只需要列出满足条件的部分城市，所以导航字母也需要根据城市动态显示。

点击字母滚动到对应位置的功能使用了 scroll-view 的 scroll-into-view，在 scroll-into-view 上动态绑定对应字母的唯一 id（变量在 Vue 的 data 中），同时在我们渲染城市的时候，需要在最外层加上属性 id，与 scroll-into-view 上的 id 要对应。

字母导航条中，我们需要设定一个 data-i，并在 click 事件中用 `e.currentTarget.dataset.i` 获取这个 data-i，并赋值给 data 中绑定 id 的变量，这样就可以直接转跳到相应的位置。

```html
<template>
  <div class="city">
    <div class="city__body">
      <div class="current-city">当前定位城市：{{ city.city }}</div>
      <scroll-view scroll-y class="alphabet" :scroll-into-view="'id-' + toView">
        <div
          :id="'id-' + index"
          v-for="(item, index) in filteredCities"
          :key="item.firstLetter + index"
        >
          <div class="weui-cells__title">{{ item.firstLetter }}</div>
          <div class="weui-cells weui-cells_after-title">
            <div
              class="weui-cell"
              v-for="(_item, _index) in item.cities"
              :key="_item.cityCode + _index"
              @click="setCity(_item)"
            >
              <div class="weui-cell__bd">{{ _item.city }}</div>
            </div>
          </div>
        </div>
      </scroll-view>
    </div>
    <div class="city__letters">
      <div
        v-for="(item, index) in nav"
        :key="item"
        @click="setAlphabet"
        :data-i="index"
      >
        {{item}}
      </div>
    </div>
  </div>
</template>
```

```javascript
<script>
export default {
	data() {
    return {
      nav: ['A', 'B', 'C', 'D', 'E', 'F', 'G', 'H', 'I', 'J', 'K', 'L', 'M', 'N', 'O', 'P', 'Q', 'R', 'S', 'T', 'U', 'V', 'W', 'X', 'Y', 'Z'],
      toView: 0,
      cities: []
    }
  },
	onLoad() {
	  // 获取城市
	  getCity().then(res => {
      if (/* 调用成功 */) {
        const categoriedCities = []
        const cities = []
        let nav = []
        for (const item of res.data.data) {
          // 提取城市首字母，放入导航列表
          const py = pinyin(item.city, { style: pinyin.STYLE_FIRST_LETTER })
          const firstLetter = py[0][0].toUpperCase()
          if (firstLetter.length === 1) {
            nav.push(firstLetter)
          }
          // 构造新的数据，加入首字母
          const newItem = {
            ...item,
            firstLetter,
            city: item.city.substring(0, item.city.length - 1)
          }
          cities.push(newItem)
        }
        // 导航里去除重复的字母
        nav = [...new Set(nav)]
        // 根据导航的字母，把上面构造的数据按照字母进行重新分类，得到最后需要的结果
        // 对每个字母，在上面构造的数据中筛选出相应字母的所有城市，放入一个对象中并 push 入最终数组
        for (const item of nav) {
          const sameLetterCities = cities.filter(i => i.firstLetter === item)
          categoriedCities.push({
            firstLetter: item,
            cities: sameLetterCities
          })
        }
        this.nav = nav
        this.cities = categoriedCities
      }
    })
	}
}
</script>
```
