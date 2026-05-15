---
name: jsdelivr-github-file-hosting
description: 使用 jsDelivr CDN + GitHub 实现零配置免费文件托管与直链分享，国内访问速度快
triggers:
  - 文件托管
  - 图床
  - 直链分享
  - jsDelivr
  - Gitee Pages 下线
  - CDN 加速
---

# jsDelivr + GitHub 零配置文件托管方案

**Gitee Pages 已于 2024 年 8 月下线。** jsDelivr + GitHub 是最佳替代方案，国内访问速度快，零配置。

---

## 核心优势

| 特性 | jsDelivr + GitHub | Gitee Pages | GitHub Pages |
|------|------------------|-------------|--------------|
| 国内访问 | ⭐⭐⭐ 有国内CDN节点 | ❌ 已下线 | ⭐ 一般 |
| 配置 | ✅ 0 配置 | - | 需要启用Pages |
| 缓存 | ✅ 全球CDN缓存 | - | 无缓存 |
| 防盗链 | ❌ 无（可直接引用） | - | 无 |
| 单文件大小 | < 50MB | - | 无限制 |

---

## jsDelivr 工作原理

```
用户请求 → jsDelivr CDN 节点 → GitHub 源站 → 缓存到节点 → 返回给用户
                         ↓
                    下次请求直接从缓存返回
```

1. **第 1 次请求**：CDN 节点从 GitHub 拉取文件并缓存
2. **第 N 次请求**：直接从就近 CDN 节点返回，速度极快

jsDelivr 在中国大陆有合法备案的节点（与阿里云/腾讯云合作），国内访问速度远超 GitHub raw。

---

## 直链格式

```
https://cdn.jsdelivr.net/gh/用户名/仓库名/文件路径
```

### 常用变体

| 格式 | 示例 |
|------|------|
| 基础格式 | `https://cdn.jsdelivr.net/gh/toolazytoname/filehost/test.md` |
| 指定版本 | `https://cdn.jsdelivr.net/gh/toolazytoname/filehost@v1.0.0/test.md` |
| 刷新缓存 | `https://cdn.jsdelivr.net/gh/toolazytoname/filehost/test.md?v=20260501` |

### 其他支持的源站

| 源站 | URL 格式 |
|------|----------|
| npm | `https://cdn.jsdelivr.net/npm/包名/文件路径` |
| WordPress | `https://cdn.jsdelivr.net/wp/plugins/插件名/...` |

---

## 使用场景

1. **博客图床**：把图片上传到 GitHub 公开仓库，用 jsDelivr 链接引用
2. **静态资源托管**：JS、CSS、字体文件
3. **开源项目 Demo**：托管项目的示例页面
4. **文件分享**：小文件的直链分享
5. **Markdown 图片**：文档中的图片引用

---

## Gitee vs GitHub API 关键差异

在使用 REST API 上传文件时，注意以下差异：

| 操作 | GitHub | Gitee |
|------|--------|-------|
| 创建文件 | `PUT` 方法 | `POST` 方法 |
| 更新文件 | `PUT` + `sha` | `PUT` + `sha` |
| 新文件 `sha` | 不需要 | `PUT` 需要，但 `POST` 不需要 |
| 自动建目录 | ✅ 支持 | ✅ 支持（嵌套路径自动创建） |

### Gitee API 上传示例（Python）

```python
import urllib.request
import json
import base64

token = "your_gitee_token"
headers = {
    "Authorization": f"token {token}",
    "Content-Type": "application/json"
}

# 创建新文件 - 使用 POST
upload_data = json.dumps({
    "message": "Add file",
    "content": base64.b64encode(content.encode('utf-8')).decode(),
    "branch": "master"
}).encode('utf-8')

req = urllib.request.Request(
    "https://gitee.com/api/v5/repos/lazywc/repo-name/contents/path/to/file.md",
    data=upload_data,
    headers=headers,
    method="POST"  # 关键：Gitee 创建文件用 POST
)

# 更新文件 - 使用 PUT + sha
upload_data = json.dumps({
    "message": "Update file",
    "content": base64.b64encode(new_content.encode('utf-8')).decode(),
    "sha": existing_file_sha,  # 必须
    "branch": "master"
}).encode('utf-8')

req = urllib.request.Request(
    "...",
    data=upload_data,
    headers=headers,
    method="PUT"  # 关键：Gitee 更新文件用 PUT
)
```

---

## 缓存策略

| 内容类型 | 缓存时间 |
|----------|----------|
| JS/CSS 等静态资源 | 7 天 |
| 带版本号的文件 | 1 年 |
| 图片/文档 | 7 天 |

**刷新缓存技巧**：如果更新了文件，在 URL 后面加 `?v=时间戳` 强制刷新。

---

## 注意事项

✅ **只能用公开仓库**（CDN 节点没有 GitHub 权限）

❌ **不支持大文件**（单文件建议 < 50MB）

✅ **完全免费**（由 Cloudflare、Fastly 等大厂赞助）

---

## 推荐仓库结构

```
filehost/ (公开仓库，用于 jsDelivr)
├── images/          # 图片素材
├── documents/       # 文档分享
├── assets/          # 静态资源
└── ...

private-files/ (私有仓库，用于存储自媒体素材、笔记等)
├── media-materials/ # 自媒体素材
├── drafts/          # 草稿
└── ...
```
