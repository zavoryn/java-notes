---
tags: [八股, 计算机网络, Cookie, Session, JWT, 跨域, CORS, OAuth]
status: 待复习
created: 2026-05-06
---

# Cookie 与认证

> [!tip] 模块速记
> **Cookie = 便签（浏览器端小数据块），Session = 档案柜（服务端存储），JWT = 自包含通行证（无状态令牌）。跨域靠 CORS（预检 OPTIONS），OAuth 2.0 靠授权码（code→后端换token）。选型口诀：单体用 Session，分布式用 JWT。**

---

## 一、Cookie vs Session vs JWT

### 1.1 三剑客对比

| 维度       | Cookie         | Session      | JWT               |
| -------- | -------------- | ------------ | ----------------- |
| **存储位置** | 浏览器            | 服务端          | 客户端（浏览器或 App）     |
| **安全性**  | 容易被篡改/XSS/CSRF | 存在服务端，相对安全   | 签名防篡改但 payload 可读 |
| **扩展性**  | 单域名            | 需共享存储（Redis） | 无状态，天然支持分布式       |
| **性能**   | 每次请求都带         | 需查服务端存储      | 本地验签，无需查库         |
| **大小限制** | 4KB            | 无限制          | 无限制但过大会影响传输       |
| **跨域**   | 不支持            | 不支持（除非共享存储）  | 天然支持              |
| **比喻**   | 随身贴的便签         | 服务端的档案柜      | 自包含的通行证           |

### 1.2 Cookie 关键属性

```
Set-Cookie: sessionId=abc123; HttpOnly; Secure; SameSite=Lax; Max-Age=3600
```

| 属性 | 作用 |
|------|------|
| **HttpOnly** | 禁止 JS 读取，防 XSS 窃取 Cookie |
| **Secure** | 仅 HTTPS 传输 |
| **SameSite** | 防 CSRF（Strict/Lax/None） |
| **Domain/Path** | 控制 Cookie 的作用范围 |
| **Max-Age** | 过期时间（秒） |

### 1.3 Session 详解

```
用户登录 → 服务端创建 Session → 返回 SessionID（通过 Cookie）→ 后续请求带 SessionID → 服务端查 Session 存储
```

**Session 在分布式系统的问题**：需要共享存储（Redis 集中式），或使用粘性会话（Sticky Session）。

### 1.4 JWT 详解

**JWT 结构**：`Header.Payload.Signature`

```
eyJhbGciOiJIUzI1NiJ9.eyJ1c2VySWQiOjEyMzQ1fQ.abc123def456

  Header            Payload            Signature
{"alg":"HS256"}  {"userId":12345}   HMAC-SHA256(Header.Payload, secret)
```

**JWT 的缺点**：
- **无法主动失效**：发了的 token 在过期前一直有效（除非引入黑名单）
- **payload 明文**：只是 Base64 编码，不是加密
- **体积较大**：比 SessionID 大很多

**比喻——JWT 是身份证，Session 是会员卡**：
- 身份证（JWT）：上面有你的信息，去哪里办事出示就行，机构不需要打电话核查（无状态）。
- 会员卡（Session）：只是一串编号，每次消费都要刷一下查后台系统（查服务端存储）。

### 1.5 面试回答模板

> **面试官：Cookie、Session、JWT 的区别？什么时候用哪个？**

"三者都是用来维护 HTTP 无状态协议下的用户身份识别。

**Cookie** 是存储在浏览器端的小型数据块，关键是要开 HttpOnly 防 XSS、Secure 仅 HTTPS、SameSite 防 CSRF。

**Session** 是服务端维护的用户状态。优点是安全（数据在服务端），缺点是在分布式系统中需要共享存储（Redis），这是有状态的。

**JWT** 是无状态的令牌。优点是利于水平扩展和跨域，缺点是无法主动失效、payload 明文可读。

**选型建议**：传统单体 Web 应用用 Session + Cookie 够用；前后端分离、微服务、移动端用 JWT 更灵活。实际生产中也可以两者结合——用 JWT 做认证，但敏感状态信息走 Redis 缓存。"

**🔗 知识串联：**
- **串联 05 HTTP**：Cookie 和 Authorization 都是通过 HTTP Header 传递的（`Set-Cookie`/`Cookie`/`Authorization: Bearer`），详见 [[05-HTTP协议/HTTP协议详解]]
- **串联 06 HTTPS**：Cookie 的 `Secure` 属性确保只在 HTTPS 连接中传输，JWT 在 HTTPS 下传输防止中间人窃取，详见 [[06-HTTPS与安全认证/HTTPS与TLS详解]]
- **串联 07 DNS与CDN**：CDN 和反向代理场景下 Cookie 可能有跨域问题，需要 CORS 配合，详见 [[07-网络应用技术/DNS与CDN]]
- **串联 08 面试**：Spring Boot 中 Cookie/Token 用 Spring Security + `@CookieValue` / `JwtTokenProvider`，Session 共享用 Spring Session + Redis，详见 [[08-综合面试题/计网面试突击]]

---

## 二、跨域问题详解（CORS）

### 2.1 同源策略

**同源 = 协议 + 域名 + 端口完全相同**。浏览器遵循同源策略，限制跨源 HTTP 请求。

### 2.2 CORS 完整请求流程

#### 简单请求

满足以下条件的是**简单请求**（不需要预检）：
- 方法：GET / HEAD / POST
- Content-Type：text/plain / multipart/form-data / application/x-www-form-urlencoded
- 不包含自定义头部

```
浏览器直接发请求，附加 Origin 头：
  Origin: https://www.example.com

服务端响应：
  Access-Control-Allow-Origin: https://www.example.com
  Access-Control-Allow-Credentials: true
```

#### 预检请求（Preflight）

不满足简单请求条件的，浏览器先发一个 **OPTIONS** 预检请求：

```
OPTIONS /api/data HTTP/1.1
Origin: https://www.example.com
Access-Control-Request-Method: PUT
Access-Control-Request-Headers: X-Custom-Header

服务端响应：
Access-Control-Allow-Origin: https://www.example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: X-Custom-Header
Access-Control-Max-Age: 86400  ← 预检结果缓存24小时

→ 预检通过后，浏览器才发真正的 PUT 请求
```

> [!warning] 面试高频
> **为什么需要预检？** 因为非简单请求（如 PUT、自定义头部）可能对服务端有破坏性影响，浏览器需要先"问一下"服务端是否允许，再决定是否发真正的请求。

### 2.3 CORS 解决方案总结

| 方案 | 适用场景 | 限制 |
|------|----------|------|
| **CORS**（推荐） | 服务端可控 | 需要服务端配合设置头部 |
| **JSONP** | 只需 GET | 只支持 GET，安全风险 |
| **反向代理** | 服务端不可控 | Nginx 代理到同域 |
| **WebSocket** | 全双工通信 | 不受同源策略限制 |

**🔗 知识串联：**
- **串联 05 HTTP**：CORS 预检用的是 OPTIONS 方法（HTTP 方法之一），响应头是标准 HTTP Header
- **串联 07 反向代理**：Nginx 反向代理可以把跨域请求代理到同域，绕过浏览器同源策略
- **串联 07 DNS与CDN**：CDN 跨域资源也需要配置 CORS 头部，否则浏览器会拦截

---

## 三、OAuth 2.0 基本流程

### 3.1 授权码模式（最常用）

```
① 用户点击"用微信登录"
    │
    ▼
② 浏览器跳转到微信授权页面
   URL: https://open.weixin.qq.com/oauth/authorize?client_id=XXX&redirect_uri=YYY&response_type=code
    │
    ▼
③ 用户在微信页面同意授权
    │
    ▼
④ 微信回调 redirect_uri，带上授权码 code
   URL: https://yourapp.com/callback?code=AUTH_CODE
    │
    ▼
⑤ 后端用 code + client_secret 向微信换取 access_token
   POST https://open.weixin.qq.com/oauth/access_token
   { code, client_id, client_secret }
    │
    ▼
⑥ 后端用 access_token 获取用户信息，完成登录
```

> [!note] 为什么需要授权码（code）？
> **安全**：access_token 是敏感的，不能直接给前端（可能泄露）。授权码只能用一次且有效期短，后端用 code + client_secret 换 token，client_secret 只在后端保存，不会暴露给浏览器。

### 3.2 OAuth 2.0 四种授权模式

| 模式 | 适用场景 | 安全性 |
|------|----------|--------|
| **授权码模式** | 有后端的 Web 应用 | **最安全**（推荐） |
| **隐式模式** | 纯前端（无后端） | 较低（token 直接给前端） |
| **密码模式** | 高度信任的第一方应用 | 低（用户密码给客户端） |
| **客户端凭证模式** | 服务器间通信（M2M） | 中（无用户参与） |

**🔗 知识串联：**
- **串联 06 HTTPS**：OAuth 2.0 的授权码和 token 传输必须走 HTTPS，防止中间人窃取
- **串联 05 HTTP 重定向**：OAuth 2.0 流程中大量使用 HTTP 302 重定向
- **串联 JWT**：OAuth 2.0 的 access_token 可以是 JWT 格式（自包含用户信息），也可以是 opaque token（需查库验证）

---

## 四、面试回答模板

> **面试官：CORS 跨域怎么解决？**

"浏览器遵循同源策略（协议+域名+端口相同）。跨域解决方案首选 CORS——服务端设置 Access-Control-Allow-Origin 等头部。对于非简单请求（如 PUT、自定义头部），浏览器先发 OPTIONS 预检请求确认服务端允许后才发真正的请求。也可以用 Nginx 反向代理把跨域请求代理到同域。"

> **面试官：OAuth 2.0 的授权码模式流程？**

"用户点击第三方登录→跳转到授权页面→用户同意→授权服务器回调带 code→后端用 code + client_secret 换 access_token→用 token 获取用户信息。关键在于 code 只用一次、有效期短，client_secret 只在后端保存，不会暴露给前端。"

---

## 相关面试题

| # | 题目 | 要点 |
|---|------|------|
| 1 | Cookie vs Session vs JWT | 便签/档案柜/通行证，选型看场景 |
| 2 | Cookie安全属性 | HttpOnly/Secure/SameSite |
| 3 | JWT优缺点 | 无状态+跨域友好 vs 无法主动失效+payload明文 |
| 4 | 跨域与CORS | 简单请求直接发+非简单请求预检OPTIONS |
| 5 | OAuth 2.0授权码模式 | code→后端换token→获取用户信息 |
| 6 | 同源策略 | 协议+域名+端口完全相同 |

---

> **相关笔记**：[[07-网络应用技术/DNS与CDN]] | [[07-网络应用技术/从URL到页面]] | [[06-HTTPS与安全认证/HTTPS与TLS详解]] | [[05-HTTP协议/HTTP协议详解]] | [[计算机网络总览]]
