# BBEnc LLM Skills — 视频批量编码对话技能组

本技能组将原 PowerShell 交互式脚本转化为 LLM 可执行的对话技能，实现相同的视频编码工作流。

## 工作流概览

```
┌─────────────────────┐
│  Skill 1: 环境检测    │  检查系统兼容性、权限、硬件信息
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│  Skill 2: 编码管线    │  选择上下游工具 → 生成 encode_single.bat
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│  Skill 3: 源分析      │  ffprobe 分析视频 → 导出 CSV 元数据
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│  Skill 4: 编码任务    │  读取 CSV + 计算参数 → 生成 encode_task_final.bat
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│  Skill 5: 多轨封装    │  选择素材文件 → 生成 ffmpeg_mux.bat
└──────────┘──────────┘
```

## 技能列表

| 技能 | 文件 | 功能 |
|------|------|------|
| Skill 1 | `skill-1-environment-check.md` | 运行环境检测 |
| Skill 2 | `skill-2-encoding-pipeline.md` | 生成编码管线批处理 |
| Skill 3 | `skill-3-source-analysis.md` | ffprobe 读取源 |
| Skill 4 | `skill-4-encoding-task.md` | 生成编码任务批处理 |
| Skill 5 | `skill-5-muxing.md` | 生成多轨道封装批处理 |

## 共享参考数据

| 文件 | 内容 |
|------|------|
| `common/presets.md` | 工具链组合、编码器预设参数 |
| `common/color-space.md` | 色彩空间映射表（SEI / CSP / RAW） |
| `common/fps-utils.md` | 帧率处理逻辑与预设 |

## 使用说明

1. LLM 加载对应技能文件作为系统提示或工具定义
2. 通过与用户的对话收集必要信息（文件路径、工具选择、参数偏好等）
3. 调用系统命令（ffprobe、ffmpeg 等）执行分析
4. 根据技能中的逻辑生成批处理文件

## 前置要求

- 用户系统上需安装 ffmpeg/ffprobe
- 如使用 VapourSynth/AviSynth，需对应安装
- 编码器（x264/x265/SVT-AV1）需用户提供路径
