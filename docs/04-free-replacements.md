# 04 免费替身与本地验证（0 元跑通全链）

本课链路天生便宜：两个数据源全免费，唯一"硬件"依赖是 Home Assistant——没有 HA 也能验证。

## 替身对照表

| 原需求 | 免费方案 | 说明 |
|---|---|---|
| NASA API key | **`DEMO_KEY`** | api.nasa.gov 公共演示 key，30 req/h/IP；本工作流每次运行只调 1 次 NeoWS，远低于限额。生产建议免费注册个人 key（邮箱即得） |
| APOD feed | **无需替身** | science.nasa.gov 新端点免 key 直连 |
| Home Assistant 实例 | **本地 HTTP 接收端** | 30 行 Python（`http.server`）起在 127.0.0.1，把 2 个 homeAssistant 节点换成 httpRequest POST 到本地端点，payload 落盘校验 |

## 本地接收端参考实现（L2-C 实测同款）

```python
# c-recv.py —— 监听 127.0.0.1:18349，按路径落盘 JSON
from http.server import HTTPServer, BaseHTTPRequestHandler
import json, os

class H(BaseHTTPRequestHandler):
    def do_POST(self):
        body = self.rfile.read(int(self.headers.get('Content-Length', 0)))
        name = 'ha-apod.json' if 'apod' in self.path else 'ha-asteroid.json'
        with open(name, 'wb') as f:
            f.write(body)
        self.send_response(200); self.end_headers()

HTTPServer(('127.0.0.1', 18349), H).serve_forever()
```

替换要点：homeAssistant 节点 → httpRequest（POST，JSON body = 原 upsert 的 attributes/state 结构）。真 HA 部署时把节点换回来，填 base URL + 长期访问令牌（HA 档案页 → Long-Lived Access Tokens）。

## 本地验证三步

1. 导入 `workflow.json`（或修复版 `workflow-custom.json`）
2. NeoWS 节点的 Query Auth 凭证先用 `DEMO_KEY`
3. HA 节点按上文换本地接收端 → 手动触发 → 检查落盘 JSON 字段齐全