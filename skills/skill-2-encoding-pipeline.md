# Skill 2: 编码管线脚本生成器

## 技能描述

通过对话引导用户选择上游工具（视频源解析器）和下游工具（编码器），生成编码管线脚本（Windows: `.bat` / macOS+Linux: `.sh`）。

## 触发条件

用户提出以下意图之一：
- 生成编码管线/管道
- 创建编码脚本
- 配置编码工具链
- 选择编码器组合

## 前置知识

- 参考 `common/presets.md` 获取完整工具链组合列表和命令模板
- 参考 `common/platform.md` 获取平台差异

---

## 对话流程

### 第一步：选择输出路径

```pseudocode
FUNCTION select_output_path(platform):
    ASK_USER "请指定编码管线脚本的保存位置（文件夹路径）"

    path = user_input
    IF NOT DIR_EXISTS(path):
        REPORT "路径不存在，请重新输入"
        RETRY

    # 确认脚本文件名
    IF platform == "Windows":
        script_name = "encode_single.bat"
    ELSE:
        script_name = "encode_single.sh"

    RETURN Join(path, script_name)
```

### 第二步：导入上游工具

```pseudocode
FUNCTION import_upstream_tools(platform):
    upstream_tools = {}

    # 根据平台过滤可用工具
    IF platform == "Windows":
        available_upstreams = ["ffmpeg", "vspipe", "avs2yuv", "avs2pipemod", "svfi"]
    ELIF platform == "macOS":
        available_upstreams = ["ffmpeg", "vspipe"]
        REPORT "ℹ️ macOS 平台：avs2yuv、avs2pipemod、SVFI 不可用"
    ELIF platform == "Linux":
        available_upstreams = ["ffmpeg", "vspipe"]
        REPORT "ℹ️ Linux 平台：avs2yuv、avs2pipemod、SVFI 不可用"

    FOR index, tool IN enumerate(available_upstreams):
        ASK_USER "[上游] ({index+1}/{len}) 导入 {tool}？（y=是，Enter 跳过）"
        IF user_choice == "y":
            tool_path = CALL find_tool(platform, tool)
            IF tool_path:
                upstream_tools[tool] = tool_path

    RETURN upstream_tools
```

#### 工具查找子流程

```pseudocode
FUNCTION find_tool(platform, tool_name):
    exe_name = PLATFORM_EXE_NAME(platform, tool_name)  # 参见 common/platform.md

    # 1. 自动搜索
    search_paths = CALL get_search_paths(platform, tool_name)
    found_path = CALL FIND_EXE(exe_name) OR SEARCH_IN(search_paths)

    IF found_path:
        ASK_USER "自动检测到 {tool_name} 位于：{found_path}
                   是否使用此文件？（Enter=确认, n=手动选择）"
        IF user_confirms:
            RETURN found_path

    # 2. 手动选择
    IF platform == "Windows":
        # 可尝试打开文件对话框（WinForms）或请求路径
        ASK_USER "请输入 {tool_name} 的完整路径"
    ELSE:
        ASK_USER "请输入 {tool_name} 的完整路径"

    # vspipe 特殊处理：检测 Y4M 参数支持
    IF tool_name == "vspipe":
        CALL detect_vspipe_y4m_arg(platform, tool_path)

    RETURN tool_path
```

#### vspipe Y4M 参数检测

```pseudocode
FUNCTION detect_vspipe_y4m_arg(platform, vspipe_path):
    # 按顺序测试三种 Y4M 参数格式
    y4m_candidates = ["-c y4m", "--container y4m", "--y4m"]

    FOR candidate IN y4m_candidates:
        result = EXECUTE "QUOTE(vspipe_path) {candidate}"
        IF result.output CONTAINS "No script file specified":
            SET session.vspipe_y4m_arg = candidate
            REPORT "✅ vspipe 使用 Y4M 参数: {candidate}"
            RETURN

    REPORT "⚠️ 无法确定 vspipe 的 Y4M 参数格式，默认使用 -c y4m"
    SET session.vspipe_y4m_arg = "-c y4m"
```

### 第三步：导入下游工具

```pseudocode
FUNCTION import_downstream_tools(platform):
    downstream_tools = {}
    available_downstreams = ["x264", "x265", "svtav1"]

    FOR index, tool IN enumerate(available_downstreams):
        ASK_USER "[下游] ({index+1}/3) 导入 {tool}？（y=是，Enter 跳过）"
        IF user_choice == "y":
            tool_path = CALL find_tool(platform, tool)
            IF tool_path:
                downstream_tools[tool] = tool_path

    # svtav1 额外提示
    IF "svtav1" IN downstream_tools:
        REPORT "ℹ️ 建议自行编译 SVT-AV1 编码器（大幅提高性能）"
        REPORT "   教程：https://iavoe.github.io/av1-web-tutorial/HTML/index.html"

    RETURN downstream_tools
```

### 第四步：验证工具组合

```pseudocode
FUNCTION validate_tool_combination(upstream_tools, downstream_tools):
    IF len(upstream_tools) == 0 OR len(downstream_tools) == 0:
        REPORT "❌ 至少需要选择一个上游工具和一个下游工具"
        REPORT "   例如 ffmpeg + x265 或 ffmpeg + svtav1"
        ABORT
```

### 第五步：选择工具链

```pseudocode
FUNCTION select_toolchain(upstream_tools, downstream_tools):
    # 生成所有可用组合
    combinations = []
    FOR upstream IN upstream_tools:
        FOR downstream IN downstream_tools:
            combinations.APPEND({
                id: len(combinations) + 1,
                name: "{upstream}_{downstream}",
                upstream: upstream,
                downstream: downstream
            })

    IF len(combinations) == 1:
        REPORT "仅一种可用组合，自动选择: {combinations[0].name}"
        RETURN combinations[0]

    # 展示选项
    DISPLAY_TABLE combinations WITH columns [ID, Preset, Upstream, Downstream]
    ASK_USER "请输入编号选择工具链"
    RETURN combinations[user_choice - 1]
```

### 第六步：生成脚本文件

```pseudocode
FUNCTION generate_pipeline_script(platform, toolchain, upstream_tools, downstream_tools):
    upstream = toolchain.upstream
    downstream = toolchain.downstream
    upstream_path = upstream_tools[upstream]
    downstream_path = downstream_tools[downstream]

    # 获取管道参数（参见 common/presets.md）
    pipe_arg = GET_PIPE_ARG(downstream)    # --demuxer y4m / --y4m / 空
    y4m_arg = GET_Y4M_ARG(upstream)        # -c y4m / -y4mp / --pipe-out / etc

    # 构建管道命令
    pipe_cmd = CALL build_pipe_command(platform, upstream, downstream, 
                                       upstream_path, downstream_path,
                                       pipe_arg, y4m_arg)

    # 构建备用命令（其他编码器）
    alt_cmds = []
    FOR alt_downstream IN downstream_tools:
        IF alt_downstream != downstream:
            alt_cmd = CALL build_pipe_command(platform, upstream, alt_downstream,
                                              upstream_path, downstream_tools[alt_downstream],
                                              GET_PIPE_ARG(alt_downstream), y4m_arg)
            alt_cmds.APPEND(alt_cmd)

    # 组合为完整脚本
    script_content = CALL build_script_content(platform, toolchain.name,
                                                pipe_cmd, alt_cmds)
    RETURN script_content
```

#### 脚本模板（伪代码 → 平台转换）

```pseudocode
FUNCTION build_script_content(platform, preset_name, main_cmd, alt_cmds):
    timestamp = CURRENT_TIMESTAMP()

    IF platform == "Windows":
        header = """
@echo off
chcp 65001 >nul
setlocal
"""
        comment_prefix = "REM"
        param_example_marker = "REM 参数示例"
        cmd_marker = "REM 指定本次所需编码命令"
        footer = """
endlocal
cmd /k
"""

    ELSE:  # macOS / Linux
        IF platform == "macOS":
            header = "#!/usr/bin/env zsh\nset -euo pipefail\n"
        ELSE:
            header = "#!/usr/bin/env bash\nset -euo pipefail\n"
        comment_prefix = "#"
        param_example_marker = "# 参数示例"
        cmd_marker = "# 指定本次所需编码命令"
        footer = """
echo ""
echo "编码完成！"
"""

    script = header
    script += "{comment_prefix} ========================================\n"
    script += "{comment_prefix} 视频编码工具调用管线\n"
    script += "{comment_prefix} 生成时间: {timestamp}\n"
    script += "{comment_prefix} 工具链: {preset_name}\n"
    script += "{comment_prefix} ========================================\n\n"
    script += "echo 开始编码任务...\n\n"
    script += "{param_example_marker}（由后续步骤编辑）\n\n"
    script += "{cmd_marker}\n\n"
    script += main_cmd + "\n\n"
    script += "{comment_prefix} ========================================\n"
    script += "{comment_prefix} 备用编码命令\n"
    script += "{comment_prefix} ========================================\n"
    FOR cmd IN alt_cmds:
        script += "{comment_prefix} {cmd}\n"
    script += "\n"
    script += footer

    RETURN script
```

### 第七步：保存与验证

```pseudocode
FUNCTION save_and_verify(platform, script_path, script_content):
    # 检查目标文件是否已存在
    IF FILE_EXISTS(script_path):
        ASK_USER "文件已存在，是否删除旧文件？（y/n）"
        IF user_confirms:
            DELETE script_path

    # 按平台规范写入
    CALL write_output_file(platform, script_path, script_content)

    # 验证
    IF platform == "Windows":
        # 检查 CRLF 换行符
        VERIFY file_has_crlf(script_path)
    ELSE:
        # 检查可执行权限
        VERIFY file_is_executable(script_path)

    REPORT "✅ 管线脚本已保存至: {script_path}"
    REPORT ""
    REPORT "提示："
    REPORT "  - x265 默认输出 .hevc 文件"
    REPORT "  - SVT-AV1 默认输出 .ivf 文件"
    REPORT "  - 如需容器格式（.mkv/.mp4），需后续使用 Skill 5 封装"
```

---

## 输出

| 平台 | 文件 | 用途 |
|------|------|------|
| Windows | `encode_single.bat` | 管道编码命令（cmd 格式） |
| macOS | `encode_single.sh` | 管道编码命令（zsh 格式） |
| Linux | `encode_single.sh` | 管道编码命令（bash 格式） |

## 依赖

无，此技能独立运行。后续 Skill 4 将引用此文件。
