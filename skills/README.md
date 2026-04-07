# BBEnc LLM Skills — 跨平台视频批量编码对话技能组

本技能组将原 PowerShell 交互式脚本转化为**跨平台**的 LLM 对话技能。
所有流程以**伪代码**描述，LLM 在执行时根据用户平台（Windows/macOS/Linux）动态生成合适的代码并运行。

## 工作流概览

```
┌──────────────────────────┐
│  平台检测                  │  首次对话自动执行
│  (common/platform.md)     │  → 设定 Shell/脚本格式/路径规范
└────────────┬─────────────┘
             ▼
┌──────────────────────────┐
│  Skill 1: 环境检测         │  检查 Shell 环境、权限、硬件信息
└────────────┬─────────────┘
             ▼
┌──────────────────────────┐
│  Skill 2: 编码管线         │  选择工具 → 生成 .bat 或 .sh
└────────────┬─────────────┘
             ▼
┌──────────────────────────┐
│  Skill 3: 源分析           │  ffprobe 分析视频 → 导出 CSV
└────────────┬─────────────┘
             ▼
┌──────────────────────────┐
│  Skill 4: 编码任务         │  CSV + 计算参数 → .bat 或 .sh
└────────────┬─────────────┘
             ▼
┌──────────────────────────┐
│  Skill 5: 多轨封装         │  选择素材 → 生成 ffmpeg 封装脚本
└──────────────────────────┘
```

## 跨平台设计

| 平台 | Shell | 脚本格式 | 编码 | 可用工具 |
|------|-------|---------|------|---------|
| Windows | PowerShell / cmd | `.bat` / `.ps1` | UTF-8 BOM + CRLF | 全部（ffmpeg, vspipe, avs2yuv, avs2pipemod, SVFI, x264, x265, SVT-AV1） |
| macOS | zsh | `.sh` | UTF-8 + LF | ffmpeg, vspipe, x264, x265, SVT-AV1 |
| Linux | bash | `.sh` | UTF-8 + LF | ffmpeg, vspipe, x264, x265, SVT-AV1 |

## 技能列表

| 技能 | 文件 | 功能 |
|------|------|------|
| Skill 1 | `skill-1-environment-check.md` | 运行环境检测（平台自适应） |
| Skill 2 | `skill-2-encoding-pipeline.md` | 生成编码管线脚本 |
| Skill 3 | `skill-3-source-analysis.md` | ffprobe 读取源 |
| Skill 4 | `skill-4-encoding-task.md` | 生成编码任务脚本 |
| Skill 5 | `skill-5-muxing.md` | 生成多轨道封装脚本 |

## 共享参考数据

| 文件 | 内容 |
|------|------|
| `common/platform.md` | **平台抽象层** — 检测规则、代码生成规范、降级策略 |
| `common/presets.md` | 工具链组合、编码器预设参数（跨平台通用） |
| `common/color-space.md` | 色彩空间映射表（SEI / CSP / RAW） |
| `common/fps-utils.md` | 帧率处理逻辑与预设 |

## 伪代码驱动设计

Skill 文件中的流程以**伪代码**形式描述，包含平台分支：

```pseudocode
IF platform == "Windows":
    # 生成 .bat 脚本
    # 使用 set VAR=value 语法
    # 使用 %VAR% 引用
ELIF platform IN ["macOS", "Linux"]:
    # 生成 .sh 脚本
    # 使用 VAR="value" 语法
    # 使用 $VAR 引用
```

LLM 读取伪代码 → 根据当前平台翻译为可执行代码 → 运行或写入脚本文件。

## 使用说明

1. LLM 加载 `system-prompt.md` 和对应技能文件
2. **首次对话时检测平台**（参见 `common/platform.md`）
3. 通过对话收集信息（文件路径、工具选择、参数偏好等）
4. 将伪代码翻译为平台代码后执行
5. 生成的脚本自动匹配目标平台格式

## 前置要求

- 用户系统上需安装 ffmpeg/ffprobe（所有平台）
- 如使用 VapourSynth，需对应安装（所有平台）
- AviSynth / avs2pipemod / SVFI 仅限 Windows
- 编码器（x264/x265/SVT-AV1）需用户提供路径或包管理器安装
