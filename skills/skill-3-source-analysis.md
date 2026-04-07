# Skill 3: ffprobe 视频源分析

## 技能描述

使用 ffprobe 分析视频源文件，提取视频元数据（分辨率、帧率、色彩空间等），检测 VFR（可变帧率）和非方形像素问题，并导出 CSV 文件供后续步骤使用。

## 触发条件

用户提出以下意图之一：
- 分析视频源
- 读取视频信息
- 生成 ffprobe CSV
- 检测视频帧率/色彩空间

---

## 对话流程

### 第一步：选择上游程序类型

```pseudocode
FUNCTION select_upstream_type(platform):
    # 根据平台过滤可用上游选项
    IF platform == "Windows":
        options = {
            "A": "ffmpeg（任意源）",
            "B": "vspipe（.vpy 源）",
            "C": "avs2yuv（.avs 源）",
            "D": "avs2pipemod（.avs 源）",
            "E": "SVFI（.ini 源）"
        }
    ELSE:  # macOS / Linux
        options = {
            "A": "ffmpeg（任意源）",
            "B": "vspipe（.vpy 源）"
        }
        REPORT "ℹ️ 当前平台不支持 avs2yuv / avs2pipemod / SVFI"

    ASK_USER "选择管道上游程序类型："
    DISPLAY options

    SET session.upstream_code = LOWERCASE(user_choice)  # a/b/c/d/e
    RETURN session.upstream_code
```

### 第二步：获取视频源（根据上游类型分支）

```pseudocode
FUNCTION acquire_video_source(platform, upstream_code):
    SWITCH upstream_code:

        CASE "a":  # ffmpeg — 直接分析视频文件
            ASK_USER "请提供视频源文件路径（如 .mp4/.mov/.mkv/.yuv/.y4m 等）"
            SET session.video_source = user_input
            VERIFY FILE_EXISTS(session.video_source)

        CASE "b", "c", "d":  # vspipe / avs2yuv / avs2pipemod — 脚本上游
            ASK_USER "请提供脚本引用的视频源文件路径（ffprobe 将分析此文件）"
            SET session.video_source = user_input
            VERIFY FILE_EXISTS(session.video_source)

            ASK_USER "输入 'y' 导入自定义脚本，Enter 为视频源生成无滤镜脚本"

            IF user_choice == "y":
                ASK_USER "请提供脚本文件路径"
                SET session.script_source = user_input
                # 验证扩展名匹配: vspipe→.vpy, avs2yuv/avs2pipemod→.avs
            ELSE:
                CALL generate_blank_script(platform, upstream_code, session.video_source)

            # avs2pipemod 额外步骤
            IF upstream_code == "d":
                # 仅 Windows
                ASK_USER "请指定 AviSynth.dll 的路径"
                SET session.avisynth_dll = user_input

        CASE "e":  # SVFI — 仅 Windows
            IF platform != "Windows":
                REPORT "❌ SVFI 仅支持 Windows 平台"
                ABORT

            # 自动搜索 SVFI 配置目录
            CALL search_svfi_configs(platform)
            ASK_USER "请提供 SVFI INI 配置文件路径"
            SET session.svfi_config = user_input

            # 从 INI 解析视频源
            CALL parse_svfi_ini(session.svfi_config)
```

#### 生成无滤镜脚本

```pseudocode
FUNCTION generate_blank_script(platform, upstream_code, video_path):
    work_dir = CALL get_work_directory(platform)
    CALL ensure_work_directory(platform)

    IF upstream_code == "b":  # VapourSynth 脚本
        script_content = """
import vapoursynth as vs
core = vs.core
src = core.lsmas.LWLibavSource(source=r"{video_path}")
# 自动生成无滤镜脚本：按需在此处加入滤镜、裁切、帧率调整等
src.set_output()
"""
        script_name = "blank_vs_script.vpy"

    ELIF upstream_code IN ["c", "d"]:  # AviSynth 脚本
        script_content = 'LWLibavVideoSource("{video_path}") # 自动生成的占位脚本，按需修改'
        script_name = "blank_avs_script.avs"

    script_path = JOIN(work_dir, script_name)
    CALL write_output_file(platform, script_path, script_content)
    SET session.script_source = script_path
    REPORT "✅ 已生成占位脚本: {script_path}"
```

#### 解析 SVFI 配置（仅 Windows）

```pseudocode
FUNCTION parse_svfi_ini(ini_path):
    content = READ_FILE(ini_path)
    
    # 查找 gui_inputs= 行
    line = FIND_LINE_STARTING_WITH(content, "gui_inputs=")
    json_str = EXTRACT_AFTER_EQUALS(line)

    # 解析 JSON
    parsed = PARSE_JSON(json_str)
    video_path = parsed["input_path"]

    # 处理 Unicode 转义和双反斜杠
    video_path = UNESCAPE_UNICODE(video_path)
    video_path = REPLACE(video_path, "\\\\", PATH_SEP)

    SET session.video_source = video_path
    SET session.svfi_task_id = parsed.get("task_id", "")

    VERIFY FILE_EXISTS(session.video_source)
```

### 第三步：定位 ffprobe

```pseudocode
FUNCTION locate_ffprobe(platform):
    exe_name = PLATFORM_EXE_NAME(platform, "ffprobe")  # ffprobe.exe 或 ffprobe

    # 自动搜索
    found = CALL FIND_EXE(exe_name)
    IF NOT found:
        search_paths = CALL get_search_paths(platform, "ffprobe")
        found = SEARCH_IN(search_paths)

    IF found:
        REPORT "✅ 找到 ffprobe: {found}"
        SET session.ffprobe_path = found
    ELSE:
        ASK_USER "请提供 ffprobe 的完整路径"
        SET session.ffprobe_path = user_input

    VERIFY FILE_EXISTS(session.ffprobe_path)
```

### 第四步：执行 ffprobe 分析

```pseudocode
FUNCTION run_ffprobe_analysis(platform):
    video = session.video_source
    ffprobe = session.ffprobe_path

    # ffprobe 命令（跨平台通用）
    cmd = 'QUOTE(ffprobe) -v quiet -hide_banner -select_streams v:0 ' +
          '-show_entries stream=r_frame_rate,avg_frame_rate,nb_frames,duration,sample_aspect_ratio ' +
          '-show_format -of json QUOTE(video)'

    result = EXECUTE cmd
    metadata = PARSE_JSON(result.output)

    SET session.metadata = {
        r_frame_rate:   metadata.streams[0].r_frame_rate,
        avg_frame_rate: metadata.streams[0].avg_frame_rate,
        nb_frames:      metadata.streams[0].nb_frames,
        duration:       metadata.format.duration,
        sar:            metadata.streams[0].sample_aspect_ratio
    }

    REPORT "基础帧率: {r_frame_rate}"
    REPORT "平均帧率: {avg_frame_rate}"
    REPORT "总帧数:   {nb_frames}"
    REPORT "时长:     {duration} 秒"
```

### 第五步：VFR 检测

```pseudocode
FUNCTION detect_vfr():
    # 参考 common/fps-utils.md 中的评分系统
    score = 0
    r_fps = PARSE_FPS(session.metadata.r_frame_rate)
    a_fps = PARSE_FPS(session.metadata.avg_frame_rate)

    # 评分规则 1: 基础帧率 ≠ 平均帧率
    IF ABS(r_fps - a_fps) / MAX(r_fps, a_fps) > 1e-9:
        score += 1

    # 评分规则 2: 估计帧率 ≠ 平均帧率
    IF session.metadata.nb_frames AND session.metadata.duration:
        e_fps = nb_frames / duration
        IF ABS(e_fps - a_fps) / MAX(e_fps, a_fps) > 1e-9:
            score += 2

    # 评分规则 3: 特殊 r_frame_rate
    IF session.metadata.r_frame_rate == "90000/1":
        score += 3

    # 评分规则 4: 大分母
    den = DENOMINATOR(session.metadata.avg_frame_rate)
    IF den > 50000:
        divisor_count = COUNT_DIVISORS(den)
        IF divisor_count <= 5:
            score += 2
        ELSE:
            score += 1

    # 报告
    IF score >= 5:
        REPORT "❌ 确定是 VFR（可变帧率），score={score}"
    ELIF score >= 2:
        REPORT "⚠️ 可能是 VFR，score={score}"
    ELIF score > 0:
        REPORT "ℹ️ 有 VFR 迹象，score={score}"
    ELSE:
        REPORT "✅ 确定是 CFR（恒定帧率）"

    IF score > 0:
        REPORT "VFR 修复建议："
        REPORT "  ffmpeg -i QUOTE(video) -r {r_fps_num}/{r_fps_den} -c:v ffv1 -level 3 -context 1 -g 180 -c:a copy output.mkv"
        REPORT "VFR 测量命令："
        REPORT "  ffmpeg -i QUOTE(video) -vf vfrdet -an -f null -"
```

### 第六步：非方形像素检测

```pseudocode
FUNCTION detect_sar():
    sar = session.metadata.sar
    IF sar AND sar != "1:1" AND sar != "N/A":
        REPORT "⚠️ 非方形像素比（SAR={sar}），可能导致画面拉伸"
        REPORT "修正方法："
        REPORT "  ffmpeg -i QUOTE(video) -c copy -aspect {sar} output.mkv"
```

### 第七步：封装格式检测

```pseudocode
FUNCTION detect_container_format(platform):
    cmd = 'QUOTE(ffprobe) -v quiet -hide_banner -show_format -of json QUOTE(video)'
    result = EXECUTE cmd
    format_info = PARSE_JSON(result.output)

    format_name = format_info.format.format_name
    file_ext = GET_EXTENSION(session.video_source)

    # 格式映射
    IF format_name MATCHES "mpeg" AND has_dvd_streams:
        SET session.video_format = "VOB"
    ELIF format_name MATCHES "mov" AND file_ext == ".mov":
        SET session.video_format = "MOV"
    ELSE:
        SET session.video_format = "std"

    REPORT "封装格式: {session.video_format}"
```

### 第八步：导出 CSV

```pseudocode
FUNCTION export_csv(platform):
    work_dir = CALL get_work_directory(platform)
    CALL ensure_work_directory(platform)

    # --- 视频信息 CSV ---
    # 根据格式选择 ffprobe 参数
    IF session.video_format == "VOB" OR session.video_format == "MOV":
        entries = "stream=width,height,pix_fmt,color_space,color_transfer,color_primaries,field_order,avg_frame_rate,nb_frames"
        csv_suffix = "_is_" + LOWERCASE(session.video_format)
    ELSE:
        entries = "stream=width,height,pix_fmt,color_space,color_transfer,color_primaries,avg_frame_rate,nb_frames,interlaced_frame,top_field_first" +
                  ":stream_tags=NUMBER_OF_FRAMES,NUMBER_OF_FRAMES-eng"
        csv_suffix = ""

    csv_cmd = 'QUOTE(ffprobe) -i QUOTE(video) -select_streams v:0 -v error -hide_banner -show_streams ' +
              '-show_entries {entries} -of csv'
    csv_output = EXECUTE csv_cmd

    v_info_path = JOIN(work_dir, "temp_v_info{csv_suffix}.csv")
    WRITE_FILE v_info_path, csv_output.output

    # --- 源信息 CSV ---
    s_info_path = JOIN(work_dir, "temp_s_info.csv")
    s_info_content = FORMAT_CSV_LINE(
        session.video_source,        # 或 script_source
        session.upstream_code,
        session.avisynth_dll OR "",
        session.svfi_config OR "",
        session.svfi_task_id OR ""
    )
    WRITE_FILE s_info_path, s_info_content

    REPORT "✅ 视频信息 CSV: {v_info_path}"
    REPORT "✅ 源信息 CSV:   {s_info_path}"
```

**CSV 列映射（标准格式）：**
- A: stream 标记
- B: width（宽度）
- C: height（高度）
- D: pix_fmt（像素格式）
- E: color_space（色彩矩阵）
- F: color_transfer（传输特性）
- G: color_primaries（三原色）
- H: avg_frame_rate（平均帧率）— MOV/VOB 中为 field_order，帧率在 I
- I: nb_frames — MOV/VOB 中为 avg_frame_rate
- J-AJ: 其他标签数据

---

## 输出

| 文件 | 位置 | 用途 |
|------|------|------|
| temp_v_info*.csv | 工作目录（`$HOME/bbenc` 或 `%USERPROFILE%\bbenc`） | 视频元数据 |
| temp_s_info.csv | 工作目录 | 源文件信息 |
| blank_avs_script.avs | 工作目录（可选，仅 Windows） | 无滤镜 AVS 脚本 |
| blank_vs_script.vpy | 工作目录（可选） | 无滤镜 VPY 脚本 |

## 依赖

- ffprobe 可执行文件（来自 ffmpeg 工具包，跨平台通用）
