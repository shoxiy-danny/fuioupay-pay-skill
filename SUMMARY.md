# 富友支付对接 Skill - 完整目录

## 主入口
- [skill.md](skill.md) - 主入口文件

## 概述与签名
- [docs/overview.md](docs/overview.md) - 完整概述、术语定义、签名说明、编码规则

## 接口文档

### 交易接口
- [docs/3.1-barcode-pay.md](docs/3.1-barcode-pay.md) - 条码支付（被扫）
- [docs/3.2-qrcode-pay.md](docs/3.2-qrcode-pay.md) - 主扫统一下单
- [docs/3.3-jsapi-pay.md](docs/3.3-jsapi-pay.md) - 公众号/服务窗/小程序下单
- [docs/3.4-order-query.md](docs/3.4-order-query.md) - 订单查询
- [docs/3.5-notify.md](docs/3.5-notify.md) - 支付结果通知（回调）
- [docs/3.6-refund.md](docs/3.6-refund.md) - 退款申请
- [docs/3.7-close.md](docs/3.7-close.md) - 关闭订单
- [docs/3.8-revoke.md](docs/3.8-revoke.md) - 撤销接口
- [docs/3.9-refund-query.md](docs/3.9-refund-query.md) - 退款查询
- [docs/3.10-history-query.md](docs/3.10-history-query.md) - 历史订单查询

### 辅助接口
- [docs/3.11-auth-code.md](docs/3.11-auth-code.md) - 授权码查询openid
- [docs/3.12-servicer-openid.md](docs/3.12-servicer-openid.md) - 服务商模式获取openid
- [docs/3.13-delegate-pay.md](docs/3.13-delegate-pay.md) - 微信委托代扣
- [docs/3.14-delegate-query.md](docs/3.14-delegate-query.md) - 微信委托代扣查询
- [docs/3.15-fee-push.md](docs/3.15-fee-push.md) - 交易手续费推送

## 对接指南
- [guide/getting-started.md](guide/getting-started.md) - 快速开始
- [guide/signature.md](guide/signature.md) - 签名详解（含示例）
- [guide/encoding.md](guide/encoding.md) - 编码与URLencode详解
- [guide/troubleshooting.md](guide/troubleshooting.md) - 常见问题与避坑指南
- [guide/api-endpoints.md](guide/api-endpoints.md) - 接口地址汇总

## 代码示例
- [examples/nodejs.md](examples/nodejs.md) - Node.js完整示例

## 术语表

| 术语 | 说明 |
|------|------|
| 主扫 | 用户主动扫付款二维码（扫码支付） |
| 被扫 | 商户扫用户二维码（条码支付） |
| 固码 | 台卡、桌贴（统一下单） |
| 刷卡 | 支付宝当面付，用户出示条码 |
| 公众号支付 | JSAPI，微信公众号内支付 |
| 条码支付 | barcode_pay，被扫 |
| 统一下单 | pre_create，主扫 |
