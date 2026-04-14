# 常见问题与避坑指南

## 验签相关

### 1. 验签失败但业务成功

**问题描述**: 富友返回的sign字段可能包含空格（换行符），导致验签失败。

**错误代码示例**:
```javascript
// 验签失败
const isValid = verifyWithPublicKey(content, data.sign, publicKey);
console.log(isValid); // false
console.log(data.sign); // "MIGfMA0GCSqGSIb3...\n "
```

**解决方案**: 验签前过滤空格和换行符
```javascript
const cleanSign = data.sign.replace(/\s/g, '');
const isValid = verifyWithPublicKey(content, cleanSign, publicKey);
```

**经验来源**: scanpay项目退款接口，1032订单已全额退款时验签可能失败但业务结果明确。

---

### 2. 响应验签不通过

**问题描述**: 自己计算的验签原文与富友返回的不一致。

**可能原因**:
1. 字段未按ASCII码排序
2. 使用了错误的编码（UTF-8而非GBK）
3. 遗漏了某些字段
4. reserved开头字段不应该参与验签

**排查步骤**:
```javascript
// 1. 确认字段排序
console.log('参与验签的字段:', sortedKeys);

// 2. 确认编码
console.log('编码后的字节:', iconv.encode(content, 'gbk'));

// 3. 对比签名原文
console.log('生成的验签原文:', signContent);
console.log('富友提供的签名原文:', originalSignContent);
```

---

## 金额相关

### 3. 金额单位错误

**问题描述**: 提交金额为1，实际扣款1分钱。

**原因**: 富友接口金额单位是**分**，不是元。

**解决方案**:
```javascript
// 错误：传入元
orderAmt: 1  // 实际扣款1分

// 正确：传入分
orderAmt: Math.round(1 * 100)  // 实际扣款1元
```

---

### 4. 退款金额超限

**问题描述**: 退款金额不能超过订单总金额。

**解决方案**:
```javascript
const totalAmtInt = Math.round(totalAmt * 100);  // 分
const refundAmtInt = Math.round(refundAmt * 100); // 分

if (refundAmtInt > totalAmtInt) {
  return { success: false, message: '退款金额不能超过订单总金额' };
}
```

---

## 订单相关

### 5. 商户订单号重复

**问题描述**: 报错"商户订单号重复"。

**原因**: 订单号必须全局唯一，不能重复使用。

**解决方案**:
```javascript
// 生成唯一订单号
function generateOrderNo(insCd) {
  const dateStr = moment().format('YYYYMMDD');
  const random = Math.random().toString().substring(2, 12);
  return `${insCd}${dateStr}${random}`;
}
```

---

### 6. 条码支付状态不确定

**问题描述**: 条码支付返回"用户支付中"，无法确定最终状态。

**原因**: 部分交易需要用户输入密码，需要主动查询。

**解决方案**: 轮询查询订单状态

```javascript
async function barcodePayWithQuery(params) {
  // 1. 发起条码支付
  const payResult = await barcodePay(params);

  // 2. 如果是pending，开始轮询
  if (payResult.result_code === '000001') {
    let elapsed = 0;
    const interval = 5000; // 5秒查一次

    while (elapsed < 60000) { // 最多查询60秒
      await new Promise(r => setTimeout(r, interval));
      elapsed += interval;

      const queryResult = await queryOrder(payResult.mchnt_order_no);

      if (queryResult.trans_stat === 'SUCCESS') {
        return { success: true, status: 'success', ...queryResult };
      }
      if (queryResult.trans_stat === 'FAIL') {
        return { success: false, status: 'failed', ...queryResult };
      }
    }

    return { success: false, status: 'timeout' };
  }

  return payResult;
}
```

**经验**: 交易发起5秒后再开始查询，间隔5秒以上。

---

### 7. 新商户无法交易

**问题描述**: 报错"无效商户"。

**原因**: 新开通提现和退货的商户，需要15分钟后才能交易。

**解决方案**: 这是富友的限制，无法绕过，只能等待。

---

### 8. 订单查询无记录

**问题描述**: 查询接口返回"订单不存在"。

**可能原因**:
1. 订单超过3天
2. 订单号不匹配
3. 商户号/机构号错误

**解决方案**:
- 3天内的订单用订单查询接口
- 3天前的订单用历史订单查询接口

---

## 退款相关

### 9. 退款单号长度超限

**问题描述**: 退款单号格式不符合要求。

**要求**: 5-30字符，只能包含字母、数字、下划线。

**解决方案**:
```javascript
function generateRefundOrderNo(origOrderNo) {
  const random2 = Math.floor(Math.random() * 100).toString().padStart(2, '0');
  const maxOrigLen = 30 - 3; // 预留 "_R" + 2位数字
  const truncatedOrderNo = origOrderNo.length > maxOrigLen
    ? origOrderNo.substring(0, maxOrigLen)
    : origOrderNo;
  return `${truncatedOrderNo}_R${random2}`;
}
```

**经验来源**: scanpay项目退款接口开发。

---

### 10. 退款失败-账户余额不足

**问题描述**: 报错"商户账户余额不足"。

**原因**: 富友退款要求商户账户有足够的可退资金。

**解决方案**: 确保商户账户有足够的退款资金。

---

## 编码相关

### 11. 中文乱码

**问题描述**: 富友返回的中文显示乱码。

**原因**: 编解码使用了错误的字符集。

**解决方案**:
```javascript
// 解码富友响应
const decoded = iconv.decode(responseBuffer, 'gbk');

// 编码请求
const encoded = iconv.encode(requestXml, 'gbk');
```

---

### 12. URL编码错误

**问题描述**: 提交数据中文变乱码。

**原因**: 中文需要进行两次GBK URL编码。

**解决方案**:
```javascript
// 两次GBK URL编码
function doubleEncode(str) {
  return gbkUrlEncode(gbkUrlEncode(str));
}

// GBK URL编码
function gbkUrlEncode(str) {
  const buf = iconv.encode(str, 'gbk');
  let encoded = '';
  for (let i = 0; i < buf.length; i++) {
    const b = buf[i];
    if ((b >= 0x61 && b <= 0x7A) || (b >= 0x41 && b <= 0x5A) ||
        (b >= 0x30 && b <= 0x39) || b === 0x2D || b === 0x5F ||
        b === 0x2E || b === 0x2A || b === 0x27 || b === 0x28 || b === 0x29) {
      // 字母数字和特定符号不编码
      encoded += String.fromCharCode(b);
    } else if (b === 0x20) {
      encoded += '+';
    } else {
      encoded += '%' + b.toString(16).toUpperCase().padStart(2, '0');
    }
  }
  return encoded;
}
```

---

## 回调相关

### 13. 回调处理失败

**问题描述**: 回调返回FAIL后富友继续回调。

**原因**: 回调处理失败时需要返回数字1。

**解决方案**:
```javascript
app.post('/callback', async (req, res) => {
  try {
    const data = parseXml(req.body);

    // 验签
    if (!verifySign(data)) {
      return res.send('FAIL'); // 验签失败，返回FAIL
    }

    // 处理业务
    await handleCallback(data);

    // 成功返回数字1
    res.send('1');
  } catch (e) {
    console.error('回调处理异常:', e);
    res.send('FAIL');
  }
});
```

**重要**: 最多回调6次，每次间隔30秒。

---

### 14. 重复回调

**问题描述**: 同一订单收到多次回调。

**解决方案**:
1. 确保幂等处理
2. 返回成功后再处理业务逻辑
3. 使用数据库事务避免重复处理

```javascript
async function handleCallback(data) {
  // 使用事务确保幂等
  await db.transaction(async () => {
    const order = await db.getOrder(data.mchnt_order_no);
    if (order.status === 'success') {
      return; // 已处理，跳过
    }
    await db.updateOrderStatus(data.mchnt_order_no, 'success');
  });
}
```

---

## 其他

### 15. 服务重启后签名失败

**问题描述**: 服务重启后签名结果与之前不一致。

**原因**: iconv-lite等模块未正确加载。

**经验来源**: scanpay项目PM2进程问题。

**解决方案**:
```javascript
// 启动时检查模块
function checkDependencies() {
  try {
    iconv.encode('测试', 'gbk');
    console.log('[OK] iconv-lite');
  } catch (e) {
    console.error('[ERROR] iconv-lite:', e.message);
    process.exit(1);
  }
}
```

---

### 16. 签名工具推荐

富友提供了在线签名工具:
- 签名计算工具
- RSA密钥对生成工具（生成1024bit、2048bit PKCS#8格式密钥对）
