---
name: github-rest-push-fallback
description: 当 git push 超时时，通过 GitHub REST API (Contents API) + base64 直接提交文件到仓库
triggers:
  - git push 超时
  - git push timeout
  - 无法 git push 时
  - GitHub API push fallback
---

# GitHub REST API Push Fallback

当 `git push` 持续超时时，用 GitHub Contents API 绕过网络问题直接提交文件。

## 前置条件

- GitHub Personal Access Token（需 `repo` 权限）
- 目标文件的当前 SHA（通过 GET 请求获取）

## 完整流程（Python）

```python
import urllib.request, json, base64

TOKEN = "ghp_xxxx"
OWNER = "toolazytoname"
REPO  = "metronome"
FILE  = "index.html"
BRANCH = "main"

headers = {
    "Authorization": f"token {TOKEN}",
    "Accept": "application/vnd.github.v3+json",
    "Content-Type": "application/json",
    "User-Agent": "Hermes-Agent/1.0"
}

# 1. 获取当前 SHA
url = f"https://api.github.com/repos/{OWNER}/{REPO}/contents/{FILE}"
req = urllib.request.Request(url, headers=headers)
with urllib.request.urlopen(req, timeout=15) as r:
    sha = json.loads(r.read())["sha"]

# 2. 读取本地文件并提交
with open(FILE, "r") as f:
    content = f.read()

payload = json.dumps({
    "message": "commit message",
    "content": base64.b64encode(content.encode()).decode(),
    "sha": sha,
    "branch": BRANCH
})

req = urllib.request.Request(url, data=payload.encode(), headers=headers, method="PUT")
with urllib.request.urlopen(req, timeout=30) as r:
    result = json.loads(r.read())
    print("SUCCESS:", result["commit"]["sha"])
```

## 关键点

- `curl` 和 `gh` CLI 在此环境不可用，必须用 Python `urllib`
- `curl: command not found` → 换 Python
- `git push timeout` → 换 REST API
- 更新文件需要当前 SHA，删除文件也需要 SHA
- 如果文件不在仓库根目录，路径写完整：`src/utils/helper.py`

## 附加：base64 嵌入 HTML 图片

用于将本地图片（如二维码）内联到 HTML，避免额外资源请求。

```python
import base64

with open('wechat_qr.jpg', 'rb') as f:
    b64 = base64.b64encode(f.read()).decode()

with open('index.html', 'r') as f:
    html = f.read()

# 替换占位符（不要在 HTML 里写 shell 命令，浏览器不执行）
html = html.replace('{{WECHAT_QR_PLACEHOLDER}}', f'data:image/jpeg;base64,{b64}')

with open('index.html', 'w') as f:
    f.write(html)
```

## 附加：Feishu 图片处理

用户发送图片到飞书后，存储在 `/opt/data/cache/images/img_*.jpg`（哈希文件名）。
vision_analyze 不支持直接文件路径分析，需用 Playwright 打开：
```javascript
await page.goto('file:///opt/data/cache/images/img_xxx.jpg');
```

读取 JPEG 尺寸可用 Playwright 的 `page.evaluate()` 获取 `img.naturalWidth/naturalHeight`。

## 适用场景

- 网络环境限制 SSH/HTTPS git 流量但允许 HTTP API
- 临时 token 无法配置 git credential helper
- git push 反复超时但 API 请求正常
