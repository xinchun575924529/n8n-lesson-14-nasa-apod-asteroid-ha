# n8n 实战课 14：NASA 每日天文图 + 最近小行星推送到 Home Assistant 仪表盘

> 官方模板：[Show NASA APOD and nearest asteroid data in Home Assistant dashboards](https://n8n.io/workflows/19849/)（模板 ID 19849，9 功能节点 + 4 张教学便签）

每天早上 6 点（或手动触发）并行跑两条分支：**APOD 分支**从 NASA 官方新 feed 拉「每日天文一图」（免 API key），清洗标题/解说/图片地址后推送到 Home Assistant 的 `sensor.nasa_apod`；**小行星分支**调 NeoWS 近地天体接口（未来 3 天窗口），按擦肩距离排序找出最近的小行星，推送到 `sensor.nasa_asteroid`（名称/距离 km 与地月倍数/直径/速度/是否潜在危险/窗口内总数）。

## 你会学到

1. **双数据源并行分支**：一个触发器带两条互不影响的分支（S3 实测：asteroid 分支炸了 APOD 照常落盘）
2. **新版 APOD feed 的字段真相**：science.nasa.gov 的 `apod-basic` 是定制扁平端点，title/explanation 是纯字符串（**不是** WP 标准 `{rendered:...}` 对象）——本课 C 真跑实锤
3. **表达式清洗三板斧**：HTML 标签剥离、`Explanation:` 前缀去除、`&nbsp;`/多空白折叠、280 字截断
4. **分支取图逻辑**：image 日优先 hdurl 回退 url；video 日取 thumbnail_url（⚠️ 新 feed 已无此字段，见坑#3）
5. **NeoWS 数据处理**：按日期分组的对象遍历、miss distance 排序、千分位格式化、地月距离换算
6. **三大实锤坑**（全部经断言复现）：NEO 空窗口守卫 / 新 feed 字段说明 / video 日 image_url 劣化（见 docs/02-pitfalls.md）

## 课程文件

| 文件 | 内容 |
|---|---|
| `workflow.json` | 官方模板原样（9 功能节点，n8n 可导入形态） |
| `workflow-custom.json` | 修复版：坑#1 NEO 空窗口守卫 + 坑#3 permalink 兜底字段 |
| `docs/00~05` | 总览 / 架构 / 坑集 / 三层验证报告 / 免费替身 / 生产加固 |
| `exercises/exercise.md` | 3 道动手练习（含答案） |

## 三层验证结论（本课全部实测通过）

- **A 单元级**：Find Nearest Asteroid Code + Set 表达式忠实移植，**22/22 断言全过**（含坑#1 双形态复现）
- **B mock 全链**：13 节点保结构 mock（NASA×2 + HA×2 替换），三场景 **15/15 断言全过**（正常日 / 视频 APOD / NEO 空窗口全链复现）
- **C 真跑**：NASA 两 API 真调（APOD 免 key + NeoWS DEMO_KEY），HA 换本地 HTTP 接收端，**端到端 0 元成功**：APOD=《The Saturn System Smörgåsbord》真图 URL、最近小行星=2015 TS238（1,813,910 km ≈ 4.7 倍地月距，total_count=25）

## 一分钟看懂它干嘛

```text
每天 06:00 / 手动
 ├─ APOD 分支：science.nasa.gov feed（免key）
 │    → 清洗标题/解说、按媒体类型取图
 │    → upsert HA sensor.nasa_apod
 └─ 小行星分支：api.nasa.gov NeoWS（未来3天）
      → 按擦肩距离排序取最近者
      → upsert HA sensor.nasa_asteroid
```