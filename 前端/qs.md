

```
  mail:

    host: smtp.qq.com

    port: 465

    username: 2061867903@qq.com

    password: wddujbtgasmycjfc

    protocol: smtps
```

username email phone password

发送验证码，一分钟内只能发送一次，验证码 5 分钟内有效，验证码存入 redis

注册，手机号，邮箱，验证码，密码

登录

忘记密码-todo

查询用户信息

修改用户信息

curl -X POST -H "Content-Type: application/json" -d '{"email": "n199m992@163.com"}' http://127.0.0.1:8888/api/sendcode

```
密码输入采用隐藏式输入，点击旁边小眼睛按钮可以显示密码
所有信息包括验证码输入之后才能点击注册按钮
登录和注册都需要对格式进行检验,密码也需要满足长度，没有满足则输入框红色提示
所有消息通知使用ElMessage,可以参考其他页面的使用
发送验证码之后显示 60 秒倒计时才能再次发送

# todo
修改密码中发送验证码应该验证是否已经注册

查询目录-查询产品
新增目录，修改目录
删除目录
新增产品
修改产品
删除产品

```

页面切换动画

```
<Transition
  name="fade-transform"
  mode="out-in"
  appear
>
  <keep-alive :include="keepAliveComponents">
    <component :is="Component" :key="route.path" />
  </keep-alive>
</Transition>

/* 页面切换动画 */
.fade-transform-leave-active,
.fade-transform-enter-active {
  transition: all 0.5s;
}

.fade-transform-enter-from {
  opacity: 0;
  transform: translateX(-30px);
}

.fade-transform-leave-to {
  opacity: 0;
  transform: translateX(30px);
}
```

