# 纯视觉 RPA 工具

基于 **屏幕截图 + 通义千问视觉模型 (Qwen-VL)** 的纯视觉 RPA 引擎。

不依赖任何 DOM / Accessibility API，仅通过"看屏幕"来理解和操作。

## 工作原理

```
截图 → 缩略图粗定位 → 全分辨率裁剪精定位 → 执行操作 → 截图验证 → 循环
```

核心流程:
1. 全屏截图 → 缩小到 1280px → 千问粗定位获得大致坐标
2. 从全分辨率截图裁剪目标区域 (400×400) → 千问精确定位
3. 坐标映射到屏幕 → 执行鼠标/键盘操作 → 截图验证

## 快速开始

### 1. 安装依赖

```bash
pip install -r requirements.txt
```

> **Linux 额外依赖**: `sudo apt install xclip scrot`
> **macOS**: 需要在「系统设置 → 隐私与安全 → 辅助功能」中授权终端/Python

### 2. 设置 API Key

```bash
# DashScope API Key
export DASHSCOPE_API_KEY="sk-your-api-key"
```

### 3. 运行

#### 交互模式（推荐入门）

```bash
python visual_rpa.py
```

然后在终端中逐条输入操作指令：

```
[0] > 点击桌面上的 Chrome 浏览器图标
OK | Chrome 浏览器已打开

[1] > 在地址栏中输入 https://www.baidu.com 并按回车
OK | 已导航到百度首页
```

#### 批量任务模式

```bash
python visual_rpa.py --mode task \
  --task "点击Chrome浏览器" \
        "在地址栏输入 https://www.baidu.com 并回车" \
        "在搜索框输入天气预报" \
        "点击百度一下按钮"
```

支持复合指令自动分解：

```bash
python visual_rpa.py --mode task \
  --task "打开微信，并打开文件传输助手聊天，在输入框输入你好，并点击发送"
```

会自动分解为多步执行。

#### 跳过验证（省 API 费用）

```bash
python visual_rpa.py --no-verify
```

### 4. 代码中调用

```python
from visual_rpa import VisualRPA

rpa = VisualRPA(
    model="qwen-vl-max-latest",
    api_key="your-api-key",    # 或设置 DASHSCOPE_API_KEY 环境变量
    verify_actions=True,
    confidence_threshold=0.5,
    post_action_wait=1.0,
)

results = rpa.run_task([
    "打开微信",
    "点击文件传输助手",
    "在输入框输入你好",
    "点击发送",
])

for r in results:
    print(f"{'OK' if r.success else 'FAIL'} {r.instruction}")
```

## OpenClaw Skill 集成

本项目提供了 [OpenClaw](https://github.com/nicepkg/openclaw) 技能插件，可以让 AI 助手直接操控桌面。

### 安装

将 `skills/visual-rpa/` 目录复制到 OpenClaw 的技能目录：

```bash
# 复制到 OpenClaw 内置技能目录
cp -r skills/visual-rpa /path/to/openclaw/skills/

# 或复制到用户技能目录
cp -r skills/visual-rpa ~/.openclaw/skills/
```

### 配置

在 OpenClaw 配置中设置 API Key：

```json
{
  "skills": {
    "entries": {
      "visual-rpa": {
        "enabled": true
      }
    }
  }
}
```

确保 `DASHSCOPE_API_KEY` 环境变量已设置。

### 使用

安装后，直接对 AI 助手说自然语言指令即可：

- "帮我打开微信，给文件传输助手发一条你好"
- "打开 Chrome 浏览器，搜索今天天气"
- "点击桌面上的计算器"

AI 助手会自动识别为桌面操作意图，调用 visual-rpa 技能完成任务。

### Skill 文件结构

```
skills/visual-rpa/
├── SKILL.md              # 技能定义（frontmatter 元数据 + LLM 指令）
└── scripts/
    └── visual_rpa.py     # RPA 引擎脚本
```

| 元数据 | 值 |
|--------|-----|
| 触发条件 | 用户提到操作桌面、点击、打开应用、输入文字等 |
| 平台 | Windows (`win32`) |
| 依赖 | Python + `DASHSCOPE_API_KEY` 环境变量 |

## 特性

- **两轮定位**: 缩略图粗定位 + 全分辨率裁剪精定位，提高坐标精度
- **复合指令分解**: 自动将多步指令拆分为原子操作逐步执行
- **操作验证**: 每步操作后截图对比验证是否成功
- **自动重试**: 定位失败或验证失败时自动重试
- **中文输入**: 通过 Windows API 剪贴板实现可靠的中文输入
- **JSON 容错**: 自动修复模型返回的畸形 JSON
- **OpenClaw 集成**: 可作为 AI 助手技能插件，自然语言驱动桌面自动化

## 参数说明

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `model` | qwen-vl-max-latest | 千问视觉模型 |
| `max_retries` | 3 | 每步最大重试次数 |
| `confidence_threshold` | 0.5 | 置信度低于此值时重试 |
| `verify_actions` | True | 是否在每步操作后验证 |
| `post_action_wait` | 1.0 | 操作后等待时间(秒) |
| `thumbnail_width` | 1280 | 缩略图最大宽度 |
| `crop_half_size` | 200 | 精定位裁剪半径(像素) |

## 日志与调试

所有截图和日志保存在 `./rpa_logs/` 目录：

```
rpa_logs/
├── rpa.log                    # 运行日志
├── ss_step0_thumb_*.png       # 缩略图
├── ss_step0_crop_*.png        # 裁剪区域
├── ss_step0_after_*.png       # 操作后截图
└── ...
```

## 已知限制

1. **延迟**: 每步操作需 3-8 秒（截图 + 两轮 API 调用 + 验证）
2. **坐标精度**: 模型返回的坐标偶有偏差，对很小的按钮可能需要重试
3. **中文输入**: 通过剪贴板粘贴实现，会覆盖用户剪贴板内容
4. **动态页面**: 如果页面有动画/加载，需要适当增大 `post_action_wait`
5. **多屏幕**: 当前仅支持主屏幕
