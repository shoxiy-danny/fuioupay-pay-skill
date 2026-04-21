---
name: fuioupay-examples-frontend
description: "富友支付主扫（统一下单）的前端轮询实现指南，包括轮询间隔和超时处理。"
trigger:
  - 前端
  - 轮询
  - polling
  - 前端轮询
  - 支付轮询
  - jsapi
  - poll
---

# 前端轮询指南

## 主扫支付前端轮询

主扫（统一下单）返回二维码后，需要前端轮询查询订单状态。

```javascript
// 主扫支付 - 前端轮询
async function pollOrderStatus(orderNo, mchntCd, orderType) {
  // 下单后3秒开始轮询
  setTimeout(() => {
    const interval = setInterval(async () => {
      try {
        const response = await fetch('/api/queryOrder', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ mchntCd, orderType, mchntOrderNo: orderNo })
        });
        const data = await response.json();

        if (data.success) {
          if (data.trans_stat === 'SUCCESS' || data.trans_stat === 'TRADE_SUCCESS') {
            clearInterval(interval);
            showSuccess('支付成功');
          } else if (data.trans_stat === 'FAIL' || data.trans_stat === 'TRADE_CLOSED') {
            clearInterval(interval);
            showError('订单已关闭或支付失败');
          }
          // NOTPAY/WAIT_BUYER_PAY 继续轮询
        }
      } catch (e) {
        console.error('查询异常:', e);
      }
    }, 5000); // 每5秒查一次
  }, 3000); // 下单后3秒开始
}
```

## 条码支付结果展示

条码支付如果是同步返回成功，前端直接展示结果；如果是pending状态，服务端会进行60秒轮询。

```javascript
// 条码支付 - 前端处理
async function barcodePay(authCode) {
  const response = await fetch('/api/barcodePay', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      mchntCd: '商户号',
      orderType: 'WECHAT',
      orderAmt: 1,
      goodsDes: '商品描述',
      authCode: authCode
    })
  });

  const data = await response.json();

  if (data.status === 'success') {
    showSuccess('支付成功');
  } else if (data.status === 'pending') {
    showInfo('等待支付中...');
    // 服务端正在进行60秒轮询，前端可以展示loading状态
  } else if (data.status === 'failed') {
    showError(data.message || '支付失败');
  }
}
```

## 订单号生成规则

```javascript
// 商户订单号生成规则
function generateOrderNo(insCd) {
  // 格式：机构编码 + 日期(yyyyMMdd) + 10位随机数
  const date = new Date();
  const dateStr = date.getFullYear().toString() +
    (date.getMonth() + 1).toString().padStart(2, '0') +
    date.getDate().toString().padStart(2, '0');
  const random = Math.floor(Math.random() * 9000000000) + 1000000000;
  return insCd + dateStr + random;
}

// 退款单号生成规则
function generateRefundOrderNo(origOrderNo) {
  // 格式：原订单号 + _R + 2位随机数
  // 总长度不超过30字符
  const random2 = Math.floor(Math.random() * 100).toString().padStart(2, '0');
  const maxOrigLen = 30 - 3; // 预留 "_R" + 2位数字
  const truncated = origOrderNo.length > maxOrigLen
    ? origOrderNo.substring(0, maxOrigLen)
    : origOrderNo;
  return `${truncated}_R${random2}`;
}
```

## 条码格式参考

| 渠道 | 条码格式 | 示例 |
|------|----------|------|
| 微信 | 18位数字，以13/15/16/18开头 | `13XXXXXXXXXXXXXXX` |
| 支付宝 | 16-32位数字或字母 | `28XXXXXXXXXXXXXXXX` |
| 云闪付 | 18位数字 | `62XXXXXXXXXXXXXXXX` |
