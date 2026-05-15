---
name: feishu-document-api
description: 通过飞书开放平台API创建和编辑飞书文档（docx），包含权限申请、API调用流程和常见问题排查
---

# 飞书文档API使用指南

## 前置条件
- 飞书开放平台应用 (App ID + App Secret)
- 应用已发布/已开通权限

## 所需权限列表

| 权限 | 说明 | 用途 |
|------|------|------|
| `docx:document:create` | 创建文档 | 创建新的飞书文档 |
| `docx:document:write` | 编辑文档内容 | 写入/更新文档内容 |
| `drive:drive` | 云盘全量访问 | 获取文档列表、管理文件 |

权限申请地址：`https://open.feishu.cn/app/{APP_ID}/auth`

## API调用流程

### 1. 获取 tenant_access_token

```python
import json
import urllib.request

def get_token(app_id, app_secret):
    url = "https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal"
    payload = json.dumps({"app_id": app_id, "app_secret": app_secret}).encode()
    headers = {"Content-Type": "application/json"}
    req = urllib.request.Request(url, data=payload, headers=headers, method="POST")
    with urllib.request.urlopen(req, timeout=10) as resp:
        result = json.loads(resp.read().decode())
        return result["tenant_access_token"]
```

### 2. 创建文档

```python
def create_document(token, title, folder_token=""):
    url = "https://open.feishu.cn/open-apis/docx/v1/documents"
    payload = json.dumps({
        "title": title,
        "folder_token": folder_token
    }).encode()
    headers = {
        "Authorization": f"Bearer {token}",
        "Content-Type": "application/json"
    }
    req = urllib.request.Request(url, data=payload, headers=headers, method="POST")
    with urllib.request.urlopen(req, timeout=10) as resp:
        result = json.loads(resp.read().decode())
        return result["data"]["document"]["document_id"]
```

### 3. 获取文档信息（确认文档存在）

```python
def get_document_info(token, document_id):
    url = f"https://open.feishu.cn/open-apis/docx/v1/documents/{document_id}"
    headers = {"Authorization": f"Bearer {token}"}
    req = urllib.request.Request(url, headers=headers, method="GET")
    with urllib.request.urlopen(req, timeout=10) as resp:
        return json.loads(resp.read().decode())
```

### 4. 写入文档内容（需要权限）

```python
def create_blocks(token, document_id, parent_block_id, blocks, index=-1):
    """
    创建内容块
    parent_block_id: 通常是 document_id 本身（page block）
    """
    url = f"https://open.feishu.cn/open-apis/docx/v1/documents/{document_id}/blocks/{parent_block_id}/children"
    payload = json.dumps({"children": blocks, "index": index}).encode()
    headers = {
        "Authorization": f"Bearer {token}",
        "Content-Type": "application/json"
    }
    req = urllib.request.Request(url, data=payload, headers=headers, method="POST")
    with urllib.request.urlopen(req, timeout=20) as resp:
        return json.loads(resp.read().decode())
```

## Block 类型说明

| block_type | 类型 | 说明 |
|------------|------|------|
| 1 | page | 页面（文档根block） |
| 2 | paragraph | 段落 |
| 3 | heading1 | 一级标题 |
| 4 | heading2 | 二级标题 |
| 5 | heading3 | 三级标题 |
| 14 | code_block | 代码块 |
| 17 | divider | 分割线 |

## 内容块示例

```python
# 二级标题
{
    "block_type": 4,
    "heading2": {
        "elements": [
            {"text_run": {"content": "📋 学员信息卡"}}
        ]
    }
}

# 段落
{
    "block_type": 2,
    "paragraph": {
        "elements": [
            {"text_run": {"content": "👤 学员姓名：_______________"}}
        ]
    }
}

# 分割线
{"block_type": 17, "divider": {}}

# 代码块
{
    "block_type": 14,
    "code_block": {
        "language": "",
        "elements": [{"text_run": {"content": "代码内容"}}]
    }
}
```

## 常见错误码

| 错误码 | 说明 | 解决方案 |
|--------|------|----------|
| 10014 | app secret invalid | 检查 App Secret 是否正确 |
| 1770001 | invalid param | 参数格式错误，检查请求参数 |
| 99991672 | No permission | 缺少对应权限，去开放平台申请 |

## 完整示例

```python
APP_ID = "cli_xxxxxxxxxx"
APP_SECRET = "xxxxxxxxxx"

token = get_token(APP_ID, APP_SECRET)
doc_id = create_document(token, "我的文档标题")

blocks = [
    {"block_type": 17, "divider": {}},
    {"block_type": 4, "heading2": {"elements": [{"text_run": {"content": "欢迎使用"}}]}},
    {"block_type": 2, "paragraph": {"elements": [{"text_run": {"content": "这是通过API创建的文档"}}]}},
]

result = create_blocks(token, doc_id, doc_id, blocks)
print(f"文档地址: https://feishu.cn/docx/{doc_id}")
```

## 注意事项

1. 创建文档不需要特殊权限，但写入内容需要 `docx:document:write` 权限
2. 权限申请后需要重新获取 token 才会生效
3. 文档根 block 的 id 就是 document_id 本身
4. 每次 API 调用建议加超时处理，大文档写入建议分批次
