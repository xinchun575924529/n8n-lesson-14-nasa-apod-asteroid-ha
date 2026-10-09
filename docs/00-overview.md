# 00 总览：这个工作流干什么

每天 06:00（scheduleTrigger）或手动触发（manualTrigger），并行执行两条分支，把两个 NASA 免费数据源推送到 Home Assistant 仪表盘。

## 数据源

| 分支 | 接口 | 凭证 |
|---|---|---|
| APOD（每日天文一图） | `science.nasa.gov/wp-json/wp/v2/apod-basic?per_page=1` | **免 key**（NASA 退役旧 api.nasa.gov APOD 端点后的官方新源） |
| 最近小行星 | `api.nasa.gov/neo/rest/v1/feed`（start/end_date 未来 3 天） | Query Auth `api_key`，免费注册；教学可用公共 `DEMO_KEY`（30 req/h/IP） |

## 输出（HA 实体）

- `sensor.nasa_apod`：title / explanation（清洗截 280 字）/ image_url / media_type / date
- `sensor.nasa_asteroid`：name / date / distance_km（千分位）/ distance_lunar（地月倍数，1 位小数）/ diameter_m / velocity_kmh / hazardous / total_count

## 卖点

- 节点少（9 功能节点）、链路短、双分支结构清晰，适合入门课
- 两数据源全免费免部署；HA 可用本地 HTTP 接收端替代做验证（见 docs/04）
- 智能家居仪表盘装饰型用例，学员成就感强