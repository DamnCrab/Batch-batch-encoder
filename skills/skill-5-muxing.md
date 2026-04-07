# Skill 5: 多轨道封装脚本生成器

## 技能描述

通过对话引导用户导入多个视频、音频、字幕、字体文件，生成 ffmpeg 多轨道封装命令脚本（Windows: `.bat` / macOS+Linux: `.sh`）。

## 触发条件

用户提出以下意图之一：
- 封装/混流视频
- 合并音视频轨道
- 生成封装脚本
- 创建 mux 命令

---

## 对话流程

### 第一步：导入工具与路径

```pseudocode
FUNCTION import_tools_and_paths(platform):
    # 1. ffprobe
    ffprobe_path = CALL locate_executable(platform, "ffprobe")

    # 2. ffmpeg
    ffmpeg_path = CALL locate_executable(platform, "ffmpeg")

    # 3. 脚本导出目录
    ASK_USER "选择导出封装脚本的文件夹"
    script_dir = user_input
    VERIFY DIR_EXISTS(script_dir)

    # 4. 封装结果导出目录
    ASK_USER "选择封装结果的导出文件夹"
    output_dir = user_input
    VERIFY DIR_EXISTS(output_dir)

    RETURN ffprobe_path, ffmpeg_path, script_dir, output_dir
```

### 第二步：循环导入素材文件

```pseudocode
FUNCTION import_media_files(platform, ffprobe_path):
    REPORT "ℹ️ 仅第一个视频文件会被用作主视频流，后续文件只添加音频、字幕等轨道"

    input_files = []
    map_args = []
    codec_args = []
    file_index = 0

    LOOP:
        ASK_USER "请提供第 {file_index + 1} 个源文件路径（输入 'q' 结束导入）"
        IF user_input == "q":
            BREAK

        file_path = user_input
        VERIFY FILE_EXISTS(file_path)

        file_ext = LOWERCASE(GET_EXTENSION(file_path))

        # 根据文件类型分析
        SWITCH file_ext:
            # --- 视频容器格式 ---
            CASE ".mkv", ".mp4", ".mov", ".f4v", ".flv", ".avi", ".m3u", ".mxv":
                result = CALL analyze_container(platform, ffprobe_path, file_path)
                APPEND result TO input_files, map_args, codec_args

            # --- 音频容器格式 ---
            CASE ".m4a", ".mka", ".mks":
                APPEND '-c:a copy' TO codec_args
                # 检查是否包含字幕流
                sub_streams = CALL probe_streams(ffprobe_path, file_path, "s")
                IF sub_streams > 0:
                    APPEND '-c:s copy' TO codec_args

            # --- 裸视频流 ---
            CASE ".hevc", ".h264", ".h265", ".avc", ".ivf", ".obu", ".265", ".264":
                fps = CALL get_video_fps(platform, ffprobe_path, file_path)
                IF fps:
                    APPEND '-r {fps} -c:v copy' TO codec_args
                ELSE:
                    REPORT "⚠️ 无法获取帧率"
                    fps = CALL resolve_fps_interactive(platform, ffprobe_path)
                    IF fps:
                        APPEND '-r {fps} -c:v copy' TO codec_args
                    ELSE:
                        REPORT "跳过此文件"
                        CONTINUE

            # --- 裸音频流 ---
            CASE ".aac", ".flac", ".mp3", ".opus", ".wav", ".eac3", ".ac3", ".dts", ".ogg", ".wma":
                APPEND '-c:a copy' TO codec_args

            # --- 字幕文件 ---
            CASE ".srt", ".ass", ".ssa":
                APPEND '-c:s copy' TO codec_args

            # --- 字体文件 ---
            CASE ".ttf", ".ttc", ".otf":
                APPEND '-c:t copy' TO codec_args

            DEFAULT:
                REPORT "⚠️ 未知文件类型: {file_ext}，尝试作为输入文件添加"

        input_files.APPEND(file_path)
        map_args.APPEND("-map {file_index}")
        file_index += 1

        # 询问是否继续
        ASK_USER "继续添加文件？输入 'y' 确认，Enter 完成"
        IF user_choice != "y":
            BREAK

    RETURN input_files, map_args, codec_args
```

#### 容器分析子流程

```pseudocode
FUNCTION analyze_container(platform, ffprobe_path, file_path):
    # 探测视频流
    v_cmd = 'QUOTE(ffprobe_path) -v quiet -print_format json -show_streams -select_streams v QUOTE(file_path)'
    v_result = EXECUTE v_cmd
    v_streams = PARSE_JSON(v_result.output).streams

    # 探测音频流
    a_cmd = 'QUOTE(ffprobe_path) -v quiet -print_format json -show_streams -select_streams a QUOTE(file_path)'
    a_result = EXECUTE a_cmd
    a_streams = PARSE_JSON(a_result.output).streams

    # 探测字幕流
    s_cmd = 'QUOTE(ffprobe_path) -v quiet -print_format json -show_streams -select_streams s QUOTE(file_path)'
    s_result = EXECUTE s_cmd
    s_streams = PARSE_JSON(s_result.output).streams

    args = []
    IF len(v_streams) > 0:
        fps = v_streams[0].r_frame_rate OR v_streams[0].avg_frame_rate
        args.APPEND("-r {fps} -c:v copy")
    IF len(a_streams) > 0:
        args.APPEND("-c:a copy")
    IF len(s_streams) > 0:
        args.APPEND("-c:s copy")

    RETURN args
```

#### 帧率交互式解决

```pseudocode
FUNCTION resolve_fps_interactive(platform, ffprobe_path):
    ASK_USER "选择帧率来源：
        1：手动输入帧率
        2：从其他封装视频文件读取（推荐）
        3：使用常用预设帧率
        q：跳过此文件"

    SWITCH user_choice:
        "1":
            ASK_USER "请输入帧率（整数/小数/分数，如 24、23.976、24000/1001）"
            RETURN PARSE_FPS(user_input)

        "2":
            ASK_USER "选择一个包含帧率信息的封装视频文件"
            ref_fps = CALL get_video_fps(platform, ffprobe_path, user_input)
            RETURN ref_fps

        "3":
            DISPLAY FPS_PRESET_TABLE  # 参考 common/fps-utils.md 常用帧率预设
            ASK_USER "请选择预设帧率编号"
            RETURN FPS_PRESETS[user_choice]

        "q":
            RETURN null
```

### 第三步：确定输出文件名

```pseudocode
FUNCTION get_mux_output_name(platform, input_files):
    default_name = GET_FILENAME_WITHOUT_EXT(input_files[0]) + "_mux"

    ASK_USER "请输入输出文件名（留空默认：{default_name}）"
    filename = user_input OR default_name

    IF NOT CALL validate_filename(platform, filename):
        REPORT "⚠️ 文件名包含非法字符，已自动修正"
        filename = SANITIZE_FILENAME(platform, filename)

    RETURN filename
```

### 第四步：选择封装容器

```pseudocode
FUNCTION select_container():
    ASK_USER "选择封装容器:
        1：MP4（适合通用）
        2：MOV（适合剪辑）
        3：MKV（兼容字幕、字体）
        4：MXF（专业用途）
        ⚠️ ffmpeg 正在弃用 MP4 时间码（pts）生成功能，届时 MP4 格式选项将不可用"

    container_map = {"1": ".mp4", "2": ".mov", "3": ".mkv", "4": ".mxf"}
    RETURN container_map[user_choice]
```

### 第五步：兼容性检查与自动修复

```pseudocode
FUNCTION check_compatibility(codec_args, container):
    # 检查 1：字体流 + 非 MKV 容器
    IF "-c:t copy" IN codec_args AND container != ".mkv":
        ASK_USER "⚠️ 检测到字体流，但 {container} 不支持字体嵌入
            d：删除字体导入
            m：改用 MKV 容器
            Enter：忽略"
        
        IF user_choice == "d":
            REMOVE "-c:t copy" FROM codec_args
        ELIF user_choice == "m":
            container = ".mkv"

    # 检查 2：字幕流 + MP4/MOV
    IF "-c:s copy" IN codec_args AND container IN [".mp4", ".mov"]:
        ASK_USER "⚠️ 字幕流在 {container} 中支持较差，大概率无法封装
            d：删除字幕导入
            m：改用 MKV 容器
            Enter：忽略"

        IF user_choice == "d":
            REMOVE "-c:s copy" FROM codec_args
        ELIF user_choice == "m":
            container = ".mkv"

    RETURN codec_args, container
```

### 第六步：生成脚本文件

```pseudocode
FUNCTION generate_mux_script(platform, ffmpeg_path, input_files, map_args, codec_args, output_path):
    # 构建 ffmpeg 命令
    input_part = ""
    FOR file IN input_files:
        input_part += '-i QUOTE(file) '

    mapping_part = ""
    FOR i, args IN enumerate(MERGE(map_args, codec_args)):
        mapping_part += '{args} '

    ffmpeg_cmd = 'QUOTE(ffmpeg_path) {input_part} {mapping_part} QUOTE(output_path)'

    # 构建完整脚本
    comment = PLATFORM_COMMENT(platform)   # "REM" or "#"
    timestamp = CURRENT_TIMESTAMP()

    IF platform == "Windows":
        script = """@echo off
chcp 65001 >nul
setlocal

{comment} ========================================
{comment} ffmpeg 封装工具
{comment} 生成时间：{timestamp}
{comment} ========================================

echo 开始封装任务...

{ffmpeg_cmd}

echo.
echo ========================================
echo  批处理执行完毕！
echo ========================================

endlocal
echo 按任意键进入命令提示符，输入 exit 退出...
cmd /k
"""
    ELSE:
        IF platform == "macOS":
            shebang = "#!/usr/bin/env zsh"
        ELSE:
            shebang = "#!/usr/bin/env bash"

        script = """{shebang}
set -euo pipefail

{comment} ========================================
{comment} ffmpeg 封装工具
{comment} 生成时间：{timestamp}
{comment} ========================================

echo "开始封装任务..."

{ffmpeg_cmd}

echo ""
echo "========================================"
echo " 封装完毕！"
echo "========================================"
"""

    RETURN script
```

### 第七步：保存与提示

```pseudocode
FUNCTION save_mux_script(platform, script_dir, script_content):
    IF platform == "Windows":
        script_name = "ffmpeg_mux.bat"
    ELSE:
        script_name = "ffmpeg_mux.sh"

    script_path = JOIN(script_dir, script_name)
    CALL write_output_file(platform, script_path, script_content)

    REPORT "✅ 封装脚本已保存至: {script_path}"
    REPORT ""
    REPORT "提示：如果音画不同步，请在 -map 和 -c 之间添加 -itoffset <秒> 参数"
```

---

## 帧率验证规则

有效帧率格式：
- 分数: `24000/1001`
- 整数: `24`
- 小数: `23.976`
- 排除: `0/0`、`0`、`0.0`

## 输出

| 平台 | 文件 | 用途 |
|------|------|------|
| Windows | `ffmpeg_mux.bat` | ffmpeg 多轨道封装命令 |
| macOS | `ffmpeg_mux.sh` | ffmpeg 多轨道封装命令 |
| Linux | `ffmpeg_mux.sh` | ffmpeg 多轨道封装命令 |

## 依赖

- ffprobe 可执行文件（跨平台）
- ffmpeg 可执行文件（跨平台）
- 此技能独立于 Skill 1-4，可单独使用
