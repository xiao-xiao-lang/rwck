# 创客次元（极光论坛）开放 API 文档

> 本文件是 API 文档的**机器可读完整版**，可直接交给 AI Agent 或代码助手使用。
> 网页版（带交互示例）：https://forum.ctspace.xyz/docs
> 版本：2026-09-26 · 对应站点 v3.0

**Base URL**：`https://forum.ctspace.xyz/api`
**认证**：`Authorization: Bearer <token>`（token 为登录返回的 JWT，或「开发者平台」签发的 API Key）
**响应**：JSON，UTF-8。错误统一为 `{ "statusCode": 400, "message": "...", "error": "Bad Request" }`

---

## 0. 给 AI Agent 的快速须知（重要，先读这段）

1. **所有接口以 `/api` 为前缀**。除 `/api/settings`、`/api/captcha`、`/api/discussions`(GET)、`/api/projects`(GET)、`/api/extensions`(GET)、`/api/stats/public`、`/api/users/:id`、`/api/shortlink/*`(GET/POST view) 外，**其余均需登录**。
2. **登录必须先过人机验证**（图文验证码 + 工作量证明 PoW），流程见第 2 节。直接 POST 账号密码会失败。
3. **注册后账号处于「未激活」状态，无法登录**。必须让用户点击激活邮件里的链接。⚠️ **注册超过 30 分钟仍未激活的账号会被系统自动删除**。
4. **API Key 只用于调用接口，不能绕过登录接口的验证码**（验证码是防机器人登录的第一道闸）。
5. **上传文件的中文文件名不要用 `multipart` 的原始文件名**（会被 latin1 误解码成乱码）。上传接口都支持额外的文本字段 `name`，请把真实文件名放进 `name`。
6. **作品源文件的下载权限**：作品有 `allowDownload` 字段。为 `false` 时请勿下载/转存源文件——这是作者的明确意愿，平台协议视其为违规行为。
7. **限流**：全局 300 次/分钟/IP；登录 10 次/分钟；注册 3 次/10 分钟；找回密码 3 次/10 分钟。超限返回 429。

---

## 1. 通用约定

| 项目 | 说明 |
|---|---|
| Base URL | `https://forum.ctspace.xyz/api` |
| 认证 | `Authorization: Bearer <token>` |
| 分页 | 列表接口统一 `?page=1&pageSize=24`，响应为 `{ items: [], total, page, pageSize }` |
| 时间 | ISO 8601（UTC），如 `2026-09-26T10:27:46.343Z` |
| 文件地址 | 返回的 `url` 形如 `/uploads/<uuid>.<ext>`，是站内相对路径，需拼上站点域名 |
| 静态文件 CORS | `/uploads/*` 允许跨域（`Access-Control-Allow-Origin: *`），可直接被第三方播放器（TurboWarp 等）读取 |
| 静态文件缓存 | `.sb3`/音视频等大文件为 `Cache-Control: immutable`（一年）。**URL 变了就代表内容变了，不要按文件名判断** |

---

## 2. 人机验证与登录

### 2.1 取验证码

```
GET /api/captcha
```

响应：

```json
{
  "token": "7ef0524e8c6a8eff1bffd3d84902bb0f",
  "image": "data:image/png;base64,iVBORw0KGgo...",
  "pow": { "challenge": "a01ea5a4f495b7c638630b0bca3d30e1", "difficulty": 4 }
}
```

- `image` 是 base64 PNG（240×78），里面是 4 位大写字母数字，**需要 OCR 或人工识别**。
- `pow.difficulty` 当前为 `4`：需要找一个 nonce，使得
  `sha256(pow.challenge + ":" + nonce)` 的十六进制前 4 位为 `0`。
- `token` **一次性使用**，有有效期；同一 token 答错次数过多会作废（需重新取）。

PoW 计算示例（Node.js）：

```js
const crypto = require('crypto');
function solvePow(challenge, difficulty) {
  let nonce = 0;
  while (true) {
    const h = crypto.createHash('sha256').update(challenge + ':' + nonce).digest('hex');
    if (h.startsWith('0'.repeat(difficulty))) return String(nonce);
    nonce++;
  }
}
```

### 2.2 登录

```
POST /api/auth/login
Content-Type: application/json

{
  "username": "你的用户名或邮箱",
  "password": "密码",
  "captchaToken": "<captcha 的 token>",
  "captchaAnswer": "图里的验证码（大小写不敏感）",
  "captchaPowNonce": "<解开 PoW 的 nonce>"
}
```

响应：

```json
{ "token": "<JWT>", "user": { "id": "...", "username": "...", "role": "member", "points": 0 } }
```

> 未激活账号会返回 401「账号未激活」。JWT 有效期 7 天。

### 2.3 注册

```
POST /api/auth/register
{ "username": "testuser", "email": "you@example.com", "password": "至少6位" }
```

- 注册**不需要**验证码（前端已移除）。若你的客户端仍传 `captchaToken/captchaAnswer/captchaPowNonce` 三元组，会被校验一次。
- 成功后服务端会发激活邮件。**邮件发送失败时账号会被回滚**（不会产生收不到邮件又占着用户名的孤儿账号）。
- ⚠️ **注册后 30 分钟内未点激活链接，账号会被自动清理**。

### 2.4 其它认证接口

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/api/auth/me` | 获取当前登录用户（需 Bearer） |
| PATCH | `/api/auth/me` | 改昵称 / 签名 / 头像 |
| POST | `/api/auth/activate` | 激活账号（body: `{ token }`，token 来自邮件链接） |
| POST | `/api/auth/resend` | 重发激活邮件（body: `{ identifier }` 邮箱或用户名） |
| POST | `/api/auth/forgot-password` | 找回密码，发邮件 |
| POST | `/api/auth/reset-password` | 重置密码（token + 新密码） |
| POST | `/api/auth/forgot-username` | 忘记用户名 |
| POST | `/api/auth/change-password` | 改密码（需登录） |
| POST | `/api/auth/change-username` | 改用户名（需登录，有频次限制） |
| POST | `/api/auth/change-email` | 改邮箱（需登录） |

---

## 3. 开发者密钥（API Key）

在站点「开发者平台」页面签发。格式 `fk_live_<随机>`，服务端只存 sha256，**明文只在创建时返回一次**。

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/api/api-keys` | 列出我的密钥（不含明文） |
| POST | `/api/api-keys` | 创建，body: `{ name }`，响应含一次性明文 |
| DELETE | `/api/api-keys/:id` | 删除 |

用法与 JWT 完全一致：`Authorization: Bearer fk_live_xxx`。

---

## 4. 作品广场（Project）

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/api/projects` | 列表。参数：`category`(game/animation/story/music/art/tutorial) `q` `sort`(score/new) `page` `pageSize` |
| GET | `/api/projects/:id` | 详情（浏览量 +1） |
| GET | `/api/projects/my` | 我的作品（需登录） |
| POST | `/api/projects` | 发布作品（需登录） |
| PATCH | `/api/projects/:id` | 编辑 / 上传新版本（需登录，仅作者或管理员） |
| DELETE | `/api/projects/:id` | 删除（仅作者或管理员） |
| POST | `/api/projects/:id/like` | 点赞/取消（需登录） |

### 4.1 发布作品

```
POST /api/projects
{
  "title": "作品名（2~255 字）",
  "summary": "一句话简介（可选，≤500）",
  "content": "正文 Markdown（可选）",
  "category": "game",
  "coverResourceId": "<先上传封面得到的 Resource.id>",
  "fileResourceId": "<先上传 .sb3 得到的 Resource.id，必填>",
  "allowDownload": true,
  "syncToForum": false,
  "forumTagIds": []
}
```

上传流程：先 `POST /api/resources/upload`（multipart，字段 `file`）拿到 `id`，再把它作为 `fileResourceId`。

### 4.2 详情响应中的关键字段

```json
{
  "id": "cmu1bhg44003282gm414v5fvf",
  "title": "泰拉瑞亚",
  "category": "game",
  "views": 22,
  "likeCount": 2,
  "allowDownload": true,
  "sb3Url": "/uploads/002fa07f-....sb3",
  "sb3Filename": "Terraria (Stamped) v0.65b.sb3",
  "shortLinkSlug": "emth4e",
  "shortLinkIsSystem": true,
  "file":   { "id": "...", "url": "/uploads/xxx.sb3", "fileType": "sb3", "filename": "xxx.sb3" },
  "cover":  { "id": "...", "url": "/uploads/xxx.png", "fileType": "image" },
  "author": { "id": "...", "username": "...", "nickname": "...", "avatar": "..." },
  "versions": [
    { "id": "...", "versionLabel": "v1.0", "changelog": "初始版本",
      "sb3Resource": { "id": "...", "url": "/uploads/xxx.sb3", "filename": "..." },
      "author": { "id": "...", "username": "..." }, "createdAt": "..." }
  ],
  "editors": [ { "name": "TurboWarp", "embedTemplate": "https://turbowarp.org/embed.html?project_url={url}" } ],
  "likedByMe": false
}
```

**字段语义**

- `allowDownload`：作者是否允许他人下载源文件。为 `false` 时请勿下载/转存 `sb3Url`。
- `shortLinkSlug`：作品的「作品网址」短码。完整地址 `https://forum.ctspace.xyz/g/<slug>`，**打开即全屏试玩、不需要登录**，且地址栏不暴露 sb3 直链。适合对外分享 / 嵌入。
- `editors[].embedTemplate`：把 `{url}` 替换成 `sb3Url` 的**绝对地址**（需 URL 编码）即可嵌入任意第三方 Scratch 播放器，例如：
  `https://turbowarp.org/embed.html?project_url=<encodeURIComponent(https://forum.ctspace.xyz/uploads/xxx.sb3)>`

### 4.3 更新作品 / 发新版本

```
PATCH /api/projects/:id
{ "newFileResourceId": "<新 sb3 的 Resource.id>", "versionLabel": "v1.1", "changelog": "修了什么" }
```

传 `newFileResourceId` 会**新增一条版本记录**并把作品当前文件指向它。也可同时传 `title/summary/content/category/coverResourceId/allowDownload`。

---

## 5. 作品短链（作品网址）

把作品变成一个可直接打开就玩的网页地址，**内容永远是最新版本**，作者更新后已发出去的链接自动变新版。

| 方法 | 路径 | 认证 | 说明 |
|---|---|---|---|
| GET | `/api/shortlink/check?slug=xxx` | 公开 | 占用检测，返回 `{ available, reason, message }` |
| GET | `/api/shortlink/:slug` | 公开 | 解析短链，返回作品信息与 `sb3Url`（**不需要登录**） |
| POST | `/api/shortlink/:slug/view` | 公开 | 上报一次访问（爬虫不计、同 IP 30 分钟去重） |
| GET | `/api/shortlink/mine?projectId=` | 需登录 | 某作品的全部短链（仅作者/管理员） |
| POST | `/api/shortlink` | 需登录 | 创建：`{ projectId, slug }` |
| PATCH | `/api/shortlink/:slug` | 需登录 | 改名：`{ slug: "新名字" }`（旧名保留为别名，继续可访问） |
| POST | `/api/shortlink/:slug/disabled` | 需登录 | 停用/恢复：`{ disabled: true }` |

**短链命名规则**：3~32 位，只用小写英文字母、数字和 `-`，不能以 `-` 开头/结尾，**至少包含一个字母**（不能纯数字），不能连续 `--`，且不能是平台保留词（`api`/`admin`/`official`/`ctspace`/`g`/`d` 等）。

**其它约定**

- `reason` 取值：`format`（格式不合法）/ `reserved`（保留词）/ `blocked`（敏感词）/ `taken`（已被占用）。
- 每个作品最多 3 条（1 条主链接 + 2 条别名）；改名产生的旧名算一条。
- **系统自动创建的主链接不可停用**（它是详情页「运行」的落点），但可以改名。
- 解析响应里的 `state` 可能是 `ok` / `disabled`（作者已停用）/ `nofile`（没有 sb3）/ `notfound`。
- 命中别名时响应带 `aliased: true` 与 `canonicalSlug`，建议据此跳转到主链接。

```json
// GET /api/shortlink/emth4e
{
  "state": "ok",
  "slug": "emth4e", "canonicalSlug": "emth4e", "aliased": false,
  "title": "泰拉瑞亚", "summary": "泰拉瑞亚", "category": "game",
  "coverUrl": "/uploads/xxx.png",
  "sb3Url": "/uploads/xxx.sb3?v=1789394834331",
  "author": { "id": "...", "username": "TaLanFurry", "nickname": "塔岚TaLan" },
  "projectId": "...", "viewCount": 12
}
```

---

## 6. 云盘（Drive）与直链

| 方法 | 路径 | 认证 | 说明 |
|---|---|---|---|
| GET | `/api/drive/files` | 需登录 | 我的文件列表 + 用量 `{ items, usage:{ used, limit } }` |
| GET | `/api/drive/usage` | 需登录 | `{ used, limit }`（limit 为字节，随用户配额/站点默认变化） |
| POST | `/api/drive/upload` | 需登录 | 上传（multipart：`file` + 可选 `name`），受用户容量约束 |
| POST | `/api/drive/files/:id/replace` | 需登录 | **覆盖上传：换内容不换直链**（`shareId` 不变） |
| PATCH | `/api/drive/files/:id` | 需登录 | 改名 / 设密码 / 改公开性 |
| POST | `/api/drive/files/:id/regenerate` | 需登录 | 重新生成 shareId（旧链接失效） |
| DELETE | `/api/drive/files/:id` | 需登录 | 删除 |
| GET | `/api/drive/share/:shareId` | 公开 | 分享页元信息（可带密码解锁，返回访问 token） |
| GET | `/api/drive/raw/:shareId` | 公开 | **直链文件流**。参数：`download=1` 强制下载；密码保护的文件需带 `token` |
| GET | `/api/drive/by-share?shareIds=a,b` | 公开 | 批量按 shareId 取元信息（发帖引用卡片用） |

**关键点**

- **直链是长期有效的**：`https://forum.ctspace.xyz/api/drive/raw/<shareId>`。用「覆盖上传」更新内容时 `shareId` 不变，**已经嵌到别人网站/帖子里的链接继续有效**。
- 直链响应带 `Access-Control-Allow-Origin: *`，可被第三方页面直接 `fetch`（用于 sb3 预览等）。
- 下载限速为**全站共享**（当前 35Mbps，多人同时下载一起分）；上传共享 80Mbps。
- 文件可设访问密码；带密码的文件用直链时必须先调 `GET /api/drive/share/:shareId?password=xxx` 拿到 `token`，再 `GET /api/drive/raw/:shareId?token=xxx`。

---

## 7. 云变量（开发者平台）

为 Scratch 类作品提供云端键值存储；配合自建云变量服务，作品内可直接读写。

| 方法 | 路径 | 说明 |
|---|---|---|
| GET/POST | `/api/cloud/projects` | 我的云变量项目 / 新建项目 |
| GET | `/api/cloud/projects/:projectId` | 项目详情（含密钥） |
| PATCH | `/api/cloud/projects/:projectId` | 修改项目 |
| DELETE | `/api/cloud/projects/:projectId` | 删除项目 |
| GET/POST | `/api/cloud/projects/:projectId/variables` | 读写变量 |
| GET | `/api/cloud/projects/:projectId/history` | 变量历史 |
| POST | `/api/cloud/projects/:projectId/reset` | 重置全部变量 |
| POST | `/api/cloud/projects/:projectId/token/regenerate` | 轮换接入密钥 |
| GET | `/api/cloud/projects/:projectId/public` | 公开读取（作品内置排行榜等场景） |
| GET/POST | `/api/cloud/projects/:projectId/dev/variables` | 开发态调试用 |
| GET | `/api/cloud/projects/:projectId/dev/history` | 开发态历史 |
| POST | `/api/cloud/projects/:projectId/dev/reset` | 开发态重置 |
| GET | `/api/cloud/admin/projects` | 管理员：全部项目 |
| POST | `/api/cloud/admin/projects/:projectId/ban` | 管理员：封禁项目 |

---

## 8. 讨论与回复（论坛）

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/api/discussions` | 列表。`?sort=new&page=1&pageSize=20`，可选 `category`/`q` |
| GET | `/api/discussions/:id` | 详情（含回复） |
| GET | `/api/discussions/seq/:seq` | 按「帖号」取详情（对应站内 `/ts/<seq>`） |
| POST | `/api/discussions` | 发帖 `{ title, content, category?, tagIds?, poll? }`（需登录，限流 4/小时） |
| PATCH | `/api/discussions/:id` | 编辑 / 置顶 / 锁定（作者或管理员） |
| DELETE | `/api/discussions/:id` | 删除 |
| POST | `/api/posts` | 回帖 `{ discussionId, content }`（限流 15/小时） |
| PATCH / DELETE | `/api/posts/:id` | 改 / 删回帖 |

---

## 9. 扩展广场（Extension）

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/api/extensions` | 列表 `?category=ui&q=&sort=score&page=1` |
| GET | `/api/extensions/:id` | 详情（含 JS 源码文本） |
| GET | `/api/extensions/:id/raw` | 下载源码（`text/plain` 附件） |
| GET | `/api/extensions/:id/raw.js` | 以 `application/javascript` 返回，供编辑器直接加载 |
| GET | `/api/extensions/my` | 我发布的扩展（需登录） |
| POST | `/api/extensions` | 发布 `{ title, summary, category, code, license }`（限流 4/天） |
| PATCH / DELETE | `/api/extensions/:id` | 编辑 / 删除（作者或管理员） |
| POST | `/api/extensions/:id/like` | 点赞/取消 |

> 安全说明：扩展代码在平台内**只作为文本存储与展示**，平台不提供任何执行入口（不 eval）。

---

## 10. 评论 / 点赞 / 收藏 / 表态

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/api/comments?targetType=extension&targetId=ext01` | 取评论（`targetType`: discussion/post/project/extension） |
| POST | `/api/comments` | 发评论 `{ targetType, targetId, content, parentId? }`（限流 15/小时） |
| PATCH / DELETE | `/api/comments/:id` | 改 / 删（作者或管理员） |
| POST | `/api/likes` | 点赞/取消 `{ targetType, targetId }` |
| POST | `/api/reactions` | 表情表态 |
| GET | `/api/reactions/summary` | 表态汇总 |
| GET/POST | `/api/bookmarks` | 收藏列表 / 新增收藏 |
| GET | `/api/bookmarks/check` | 是否已收藏 |
| POST/DELETE | `/api/follow-tags` | 关注 / 取关标签 |
| GET | `/api/follow-tags` | 我关注的标签 |

---

## 11. 资源上传

| 方法 | 路径 | 说明 |
|---|---|---|
| POST | `/api/resources/upload` | 上传文件（multipart：`file` + `name`），返回 `{ id, url, filename, size, fileType }` |
| POST | `/api/resources/netdisk` | 登记网盘外链（`{ provider, link, extractCode?, requireLogin?, requireReward? }`） |
| GET | `/api/resources` | 按帖子/回复取资源附件 |
| GET | `/api/resources/works` | 某用户的作品集（sb3） |
| GET | `/api/resources/can-upload-video` | 当前用户是否允许上传视频 |
| PATCH / DELETE | `/api/resources/:id` | 改 / 删资源 |

> 上传 sb3 上限 200MB。**中文文件名请用 `name` 字段传**，不要依赖 `multipart` 的原始文件名。

---

## 12. 用户与社交

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/api/users/:id` | 用户公开资料（含 `level`/`exp`/`points`） |
| GET | `/api/users/by-username/:username` | 按用户名查 |
| GET | `/api/users/:id/followers` / `/following` | 粉丝 / 关注列表 |
| GET | `/api/users/:id/is-following` | 我是否关注了他 |
| POST | `/api/users/:id/follow` | 关注 / 取关 |
| GET | `/api/users/active` | 活跃用户 |
| GET | `/api/users/leaderboard` | 排行榜 |
| POST | `/api/users/accept-policy` | 同意最新政策（隐私/条款/公约） |
| POST | `/api/transfer` | 给用户转积分 |

---

## 13. 通知 / 签到 / 搜索 / 标签 / 统计 / 站点设置

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/api/notifications` | 我的通知 |
| POST | `/api/notifications/:id/read` | 单条已读 |
| POST | `/api/notifications/read-all` | 全部已读 |
| GET | `/api/checkin/status` | 签到状态 |
| POST | `/api/checkin` | 签到 |
| GET | `/api/search?q=&type=` | 全站搜索（讨论/作品/扩展/用户） |
| GET | `/api/tags` | 标签列表 |
| POST / PATCH / DELETE | `/api/tags`、`/api/tags/:id` | 标签管理（管理员） |
| GET | `/api/stats/public` | 公开统计：`{ users, posts, discussions, todayVisits, hot[] }` |
| POST | `/api/stats/visit` | 上报一次访问 |
| GET | `/api/settings` | **公开站点设置**（见下） |
| POST | `/api/reports` | 举报 |
| GET | `/api/changelogs` | 更新日志 |

`GET /api/settings` 公开字段：`siteName` `favicon` `bannerEnabled` `bannerText` `bannerColor` `editors`（编辑器嵌入模板 JSON 字符串）`rateLimit`（限流配置 JSON）`adEnabled` `adImageUrl` `adLinkUrl` `adTitle` `chatEnabled` `cloudDefaultQuotaMB` `policyVersion` `privacyPolicy` `serviceTerms` `communityGuidelines`。

> `editors` 是**字符串形式的 JSON**，需二次 `JSON.parse`。示例：
> `[{"name":"【推荐】TurboWarp","embedTemplate":"https://turbowarp.org/embed.html?project_url={url}"}, ...]`

---

## 14. 商店与订单

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/api/shop/products` | 商品列表（需登录） |
| GET | `/api/shop/products/:id` | 商品详情 |
| POST | `/api/shop/checkout` | 下单：`{ productId, quantity, payMethod, addressId?, variantSku? }`。`payMethod`: `points` / `wxpay` / `alipay` / `points_wxpay` / `points_alipay` |
| GET | `/api/shop/my/orders` | 我的订单 |
| POST | `/api/shop/my/orders/:id/cancel` | 取消订单 |
| POST | `/api/shop/my/orders/:id/confirm` | 确认收货 |
| GET/POST | `/api/addresses` | 收货地址列表 / 新增 |
| PATCH / DELETE | `/api/addresses/:id` | 改 / 删地址 |
| POST | `/api/addresses/:id/default` | 设为默认地址 |

商品类型 `type`：`virtual`（会员/权益）、`cloud`（云盘容量，购买后在用户容量上叠加）、`physical`（实物，需收货地址）、`digital`（数字商品）。

---

## 15. OrigMark 原创作品存证

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/api/origmark/public-key` | RSA-2048 公钥（用于离线校验签名） |
| GET | `/api/origmark/verify/:certNumber` | **验证某个证书编号的真伪** |
| GET | `/api/origmark/registrations` | 公开登记列表 |
| GET | `/api/origmark/registrations/:id` | 登记详情 |
| GET | `/api/origmark/stats` | 统计 |
| GET | `/api/origmark/preview-meta/:certNumber` | 预览元信息 |
| GET | `/api/origmark/preview-sb3/:certNumber` | 预览用 sb3（有 Referer/Origin 白名单） |
| GET | `/api/origmark/raw/:certNumber` | 原始文件（需作者审批通过） |
| POST | `/api/origmark/registrations/from-forum` | 从论坛作品发起登记 |
| POST | `/api/origmark/registrations/external` | 外部作品登记 |
| POST | `/api/origmark/registrations/:id/download-request` | 申请下载（需作者审批） |
| POST | `/api/origmark/registrations/:id/report` | 举报侵权 |

---

## 16. 其它模块（路径索引）

> 以下模块接口较多且偏站内使用，这里只列路径；字段以实际响应为准。

**私信 / 群聊（IM）**：`/api/im/dm/threads`、`/api/im/dm/:threadId/messages`、`/api/im/groups`、`/api/im/groups/:groupId/messages`、`/api/im/friends`、`/api/im/friends/request`、`/api/im/upload`、`/api/im/report` 等（约 30 个端点）。

**客服会话**：`/api/chat/conversations`、`/api/chat/conversations/:id/messages`。

**在线直播**：`/api/live/current`、`/api/live/session/:id`、`/api/live/session/:id/chat`、`/api/live/session/:id/qa`、`/api/live/session/:id/lottery`。

**在线状态**：`GET /api/presence/online`、`POST /api/presence/heartbeat`。

**投票**：`/api/polls`、`/api/polls/:id/vote`。

**财务公开**：`GET /api/finance`、`GET /api/finance/summary`。

**后台（仅管理员）**：`/api/admin/users`、`/api/admin/settings`、`/api/admin/stats`、`/api/admin/broadcast`、`/api/shop/admin/products`、`/api/origmark/admin/*`、`/api/reports/admin` 等。

---

## 17. 错误码

| HTTP | 含义 | 常见 message |
|---|---|---|
| 400 | 参数错误 | `人机验证失败，请刷新验证码后重新输入` / `云空间已满…` |
| 401 | 未登录 / token 失效 / 未激活 | `账号未激活，请先查收激活邮件完成激活后再登录。` |
| 403 | 无权限 | `只有作者本人可以为作品创建短链` |
| 404 | 资源不存在 | `作品不存在` |
| 409 | 冲突 | `用户名或邮箱已存在` / `这个名字刚被别人抢走了，换一个吧` |
| 429 | 触发限流 | `请求过于频繁，请稍后再试` |

---

## 附：完整路由索引

本项目共 276 条路由，完整列表可通过 `GET /api/docs`（Swagger 自动生成）查看。本文件覆盖了其中的公开与开发者常用部分。

---

*本文档由站方手写维护，与线上实现对齐；如发现不一致，以线上实际响应为准。*
