# 03 三层验证报告（L2 全程实录）

## A 单元级：22/22 全绿 ✅

忠实移植 Find Nearest Asteroid Code 节点 + Set APOD Fields / Set Asteroid Data 表达式逻辑，22 条断言：

- APOD 清洗：标题去 HTML 标签、`Explanation:` 前缀剥离、`&nbsp;`/多空白折叠、280 字截断（277+`...`）、image 优先 hdurl 回退 url、video 取 thumbnail_url 回退链、缺字段不抛错（A1–A12）
- NEO 处理：按 miss distance 排序取最近、括号剥离、千分位格式化、lunar 1 位小数、直径均值取整、hazardous 布尔透传、total_count（A13–A20）
- **坑#1 复现**（A21/A22）：NEO 空数组 / 空对象 → TypeError 抛错，两种形态全覆盖

## B mock 全链：15/15 全绿 ✅

13 节点全名全连线保留；2 个 NASA HTTP 换 Code 读固定样例（mock-scenario.json），2 个 homeAssistant 节点换 Code 构造 upsert payload 写盘。导入/执行走 CLI（`n8n import:workflow` + `n8n execute --id`）。

| 场景 | 结果 |
|---|---|
| S1 正常日 | APOD 清洗+hdurl、asteroid 取最近 BB2（120 万 km）、千分位/hazardous/total_count 全对 |
| S2 视频 APOD | 取 thumbnail_url、media_type=video 透传 |
| S3 NEO 空窗口 | 执行失败（坑#1 全链复现）；**APOD 并行分支照常成功落盘，互不影响** |

## C 真跑：端到端成功 ✅（0 元）

NASA 两 HTTP 真调（APOD 免 key；NeoWS `api_key=DEMO_KEY` 静态参数替代凭证，全程仅 2 次调用远低于 30/h 限额）；2 个 HA 节点改 POST 本地 HTTP 接收端（127.0.0.1）。

**真实输出证据**：

```json
// sensor.nasa_apod（date=2026-10-08）
{"title":"The Saturn System Smörgåsbord",
 "explanation":"How was the Saturn system imaged so clearly? ...（清洗截280字）",
 "image_url":"https://assets.science.nasa.gov/dynamicimage/.../...Sat2_1.5x_Labelled.png?...",
 "media_type":"image"}

// sensor.nasa_asteroid
{"name":"2015 TS238","date":"2026-10-08",
 "distance_km":"1,813,910","distance_lunar":"4.7",
 "diameter_m":36,"velocity_kmh":"56,916",
 "hazardous":false,"total_count":25}
```

**C 关两个关键实锤**：
1. 坑#2 反转——`apod-basic` 是扁平端点纯字符串，模板原样可用
2. 坑#3 新发现——feed 无 `thumbnail_url`、`url`=文章页；video 日 image_url 劣化