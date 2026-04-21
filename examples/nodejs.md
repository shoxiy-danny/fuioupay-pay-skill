---
name: fuioupay-examples-nodejs
description: "富友支付接口的完整Node.js客户端实现，包括签名、编码、请求和验签全流程。"
trigger:
  - Node.js
  - nodejs
  - SDK
  - 示例
  - javascript
  - axios
  - 客户端
---

# 富友支付 Node.js SDK 示例

## 安装依赖

```bash
npm install axios xml2js iconv-lite crypto
```

## 完整客户端实现

```javascript
const axios = require('axios');
const xml2js = require('xml2js');
const iconv = require('iconv-lite');
const crypto = require('crypto');

// ==================== 工具函数 ====================

/**
 * 构建签名原文
 * @param {Object} params - 请求参数
 * @returns {string} 签名原文
 */
function buildSignContent(params) {
  const sortedKeys = Object.keys(params).sort();
  const parts = [];
  for (const key of sortedKeys) {
    if (key === 'sign' || key.startsWith('reserved')) {
      continue;
    }
    const value = params[key] !== undefined && params[key] !== null ? params[key] : '';
    parts.push(`${key}=${value}`);
  }
  return parts.join('&');
}

/**
 * RSA签名
 */
function signWithPrivateKey(content, privateKey) {
  const contentBuffer = iconv.encode(content, 'gbk');
  const sign = crypto.createSign('RSA-MD5');
  sign.update(contentBuffer);
  return sign.sign(privateKey, 'base64');
}

/**
 * RSA验签
 */
function verifyWithPublicKey(content, signature, publicKey) {
  try {
    const cleanSignature = signature.replace(/\s/g, '');
    const contentBuffer = iconv.encode(content, 'gbk');
    const verify = crypto.createVerify('RSA-MD5');
    verify.update(contentBuffer);
    return verify.verify(publicKey, cleanSignature, 'base64');
  } catch (e) {
    console.error('验签失败:', e.message);
    return false;
  }
}

/**
 * GBK URL编码
 */
function gbkUrlEncode(str) {
  const buf = iconv.encode(str, 'gbk');
  let encoded = '';
  for (let i = 0; i < buf.length; i++) {
    const b = buf[i];
    if ((b >= 0x61 && b <= 0x7A) || (b >= 0x41 && b <= 0x5A) ||
        (b >= 0x30 && b <= 0x39) || b === 0x2D || b === 0x5F ||
        b === 0x2E || b === 0x2A || b === 0x27 || b === 0x28 || b === 0x29) {
      encoded += String.fromCharCode(b);
    } else if (b === 0x20) {
      encoded += '+';
    } else {
      encoded += '%' + b.toString(16).toUpperCase().padStart(2, '0');
    }
  }
  return encoded;
}

/**
 * 两次GBK URL编码
 */
function doubleEncode(str) {
  return gbkUrlEncode(gbkUrlEncode(str));
}

/**
 * URL解码
 */
function urlDecode(str) {
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

/**
 * 两次URL解码+GBK
 */
function doubleDecodeAndGbk(str) {
  const decoded1 = urlDecode(str);
  const decoded2 = urlDecode(decoded1);
  const gbkBytes = [];
  for (let i = 0; i < decoded2.length; i++) {
    gbkBytes.push(decoded2.charCodeAt(i));
  }
  return iconv.decode(Buffer.from(gbkBytes), 'gbk');
}

/**
 * 解析XML
 */
async function parseXml(xml) {
  const parser = new xml2js.Parser({ explicitArray: false });
  const result = await parser.parseStringPromise(xml);
  return result.xml;
}

/**
 * 构建XML请求
 */
function buildXmlRequest(params) {
  let xml = '<?xml version="1.0" encoding="GBK" standalone="yes"?><xml>';
  for (const [key, value] of Object.entries(params)) {
    xml += `<${key}>${value !== undefined && value !== null ? value : ''}</${key}>`;
  }
  xml += '</xml>';
  return xml;
}

/**
 * 生成随机字符串
 */
function generateRandomStr(length = 32) {
  const chars = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789';
  let result = '';
  for (let i = 0; i < length; i++) {
    result += chars.charAt(Math.floor(Math.random() * chars.length));
  }
  return result;
}

/**
 * 生成订单号
 */
function generateOrderNo(insCd) {
  const date = new Date();
  const dateStr = date.getFullYear().toString() +
    (date.getMonth() + 1).toString().padStart(2, '0') +
    date.getDate().toString().padStart(2, '0');
  const random = Math.floor(Math.random() * 9000000000) + 1000000000;
  return insCd + dateStr + random;
}

// ==================== 富友支付客户端 ====================

class FuiouPay {
  constructor(config) {
    this.insCd = config.insCd;
    this.mchntCd = config.mchntCd;
    this.termId = config.termId || '88888888';
    this.privateKey = config.privateKey;
    this.publicKey = config.publicKey;
    this.notifyUrl = config.notifyUrl;
    this.baseUrl = config.baseUrl || 'https://spay-cloud2.fuioupay.com';
  }

  /**
   * 发送请求
   */
  async request(path, params) {
    // 生成随机字符串
    params.random_str = generateRandomStr();

    // 构建签名原文并签名
    const signContent = buildSignContent(params);
    params.sign = signWithPrivateKey(signContent, this.privateKey);

    // 构建XML
    const xmlRequest = buildXmlRequest(params);

    // 两次编码
    const encodedXml = doubleEncode(xmlRequest);

    // 发送请求
    const url = `${this.baseUrl}${path}`;
    const response = await axios.post(url, `req=${encodedXml}`, {
      headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
      responseType: 'arraybuffer',
      timeout: 30000
    });

    // 解码响应
    const encodedStr = response.data.toString('latin1');
    const xmlResponse = doubleDecodeAndGbk(encodedStr);

    // 解析XML
    const result = await parseXml(xmlResponse);

    return result;
  }

  /**
   * 统一下单（主扫）
   */
  async createOrder(params) {
    const now = new Date();
    const txnBeginTs = now.getFullYear().toString() +
      (now.getMonth() + 1).toString().padStart(2, '0') +
      now.getDate().toString().padStart(2, '0') +
      now.getHours().toString().padStart(2, '0') +
      now.getMinutes().toString().padStart(2, '0') +
      now.getSeconds().toString().padStart(2, '0');

    return this.request('/preCreate', {
      version: '1',
      ins_cd: this.insCd,
      mchnt_cd: params.mchntCd || this.mchntCd,
      term_id: this.termId,
      order_type: params.orderType,
      mchnt_order_no: params.mchntOrderNo || generateOrderNo(this.insCd),
      order_amt: Math.round(params.orderAmt * 100), // 元转分
      goods_des: params.goodsDes,
      txn_begin_ts: txnBeginTs,
      notify_url: params.notifyUrl || this.notifyUrl,
      term_ip: params.termIp || '127.0.0.1'
    });
  }

  /**
   * 条码支付（被扫）
   */
  async barcodePay(params) {
    const now = new Date();
    const txnBeginTs = now.getFullYear().toString() +
      (now.getMonth() + 1).toString().padStart(2, '0') +
      now.getDate().toString().padStart(2, '0') +
      now.getHours().toString().padStart(2, '0') +
      now.getMinutes().toString().padStart(2, '0') +
      now.getSeconds().toString().padStart(2, '0');

    return this.request('/micropay', {
      version: '1',
      ins_cd: this.insCd,
      mchnt_cd: params.mchntCd || this.mchntCd,
      term_id: this.termId,
      order_type: params.orderType,
      mchnt_order_no: params.mchntOrderNo || generateOrderNo(this.insCd),
      order_amt: Math.round(params.orderAmt * 100),
      goods_des: params.goodsDes,
      txn_begin_ts: txnBeginTs,
      auth_code: params.authCode,
      term_ip: params.termIp || '127.0.0.1'
    });
  }

  /**
   * 订单查询
   */
  async queryOrder(params) {
    return this.request('/commonQuery', {
      version: '1',
      ins_cd: this.insCd,
      mchnt_cd: params.mchntCd || this.mchntCd,
      term_id: this.termId,
      order_type: params.orderType,
      mchnt_order_no: params.mchntOrderNo
    });
  }

  /**
   * 退款申请
   */
  async refund(params) {
    return this.request('/commonRefund', {
      version: '1',
      ins_cd: this.insCd,
      mchnt_cd: params.mchntCd || this.mchntCd,
      term_id: this.termId,
      mchnt_order_no: params.mchntOrderNo,
      order_type: params.orderType,
      refund_order_no: params.refundOrderNo,
      total_amt: Math.round(params.totalAmt * 100),
      refund_amt: Math.round(params.refundAmt * 100),
      operator_id: params.operatorId || ''
    });
  }

  /**
   * 退款查询
   */
  async queryRefund(params) {
    return this.request('/refundQuery', {
      version: '1',
      ins_cd: this.insCd,
      mchnt_cd: params.mchntCd || this.mchntCd,
      term_id: this.termId,
      refund_order_no: params.refundOrderNo
    });
  }

  /**
   * 关闭订单
   */
  async closeOrder(params) {
    return this.request('/closeorder', {
      version: '1',
      ins_cd: this.insCd,
      mchnt_cd: params.mchntCd || this.mchntCd,
      term_id: this.termId,
      mchnt_order_no: params.mchntOrderNo,
      order_type: params.orderType
    });
  }

  /**
   * 解析回调
   */
  async parseCallback(req) {
    const encodedXml = req.body.req || req.query.req;
    const xmlData = doubleDecodeAndGbk(encodedXml);
    return this.parseXml(xmlData);
  }

  /**
   * 验签回调
   */
  verifyCallback(data) {
    const cleanSign = data.sign.replace(/\s/g, '');
    const verifyContent = buildSignContent(data);
    return verifyWithPublicKey(verifyContent, cleanSign, this.publicKey);
  }

  /**
   * 解析XML
   */
  async parseXml(xml) {
    return parseXml(xml);
  }
}

// ==================== 使用示例 ====================

// 创建客户端
const payClient = new FuiouPay({
  insCd: '08XXXXXXX',
  mchntCd: '000XXXXXXX',
  termId: '88888888',
  privateKey: '-----BEGIN PRIVATE KEY-----\n...\n-----END PRIVATE KEY-----',
  publicKey: '-----BEGIN PUBLIC KEY-----\n...\n-----END PUBLIC KEY-----',
  notifyUrl: 'https://your-domain.com/callback',
  baseUrl: 'https://spay-cloud2.fuioupay.com'
});

// 1. 统一下单
async function createOrderExample() {
  const result = await payClient.createOrder({
    orderType: 'WECHAT',
    orderAmt: 1,
    goodsDes: '测试商品'
  });

  if (result.result_code === '000000') {
    console.log('二维码:', result.qr_code);
    console.log('订单号:', result.reserved_fy_order_no);
  }
}

// 2. 条码支付
async function barcodePayExample() {
  const result = await payClient.barcodePay({
    orderType: 'WECHAT',
    orderAmt: 1,
    goodsDes: '测试商品',
    authCode: '134XXXXXXXXX' // 用户付款码
  });

  if (result.result_code === '000000' && result.pay_msg === 'SUCCESS') {
    console.log('支付成功');
  } else if (result.result_code === '000001') {
    // 用户支付中，需要轮询
    console.log('等待付款，开始轮询...');
    // 轮询查询...
  }
}

// 3. 订单查询
async function queryOrderExample() {
  const result = await payClient.queryOrder({
    orderType: 'WECHAT',
    mchntOrderNo: '20240101120000001'
  });

  console.log('状态:', result.trans_stat);
  console.log('金额:', result.order_amt);
}

// 4. 退款
async function refundExample() {
  const result = await payClient.refund({
    orderType: 'WECHAT',
    mchntOrderNo: '20240101120000001',
    refundOrderNo: '20240101120000001_R01',
    totalAmt: 1,
    refundAmt: 0.5
  });

  if (result.result_code === '000000') {
    console.log('退款成功');
  }
}

// 5. 处理回调
async function handleCallbackExample(req, res) {
  const data = await payClient.parseCallback(req);

  if (!payClient.verifyCallback(data)) {
    return res.send('FAIL');
  }

  if (data.result_code === '000000') {
    // 更新订单状态
    console.log('订单支付成功:', data.mchnt_order_no);
  }

  res.send('1');
}

module.exports = { FuiouPay };
```
