# Skill 2: 编码管线批处理生成器

## 技能描述

通过对话引导用户选择上游工具（视频源解析器）和下游工具（编码器），生成 `encode_single.bat` 管线批处理文件，其中包含管道式编码命令。

## 触发条件

用户提出以下意图之一：
- 生成编码管线/管道
- 创建 encode_single.bat
- 配置编码工具链
- 选择编码器组合

## 前置知识

参考 `common/presets.md` 获取完整的工具链组合列表和命令模板。

## 对话流程

### 第一步：选择输出路径

**询问用户：**
> 请指定 encode_single.bat 批处理文件的保存位置（文件夹路径）。

**验证：** 路径必须存在且可写。

### 第二步：导入上游工具

按顺序逐一询问用户是否导入以下 5 种上游工具：

```
[上游] (1/5) 导入 ffmpeg 可执行文件？（y=是，Enter 跳过）
[上游] (2/5) 导入 vspipe 可执行文件？（y=是，Enter 跳过）
[上游] (3/5) 导入 avs2yuv 可执行文件？（y=是，Enter 跳过）
[上游] (4/5) 导入 avs2pipemod 可执行文件？（y=是，Enter 跳过）
[上游] (5/5) 导入 svfi 可执行文件？（y=是，Enter 跳过）
```

**对于每个选择 "y" 的工具：**

1. **自动搜索：** 在以下位置查找可执行文件：
   - 脚本/工作目录
   - 环境变量 PATH
   - 工具特定路径：
     - vspipe: `C:\Program Files\VapourSynth\core\`
     - svfi: `{盘符}:\SteamLibrary\steamapps\common\SVFI\`

2. **如果找到：**
   > 自动检测到 {tool} 位于：{path}
   > 是否使用此文件？（Enter=确认, n=手动选择）

3. **如果未找到：** 请求用户提供路径
   - svfi 提示: `SVFI（one_line_shot_args.exe）Steam 发布版的路径是 X:\SteamLibrary\steamapps\common\SVFI\`
   - vspipe 提示: `安装版 VapourSynth 的默认可执行文件路径是 C:\Program Files\VapourSynth\core\vspipe.exe`

4. **vspipe 特殊处理：** 检测 Y4M 参数支持
   - 按顺序测试：`-c y4m`、`--container y4m`、`--y4m`
   - 执行命令并检查输出是否包含 `"No script file specified"`
   - 记录成功的参数格式

### 第三步：导入下游工具

按顺序逐一询问用户是否导入以下 3 种下游编码器：

```
[下游] (1/3) 导入 x264？（y=是，Enter 跳过）
[下游] (2/3) 导入 x265？（y=是，Enter 跳过）
[下游] (3/3) 导入 svtav1？（y=是，Enter 跳过）
```

**同样执行自动搜索和手动选择流程。**

**svtav1 额外提示：**
> 建议自行编译 SVT-AV1 编码器（大幅提高性能）
> 编译教程：https://iavoe.github.io/av1-web-tutorial/HTML/index.html

### 第四步：验证工具组合

**检查：** 至少需要一个上游工具和一个下游工具。

**错误处理：**
> 至少需要选择一个上游工具和一个下游工具（例如 ffmpeg + x265 或 ffmpeg + svtav1）

### 第五步：选择工具链

**展示所有可用的工具链组合：**

```
ID     Preset                 Upstream     Downstream
──────────────────────────────────────────────────────────
[1]    ffmpeg_x264            ffmpeg       x264
[2]    ffmpeg_x265            ffmpeg       x265
...
```

仅显示用户已导入工具能组成的组合。

**如果只有一种组合：** 自动选择。

**如果有多种组合：** 请求用户输入编号选择。

### 第六步：生成批处理文件

**根据选定的工具链，使用命令模板生成管道命令。**

命令模板（参见 `common/presets.md`）根据上游工具不同：

| 上游 | 模板 |
|------|------|
| ffmpeg | `"{upstream}" %ffmpeg_params% -f yuv4mpegpipe -an -strict unofficial - \| "{downstream}" {pipe_arg} %{encoder}_params%` |
| vspipe | `"{upstream}" %vspipe_params% {y4m_arg} - \| "{downstream}" {pipe_arg} %{encoder}_params%` |
| avs2yuv | `"{upstream}" %avs2yuv_params% - \| "{downstream}" {pipe_arg} %{encoder}_params%` |
| avs2pipemod | `"{upstream}" %avs2pipemod_params% -y4mp \| "{downstream}" {pipe_arg} %{encoder}_params%` |
| svfi | `"{upstream}" %svfi_params% --pipe-out \| "{downstream}" {pipe_arg} %{encoder}_params%` |

**批处理文件模板：**

```batch

@echo off
chcp 65001 >nul
setlocal

REM ========================================
REM 视频编码工具调用管线
REM 生成时间: {当前时间}
REM 工具链（变更时需指定）: {选定的preset名}
REM ========================================

echo.
echo 开始编码任务...
echo.

REM 参数示例（由后续脚本编辑）
REM set ffmpeg_params=-i input.mkv -an -f yuv4mpegpipe -strict unofficial
REM set x265_params=--y4m - -o output.hevc
REM set svtav1_params=-i - -b output.ivf

REM 指定本次所需编码命令

{主命令}

REM ========================================
REM 备用编码命令（手动切换，只导入一种编码器则留空）
REM ========================================

{备用命令，以 REM 注释}

echo.
echo 编码完成！输入 exit 退出...
echo.

endlocal
cmd /k
```

### 第七步：保存与验证

1. 如果目标文件已存在，询问是否删除旧文件
2. 以 UTF-8 BOM 编码、CRLF 换行符写入文件
3. 验证文件格式（检查 CR/LF 配对）
4. 显示额外提示：
   - x265 默认输出 `.hevc` 文件
   - SVT-AV1 默认输出 `.ivf` 文件
   - 如需容器格式，需后续封装

## 输出

- `encode_single.bat`：包含管道编码命令的批处理模板文件

## 依赖

无，此技能独立运行。后续 Skill 4 将引用此文件。
