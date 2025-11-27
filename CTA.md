# CTA（聯絡我）功能教學 📨

說明
- 目的：當訪客填寫並送出「聯絡我」表單時，系統會將訊息寄到你的信箱（站方信箱），同時寄出一封確認 / 收據郵件給訪客（使用者）的 Email。這能讓使用者知道你的訊息已收到，也能降低漏信風險。
- 重要限制：純粹用 `mailto:` 的 fallback 只能打開訪客的 Email 用戶端，無法自動由伺服器替你寄出郵件給使用者或站方，因此要實作「雙寄送（owner + visitor）」功能，推薦使用後端（Server）或 serverless（Functions / API）來寄送。

概觀選項（依推薦順序）
1. Serverless / Email API（推薦）
   - 使用 SendGrid、Mailgun、或 SMTP relay（例如 Amazon SES）等服務，透過 provider 的 REST API 或官方套件寄信。
   - 好處：簡單、可擴展、通常有 deliverability、追蹤與範本功能。
2. 傳統後端（Node.js + Express + Nodemailer）
   - 若你已經有後端伺服器，可以使用 Nodemailer 來透過 SMTP 寄信（例如透過公司郵件或 Gmail SMTP）。
3. mailto（fallback）
   - 無伺服器情況下可以利用 mailto: 開啟使用者郵件用戶端來寄信給你（無法自動寄給使用者）。

基本流程
1. 使用者填寫表單（name、email、phone、message）。
2. 前端發送 POST 到你的 API：`/api/contact`（或 serverless endpoint）。
3. 伺服器驗證輸入（必填格式、避免注入攻擊）。
4. 伺服器呼叫 Email API 或 SMTP 寄出兩封郵件：
   - 寄給站方（owner）：含完整內容、來信人資訊（name, email, phone）、IP、User-Agent（可選）
   - 寄給使用者（visitor）：感謝信、參考內容與回覆方式（或你將如何聯絡）、聯絡時間預估
5. 伺服器回應前端，前端顯示成功或錯誤通知（toast）。

---

A. 範例 A：Serverless + SendGrid（推薦）

1) 安裝（本地測試/Node.js function）：
   - `npm install @sendgrid/mail dotenv`（server 或 serverless 設定環境變數 `SENDGRID_API_KEY`）

2) 範例 function（Node.js，Express / Netlify / Vercel 都相似）

Node.js - Express 範例（app.js）:

```js
require('dotenv').config();
const express = require('express');
const sgMail = require('@sendgrid/mail');
const cors = require('cors');

const app = express();
app.use(cors());
app.use(express.json());

sgMail.setApiKey(process.env.SENDGRID_API_KEY);

app.post('/api/contact', async (req, res) => {
  try {
    const { name, email, phone, message } = req.body;
    // 驗證資料
    if (!name || !email || !message) return res.status(400).json({ error: '缺少欄位' });

    // 寄給站方
    const ownerMsg = {
      to: process.env.OWNER_EMAIL,
      from: process.env.SEND_FROM, // 你在 SendGrid 裡核准的寄件地址
      subject: `[網站聯絡] ${name}`,
      html: `
        <p>姓名：${name}</p>
        <p>Email：${email}</p>
        <p>電話：${phone || 'N/A'}</p>
        <p>內容：${message}</p>
      `,
    };

    // 寄給使用者（確認信）
    const visitorMsg = {
      to: email,
      from: process.env.SEND_FROM,
      subject: `已收到您的訊息 — ${process.env.SITE_TITLE || '聯絡表單'}`,
      html: `
        <p>Hi ${name}，</p>
        <p>感謝您聯絡我們，我們已收到您的訊息，會在 48 小時內回覆。</p>
        <hr/>
        <p>您輸入的內容：</p>
        <p>${message}</p>
        <p>若要回覆，請用此郵件回覆或稍後等待我們的回覆。</p>
      `,
    };

    // 兩個 Promise 同時寄出
    await Promise.all([sgMail.send(ownerMsg), sgMail.send(visitorMsg)]);

    return res.json({ ok: true });
  } catch (err) {
    console.error('contact error', err);
    return res.status(500).json({ error: 'internal' });
  }
});

app.listen(process.env.PORT || 3000);
```

3) 環境變數（.env）:
```
SENDGRID_API_KEY=SG.xxxxxx
SEND_FROM=no-reply@yourdomain.com
OWNER_EMAIL=alanchen20051125@gmail.com
SITE_TITLE=陳韡侖的作品集
```

4) 前端 fetch 呼叫範例（你的 index.html 中，可用 fetch）:

```js
async function submitContact(data) {
  try {
    const res = await fetch('/api/contact', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data),
    });
    const r = await res.json();
    if (!res.ok) throw new Error(r.error || 'Failed');
    // show success toast
    return r;
  } catch (err) {
    throw err;
  }
}
```

---

B. 範例 B：Node.js + Express + Nodemailer（SMTP）

1) 安裝：`npm install express nodemailer dotenv cors`

2) 範例：

```js
require('dotenv').config();
const express = require('express');
const nodemailer = require('nodemailer');
const app = express();
app.use(express.json());

const transporter = nodemailer.createTransport({
  host: process.env.SMTP_HOST,
  port: Number(process.env.SMTP_PORT || 587),
  secure: process.env.SMTP_PORT === '465',
  auth: {
    user: process.env.SMTP_USER,
    pass: process.env.SMTP_PASS,
  },
});

app.post('/api/contact', async (req, res) => {
  try {
    const { name, email, phone, message } = req.body;
    if (!name || !email || !message) return res.status(400).json({ error: '缺少欄位' });

    const ownerMail = {
      from: process.env.SEND_FROM,
      to: process.env.OWNER_EMAIL,
      subject: `[網站聯絡] ${name}`,
      html: `<p>姓名：${name}</p><p>Email：${email}</p><p>電話：${phone || 'N/A'}</p><p>${message}</p>`,
    };

    const userMail = {
      from: process.env.SEND_FROM,
      to: email,
      subject: `我們已收到您的訊息` ,
      html: `<p>Hi ${name}，我們已收到您的訊息：</p><p>${message}</p>`,
    };

    await Promise.all([transporter.sendMail(ownerMail), transporter.sendMail(userMail)]);
    return res.json({ ok: true });
  } catch (err) { console.error(err); return res.status(500).json({ error: err.message }); }
});

app.listen(process.env.PORT || 3000);
```

.env 設定：
```
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your-smtp-user
SMTP_PASS=your-smtp-pass
SEND_FROM=no-reply@yourdomain.com
OWNER_EMAIL=alanchen20051125@gmail.com
```

注意：使用 Gmail SMTP 如果不是專門的 App Password/OAuth，可能無法寄信，請參考 Gmail/Google 帳號安全設定或使用 SendGrid / Mailgun 等服務。

---

C. 簡易 fallback： mailto（僅在無後端時使用）
- 優點：不需要伺服器。
- 缺點：只能打開使用者郵件用戶端；無法自動由後端寄出信給使用者與你。

範例（JavaScript 產生 mailto）：
```js
const name = 'John';
const email = 'john@example.com';
const message = 'Hello';
const mailto = `mailto:alanchen20051125@gmail.com?subject=${encodeURIComponent('[網站聯絡] ' + name)}&body=${encodeURIComponent(message + '\nFrom: ' + name + ' (' + email + ')')}`;
window.location.href = mailto;
```

---

表單前端修改建議（index.html）
- 將 `<form id="contact-form">` 的 `action` 改為 `javascript:void(0)` ，送出時執行 fetch。
- 送出時禁用按鈕、防止重複送出、驗證欄位。
- 顯示 spinner / toast / 送出成功訊息。

簡易範例：
```js
document.getElementById('contact-form').addEventListener('submit', async (e) => {
  e.preventDefault();
  const formData = Object.fromEntries(new FormData(e.target).entries());
  try {
    await submitContact(formData); // 呼叫上述 fetch
    // show success
  } catch (err) {
    // show error
  }
});
```

資料驗證與資安
- 一定要在 server 端驗證並 sanitize 使用者輸入（避免 HTML injection）。
- 防止濫用：加速率限制（rate-limit），或加上 CAPTCHA（reCAPTCHA v2/v3）或簡單的 anti-bot 機制。
- 不要在前端放 API KEY 或任何敏感資訊。
- 郵件 headers：使用 `replyTo` 指向使用者 email（使你回信時可直接回到使用者），但寄件人（From）通常要是你的信箱或已經在 Email provider 驗證的地址，否則郵件可能被擋掉。

測試步驟
1. 將後端程式啟動於本機（例如 `PORT=3000 node app.js`）。
2. 本機測試：使用 Postman 或 curl POST 到 `/api/contact`，檢查回應、並檢查收件箱是否兩封都收到。
3. 前端測試：連至本地前端（檔案或本機伺服器），填寫表單並送出。
4. 檢查 spam 資料夾或送信者 address，若未收到，要檢查：SPF / DKIM/ DMARC 設定與寄件者 address 是否正確。

部署注意
- Vercel / Netlify 支援 serverless functions，將相同 Node.js 邏輯放在相對應的位置（Netlify: `netlify/functions/contact.js`，Vercel: API route）。
- 在部署環境中設定環境變數（API Key、OWNER_EMAIL、SEND_FROM）。

其他加值建議
- 以 Email 範本服務管理郵件內容（SendGrid Template、Mailgun Templates）以方便 AB 測試與追蹤。
- 監控系統：當表單大量送出（濫用）時，記錄 IP & 紀錄至資料庫，並通知管理員（e.g., Slack webhook）。

範例 - Local quick test (curl)：
```
curl -X POST http://localhost:3000/api/contact -H 'Content-Type: application/json' -d '{"name":"test","email":"test@example.com","phone":"+886123456789","message":"Hello"}'
```

---

常見問題 FAQ
Q: 我可以用前端直接寄兩封 email 給站方與使用者嗎？
A: 前端只能呼叫 `mailto` 或透過後端 API 來要求後端寄信。要自動寄給雙方必需透過後端（server或serverless）。

Q: 為何我的郵件會被判為垃圾郵件？
A: 檢查寄件地址、SPF、DKIM、DMARC 設定，使用寄送服務（SendGrid、Mailgun、SES）可大幅改善送達率。

Q: 使用 Gmail SMTP 會有問題嗎？
A: Gmail 有 security 限制，若使用 OAuth2 或 App Password，效果較佳。建議使用專業寄信服務（SendGrid/SES）以提高穩定度。

---

如果你需要，我可以：
- 幫你把 `index.html` 的表單改為使用 fetch 串接上述 API（Node.js + SendGrid 範例），包含測試用 Postman 指引。
- 幫你新增 reCAPTCHA 或 rate limiting 的範例。
- 幫你把 Netlify / Vercel 的 function 範例實作到你的 repo

祝你整合順利！如要我直接為你新增 serverless 範例或修改 `index.html` 來對接 API，請告訴我想用哪種 provider（SendGrid / Nodemailer / Netlify / Vercel / 等）。