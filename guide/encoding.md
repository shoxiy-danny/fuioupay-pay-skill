---
name: fuioupay-guide-encoding
description: "富友支付接口的GBK编码要求和URL双重编码规则，含Node.js实现示例。"
trigger:
  - 编码
  - GBK
  - URL编码
  - encode
  - urlencode
  - gbk
  - iconv
  - 两次encode
---

# 编码与URL编码详解

## 字符编码

### GBK编码（必须）

富友接口**必须使用GBK编码**，不能使用UTF-8。

```javascript
const iconv = require('iconv-lite');

// 字符串转GBK Buffer
const gbkBuffer = iconv.encode('商品描述', 'gbk');

// GBK Buffer转字符串
const str = iconv.decode(gbkBuffer, 'gbk');
```

### 常见编码错误

| 错误做法 | 正确做法 |
|----------|----------|
| `Buffer.from(str)` | `iconv.encode(str, 'gbk')` |
| `Buffer.from(str, 'utf-8')` | `iconv.encode(str, 'gbk')` |
| `str.toString()` | `iconv.decode(buffer, 'gbk')` |

## URL编码

### 两次GBK URL Encode

**重要**: 报文中有中文需要进行两次encode操作！

```javascript
// 原始XML
const xml = `<?xml version="1.0" encoding="GBK"?>
<xml>
    <goods_des>商品描述</goods_des>
</xml>`;

// 第一次GBK URL编码
const encoded1 = gbkUrlEncode(xml);

// 第二次GBK URL编码
const encoded2 = gbkUrlEncode(encoded1);

// 最终发送
const formData = `req=${encoded2}`;
```

### GBK URL编码实现

```javascript
function gbkUrlEncode(str) {
  // 1. 先转GBK
  const buf = iconv.encode(str, 'gbk');

  let encoded = '';
  for (let i = 0; i < buf.length; i++) {
    const b = buf[i];

    // 2. 字母数字和特定符号不编码
    if ((b >= 0x61 && b <= 0x7A) ||  // a-z
        (b >= 0x41 && b <= 0x5A) ||  // A-Z
        (b >= 0x30 && b <= 0x39) ||  // 0-9
        b === 0x2D ||  // -
        b === 0x5F ||  // _
        b === 0x2E ||  // .
        b === 0x2A ||  // *
        b === 0x27 ||  // '
        b === 0x28 ||  // (
        b === 0x29) {   // )
      encoded += String.fromCharCode(b);
    } else if (b === 0x20) {  // 空格转+
      encoded += '+';
    } else {
      // 3. 其他字符编码为%XX
      encoded += '%' + b.toString(16).toUpperCase().padStart(2, '0');
    }
  }
  return encoded;
}
```

### URL解码

```javascript
function urlDecode(str) {
  // 处理+为空格
  str = str.replace(/\+/g, ' ');

  const bytes = [];
  for (let i = 0; i < str.length; i++) {
    if (str[i] === '%' && i + 2 < str.length) {
      bytes.push(parseInt(str.substring(i + 1, i + 3), 16));
      i += 2;
    } else {
      bytes.push(str.charCodeAt(i));
    }
  }

  return Buffer.from(bytes).toString('latin1');
}

function doubleUrlDecode(str) {
  return urlDecode(urlDecode(str));
}

function doubleDecodeAndGbk(str) {
  // 两次URL decode后转GBK字符串
  const decoded1 = urlDecode(str);
  const decoded2 = urlDecode(decoded1);
  const gbkBytes = [];
  for (let i = 0; i < decoded2.length; i++) {
    gbkBytes.push(decoded2.charCodeAt(i));
  }
  return iconv.decode(Buffer.from(gbkBytes), 'gbk');
}
```

## 完整请求示例

```javascript
const axios = require('axios');
const iconv = require('iconv-lite');
const crypto = require('crypto');

async function sendRequest(params) {
  // 1. 构建XML（所有字段，即使是空值也要带上）
  let xml = '<?xml version="1.0" encoding="GBK" standalone="yes"?><xml>';
  for (const [key, value] of Object.entries(params)) {
    xml += `<${key}>${value !== undefined && value !== null ? value : ''}</${key}>`;
  }
  xml += '</xml>';

  console.log('XML:', xml);

  // 2. 签名
  const signContent = buildSignContent(params);
  const sign = signWithPrivateKey(signContent, privateKey);
  params.sign = sign;

  // 3. 重新构建XML（包含sign）
  xml = '<?xml version="1.0" encoding="GBK" standalone="yes"?><xml>';
  for (const [key, value] of Object.entries(params)) {
    xml += `<${key}>${value !== undefined && value !== null ? value : ''}</${key}>`;
  }
  xml += '</xml>';

  // 4. 两次GBK URL编码
  const encodedXml = doubleEncode(xml);

  // 5. 发送请求
  const response = await axios.post(url, `req=${encodedXml}`, {
    headers: {
      'Content-Type': 'application/x-www-form-urlencoded'
    },
    responseType: 'arraybuffer',
    timeout: 30000
  });

  // 6. 解码响应
  const encodedStr = response.data.toString('latin1');
  const xmlResponse = doubleDecodeAndGbk(encodedStr);

  return xmlResponse;
}
```

## 编码对照表

### URL编码规则

| 字符 | URL编码 | 说明 |
|------|---------|------|
| 空格 | `+` 或 `%20` | |
| 中文 | `%XX%XX` | GBK双字节，每个字节单独编码 |
| 字母 | 不编码 | a-z, A-Z |
| 数字 | 不编码 | 0-9 |
| `-_.*'()` | 不编码 | 保持原样 |

### GBK编码示例

| 字符 | GBK字节 | URL编码后 |
|------|---------|-----------|
| 中 | `0xD6 0xD0` | `%D6%D0` |
| 文 | `0xCE 0xC4` | `%CE%C4` |
| A | `0x41` | `A` |
| 1 | `0x31` | `1` |
