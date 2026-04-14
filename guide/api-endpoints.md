# 接口地址汇总

## 生产环境

| 接口 | 地址 |
|------|------|
| **地址1** | `https://spay-cloud.fuioupay.com` |
| **地址2** | `https://spay-cloud2.fuioupay.com` |

两个地址均支持HTTP和HTTPS。

## 测试环境

| 接口 | 地址 |
|------|------|
| **测试地址** | `https://fundwx.payfuiouo2o.com` |

## 接口路径

| 接口 | 路径 | 说明 |
|------|------|------|
| 条码支付 | `/micropay` | 商户扫用户二维码 |
| 统一下单 | `/preCreate` | 用户扫商户二维码 |
| 订单查询 | `/commonQuery` | 查询订单状态 |
| 退款申请 | `/commonRefund` | 发起退款 |
| 退款查询 | `/commonRefundQuery` | 查询退款状态 |
| 关闭订单 | `/close` | 关闭未支付订单 |
| 撤销 | `/reverse` | 撤销交易 |
| 历史查询 | `/hisQuery` | 查询3天前订单 |

## 完整URL示例

### 生产环境

```
# 条码支付
https://spay-cloud2.fuioupay.com/micropay

# 统一下单
https://spay-cloud2.fuioupay.com/preCreate

# 订单查询
https://spay-cloud2.fuioupay.com/commonQuery

# 退款申请
https://spay-cloud2.fuioupay.com/commonRefund

# 退款查询
https://spay-cloud2.fuioupay.com/commonRefundQuery
```

### 测试环境

```
# 条码支付
https://fundwx.payfuiouo2o.com/micropay

# 统一下单
https://fundwx.payfuiouo2o.com/preCreate

# 订单查询
https://fundwx.payfuiouo2o.com/commonQuery

# 退款申请
https://fundwx.payfuiouo2o.com/commonRefund
```

## 回调地址

回调地址不是在接口地址中配置的，而是在请求参数中通过 `notify_url` 字段指定的。

```xml
<notify_url>https://your-domain.com/callback</notify_url>
```

### 回调地址要求

1. 必须是可以外网访问的HTTPS地址
2. 回调是POST请求
3. 回调参数在XML报文中，参数名为`req`
4. 商户接收成功处理后返回**数字1**

## 主备切换

富友接口支持主备切换，当主地址不可用时自动切换到备用地址。

建议在代码中实现自动切换：

```javascript
const endpoints = [
  'https://spay-cloud.fuioupay.com',
  'https://spay-cloud2.fuioupay.com'
];

async function requestWithFailover(path, data) {
  for (const base of endpoints) {
    try {
      const url = `${base}${path}`;
      const response = await sendRequest(url, data);
      return response;
    } catch (e) {
      console.error(`${base} 失败，尝试下一个...`);
    }
  }
  throw new Error('所有地址均失败');
}
```

## 测试参数

> 注意：测试环境参数仅供测试使用。

| 参数 | 测试值 | 说明 |
|------|--------|------|
| 商户号 | `0002900F0370542` | |
| 机构号 | `08A9999999` | |
| Appid(微信) | `wxfa089da95020ba1a` | |
| 终端号 | `88888888` | 无真实终端时使用 |
| 私钥 | 见下方 | 用于请求签名 |
| 公钥 | 见下方 | 用于响应验签 |

### 测试私钥

```
MIICdgIBADANBgkqhkiG9w0BAQEFAASCAmAwggJcAgEAAoGBAJgAzD8fEvBHQTyxUEeK963mjziMWG7nxpi+pDMdtWiakc6xVhhbaipLaHo4wVI92A2wr3ptGQ1/YsASEHm3m2wGOpT2vrb2Ln/S7lz1ShjTKaT8U6rKgCdpQNHUuLhBQlpJer2mcYEzG/nGzcyalOCgXC/6CySiJCWJmPyR45bJAgMBAAECgYBHFfBvAKBBwIEQ2jeaDbKBIFcQcgoVa81jt5xgz178WXUg/awu3emLeBKXPh2i0YtN87hM/+J8fnt3KbuMwMItCsTD72XFXLM4FgzJ4555CUCXBf5/tcKpS2xT8qV8QDr8oLKA18sQxWp8BMPrNp0epmwun/gwgxoyQrJUB5YgZQJBAOiVXHiTnc3KwvIkdOEPmlfePFnkD4zzcv2UwTlHWgCyM/L8SCAFclXmSiJfKSZZS7o0kIeJJ6xe3Mf4/HSlhdMCQQCnTow+TnlEhDTPtWa+TUgzOys83Q/VLikqKmDzkWJ7I12+WX6AbxxEHLD+THn0JGrlvzTEIZyCe0sjQy4LzQNzAkEAr2SjfVJkuGJlrNENSwPHMugmvusbRwH3/38ET7udBdVdE6poga1Z0al+0njMwVypnNwy+eLWhkhrWmpLh3OjfQJAI3BV8JS6xzKh5SVtn/3Kv19XJ0tEIUnn2lCjvLQdAixZnQpj61ydxie1rggRBQ/5vLSlvq3H8zOelNeUF1fT1QJADNo+tkHVXLY9H2kdWFoYTvuLexHAgrsnHxONOlSA5hcVLd1B3p9utOt3QeDf6x2i1lqhTH2w8gzjv snx13tWqg==
```

### 测试公钥

```
MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQCBv9K+jiuHqXIehX81oyNSD2RfVn+KTPb7NRT5HDPFE35CjZJd7Fu40r0U2Cp7Eyhayv/mRS6ZqvBT/8tQqwpUExTQQBbdZjfk+efb9bF9a+uCnAg0RsuqxeJ2r/rRTsORzVLJy+4GKcv06/p6CcBc5BI1gqSKmyyNBlgfkxLYewIDAQAB
```
