# 富友支付 - 概述与签名说明

## 术语定义

| 术语 | 说明 |
|------|------|
| 主扫 | 用户主动扫付款二维码，也叫扫码支付，动态二维码支付 |
| 被扫 | 商户扫用户，也叫条码支付 |
| 固码支付 | 台卡、桌贴。使用公众号、服务窗统一下单 |
| 刷卡支付 | 支付宝称当面付，被扫：用户打开微信/支付宝，受理端扫码 |
| 公众号支付 | 用户进入微信公众号通过JSAPI完成资金划扣 |
| 扫码支付(主扫) | 用户扫商户二维码完成支付 |
| APP支付 | APP集成微信/支付宝SDK，跳转完成支付 |

## 重要规则

1. **交易金额以分为单位**，不带小数点
2. **回调成功返回数字1**，不然回调会继续发起，间隔30S，共回调5次
3. **退款只有当商户账户有资金才能退款**
4. **新开通提现和退货的商户，15分钟后才能进行交易**
5. **订单查询可查询3天内的订单**，超过3天使用历史查询接口
6. **订单状态以查询为主，回调为辅**
7. **商户订单号必须全局唯一**

## 通讯方式

- 通信采用HTTP协议
- POST表单发送/接收XML格式报文
- POST参数名为 `req`
- **报文中有中文需要进行两次encode操作**

```java
// Java编解码示例
URLDecoder.decode(reqXml, "GBK");  // 解码
URLEncoder.encode(rspXml, "GBK");  // 编码
```

## 签名机制

### 算法
- **非对称加密**: RSA 1024bit
- **签名算法**: MD5WithRSA
- **编码**: GBK

### 签名原文计算
1. 将接口文档中的每一个字段（**sign及reserved开头字段除外**）以字典顺序排序
2. 按照 `key1=value1&key2=value2.....` 的顺序拼接
3. 对得到的字符串进行RSA签名

### 重要说明
> sign及reserved开头字段除外的其他非必填字段也需要参与验签。
> 富友会根据业务需求新增reserved开头字段，这些字段都不参与验签。

### 签名示例

**请求参数:**
```xml
<?xml version="1.0" encoding="GBK" standalone="yes"?>
<xml>
    <term_id>12345678</term_id>
    <random_str>d0194c1024f180065d2434fa8b6a2f82</random_str>
    <ins_cd>08A9999999</ins_cd>
    <version>1</version>
    <mchnt_cd>0002900F0370542</mchnt_cd>
    <goods_des>卡盟测试</goods_des>
    <order_amt>1</order_amt>
    <order_type>WECHAT</order_type>
    <term_ip>127.0.0.1</term_ip>
    <txn_begin_ts>20201201151802</txn_begin_ts>
    <sign>DXNr4zU78EF7R/dknRsVYNWwJ29M6l4YMZrOjqZbNYW9m90yq/n76lO4sS4r8rls0...</sign>
</xml>
```

**签名原文:**
```
addn_inf=&curr_type=&goods_des=卡盟测试&goods_detail=asasda&goods_tag=&ins_cd=08A9999999&mchnt_cd=0002900F0370542&mchnt_order_no=202012011518020724446&notify_url=https://mail.qq.com/cgi-bin/frame_html?sid=pEYG5nBgQiNVqANe&r=4a6c47ad7d279a80630dec073cda96e2&order_amt=1&order_type=WECHAT&random_str=d0194c1024f180065d2434fa8b6a2f82&term_id=12345678&term_ip=127.0.0.1&txn_begin_ts=20201201151802&version=1
```

### 测试环境参数

| 参数 | 值 |
|------|-----|
| 测试商户号 | `0002900F0370542` |
| 测试机构号 | `08A9999999` |
| 测试Appid | `wxfa089da95020ba1a`（微信）|

> 注意：测试环境银行间连生产测试账号是生产环境用来进行支付业务体验的账号，交易中产生的一切信息均为生产环境数据。请务必使用小于1元的小金额，并请务必在支付当日完成退款。

## goods_detail字段说明

### 支付宝格式
```json
[{
    "goods_id": "apple-01",           // 商品编号 必填
    "goods_name": "单品",              // 商品名称 必填
    "quantity": "1",                  // 商品数量 必填
    "price": "0.21"                   // 商品单价，单位为元 必填
}]
```

### 微信格式
```json
{
    "cost_price": 608800,             // 订单原价 单位分 非必填
    "receipt_id": "wx123",            // 商品小票ID 非必填
    "goods_detail": [{
        "goods_id": "12345",           // 商品编码 必填
        "goods_name": "iPhone6s 32G",  // 商品名称 非必填
        "quantity": 1,                 // 商品数量 必填 int
        "price": 528800               // 商品单价 单位分 必填 int
    }]
}
```

## 订单类型

| order_type | 说明 |
|------------|------|
| WECHAT | 微信 |
| ALIPAY | 支付宝 |
| UNIONPAY | 银联二维码 |
| BESTPAY | 翼支付 |
| JSAPI | 公众号支付 |
| FWC | 支付宝服务窗/小程序 |
| LETPAY | 微信小程序 |
| MPAY | 云闪付小程序 |
| PY68 | 银联分期-商户贴息 |
| PY69 | 银联分期-持卡人贴息 |
| JDBT | 京东白条 |

## 交易状态 (trans_stat)

| 状态 | 说明 |
|------|------|
| SUCCESS/TRADE_SUCCESS | 交易成功 |
| FAIL/TRADE_CLOSED | 交易失败/已关闭 |
| NOTPAY/WAIT_BUYER_PAY | 未支付 |
