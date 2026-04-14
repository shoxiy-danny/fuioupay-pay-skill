# 富友支付错误码对照表

## 通用错误码

| 错误码 | 说明 | 处理方式 |
|--------|------|----------|
| 000000 | 成功 | 处理成功 |
| 000001 | 业务处理中 | 条码支付需轮询 |
| 010003 | Parameter error | 检查参数 |
| 010004 | Signature verification failed | 检查签名 |
| 010005 | Merchant not found | 检查商户号 |
| 010006 | Order not found | 订单不存在 |
| 010007 | Order already paid | 订单已支付 |
| 010008 | Order expired | 订单过期 |

## 条码支付特有码

| 错误码 | pay_msg | 说明 | 处理 |
|--------|---------|------|------|
| 000000 | SUCCESS | 支付成功 | 直接返回成功 |
| 000000 | （空） | 等待用户输入密码 | 进入60秒轮询 |
| 000001 | WAIT_BUYER_PAY | 等待用户支付 | 进入60秒轮询 |
| 030010 | 等待用户支付中 | 用户扫码成功等待输入密码 | 进入60秒轮询 |

## 条码支付result_code判断顺序

```
1. result_code=000000 且 pay_msg=SUCCESS → 成功
2. result_code in [000001, 030010, 000000] → 进入60秒轮询
3. 其他 → 失败
```

## 订单查询trans_stat状态码

| 状态码 | 说明 |
|--------|------|
| SUCCESS / TRADE_SUCCESS | 支付成功 |
| FAIL / TRADE_CLOSED | 支付失败/已关闭 |
| NOTPAY / WAIT_BUYER_PAY | 待支付 |
| REFUND | 已退款 |
| REVOKED | 已撤销 |
| USERPAYING | 用户支付中 |
| PAYERROR | 支付失败 |

## 退款查询trans_stat状态码

| 状态码 | 说明 |
|--------|------|
| SUCCESS | 退款成功 |
| PAYERROR | 退款失败 |
