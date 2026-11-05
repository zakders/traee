# traee

Trae SOLO 每日签到（GitHub Actions 定时任务）。

## 当前状态：✅ 运行中

`Trae Daily Checkin` 工作流已启用，每天 UTC 01:30（北京时间 09:30）执行。
2026-11-05 手动触发两次验证均 `success`：

| 运行 | 结果 | 说明 |
| :--- | :--- | :--- |
| 首次 | `✅ 成功 1 / 已签 0 / 失败 0` | 缓存未命中，签到成功并写入加密状态 |
| 第二次 | `✅ 成功 0 / 已签 1 / 失败 0` | `Cache restored successfully`，解密还原成功，当日已签正确跳过 |
| 锁定上游 SHA 后 | `✅ 成功 0 / 已签 1 / 失败 0` | `HEAD is now at d289f0d`，固定版本拉取 + 缓存恢复正常 |

## 曾出过的问题与修复

**症状**：2026-11-03 起连续 3 天失败，报
`refresh http=401 code=20101 refresh token is invalid`。

**根因**：Trae 的 `refreshToken` 每次续期都会**轮换**，旧的立即作废（上游对 `20101` 的定义就是「已失效或被轮换掉」）。GitHub Actions 每次都是全新容器，而旧版工作流只从 Secrets 读取、没有回写，轮换后的新 token 随容器一起丢弃，第二天仍拿旧值去换 → 全线失效。

**修复**：

1. 从本机 Trae CN 客户端重新提取凭据，更新 `TRAE_REFRESH_TOKEN` 与 4 个设备变量。
2. 新增 `TRAE_CACHE_KEY` Secret（随机长串），工作流用 `actions/cache` + `openssl aes-256-cbc` 持久化轮换状态：恢复并解密 → 签到 → 加密并保存。缓存内容始终是**加密后**的 `state.enc`，明文不入库、不提交。
3. 新增 `Verify checkin result` 步骤。签到脚本在 token 失效时仍以 **exit 0** 结束，日志是「✅ 成功 0 / ❌ 失败 1」但步骤显示绿色——旧版就是这样静默失败了好几天。现在按统计行判断，成功 + 已签为 0 就直接让工作流失败。

> ⚠️ 状态文件由脚本写在**脚本所在目录** `script/` 下（`HERE = dirname(__file__)`），
> 不是仓库根目录。打包/还原缓存时必须保留 `script/` 前缀，直接照抄上游 `daily.yml` 会漏掉它们。

## Secrets 清单

| 变量 | 说明 |
| :--- | :--- |
| `TRAE_REFRESH_TOKEN` | 登录凭据，形如 `xxx=.18dxxx`；**不是** `eyJ` 开头的 accessToken |
| `TRAE_DEVICE_KEY_PEM` / `TRAE_DEVICE_PUB_PEM` | 设备 ECDSA 密钥对，续期签名必需 |
| `TRAE_DEVICE_ID` / `TRAE_MACHINE_ID` | 客户端真实设备号 |
| `TRAE_CACHE_KEY` | 加密轮换缓存的随机密钥，**不要删、不要改**（改了旧缓存无法解密，但下一轮会自动重建） |
| `PUSHPLUS_TOKEN` | PushPlus 推送，已配置 |
| `QYWX_TOKEN` | ⚠️ 工作流引用了但未配置，值为空。需要企业微信推送才补，否则忽略 |

设备密钥必须与 token 从**同一台机器、同一次登录**导出，混用会报 `10101` / `20403`。

## 以后 token 又失效了怎么办

1. 打开 Trae CN 客户端，退出登录并重新登录一次。
2. 提取新凭据（脚本零依赖、纯本地、不联网）：
   ```bash
   python trae_get_token.py                 # TRAE_REFRESH_TOKEN
   python trae_get_token.py --export-keys   # 4 个设备变量
   ```
   脚本地址：<https://github.com/L0NE-6/Trae-AutoCheckin/blob/main/trae_get_token.py>
3. 更新上表前 5 个 Secrets（网页 Settings → Secrets and variables → Actions，或 `gh secret set NAME -R zakders/traee`）。
4. 手动跑一次确认统计行是「✅ 成功 1」或「☑️ 已签 1」：
   ```bash
   gh workflow run checkin.yml -R zakders/traee
   gh run watch --exit-status
   ```
   **不要只看步骤是否绿色**，看 `Verify checkin result` 输出的统计。

> 同一个账号只在一处刷新（青龙 / 本地 / Actions 三选一），并发刷新会互相把 token 顶失效。
> 工作流已用 `concurrency` 组避免自身并发。

## 其他已知事项

- ⚠️ **Actions Cache 有过期策略**：缓存连续约 7 天未被访问会被 GitHub 自动清理。每天正常跑会一直续期；但如果工作流停跑或被禁用超过一周，缓存丢失后下一轮会回退到 Secrets 里的旧 token，可能再次报 `20101`。此时按上面「以后 token 又失效了怎么办」重新提取一次即可。彻底消除这个隐患需要把轮换后的 token 回写进 Secrets（需额外 PAT 与 `secrets` 写权限），当前未做。
- ✅ **上游脚本已锁定**：`L0NE-6/Trae-AutoCheckin@d289f0d7c2a232936235df96bbffbbeaefcfc771`（2026-11-03，v1.0.9），在 `.github/workflows/checkin.yml` 的 `fetch scripts` 步骤用 `ref:` 固定。上游 `main` 后续改动不会再影响本仓库。
  - 升级方式：先在新 SHA 上手动跑一次 `workflow_dispatch` 验证签到统计正常，再改 `ref`；不要直接跟 `main`。
  - ⚠️ 锁 SHA 挡不住上游**删库 / 改名**（仓库消失时 checkout 会直接失败）。要彻底摆脱外部依赖，需把 `trae_checkin.py` / `trae_credit_monitor.py` vendor 进本仓库并改为直接运行。
- 运行日志里的 `Node.js 20 is deprecated`（checkout / cache 动作）与 `ubuntu-latest` 迁移提示是上游动作的兼容性提示，不影响当前运行。
- 本仓库为 public，但缓存中的凭据已用 `TRAE_CACHE_KEY` 加密，明文状态文件从不提交。
