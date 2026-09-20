# 成都好室友（150分）

> 考点：`.pyc` 反编译、Base64 图片提取、二维码结构、定位图案修复

## 题面

> 从成都省四川市重庆区来的好室友 PYC，想让你 cos0 = 1

拿到一个 `.pyc` 文件，需要从中提取信息并获取 Flag。

## 初步分析

### 1. 反编译 `.pyc` 文件

`.pyc` 是 Python 字节码文件，无法直接阅读。使用反编译工具将其还原为 Python 源码：

```
# 方案一：uncompyle6（经典）
pip install uncompyle6
uncompyle6 challenge.pyc > challenge.py

# 方案二：pycdc（支持更新版本 Python）
./pycdc challenge.pyc > challenge.py

# 方案三：在线工具
# https://tool.lu/pyc/
# https://decompiler.com/
```

反编译后发现：

- `IMAGE_BASE64` 变量存储了一张 PNG 图片的 Base64 编码。

- 关键函数 `decode_image()` 反编译结果为 `pass`，函数体未完全还原。

```
# WARNING: Decompyle incomplete

def decode_image(encoded_data=None, output_file=None):
    pass
```

**结论**：出题人隐藏了图片处理逻辑，需要手动解码并处理图片。

### 2. 提取并解码图片

将 `IMAGE_BASE64` 解码为 PNG 文件：

可以用CyberChef工具下载或py脚本（下附脚本）

```py
import base64

IMAGE_BASE64 = "iVBORw0KGgoAAAANSUhEUgAA...（完整字符串）"

with open("flag.png", "wb") as f:
    f.write(base64.b64decode(IMAGE_BASE64))
```

得到的图片是一张二维码，但三个定位角（左上、右上、左下）为白色（缺失），无法正常扫码。

## 核心解法：修复二维码定位图案

### 1. 二维码定位图案原理

国际标准（ISO/IEC 18004）规定，二维码的三个角必须包含**定位图案（寻像图形）**，其比例为固定的 **1:1:3:1:1**（黑:白:黑:白:黑），标准大小为 **7×7** 模块。

出题人只能将定位角涂白（表面破坏），但**无法改变二维码的数据区域和模块尺寸标准**。只要数据区域完整，即可通过测量模块宽度，按标准比例重绘定位图案。、

### 2. 修复步骤

1. **测量模块大小**：从图片中心完整的数据区域，扫描黑白跳变，计算单个模块的像素宽度。
2. **生成标准回字形**：按 1:1:3:1:1 比例绘制 7×7 定位图案。
3. **覆盖到三个角**：将生成的图案粘贴到图片的左上、右上、左下位置。

### 3. 修复脚本

```py
from PIL import Image, ImageDraw

def estimate_module_size(img):
    """通过扫描图像中心区域的黑白跳变估算单个模块的像素宽度"""
    gray = img.convert('L')
    w, h = img.size
    y = h // 2
    x_start = w // 4
    x_end = w * 3 // 4

    pixels = []
    for x in range(x_start, x_end):
        val = 0 if gray.getpixel((x, y)) < 128 else 1
        pixels.append(val)

    last = pixels[0]
    dists = []
    dist = 0
    for p in pixels:
        if p != last:
            if dist > 0:
                dists.append(dist)
            dist = 1
            last = p
        else:
            dist += 1
    if dist > 0:
        dists.append(dist)

    if not dists:
        return 21
    dists.sort()
    return dists[len(dists) // 2]

def draw_finder_pattern(draw, x, y, size):
    """绘制标准 7x7 回字形定位图案（1:1:3:1:1 比例）"""
    pattern = [
        [1,1,1,1,1,1,1],
        [1,0,0,0,0,0,1],
        [1,0,1,1,1,0,1],
        [1,0,1,1,1,0,1],
        [1,0,1,1,1,0,1],
        [1,0,0,0,0,0,1],
        [1,1,1,1,1,1,1]
    ]
    for row in range(7):
        for col in range(7):
            px = x + col * size
            py = y + row * size
            color = (0, 0, 0) if pattern[row][col] == 1 else (255, 255, 255)
            draw.rectangle([px, py, px + size - 1, py + size - 1], fill=color)

# 打开图片
img = Image.open("flag.png").convert("RGB")
w, h = img.size

# 估算模块大小
module_size = estimate_module_size(img)
print(f"模块大小: {module_size} px")

# 如果有边框，手动设置偏移量（本例无需，图片无多余边框）
border = 0

# 在三个角绘制定位图案
draw = ImageDraw.Draw(img)
draw_finder_pattern(draw, border, border, module_size)                              # 左上
draw_finder_pattern(draw, w - 7*module_size - border, border, module_size)          # 右上
draw_finder_pattern(draw, border, h - 7*module_size - border, module_size)          # 左下

img.save("flag_fixed.png")
print("修复完成！")
```

## FLAG

moectf{PYC_p1us_QRc0dE_M1ssing_c0rN3r}