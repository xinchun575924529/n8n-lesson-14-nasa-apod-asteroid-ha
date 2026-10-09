# 01 架构与节点拆解

## 拓扑

```text
When 6AM Daily (scheduleTrigger 1.3) ──┬─→ Fetch NASA APOD Data (httpRequest 4.4)
Manual Test Trigger (manualTrigger) ──┤        ↓
                                      │   Set APOD Fields (set 3.4)
                                      │        ↓
                                      │   Send APOD to Home Assistant (homeAssistant)
                                      │
                                      └─→ Fetch Asteroid Feed (httpRequest 4.4)
                                               ↓
                                          Set Asteroid Data (set 3.4)
                                               ↓
                                          Find Nearest Asteroid (code 2)
                                               ↓
                                          Send Asteroid to Home Assistant (homeAssistant)
```

两条分支共享触发器、完全并行、互不阻塞——B 关 S3 实测：asteroid 分支抛错，APOD 分支照常成功落盘。

## 关键节点

### Fetch NASA APOD Data
- GET `https://science.nasa.gov/wp-json/wp/v2/apod-basic?per_page=1`，retryOnFail 3 次
- 响应是**数组**，n8n httpRequest 自动拆 item（per_page=1 → 1 item），无需改模板

### Set APOD Fields（5 个表达式）
1. `title`：`String($json.title).replace(/<[^>]*>/g, '')` —— 去 HTML 标签
2. `explanation`：剥 `<strong>Explanation:</strong>` 前缀 → 去所有标签 → `&nbsp;` 与多空白折叠 → 截 280 字（277+`...`）
3. `image_url`：`media_type === 'image' ? (hdurl || url) : (thumbnail_url || url)` —— 分支取图
4. `media_type` / `date`：透传

### Fetch Asteroid Feed
- GET `https://api.nasa.gov/neo/rest/v1/feed`，Query Auth `api_key`
- `start_date`/`end_date` 用 Luxon：`{{ $now.toFormat('yyyy-MM-dd') }}` / `{{ $now.plus({days:3}).toFormat('yyyy-MM-dd') }}`

### Set Asteroid Data
- 透传 `element_count` → `count`、`near_earth_objects` → `raw_objects`

### Find Nearest Asteroid（Code）
- 遍历按日期分组的 `near_earth_objects` 对象，展平成数组
- 名称剥括号、距离/速度千分位格式化、直径取 min/max 均值取整、lunar 1 位小数
- 按 `miss_km_raw` 升序排序取第一个（最近者）
- ⚠️ 原模板**没有空窗口守卫**——见 docs/02 坑#1

### Send * to Home Assistant（×2）
- homeAssistant 节点 state upsert：`sensor.nasa_apod` / `sensor.nasa_asteroid`
- ⚠️ 通过 API upsert 的 sensor 在 HA 重启后消失（模板便签自述），教学需讲清（见 docs/05）