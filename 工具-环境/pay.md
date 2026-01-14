将会员时长放到 token 中，校验时检验

```jsx
创建订单：
POST /api/orders
请求体：
{
  "plan_id": 2,
  "payment_channel": "alipay"
}
返回：
{
  "order_id": 123,
  "payment_url": "<https://pay.example.com/xxx>"
}

查询订单状态：
GET /api/orders/{order_id}
返回：
{
  "status": "pending" | "paid" | "failed"
}

支付回调-三方：
POST /api/payments/callback
请求体（视支付平台而定）：
{
  "order_id": 123,
  "status": "success",
  "channel": "alipay"
}
服务端操作：
- 修改订单状态为 paid
- 更新 user 表 usage_expire_time 为 max(当前时间, 原始 usage_expire_time) + plan.duration_days
```