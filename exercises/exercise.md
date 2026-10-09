# 动手练习（含答案）

## 练习 1：复现坑#1 并打守卫
在 B mock 环境把 NeoWS 返回的 `near_earth_objects` 改成 `{}`，手动触发。asteroid 分支报什么错？APOD 分支受影响吗？然后给 Code 节点补守卫。

<details><summary>答案</summary>
报 `TypeError: Cannot read properties of undefined (reading 'name')`——`allAsteroids` 为空时 `allAsteroids[0]` 是 undefined。APOD 分支**不受影响**（并行分支隔离，B-S3 实测）。守卫打法：循环前 `const neoData = $input.first().json.raw_objects || {}`，排序前 `if (!allAsteroids.length) return [{json:{name:'(none in 3-day window)', ..., total_count: 0}}]`。修复版 workflow-custom.json 已含。
</details>

## 练习 2：验证坑#2 的反转
直接浏览器/curl 访问 `https://science.nasa.gov/wp-json/wp/v2/apod-basic?per_page=1`，看返回 JSON 的 `title` 和 `explanation` 字段。它们是字符串还是 `{rendered: ...}` 对象？为什么这不遵循 WP REST 惯例？

<details><summary>答案</summary>
是**纯字符串**。`apod-basic` 是 NASA 定制的扁平端点（旧 api.nasa.gov APOD 端点退役后的迁移源），不是 WP 标准 post 类型，所以 title/explanation 没有 `rendered` 包裹。教学点：预判坑必须真数据验证，别凭框架经验改代码。
</details>

## 练习 3：给 video 日补真缩略图
新 feed 没有 `thumbnail_url`，video 日 `image_url` 会退化成文章页链接。在 `Set APOD Fields` 之后加一个 HTTP 节点，当 `media_type=video` 时调 YouTube oEmbed 取真缩略图。写出思路与关键表达式。

<details><summary>答案</summary>
思路：`url` 字段在 video 日是 YouTube 嵌入链接（APOD 视频几乎都托管在 YouTube）。加 httpRequest 节点 GET `https://www.youtube.com/oembed?url={{ encodeURIComponent($json.url) }}&format=json`，返回的 `thumbnail_url` 即为真缩略图。用 If 节点 `{{ $json.media_type === 'video' }}` 分流，video 支走 oEmbed 后合并。生产还需 try 兜底（文章页 URL 非 YouTube 时 oEmbed 404）。
</details>