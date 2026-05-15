---
name: qr-code-embedding
description: QR码嵌入静态页面的最佳实践——尺寸、裁切、CSS适配，以及GitHub REST API推送流程
---

# QR码嵌入静态页面最佳实践

## 核心经验

**对于已经裁成正方形的QR码图片，`object-fit: cover` 优于 `object-fit: contain`。**

- `object-fit: contain`：保持比例完整显示 → 周围留白 → 实际可见区域远小于容器
- `object-fit: cover`：填满容器裁切多余 → 充分利用每一像素 → **扫码体验最佳**

QR码本身就是方正的，无需保持比例展示。

## 推荐参数

```css
.qr-img {
    width: 150px;        /* 不要小于120px，扫码困难 */
    height: 150px;       /* 宽高相等 */
    border-radius: 16px;
    box-shadow: 0 4px 16px rgba(0,0,0,0.15);
    object-fit: cover;   /* 用 cover，不用 contain */
    object-position: center;
    background: #fff;    /* 白底，防止透明图露出背景 */
}
```

## 图片预处理

裁成正方形（以宽为基准裁高，或以高为基准裁宽）：
```bash
ffmpeg -i input.jpg -vf "crop=min(min(iw,ih)\,400):min(min(iw,ih)\,400)" -q 5 output_sq.jpg
```

## 面板内容溢出处理

若面板内还有其他控件，QR区放底部时：
```css
.panel-inner {
    max-height: 700px;       /* 足够容纳所有内容 */
    overflow-y: auto;        /* 内容多时滚动，不影响QR显示 */
    scrollbar-width: thin;
}
```

## 推送流程（GitHub REST API）

详见 skill: `github-rest-push-fallback`

```python
# 1. GET 获取当前 SHA
# 2. base64 编码文件内容
# 3. PUT 提交，包含 sha 和 commit message
```

## 替换 HTML 中的 base64 QR 码

**HTML 中的 `<img>` 标签没有 `</img>` 闭合标签**（自闭合或直接断行），不能用 `find('</img>')` 定位。

正确做法：用正则匹配整个 img 标签（含 src 属性），再整段替换：

```python
import re, base64

with open('new_qr.jpg', 'rb') as f:
    new_b64 = base64.b64encode(f.read()).decode()

with open('index.html', 'r', encoding='utf-8') as f:
    html = f.read()

pattern = r'<img class="qr-img" src="data:image/jpeg;base64,[^"]+"'
matches = list(re.finditer(pattern, html))

for m in matches:
    # 往后搜200字符找关联标签判断是微信还是支付宝
    segment = html[m.end():m.end()+200]
    if '支付宝' in segment:
        new_tag = f'<img class="qr-img" src="data:image/jpeg;base64,{new_b64}"'
        html = html[:m.start()] + new_tag + html[m.end():]
```

## 支付宝截图裁切经验值

- 原图：1080×1618px
- 截取 `y=900` 开始，`h=718px` → 含"推荐使用支付宝"文字引导区 + 完整二维码
- 缩放至 300×300px（正方形 crop + scale）

> ⚠️ **vision_analyze 在此环境无法分析用户上传到对话的图片**（始终返回"看不到"）。若需确认裁切位置：生成多个 y 坐标候选版本让用户选择，或直接使用经验值尝试。

## 调试技巧

- 先本地预览（`python3 -m http.server 9999`）再推送
- QR码尺寸**至少120px以上**，实际扫码推荐150-200px
