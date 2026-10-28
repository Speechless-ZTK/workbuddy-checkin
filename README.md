# 🐱 WorkBuddy 签到助手 · macOS 适配版

> [!NOTE]
> **一句话**：把官方「签到助手」skill 在 **macOS** 上跑通、修好、并加上 **微信每日播报** 的自动化工具集。
> 核心能力 = **每日自动签到 + 派猫猫旅行 + 微信推送**。

## 📋 速览

| 项目 | 内容 |
|---|---|
| 基于 | 官方 skill `totorosir-workbuddy-checkin` |
| 适用 | macOS 客户端（5.6.2+） |
| 核心功能 | 每日签到 · 派猫猫旅行 · 微信(PushPlus)播报 |
| 授权通道 | 通道 B：网页会话授权（macOS 下唯一可行） |
| 触发方式 | WorkBuddy 自动化，每天 **09:00** |
| 仓库状态 | 公开 · `Speechless-ZTK/workbuddy-checkin` |

---

## 一、解决了什么问题

> [!WARNING]
> macOS 上原 skill **直接运行会静默失败**——自动化每天 09:00 触发，但你以为签了其实没签。

| 问题 | 现象 | 本仓库解法 |
|---|---|---|
| 登录态通道不可用 | 客户端加密登录态，本机解不出 → 通道 A/C 走不通 | 改用 **通道 B**（浏览器授权码，本地存凭据） |
| 本地代理 TLS 拦截 | 客户端起的本地代理对目标站用自签证书 → `[SSL: CERTIFICATE_VERIFY_FAILED]` | 请求 **强制直连**，绕过本地代理 |
| `--check` 假阳性 | 查询失败仍报「已签」，误导排查 | 改如实返回 `error / query_failed` |
| 旅行派遣谎报 | depart 空 payload 被 400 拒，却报「已派出」 | 必须带 `location_id`，并严格校验成败 |

---

## 二、有什么功能

| 功能 | 说明 |
|---|---|
| 🔑 每日签到 | 自动签「Buddy 加油站」，领积分（+100/天），查连续天数与余额 |
| 🐱 派猫猫旅行 | 到达自动领旅行积分；空闲自动派往新目的地（咖啡馆 / 商场 / 健身房 / 古镇） |
| 🔄 先领后派 | 每天按「先领到达奖励 → 再派新行程」执行，幂等安全 |
| 📲 微信推送 | PushPlus 中转，把每日结果（签到 + 🌏 旅行）推到微信 |
| 🚨 失败提醒 | 签到 / 旅行失败主动推微信，第一时间发现断签 |
| 🧭 通道择一 | `auto_checkin.py` 自动在本机登录态 / 网页授权间择一 |

---

## 三、增加了什么（相对原 skill）

| 改动点 | 说明 |
|---|---|
| 网络层绕过本地代理 | 强制直连 + 极端 SSL 兜底重试，解决 09:00 全失败 |
| 修复 `--check` 假阳性 | 查询失败如实报错 |
| 修复旅行派遣谎报 | depart 必带 `location_id`，未指定自动随机选 1–4；严格校验 `http==200 且 code==0` |
| 旅行状态诚实播报 | 区分「已领取 / 已派出(含目的地) / 旅行中 / 派遣满 / 不可用」 |
| 微信每日播报 | `notify_config.json`（扁平结构，PushPlus） |
| 每日 09:00 自动化 | recurring 自动化，托管 Python 跑统一入口 |

> [!IMPORTANT]
> 原 skill 推送模块只认 **扁平旧结构**（`pushplus_token` 顶层），新结构 `channels.wechat.*` 只显示不生效——本仓库已统一用扁平写法。

---

## 四、使用方式

### 命令速查

```bash
python3 auto_checkin.py                # 完整：签到 + 旅行 + 推送
python3 auto_checkin.py --check        # 仅查询（已修假阳性）
python3 auto_checkin.py --no-notify    # 不推送
python3 auto_checkin.py --no-travel    # 不玩旅行
```

<details>
<summary>📖 完整配置步骤（点击展开）</summary>

**1. 准备**
将 5 个 `.py` + `templates/` 落位到 `~/.workbuddy/scripts/`；脚本已锁定 WorkBuddy 托管 Python 路径（按需改成你本机路径）。

**2. 通道 B 授权（一次性）**
1. 浏览器登录 https://www.workbuddy.cn（与客户端同一账号）；
2. F12 → Network → 刷新 → 任一发往 `www.workbuddy.cn` 的请求右键 **Copy as cURL**；
3. `python3 web_auth.py --setup-file /path/to/curl.txt`（凭据存本机 `checkin-cred.json`，0600，不外传）。
   > 若 Cookie 是 HttpOnly 读不到，可改用页面 Console 注入授权码（见 `templates/auth-snippet.js`）或导出 HAR：`--setup-har`。

**3. 微信推送（可选）**
1. 注册 https://www.pushplus.plus，复制「一对一推送」Token；
2. `cp notify_config.json.example notify_config.json` 填入 Token；
   （`success_notify: true` = 每日都推；`false` = 仅失败推）
3. 测试：`python3 push_message.py --title 测试 --content 测试`。

**4. 自动化**
每天 09:00 由 WorkBuddy 自动化触发统一入口；正常日收微信日报，失败日收失败提醒。

</details>

---

## 五、带来的效果

> [!TIP]
> 设一次，之后全自动，且 **报的都是真话**。

- ✅ **真正无人值守**：修复前 09:00 每天静默失败；修复后每天自动签 + 派旅行。
- ✅ **诚实可靠**：消灭 `check` 假阳性与派遣谎报。
- ✅ **断签可感知**：微信失败提醒第一时间发现异常。
- ✅ **旅行收益闭环**：到达自动领、空闲自动派，播报如实反映每一步。

---

## 六、已知边界 / 免责

> [!CAUTION]
> 通道 B 的网页会话 Cookie **会过期**；过期后自动化报 `auth_failed`，需重新走一次「Copy as cURL → 授权」（约 30 秒）。

- 凭据（`checkin-cred.json`）与推送 Token（`notify_config.json`）**不入库**，需本地自配，切勿提交（`.gitignore` 已强制排除）。
- 仅用于个人自动化签到，请遵守 WorkBuddy 服务条款；官方接口字段若调整需相应适配脚本。
