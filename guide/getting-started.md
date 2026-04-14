# 快速开始

## 环境准备

### 1. 安装依赖

```bash
npm init -y
npm install axios xml2js iconv-lite crypto
```

### 2. 配置参数

```javascript
const config = {
  // 机构号
  insCd: '08XXXXXXX',
  // 商户号
  mchntCd: '000XXXXXXX',
  // 终端号（无真实终端统一填88888888）
  termId: '88888888',
  // 回调地址（需外网可访问）
  notifyUrl: 'https://your-domain.com/callback',
  // RSA私钥（PKCS#8格式）
  privateKey: '-----BEGIN PRIVATE KEY-----\n...\n-----END PRIVATE KEY-----',
  // 富友公钥
  fuiouPublicKey: '-----BEGIN PUBLIC KEY-----\n...\n-----END PUBLIC KEY-----'
};
```

### 3. 创建客户端

```javascript
const { FuiouPay } = require('./fuioupay');

const payClient = new FuiouPay({
  insCd: '08XXXXXXX',
  mchntCd: '000XXXXXXX',
  termId: '88888888',
  privateKey: 'your-private-key',
  publicKey: 'fuiou-public-key',
  notifyUrl: 'https://your-domain.com/callback',
  // 生产环境
  baseUrl: 'https://spay-cloud2.fuioupay.com'
});
```

## 接口调用

### 统一下单（主扫）

```javascript
// 生成订单
const order = await payClient.createOrder({
  goodsDes: '测试商品',
  orderAmt: 1,    // 元，会自动转分
  orderType: 'WECHAT'
});

// 返回二维码内容
console.log(order.qrCode);  // 用于展示二维码
console.log(order.mchntOrderNo);  // 商户订单号
```

### 条码支付（被扫）

```javascript
// 扫码收款
const result = await payClient.barcodePay({
  goodsDes: '测试商品',
  orderAmt: 1,
  orderType: 'WECHAT',
  authCode: '134XXXXXXXXX'  // 用户支付宝/微信付款码
});

// 返回支付结果
// 如果返回pending，需要轮询查询
if (result.status === 'pending') {
  // 轮询查询
  const queryResult = await payClient.queryOrder(result.mchntOrderNo);
}
```

### 订单查询

```javascript
const result = await payClient.queryOrder({
  mchntOrderNo: '20240101120000001',
  orderType: 'WECHAT'
});

console.log(result.transStat);  // SUCCESS/FAIL/NOTPAY
```

### 退款

```javascript
const result = await payClient.refund({
  mchntOrderNo: '20240101120000001',
  orderType: 'WECHAT',
  totalAmt: 1,      // 原订单金额（元）
  refundAmt: 0.5    // 退款金额（元）
});
```

### 处理回调

```javascript
app.post('/callback', async (req, res) => {
  const xmlData = req.body;  // 获取XML报文体

  // 解析XML
  const data = await payClient.parseXml(xmlData);

  // 验签
  const isValid = payClient.verifySign(data);
  if (!isValid) {
    return res.send('FAIL');
  }

  // 处理业务
  if (data.result_code === '000000') {
    // 更新订单状态
    await updateOrderStatus(data.mchnt_order_no, 'success');
  }

  // 返回成功
  res.send('1');
});
```

## 交易金额转换

```javascript
// 元转分
function yuanToFen(yuan) {
  return Math.round(yuan * 100);
}

// 分转元
function fenToYuan(fen) {
  return (fen / 100).toFixed(2);
}
```

## 订单号生成

```javascript
// 商户订单号规则：机构编码 + 日期 + 随机数
function generateOrderNo(insCd) {
  const date = new Date();
  const dateStr = date.getFullYear().toString() +
    (date.getMonth() + 1).toString().padStart(2, '0') +
    date.getDate().toString().padStart(2, '0');
  const random = Math.floor(Math.random() * 9000000000) + 1000000000;
  return insCd + dateStr + random;
}
```

## 常用接口地址

| 环境 | 地址 |
|------|------|
| 生产地址1 | https://spay-cloud.fuioupay.com |
| 生产地址2 | https://spay-cloud2.fuioupay.com |
| 测试地址 | https://fundwx.payfuiouo2o.com |

### 接口路径
- 条码支付: `/micropay`
- 统一下单: `/preCreate`
- 订单查询: `/commonQuery`
- 退款申请: `/commonRefund`
- 退款查询: `/commonRefundQuery`
- 关闭订单: `/close`
- 撤销: `/reverse`
