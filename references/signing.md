---
name: fuioupay-signing
description: "富友支付签名算法：RSA 1024bit + MD5WithRSA，GBK编码，签名原文构建规则，Node.js实现。"
trigger:
  - 签名
  - signature
  - RSA
  - MD5WithRSA
  - buildSignContent
  - 签名原文
  - 签名算法
---

# 富友支付 - 签名机制

## 算法规格

| 项目 | 说明 |
|------|------|
| 算法 | RSA 1024bit + MD5WithRSA |
| 编码 | GBK |
| 签名原文格式 | `key1=value1&key2=value2&...` |
| 签名值编码 | Base64 |
| 签名方向 | 商户使用RSA私钥签名，富友用公钥验签 |

## 签名流程

### 第一步：构建签名原文

将所有参与签名的参数按以下规则处理：

1. **排除字段**：`sign` 字段和所有 `reserved` 开头的字段不参与签名
2. **排序**：按字段名的ASCII码从小到大排序
3. **拼接**：`key1=value1&key2=value2&...` 格式拼接
4. **编码**：使用GBK编码

### 第二步：计算签名

使用商户的RSA私钥（PKCS#8格式）对签名原文进行MD5WithRSA签名，输出Base64编码。

### 第三步：发送请求

将签名值放入请求参数的 `sign` 字段中。

## Node.js 实现

```javascript
const crypto = require('crypto');
const iconv = require('iconv-lite');

/**
 * 构建签名原文
 * @param {Object} params - 请求参数对象
 * @returns {string} GBK编码的签名原文
 */
function buildSignContent(params) {
  // 1. 过滤掉 sign 和 reserved 开头字段
  const filtered = Object.entries(params)
    .filter(([key]) => key !== 'sign' && !key.startsWith('reserved'))
    .sort(([a], [b]) => a.localeCompare(a, 'ASCII'))
    .sort(([a], [b]) => a < b ? -1 : a > b ? 1 : 0);
  
  // 按ASCII排序
  const sorted = Object.keys(params).sort();
  const parts = [];
  for (const key of sorted) {
    if (key === 'sign' || key.startsWith('reserved')) continue;
    const value = params[key] !== undefined && params[key] !== null 
      ? String(params[key]) 
      : '';
    parts.push(`${key}=${value}`);
  }
  
  // GBK编码
  const signContent = parts.join('&');
  return iconv.encode(signContent, 'gbk');
}

/**
 * RSA签名
 * @param {string} content - GBK编码的签名原文
 * @param {string} privateKey - PKCS#8格式RSA私钥
 * @returns {string} Base64编码的签名值
 */
function sign(content, privateKey) {
  const sign = crypto.createSign('MD5WithRSA');
  sign.update(content);
  return sign.sign(privateKey, 'base64');
}
```

## 常见错误

| 错误做法 | 正确做法 |
|----------|----------|
| UTF-8编码 | GBK编码 |
| 字段未排序 | 按ASCII排序 |
| sign参与签名 | sign字段不参与 |
| reserved字段参与签名 | reserved开头字段不参与 |

## 响应验签

富友返回的响应同样带有 `sign` 字段，验签时需：

1. 取出 `sign` 值并清除可能的空格/换行符
2. 将除 `sign` 外的所有参数字段按同样规则构建验签原文
3. 用富友提供的公钥验签

```javascript
// 清除sign中的空格和换行符（常见问题！）
const cleanSign = data.sign.replace(/\s/g, '');
```

详见 [常见问题与避坑指南](../guide/troubleshooting.md)。
