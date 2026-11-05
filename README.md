# traee

Trae SOLO 每日签到（GitHub Actions 定时任务）。

## 当前状态：⚠️ 已停用

`Trae Daily Checkin` 工作流目前为 `disabled_manually`，**不会执行签到**。

原因：`TRAE_REFRESH_TOKEN` 已失效，接口返回
`refresh http=401 code=20101 refresh token is invalid`。
2026-11-03 起连续失败；此前的 11-02 及更早均成功。

注意：脚本在 token 失效时仍以 exit 0 结束，日志显示「✅ 成功 0 / ❌ 失败 1」，
步骤看起来是绿的，但实际没有签到成功。判断真实结果要看这段统计，不要只看步骤状态。

## 恢复步骤

1. 重新登录 Trae，获取新的 refresh token（以及配套的 DEVICE_ID / MACHINE_ID / KEY_PEM / PUB_PEM）。
2. 在本仓库 Settings → Secrets and variables → Actions 中更新对应 secrets。
   - 若需要企业微信推送，还需补上 `QYWX_TOKEN`（当前 workflow 引用了它，但 secret 未配置，值为空）。
   - `PUSHPLUS_TOKEN` 已配置，PushPlus 推送正常。
3. 手动触发一次 `workflow_dispatch` 验证，确认日志里「✅ 成功 1」。
4. 重新启用定时任务：
   ```
   gh workflow enable checkin.yml -R zakders/traee
   ```

## 其他已知风险

- 签到脚本通过 `actions/checkout` 从第三方仓库 `L0NE-6/Trae-AutoCheckin` 动态拉取，
  未锁定 commit SHA。上游改名、删库或改动都可能导致本工作流失效。
  建议后续改为固定 commit SHA 或将脚本 vendor 进本仓库。
