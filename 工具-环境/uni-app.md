---
title: "uni-app"
date: 2025-08-04
categories:
  - utils
tags:
  - uni-app
---

# 下载

https://www.dcloud.io/hbuilderx.html

下载插件scss/sass编译：https://ext.dcloud.net.cn/plugin?name=compile-node-sass

* 切换快捷键方案：工具-预设快捷方案
* 工具-设置-打开Setting.json配置字体，大小
* 参考文档：https://uniapp.dcloud.net.cn/

新建vue3项目



# uni_app_base

* 创建一个 uni-app 基础项目-vue3-其他动不用勾选
* manifest 中填写 AppID
* 入口 main.js

下载 http 请求库

pnpm add @escook/request-miniprogram

使用vuex 的状态管理

pages.json 中写 页面路由和tabBar

# 一键登录与用户状态管理

## 登录相关接口

### 前端 wx.login 获取临时code

使用 uni 提供的接口

~~~vue
try {
  // 获取微信登录code
  const loginRes = await uni.login({
    provider: 'weixin'
  })

  console.log('微信登录code:', loginRes.code)
  // 调用登录接口
  ...

} catch (error) {
  console.error('登录失败:', error)
  uni.showToast({
    title: '登录失败: ' + error.message,
    icon: 'none'
  })
}
~~~

### 后端 code2Session

传入： 小程序 appId， 小程序 appSecret， code， grant_type
返回： session_key，unionid(同个微信账号在不同平台的unionid 都是相同的)，openid

~~~
示例返回：
{\"session_key\":\"7htQvdCCVR2eRgMFj/SneQ==\",\"openid\":\"o1Gqe6DPXUnYqa3MdNMyOlwFJ22w\"}","level":"info","span":"f6af84092c254d93","trace":"0f3cfd7df0efe9e7d88212c8779fc838"}
~~~

https://developers.weixin.qq.com/miniprogram/dev/OpenApiDoc/user-login/code2Session.html

### 前端 wx.getUserProfile 获取用户信息

https://developers.weixin.qq.com/miniprogram/dev/api/open-api/user-info/wx.getUserProfile.html

直接获取用户的相关信息

~~~
# res.userInfo
{"nickName":"微信用户","gender":0,"language":"","city":"","province":"","country":"","avatarUrl":"https://thirdwx.qlogo.cn/mmopen/vi_32/POgEwh4mIHO4nibH0KlMECNjjGxQUq24ZEaGT4poC6icRiccVGKSyXwibcPq4BWmiaIGuG1icwxaQX6grC9VemZoJ8rg/132","is_demote":true}
~~~

~~~
wx.getUserProfile({
  desc: '用于完善用户资料', // 声明获取用户个人信息后的用途
  success: (res) => {
    console.log('用户信息:', JSON.stringify(res.userInfo))
    // 显示成功提示
    uni.showToast({
      title: '获取用户信息成功',
      icon: 'success'
    })
  },
  fail: (err) => {
    console.error('获取用户信息失败:', err)
    uni.showToast({
      title: '获取用户信息失败',
      icon: 'none'
    })
  }
})
~~~



### 点击获取用户头像，点击获取用户名

```
<!-- 头像选择按钮 -->
<button class="avatar-wrapper" open-type="chooseAvatar" @chooseavatar="onChooseAvatar">
    <image class="avatar" :src="avatarUrl"></image>
</button>

<!-- 头像处理方法 -->
onChooseAvatar(e) {
    console.log("选择头像:", e.detail.avatarUrl)
    this.avatarUrl = e.detail.avatarUrl
}

<!-- 昵称输入框 -->
<view class="row">
    <view class="text1">昵称：</view>
    <input type="nickname" class="weui-input" name="nickname" placeholder="请输入昵称"/>
</view>

<!-- 昵称处理方法 -->
async formSubmit(e){
    console.log("nickname: " + e.detail.value.nickname)

    // 获取昵称
    const nickname = e.detail.value.nickname
    if (!nickname) {
        uni.showToast({
            title: '请输入昵称',
            icon: 'none'
        })
        return
    }
    
    // 后续登录逻辑...
}
```



### 获取手机号

收费功能，暂不考虑

## 一键登录

1. 调用wx.getUserProfile 获取用户信息(昵称，头像，性别)
2. wx.login 获取临时code
3. 传入后端自定义接口调用code2Session 完成登录
4. 后端判断用户数据是否存在，不存在则存储用户数据
5. 返回 access_token
6. 小程序记录 access_token
7. 小程序传入access_token，调用获取用户信息的接口，存储用户信息

## 用户信息管理

### 检查是否为登录状态

1. access_token 存在并且没有过期
2. 能调用获取用户信息的接口，并更新用户信息

### 登录状态恢复

1. 检查登录状态
2. 如果检查结果为未登录调用一键登录的逻辑

### 退出登录

清空保存的 token 和用户信息



