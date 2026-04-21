---
name: fuioupay-guide-signature
description: "富友支付接口的RSA签名算法详解，包括签名原文构建、Node.js实现和常见错误。"
trigger:
  - 签名
  - signature
  - RSA
  - MD5WithRSA
  - buildSignContent
  - sign
---

# 签名详解

## 签名算法

- **非对称加密**: RSA 1024bit
- **签名算法**: MD5WithRSA
- **字符编码**: GBK

## 签名流程

### 1. 构建签名原文

将所有参与签名的参数（sign字段和reserved开头字段除外）按以下规则处理：

1. **排除字段**: `sign` 和所有 `reserved` 开头的字段
2. **排序**: 按字段名的ASCII码从小到大排序
3. **拼接**: 按 `key1=value1&key2=value2&...` 格式拼接
4. **编码**: 使用GBK编码

### 2. 计算签名

使用RSA私钥对签名原文进行MD5WithRSA签名，输出Base64编码。

### 3. 签名原文示例

假设请求参数如下：

```xml
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
</xml>
```

**签名原文（已排序）：**

```
addn_inf=&curr_type=&goods_des=卡盟测试&goods_detail=&goods_tag=&ins_cd=08A9999999&mchnt_cd=0002900F0370542&mchnt_order_no=202012011518020724446&notify_url=https://mail.qq.com/cgi-bin/frame_html?sid=pEYG5nBgQiNVqANe&r=4a6c47ad7d279a80630dec073cda96e2&order_amt=1&order_type=WECHAT&random_str=d0194c1024f180065d2434fa8b6a2f82&term_id=12345678&term_ip=127.0.0.1&txn_begin_ts=20201201151802&version=1
```

## Node.js签名实现

```javascript
const crypto = require('crypto');
const iconv = require('iconv-lite');

/**
 * 构建签名原文
 * @param {Object} params - 请求参数对象
 * @returns {string} 签名原文
 */
function buildSignContent(params) {
  // 1. 获取所有key并排序
  const sortedKeys = Object.keys(params).sort();

  // 2. 排除sign和reserved开头字段，拼接key=value
  const parts = [];
  for (const key of sortedKeys) {
    if (key === 'sign' || key.startsWith('reserved')) {
      continue;
    }
    const value = params[key] !== undefined && params[key] !== null ? params[key] : '';
    parts.push(`${key}=${value}`);
  }

  // 3. 返回用&拼接的字符串
  return parts.join('&');
}

/**
 * RSA签名
 * @param {string} content - 签名原文
 * @param {string} privateKey - PKCS#8格式私钥
 * @returns {string} Base64编码的签名
 */
function signWithPrivateKey(content, privateKey) {
  // 1. 使用GBK编码
  const contentBuffer = iconv.encode(content, 'gbk');

  // 2. 创建signer
  const sign = crypto.createSign('RSA-MD5');

  // 3. 更新内容
  sign.update(contentBuffer);

  // 4. 签名（Base64输出）
  return sign.sign(privateKey, 'base64');
}

/**
 * RSA验签
 * @param {string} content - 签名原文
 * @param {string} signature - Base64编码的签名
 * @param {string} publicKey - 富友公钥
 * @returns {boolean} 验签结果
 */
function verifyWithPublicKey(content, signature, publicKey) {
  try {
    // 1. 清理签名（移除空格和换行）
    const cleanSignature = signature.replace(/\s/g, '');

    // 2. 使用GBK编码
    const contentBuffer = iconv.encode(content, 'gbk');

    // 3. 创建verifier
    const verify = crypto.createVerify('RSA-MD5');
    verify.update(contentBuffer);

    // 4. 验签
    return verify.verify(publicKey, cleanSignature, 'base64');
  } catch (e) {
    console.error('验签失败:', e.message);
    return false;
  }
}
```

## 验签注意事项

### 1. 响应中的sign验签

富友返回的sign字段可能包含**空格或换行符**，验签前必须清理：

```javascript
// 错误写法
const isValid = verifyWithPublicKey(content, data.sign, publicKey);

// 正确写法
const cleanSign = data.sign.replace(/\s/g, '');
const isValid = verifyWithPublicKey(content, cleanSign, publicKey);
```

### 2. 响应验签原文构建

响应报文验签时，同样需要排除sign和reserved开头字段，按ASCII排序拼接：

```javascript
// 响应参数
{
  ins_cd: '08A9999999',
  mchnt_cd: '0002900F0370542',
  result_code: '000000',
  result_msg: 'SUCCESS',
  sign: '...'
}

// 验签原文
ins_cd=08A9999999&mchnt_cd=0002900F0370542&result_code=000000&result_msg=SUCCESS
```

## 常见签名错误

| 错误原因 | 解决方案 |
|----------|----------|
| 编码错误 | 必须使用GBK编码，不要用UTF-8 |
| 排序错误 | 按ASCII码排序，不是按字典序 |
| 遗漏字段 | 非必填字段也要参与签名（除reserved开头） |
| sign未排除 | sign字段不参与签名 |
| 签名原文格式错误 | 必须是key=value&key=value格式 |
