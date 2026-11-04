# Day05 笔记

> [!info] 项目
> 苍穹外卖 · 第 5 天
>
> 涉及内容：微信小程序登录流程、微信授权接口参数说明
>
> 上一篇：[[苍穹外卖/day04]]

- **tags**: #苍穹外卖 #微信小程序 #JWT

---

## 1. 微信小程序登录流程

```
小程序端                          后端服务器                    微信服务器
  │                                │                              │
  │  1. wx.login() 获取 code        │                              │
  │───────────────────────────────>│                              │
  │                                │  2. 用 code + appid + secret  │
  │                                │  调用微信接口                 │
  │                                │─────────────────────────────>│
  │                                │                              │
  │                                │  3. 返回 openid + session_key │
  │                                │<─────────────────────────────│
  │                                │                              │
  │  4. 返回 JWT token             │                              │
  │<───────────────────────────────│                              │
```

流程说明：

1. 小程序端调用 `wx.login()` 获取临时登录凭证 **code**
2. 后端服务器用 `code + appid + secret` 调用微信接口（`https://api.weixin.qq.com/sns/jscode2session`）
3. 微信服务器返回 **openid**（用户唯一标识）+ **session_key**（会话密钥）
4. 后端校验成功后，生成 **JWT token** 返回给小程序端

---

## 2. 微信授权接口参数说明

| 参数 | 说明 |
|---|---|
| `appid` | 小程序的唯一标识（在微信公众平台申请的） |
| `secret` | 小程序的密钥（证明你是这个小程序的开发者） |
| `js_code` | 小程序端调用 `wx.login()` 获取的临时登录凭证 |
| `grant_type` | 固定值 `authorization_code`，表示授权码模式 |
| `WX_LOGIN_URL` | 微信的接口地址，用来换取 `openid`（即 `https://api.weixin.qq.com/sns/jscode2session`） |

---

## 3. 今日重点

> [!star] 重点回顾
>
> 1. **微信登录流程 4 步**：`wx.login()` 拿 code → 后端用 code + appid + secret 调微信接口 → 微信返回 openid + session_key → 后端签发 JWT 返回小程序。
> 2. **关键概念区分**：
>    - `openid`：用户唯一标识，全局不可逆
>    - `session_key`：会话密钥，不可下发到小程序端
>    - `js_code`：临时登录凭证，5 分钟有效，只能使用一次
> 3. **5 个授权参数**：appid、secret、js_code、grant_type（固定 `authorization_code`）、WX_LOGIN_URL（`/sns/jscode2session`）。

---

## 4. 面试问题

> [!question] 🎤 面试题
> **问题**：说下微信小程序的登录流程？openid、session_key、js_code 各是什么？session_key 为什么不能下发到小程序端？

**参考答案**：

1. **登录流程**：
   - 小程序端 `wx.login()` 获取临时凭证 **js_code**；
   - 后端用 `js_code + appid + secret` 调用微信接口 `jscode2session`；
   - 微信服务器返回 **openid**（用户唯一标识）+ **session_key**（会话密钥）；
   - 后端校验/落库后，签发 **JWT** 返回给小程序端，后续请求带 JWT 访问。
2. **三个概念区分**：
   - **openid**：用户在本小程序内的唯一标识，可逆映射；
   - **js_code**：临时登录凭证，**5 分钟有效、只能用一次**；
   - **session_key**：会话密钥，用于**解密**用户数据（如手机号）。
3. **session_key 不能下发到小程序端**：
   - 它是敏感密钥，一旦泄露，攻击者可伪造/解密用户数据；
   - 只能保存在服务端，需要解密时由后端用 session_key 完成，再返回明文结果给前端。
