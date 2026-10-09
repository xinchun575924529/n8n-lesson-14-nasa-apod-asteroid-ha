# 02 坑集：3 大实锤 + 3 条提示（全部经断言/真跑复现）

## 🔴 坑#1 · NEO 空窗口崩溃（A21/A22 + B-S3 实锤复现）

**现象**：未来 3 天窗口内如果没有近地小行星（极少见但可能），`allAsteroids` 为空 → `allAsteroids[0]` = undefined → 访问 `.name` 抛 `TypeError: Cannot read properties of undefined (reading 'name')`，asteroid 整条分支被阻断。`near_earth_objects` 为空对象（`{}`）时同样炸。

**根因**：原模板 `Find Nearest Asteroid` Code 节点无守卫。

**处置（已固化进 workflow-custom.json）**：Code 开头加守卫——

```js
const neoData = $input.first().json.raw_objects || {};
// ...展平循环不变...
if (!allAsteroids.length) {
  return [{ json: {
    name: '(none in 3-day window)',
    date: $input.first().json.start_date || '',
    distance_km: '-', distance_lunar: '-', diameter_m: 0,
    velocity_kmh: '-', hazardous: false, total_count: 0
  } }];
}
```

**B-S3 全链证据**：asteroid 分支执行失败，但并行的 APOD 分支不受影响照常成功——双分支隔离性是模板设计的亮点。

## 🟡 坑#2（预判不成立，反转案例）· APOD title/explanation 疑为 `{rendered}` 对象

**预判**：science.nasa.gov 是 WordPress JSON，WP REST 的 title/explanation 通常包一层 `{rendered: ...}` 对象，模板 `String($json.title)` 会变 `[object Object]`。

**C 真跑反转**：`apod-basic` 是 **NASA 定制扁平端点**，title/explanation 是**纯字符串**，模板表达式无需改动 ✅。explanation 内的 `<strong>Explanation:</strong>` 与 `<a>` 标签被表达式正确剥离。

**教学价值**：预判坑必须用真数据验证，别凭框架经验改代码。

## 🟡 坑#3 · 新 feed 字段漂移：video 日 image_url 劣化（C 真跑新发现）

**现象**：`apod-basic` 端点**没有 `thumbnail_url` 字段**，且 `url` 是**文章页 URL 不是媒体文件**。

- image 日：无碍，`hdurl` 恒有（C 真跑实测拿到真图 URL）
- video 日：模板取图链 `thumbnail_url || url` 退化成文章页链接 → HA 仪表盘 Picture 卡**裂图**

**处置（已固化进 workflow-custom.json）**：Set APOD Fields 补 `permalink = {{ $json.url }}` 字段，video 日仪表盘至少能做成可点击的链接卡；进阶可再调 YouTube oEmbed 取缩略图。

## 🟢 提示级（模板已知/可规避）

| # | 提示 | 处置 |
|---|---|---|
| 4 | HA sensor API upsert 重启即失（模板便签自述） | docs/05 给 `configuration.yaml` 持久化方案 |
| 5 | DEMO_KEY 限流 30 req/h/IP | 教学够；生产建议 api.nasa.gov 免费注册个人 key |
| 6 | scheduleTrigger 无 timezone 设置 | 按 n8n 实例时区跑（settings.timezone 或 GENERIC_TIMEZONE），教学需说明 |