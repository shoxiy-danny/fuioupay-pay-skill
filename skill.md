---
name: fuioupay-pay-skill
display_name: 富友支付对接 Skill
description: 富友支付接口对接指南。包含下单(3.2)、条码支付(3.1)、订单查询(3.4)、退款(3.6)等完整交易流程。涵盖中文编码、签名算法、XML构建、异步轮询、回调处理等核心问题解决方案。
version: 1.0.0
---

# 富友支付对接 Skill

帮助AI智能体快速对接富友支付系统的交易接口。

## 快速索引

### 核心文档
- [概述与签名说明](docs/overview.md)
- [错误码对照表](docs/error-codes.md)
- [快速开始](guide/getting-started.md)
- [签名详解](guide/signature.md)
- [常见问题与避坑](guide/troubleshooting.md)

### 接口文档
- [3.1 条码支付](docs/3.1-barcode-pay.md) - 商户扫用户二维码
- [3.2 主扫统一下单](docs/3.2-qrcode-pay.md) - 用户扫商户二维码
- [3.3 公众号/服务窗下单](docs/3.3-jsapi-pay.md) - JSAPI支付
- [3.4 订单查询](docs/3.4-order-query.md) - 查询订单状态
- [3.5 支付结果通知](docs/3.5-notify.md) - 异步回调
- [3.6 退款申请](docs/3.6-refund.md) - 退款
- [3.7 关闭订单](docs/3.7-close.md) - 关闭未支付订单
- [3.8 撤销接口](docs/3.8-revoke.md) - 撤销交易
- [3.9 退款查询](docs/3.9-refund-query.md) - 查询退款状态
- [3.10 历史订单查询](docs/3.10-history-query.md) - 查询3天前订单
- [3.11 授权码查询openid](docs/3.11-auth-code.md) - 获取用户openid
- [3.12 服务商模式获取openid](docs/3.12-servicer-openid.md) - 服务商模式获取子商户用户openid

### 代码示例
- [Node.js示例](examples/nodejs.md)

## 关键规则

| 规则 | 说明 |
|------|------|
| 金额单位 | **分**，不带小数点 |
| 回调返回 | 成功返回 **数字1** |
| 新商户限制 | 开通后 **15分钟** 才能交易 |
| 订单查询 | 可查 **3天内** 订单 |
| 订单号唯一性 | 必须全局唯一 |

## 接口地址

### 生产环境
- 地址1: `https://spay-cloud.fuioupay.com` (http/https均支持)
- 地址2: `https://spay-cloud2.fuioupay.com` (http/https均支持)

### 测试环境
- 地址: `https://fundwx.payfuiouo2o.com`

## 签名机制

- **算法**: RSA 1024bit + MD5WithRSA
- **编码**: GBK
- **原文**: 按ASCII字典序 `key=value&...`
- **排除**: `sign`字段和`reserved`开头的字段不参与签名

## 通讯方式

- HTTP POST
- Content-Type: `application/x-www-form-urlencoded`
- 参数名: `req`
- XML格式，中文需**两次GBK URL encode**

## 使用示例

```javascript
// 调用统一下单
const result = await payClient.createOrder({
  mchntCd: '商户号',
  orderAmt: 1,      // 元转为分
  orderType: 'WECHAT',
  goodsDes: '商品描述'
});
```

## 避坑重点

1. **验签sign含空格**: 富友返回的sign可能包含换行符，验签前需 `.replace(/\s/g, '')`
2. **退款单号规则**: 5-30字符，只能含字母数字下划线
3. **条码支付异步**: 需要轮询查询，建议5秒后开始，间隔5秒以上
4. **GBK编码**: 必须使用iconv-lite等库处理，不要用默认编码
5. **URL双重encode**: 中文必须进行两次GBK URL编码

## 对接检查清单

- [ ] 私钥格式为PKCS#8（`-----BEGIN PRIVATE KEY-----`）
- [ ] 签名算法使用RSA-MD5
- [ ] 签名原文中空字段保留为`field=`
- [ ] XML中所有非reserved字段都出现，空字段为`<field></field>`
- [ ] 中文字段进行两次GBK URL编码
- [ ] 条码支付时判断`result_code`，非SUCCESS需进入轮询
- [ ] 回调验签时排除reserved开头字段
- [ ] 条码支付服务端等待最多60秒
- [ ] 主扫前端轮询：下单后3秒开始，每5秒一次
- [ ] 金额单位使用分，不是元
