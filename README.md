# WorkBuddy 签到助手 · macOS 适配版（每日自动签到 + 派猫猫旅行）

## 一、这是一份什么说明

本仓库是基于 WorkBuddy 官方「签到助手」skill（`totorosir-workbuddy-checkin`）改造而来的**自动化签到工具集**，专门在 **macOS 客户端环境**下做了适配与修复。

原 skill 的定位：通过接口直签的方式，自动完成 WorkBuddy「Buddy 加油站」每日签到，并顺带完成「派猫猫旅行」玩法，**全程无需点击 GUI**。

本仓库在其基础上，解决了 macOS 上原本跑不通自动化的若干拦路问题，并补齐了推送与诚实播报能力——让这套东西真正做到"设一次、之后全自动、且报的都是真话"。

---

## 二、解决了什么问题

在 macOS 客户端（5.6.2+）上，原 skill 直接运行会遇到以下硬阻塞：

1. **登录态通道不可用**
   macOS 客户端把登录态加密存储，本机拿不到解密密钥，导致通道 A（本机登录态）与通道 C（长效令牌本机导出）都走不通。本仓库改用**通道 B（网页会话授权）**：在浏览器登录态下导出一段授权码，由脚本本地保存凭据，客户端不开也能签。

2. **自动化触发即失败（本地代理 TLS 拦截）**
   WorkBuddy 客户端会在本机起一个本地代理（如 `HTTPS_PROXY=127.0.0.1:61149`）。09:00 自动化运行时，该代理对 `www.workbuddy.cn` 做了 TLS 拦截并使用自签名证书，urllib 证书校验失败 → 每天 09:00 触发但全部 `[SSL: CERTIFICATE_VERIFY_FAILED]` 失败，实际根本没签到。

3. **`--check` 假阳性**
   原脚本在查询失败时也会返回"已签"，误导排查（看上去"已签"其实是网络全挂）。

4. **旅行派遣谎报**
   旅行 dispatch 接口必须传目的地 `location_id`（1–4），原脚本传空 payload → 服务器 `HTTP 400` 拒绝；但脚本"只要调过就报已派出"，导致出现"积分领了但猫猫压根没派出去"的假象。

---

## 三、有什么功能

- **每日签到**：自动完成「Buddy 加油站」签到，领取积分（正常 +100/天），可查询连续签到天数与积分余额。
- **派猫猫旅行**：在签到之外自动玩「派猫猫旅行」——猫猫到达后自动领取旅行积分，空闲时自动派往新目的地（四选一：咖啡馆 / 商场 / 健身房 / 古镇，奖励区间相同 5~10 分）。
- **先领后派**：每天运行按"先领到达奖励、再派遣新行程"的顺序执行，幂等安全。
- **微信推送（PushPlus 中转）**：把每日结果推到微信，含签到结论 + 🌏 旅行状态。
- **失败提醒**：签到 / 旅行失败时主动推微信，第一时间发现断签。
- **统一入口 + 通道择一**：`auto_checkin.py` 作为统一入口，自动在本机登录态通道与网页授权通道之间择一。

---

## 四、增加了什么（相对原 skill 的改动）

| 改动点 | 说明 |
|---|---|
| 网络层绕过本地代理 | 请求强制直连目标域名（使用系统真实证书），极端 SSL 异常时兜底忽略证书重试，彻底解决 09:00 全失败 |
| 修复 `--check` 假阳性 | 查询失败如实返回 `error / query_failed`，不再谎报"已签" |
| 修复旅行派遣谎报 | depart 必须带 `location_id`；未指定时自动从 `travel/config` 随机选 1–4 再派；严格校验 `http==200 且 code==0` 才算派遣成功 |
| 旅行状态诚实播报 | `_fmt_travel()` 区分「已领取 / 已派出（含目的地名）/ 旅行中 / 派遣满 / 接口不可用」，不再无脑报"已派出" |
| 微信每日播报 | `notify_config.json`（扁平结构，PushPlus）开启每日推送，内容含签到结论 + 旅行状态 |
| 每日 09:00 自动化 | 注册 recurring 自动化，每天 09:00 用 WorkBuddy 托管 Python 运行统一入口 |

> ⚠️ 踩坑备忘：原 skill 的推送模块只认**扁平旧结构**（`pushplus_token` 放顶层），新结构 `channels.wechat.*` 只在诊断里显示、实际发送取不到，本仓库已统一用扁平写法。

---

## 五、使用方式

### 1. 准备
- macOS + 已登录的 WorkBuddy 客户端
- WorkBuddy 自带托管 Python（仓库内脚本已锁定示例路径 `/Users/orion/.workbuddy/binaries/python/versions/3.13.12/bin/python3`，按需改成你本机路径）
- 将 5 个 `.py` 脚本 + `templates/` 落位到 `~/.workbuddy/scripts/`

### 2. 通道 B 授权（一次性）
1. 浏览器打开并登录 https://www.workbuddy.cn （与客户端同一账号）；
2. F12 → Network，刷新页面，任一条发往 `www.workbuddy.cn` 的请求右键 → Copy as cURL；
3. 把 cURL 存成文件后运行：
   ```bash
   python3 web_auth.py --setup-file /path/to/curl.txt
   ```
   凭据存本机 `~/.workbuddy/checkin-cred.json`（cookie 模式，权限 0600，不外传）。

   > 若页面 Cookie 是 HttpOnly 读不到，可改用页面 Console 注入授权码（见 `templates/auth-snippet.js`）或导出 HAR：`--setup-har`。

### 3. 微信推送（可选）
1. 注册 https://www.pushplus.plus，复制「一对一推送」Token；
2. 复制模板并填入 Token：
   ```bash
   cp notify_config.json.example notify_config.json
   # 编辑 notify_config.json，填入 pushplus_token
   # success_notify: true  = 每日都推（含正常签到）；false = 仅失败推
   ```
3. 测试：`python3 push_message.py --title 测试 --content 测试`

### 4. 手动运行 / 验证
```bash
python3 auto_checkin.py                # 完整执行：签到 + 旅行 + 推送
python3 auto_checkin.py --check        # 仅查询（已修复假阳性）
python3 auto_checkin.py --no-notify    # 不推送
python3 auto_checkin.py --no-travel    # 不玩旅行
```

### 5. 自动化（每天 09:00）
由 WorkBuddy 自动化定时触发统一入口；正常签到日你会收到微信日报（签到结论 + 旅行状态），失败时收到失败提醒。

---

## 六、带来的效果

- **真正无人值守**：修复前 09:00 自动化每天静默失败（你以为签了其实没签）；修复后每天自动签 + 派旅行，无需手动点。
- **诚实可靠**：彻底消灭 `check` 假阳性与派遣谎报——脚本报"已派出"就真的派出、报"已签"就真的签了。
- **断签可感知**：微信失败提醒让偶发的网络 / 凭据过期问题第一时间被发现，不再"悄悄没签"。
- **旅行收益闭环**：到达自动领积分、空闲自动再派，积分收益最大化，且播报如实反映每一步动作。

---

## 七、已知边界 / 免责

- 通道 B 的网页会话 Cookie 会过期；过期后自动化会报 `auth_failed`，需重新走一次「Copy as cURL → 授权」（约 30 秒）。
- 凭据（`~/.workbuddy/checkin-cred.json`）与推送 Token（`notify_config.json`）**不纳入本仓库**，需本地自行配置，切勿提交。仓库 `.gitignore` 已强制排除这两类文件与 `*.log`。
- 本仓库仅用于个人自动化签到，请遵守 WorkBuddy 服务条款；官方接口字段若调整，可能需相应适配脚本。
