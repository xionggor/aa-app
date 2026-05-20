# AA记账 修改清单

## 待修复 Bug

### Bug 1: 账单详情页空白（刷新后）
**根因：** `renderBillDetail()` 不调 API 兜底加载。页面刷新后 `_cachedProjects` 清空，`DB.projects` 中的项目是概要数据（无 bills），导致 `p.bills.find()` 找不到账单，页面空白。
**修复：** `renderBillDetail()` 增加 API 兜底获取逻辑，与 `renderSettle()`/`renderRoom()` 一致。

### Bug 2: 后加入成员不分摊旧账单
**根因：** `saveBill()` 在创建时快照 `split = p.members.slice()`。有新成员加入后，旧账单的 `split` 数组不含新成员，结算时遍历 `bill.split` 自然不会算他。
**修复：** 全员平分创建的账单加 `splitAll: true` 标记，展示/结算时动态用 `p.members` 代替 `bill.split`。

### Bug 3: 成员信息弹窗收款码不显示
**根因：** `showMemberInfo()` 的 `hasCode` 依赖 `p.memberInfos`，但 Worker 的 `sanitizeProject()` 不返回此字段，`hasCode` 永远 `false`。
**修复：** 去掉 `hasCode` 条件判断，直接调 `/api/user/:id/payment-code` API，404 静默忽略。

### Bug 4: 加入项目后成员头像不全
**根因：** `renderRoom()` 渲染成员头像时用 `findUser(id)`，但 `DB.users` 只存了 mock 数据（GG/小王），API 注册的真实用户查不到，被 `.filter(Boolean)` 过滤掉。
**修复：** `findUser()` 找不到时，调 Worker 新增的 `GET /api/user/:id` 接口获取，缓存回 `DB.users`。

### Bug 5: 数字键盘消失无法继续
**状态：** 已修复 ✅（金额显示区点击可重新唤起键盘）

### Bug 6: 账号设置点击更换头像无反应
**根因：** `showAvatarPicker()` 函数中 `overlay` 和 `content` 变量未声明，直接使用导致 ReferenceError，弹窗无法弹出。
**修复：** 在函数开头添加 `const overlay = document.getElementById('modal-overlay'); const content = document.getElementById('modal-content');`

### Bug 7: Worker 全部 API 返回 500（bcryptjs 不兼容 CF Workers）
**根因：** `auth.ts` 使用 `bcryptjs`，该库依赖 Node.js 内部模块，Cloudflare Workers 不支持。
**修复：** 替换为 Web Crypto API（PBKDF2 + SHA-256），Worker 原生支持，无需第三方库。
**影响：** KV 中旧密码哈希与新算法不兼容，已清理 KV 重新注册。

### Bug 8: join 输入框 text-transform:uppercase 让用户看到大写字母
**根因：** CSS 样式 `text-transform:uppercase` 让输入的分享码显示为大写，用户困惑。
**修复：** 改为 `lowercase` + `letter-spacing:2px`，更清晰。
**注意：** 部署 Worker 时必须使用 `--config wrangler.toml`，否则 KV 绑定不生效。

## 功能修改

### P6: 货币符号改为 $（USD）
**范围：** 前端 25 处 + Worker 3 处，全局替换 ¥ → $
**涉及页面：** 房间页、数字键盘、账单明细、结算页、首页、日志

### P7: 结算页加入实时汇率
**位置：** 结算页（`renderSettle()`）
**方案：** 在转账列表下方显示 `≈ ¥xxx（汇率 x.xx）`，调 `open.er-api.com/v6/latest/USD` 获取实时 USD/CNY 汇率
**缓存：** 内存缓存，避免每次渲染都请求

### P8: Worker 新增 GET /api/user/:id
**用途：** 返回用户基本信息（username、avatar、avatarBg），供前端查询 API 注册用户的头像和名字
**鉴权：** authMiddleware（需登录即可，不限制 admin）
**响应：** `{ id, username, avatar, avatarBg }`
