---
source_file: SKILL.md
source_commit: 41bbe19d1a1a7eaab5e7bb9050a417e5c6cffc8f
source_sha256: 2efca615ce55a3edd8fc05c779068a8085816617991987e446606403cd3abb22
translated_at: 2026-08-21
---

<!-- translation-header -->
> **譯文說明**｜本檔是 [`SKILL.md`](SKILL.md) 的台灣正體中文譯文，供閱讀理解之用；原檔一字未改，實際使用請以原文為準。
>
> **原檔 YAML frontmatter**
> - `name`: `slack-gif-creator`
> - `description`（中譯）：建立專為 Slack 最佳化的動態 GIF 的知識與工具。提供限制條件、驗證工具與動畫概念。當使用者要求製作 Slack 用的動態 GIF（如「幫我做一個 X 做 Y 的 GIF 用於 Slack」）時，請使用本 skill。
> - `license`: Complete terms in LICENSE.txt
<!-- /translation-header -->

# Slack GIF Creator

一個提供工具與知識的工具組，用於建立專為 Slack 最佳化的動態 GIF。

## Slack 需求

**尺寸：**
- 表情符號 GIF：128x128（建議）
- 訊息 GIF：480x480

**參數：**
- FPS：10-30（越低，檔案越小）
- 色彩數：48-128（越少，檔案越小）
- 時長：表情符號 GIF 請控制在 3 秒以內

## 核心工作流程

```python
from core.gif_builder import GIFBuilder
from PIL import Image, ImageDraw

# 1. Create builder
builder = GIFBuilder(width=128, height=128, fps=10)

# 2. Generate frames
for i in range(12):
    frame = Image.new('RGB', (128, 128), (240, 248, 255))
    draw = ImageDraw.Draw(frame)

    # Draw your animation using PIL primitives
    # (circles, polygons, lines, etc.)

    builder.add_frame(frame)

# 3. Save with optimization
builder.save('output.gif', num_colors=48, optimize_for_emoji=True)
```
## 繪製圖形

### 處理使用者上傳的圖片
如果使用者上傳了圖片，考慮他們是否想要：
- **直接使用**（例如：「讓這個動起來」、「把這個分割成影格」）
- **作為靈感參考**（例如：「做一個類似這個的東西」）

使用 PIL 載入並處理圖片：
```python
from PIL import Image

uploaded = Image.open('file.png')
# Use directly, or just as reference for colors/style
```
### 從頭開始繪製
從頭開始繪製圖形時，使用 PIL ImageDraw 基本圖形：

```python
from PIL import ImageDraw

draw = ImageDraw.Draw(frame)

# Circles/ovals
draw.ellipse([x1, y1, x2, y2], fill=(r, g, b), outline=(r, g, b), width=3)

# Stars, triangles, any polygon
points = [(x1, y1), (x2, y2), (x3, y3), ...]
draw.polygon(points, fill=(r, g, b), outline=(r, g, b), width=3)

# Lines
draw.line([(x1, y1), (x2, y2)], fill=(r, g, b), width=5)

# Rectangles
draw.rectangle([x1, y1, x2, y2], fill=(r, g, b), outline=(r, g, b), width=3)
```
**不要使用：** 表情符號字體（跨平台不可靠）或假設本 skill 中存在預先打包的圖形。

### 讓圖形看起來精緻

圖形應看起來精緻且有創意，而非基本。以下是方法：

**使用較粗的線條**——輪廓線與線條始終設定 `width=2` 或更高。細線（width=1）看起來粗糙而業餘。

**增加視覺深度**：
- 為背景使用漸層（`create_gradient_background`）
- 疊加多個形狀以增加複雜性（例如，在星形內放一個較小的星形）

**讓形狀更有趣**：
- 不要只畫一個普通的圓形——加入高光、環形或圖案
- 星形可以有光暈效果（在後方繪製較大的半透明版本）
- 結合多個形狀（星形 + 閃光、圓形 + 環形）

**注重色彩**：
- 使用鮮豔、互補的色彩
- 增加對比度（淺色形狀用深色輪廓，深色形狀用淺色輪廓）
- 考慮整體構圖

**對於複雜形狀**（愛心、雪花等）：
- 使用多邊形與橢圓形的組合
- 仔細計算點位以確保對稱性
- 加入細節（愛心可以有高光曲線，雪花有複雜的分支）

要有創意並注重細節！一個好的 Slack GIF 應該看起來精緻，而非像佔位符號圖形。

## 可用工具

### GIFBuilder（`core.gif_builder`）
組合影格並為 Slack 最佳化：
```python
builder = GIFBuilder(width=128, height=128, fps=10)
builder.add_frame(frame)  # Add PIL Image
builder.add_frames(frames)  # Add list of frames
builder.save('out.gif', num_colors=48, optimize_for_emoji=True, remove_duplicates=True)
```
### 驗證器（`core.validators`）
檢查 GIF 是否符合 Slack 需求：
```python
from core.validators import validate_gif, is_slack_ready

# Detailed validation
passes, info = validate_gif('my.gif', is_emoji=True, verbose=True)

# Quick check
if is_slack_ready('my.gif'):
    print("Ready!")
```
### 緩動函式（`core.easing`）
使用平滑動態取代線性運動：
```python
from core.easing import interpolate

# Progress from 0.0 to 1.0
t = i / (num_frames - 1)

# Apply easing
y = interpolate(start=0, end=400, t=t, easing='ease_out')

# Available: linear, ease_in, ease_out, ease_in_out,
#           bounce_out, elastic_out, back_out
```
### 影格輔助工具（`core.frame_composer`）
常見需求的便利函式：
```python
from core.frame_composer import (
    create_blank_frame,         # Solid color background
    create_gradient_background,  # Vertical gradient
    draw_circle,                # Helper for circles
    draw_text,                  # Simple text rendering
    draw_star                   # 5-pointed star
)
```
## 動畫概念

### 震動／抖動
以振盪方式偏移物件位置：
- 使用 `math.sin()` 或 `math.cos()` 搭配影格索引
- 加入小幅隨機變化以增加自然感
- 套用於 x 和／或 y 位置

### 脈動／心跳
有節奏地縮放物件大小：
- 使用 `math.sin(t * frequency * 2 * math.pi)` 實現平滑脈動
- 心跳效果：兩次快速脈動後暫停（調整正弦波）
- 在基本大小的 0.8 到 1.2 之間縮放

### 彈跳
物件掉落並彈起：
- 落地時使用 `interpolate()` 搭配 `easing='bounce_out'`
- 下落時使用 `easing='ease_in'`（加速）
- 每影格增加 y 速度以模擬重力

### 旋轉
圍繞中心旋轉物件：
- PIL：`image.rotate(angle, resample=Image.BICUBIC)`
- 搖擺效果：使用正弦波控制角度，而非線性

### 淡入／淡出
逐漸出現或消失：
- 建立 RGBA 圖片，調整 alpha 通道
- 或使用 `Image.blend(image1, image2, alpha)`
- 淡入：alpha 從 0 到 1
- 淡出：alpha 從 1 到 0

### 滑動
從畫面外移動到目標位置：
- 起始位置：影格邊界之外
- 終止位置：目標位置
- 使用 `interpolate()` 搭配 `easing='ease_out'` 實現平滑停止
- 超衝效果：使用 `easing='back_out'`

### 縮放
縮放並定位以產生縮放效果：
- 放大：從 0.1 縮放到 2.0，裁切中心
- 縮小：從 2.0 縮放到 1.0
- 可加入動態模糊以增強戲劇感（PIL 濾鏡）

### 爆炸／粒子噴射
建立向外放射的粒子：
- 以隨機角度與速度生成粒子
- 每影格更新粒子：`x += vx`、`y += vy`
- 加入重力：`vy += gravity_constant`
- 隨時間淡出粒子（降低 alpha）

## 最佳化策略

只有在有請求縮小檔案大小時，才實作以下幾種方法：

1. **較少影格**——降低 FPS（用 10 取代 20）或縮短時長
2. **較少色彩**——`num_colors=48` 取代 128
3. **較小尺寸**——128x128 取代 480x480
4. **移除重複影格**——在 save() 中使用 `remove_duplicates=True`
5. **表情符號模式**——`optimize_for_emoji=True` 自動最佳化

```python
# Maximum optimization for emoji
builder.save(
    'emoji.gif',
    num_colors=48,
    optimize_for_emoji=True,
    remove_duplicates=True
)
```
## 設計理念

本 skill 提供：
- **知識**：Slack 的需求與動畫概念
- **工具**：GIFBuilder、驗證器、緩動函式
- **彈性**：使用 PIL 基本圖形建立動畫邏輯

它**不**提供：
- 僵化的動畫樣板或預製函式
- 表情符號字體渲染（跨平台不可靠）
- 內建於 skill 的預先打包圖形庫

**關於使用者上傳的注意事項**：本 skill 不包含預製圖形，但如果使用者上傳了圖片，請使用 PIL 載入並處理——根據他們的請求判斷是直接使用還是只作為靈感參考。

發揮創意！結合各種概念（彈跳 + 旋轉、脈動 + 滑動等），充分發揮 PIL 的所有功能。

## 相依套件

```bash
pip install pillow imageio numpy
```
