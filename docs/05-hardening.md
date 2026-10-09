# 05 生产加固清单

## 1. NEO 空窗口守卫（必修）
原模板无守卫，直接使用 `workflow-custom.json`（已含守卫），或按 docs/02 坑#1 手动补。

## 2. video 日图片兜底
新 feed 无 `thumbnail_url`。修复版已加 `permalink` 字段；追求完美可在 Set 后加 HTTP 节点调 YouTube oEmbed（`https://www.youtube.com/oembed?url=...`）取 `thumbnail_url`。

## 3. HA 实体持久化
API upsert 的 sensor 在 HA 重启后消失（模板便签自述）。生产建议：
- 在 `configuration.yaml` 用 `template:` 平台定义常驻 sensor，attribute 从 `sensor.nasa_apod` 镜像；或
- 改用 MQTT（`mqtt` 集成 + `retain: true`）替代 REST upsert。

## 4. API key 升级
`DEMO_KEY` 仅 30 req/h/IP 且与其他使用者共享限额。api.nasa.gov 免费注册个人 key（1 分钟），限额 1000 req/h。

## 5. 定时时区
scheduleTrigger 无 timezone 字段，按 n8n 实例时区执行。确认实例 `GENERIC_TIMEZONE` 或工作流 `settings.timezone` 设为你的本地时区（如 `Asia/Shanghai`），否则「每天 6 点」会按 UTC 跑偏 8 小时。

## 6. 错误网
两条分支各接一个 Error Trigger 或把 httpRequest 的 `onError: continueRegularOutput` 打开 + 后置 If 判空，避免 NASA 接口瞬断时执行记录炸红。

## 7. NeoWS 窗口
模板窗口 = 未来 3 天。想更密/更稀，改 `Fetch Asteroid Feed` 的 `end_date` 表达式 `$now.plus({days:N})` 即可；注意 NeoWS 单次最多 7 天。