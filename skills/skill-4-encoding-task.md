# Skill 4: 编码任务脚本生成器

## 技能描述

读取 Skill 2 生成的管线模板和 Skill 3 生成的 CSV 元数据，计算视频编码参数（关键帧间隔、色彩空间、动态搜索范围等），注入到脚本模板中，生成最终编码任务脚本。

## 触发条件

用户提出以下意图之一：
- 生成编码任务
- 计算编码参数
- 创建编码脚本
- 注入编码参数到模板

## 前置要求

- Skill 2 的管线脚本（`encode_single.bat` 或 `encode_single.sh`）
- Skill 3 的 CSV 文件（`temp_v_info*.csv` + `temp_s_info.csv`）

---

## 对话流程

### 第一步：自动读取 CSV 文件

```pseudocode
FUNCTION read_csv_files(platform):
    work_dir = CALL get_work_directory(platform)

    # 查找最新的视频信息 CSV（按文件修改时间降序）
    IF platform == "Windows":
        EXECUTE "Get-ChildItem -Path '{work_dir}' -Filter 'temp_v_info*.csv' | Sort-Object LastWriteTime -Descending | Select-Object -First 1"
    ELSE:
        EXECUTE "ls -t '{work_dir}'/temp_v_info*.csv 2>/dev/null | head -1"

    IF NOT found:
        REPORT "❌ 未找到 ffprobe CSV 文件，请先运行 Skill 3"
        ABORT

    # 读取源信息 CSV
    s_info_path = JOIN(work_dir, "temp_s_info.csv")
    IF NOT FILE_EXISTS(s_info_path):
        REPORT "❌ 未找到源信息 CSV 文件，请先运行 Skill 3"
        ABORT

    ffprobe_csv = PARSE_CSV(v_info_path, headers=[A,B,C,D,E,F,G,H,I,J,K,...AJ])
    source_csv = PARSE_CSV(s_info_path, headers=[SourcePath,UpstreamCode,Avs2PipeModDllPath,SvfiConfigInput,SvfiTaskId])

    REPORT "✅ 读取视频信息: {v_info_filename}"
    REPORT "✅ 读取源信息: temp_s_info.csv"
```

### 第二步：隔行扫描检测

```pseudocode
FUNCTION detect_interlace(v_info_filename, ffprobe_csv):
    # 判断文件格式
    is_vob = v_info_filename CONTAINS "_vob"
    is_mov = v_info_filename CONTAINS "_mov"

    SET session.is_interlaced = false
    SET session.is_tff = false

    IF is_mov OR is_vob:
        # MOV/VOB: H 列为 field_order
        field_order = LOWERCASE(TRIM(ffprobe_csv.H))

        SWITCH field_order:
            "progressive":    is_interlaced=false, is_tff=false
            "tt", "bt":       is_interlaced=true,  is_tff=true
            "bb", "tb":       is_interlaced=true,  is_tff=false
            "unknown", "":    is_interlaced=false,  is_tff=false (WARN)
            DEFAULT:          is_interlaced=false,  is_tff=false (WARN)

    ELSE:
        # 标准格式: J=interlaced_frame, K=top_field_first
        interlaced_frame = PARSE_INT(ffprobe_csv.J, default=0)
        top_field_first = PARSE_INT(ffprobe_csv.K, default=-1)

        is_interlaced = (interlaced_frame == 1)

        IF is_interlaced:
            is_tff = (top_field_first == 1)  # 1=TFF, 0/-1=BFF
        ELSE:
            is_tff = false

    SET session.is_interlaced = is_interlaced
    SET session.is_tff = is_tff

    IF is_interlaced:
        REPORT "⚠️ 检测到隔行扫描源 (场序: {TFF if is_tff else BFF})"
```

**隔行扫描编码器参数映射（跨平台通用）：**

| 编码器 | TFF | BFF |
|--------|-----|-----|
| avs2pipemod | `-y4mt` | `-y4mb` |
| x264 | `--tff` | `--bff` |
| x265 | `--interlace 1` | `--interlace 2` |
| SVT-AV1 | 不支持 | 不支持 |

### 第三步：计算编码参数

**以下参数对所有编码器均自动计算，与平台无关。**

#### 3.1 分辨率参数

```pseudocode
FUNCTION calc_resolution(ffprobe_csv):
    w = ffprobe_csv.B
    h = ffprobe_csv.C
    SET x264_res = "--input-res {w}x{h}"
    SET x265_res = "--input-res {w}x{h}"
    SET svtav1_res = "-w {w} -h {h}"
```

#### 3.2 色彩空间 SEI

```pseudocode
FUNCTION calc_color_sei(ffprobe_csv):
    # 参考 common/color-space.md 映射表
    matrix = ffprobe_csv.E
    transfer = ffprobe_csv.F
    primaries = ffprobe_csv.G

    # x264 参数
    x264_sei = CALL map_sei_x264(matrix, transfer, primaries)
    # x265 参数
    x265_sei = CALL map_sei_x265(matrix, transfer, primaries)
    # SVT-AV1 参数（数字枚举）
    svtav1_sei = CALL map_sei_svtav1(matrix, transfer, primaries)
```

#### 3.3 帧率参数

```pseudocode
FUNCTION calc_fps(ffprobe_csv, is_vob, is_mov):
    # 参考 common/fps-utils.md
    IF is_vob OR is_mov:
        fps_string = ffprobe_csv.I   # MOV/VOB 帧率在 I 列
    ELSE:
        fps_string = ffprobe_csv.H

    SET ffmpeg_fps = "-r {fps_string}"
    SET x264_fps = "--fps {fps_string}"
    SET x265_fps = "--fps {fps_string}"

    # SVT-AV1 需要分数格式
    IF fps_string CONTAINS "/":
        num, den = SPLIT(fps_string, "/")
        SET svtav1_fps = "--fps-num {num} --fps-denom {den}"
    ELSE:
        SET svtav1_fps = "--fps {fps_string}"
```

#### 3.4 关键帧间隔（交互式）

```pseudocode
FUNCTION calc_keyint(fps_float):
    # 依次为每个编码器询问
    FOR encoder IN ["x264", "x265", "SVT-AV1"]:
        ASK_USER "请指定 {encoder} 的最大关键帧间隔秒数（正整数）：
            参考范围：[低功耗/多轨剪辑：6-7 | 一般：8-10 | 高：11-13+]
            提示：分辨率高于 2560x1440 建议偏小"

        seconds = PARSE_INT(user_input)

        IF encoder IN ["x264", "x265"]:
            keyint = ROUND(fps_float * seconds)  # 银行家舍入法
            SET {encoder}_keyint = "--keyint {keyint}"
        ELIF encoder == "SVT-AV1":
            SET svtav1_keyint = "--keyint {seconds}s"
```

#### 3.5 RC Lookahead

```pseudocode
FUNCTION calc_rc_lookahead(fps_float, bframes):
    frames = ROUND(fps_float * 1.8)
    IF frames <= bframes:
        frames = bframes + 1
    SET rc_lookahead = "--rc-lookahead {frames}"
```

#### 3.6 x265 动态搜索范围 (MERange)

```pseudocode
FUNCTION calc_merange(width, height):
    # 参考 common/presets.md 查找表
    IF width >= 3840 AND height >= 2160: merange = 56
    ELIF width >= 2560 AND height >= 1440: merange = 52
    ELIF width >= 1920 AND height >= 1080: merange = 48
    ELIF width >= 1280 AND height >= 720: merange = 40
    ELSE: merange = 36
    SET x265_merange = "--merange {merange}"
```

#### 3.7 x265 子像素搜索 (Subme)

```pseudocode
FUNCTION calc_subme(fps_float):
    IF fps_float < 25: subme = 3
    ELIF fps_float <= 48: subme = 4
    ELIF fps_float <= 60: subme = 5
    ELSE: subme = 6
    SET x265_subme = "--subme {subme}"
```

#### 3.8 x265 PME 与 NUMA 线程池（平台相关）

```pseudocode
FUNCTION calc_x265_threading(platform):
    # PME: CPU 核心数 > 36 时启用
    cpu_cores = CALL get_cpu_core_count(platform)
    IF cpu_cores > 36:
        SET x265_pme = "--pme"
    ELSE:
        SET x265_pme = ""

    # NUMA: 多 CPU 时配置线程池
    numa_count = CALL get_numa_node_count(platform)
    IF numa_count > 1:
        ASK_USER "检测到 {numa_count} 处 NUMA 节点，请指定使用一处节点（范围：0-{numa_count-1}）"
        # 生成 --pools 参数
        pools_parts = []
        FOR i IN range(numa_count):
            IF i == user_choice:
                pools_parts.APPEND("+")
            ELSE:
                pools_parts.APPEND("-")
        SET x265_pools = "--pools '{JOIN(pools_parts, ',')}'"
    ELSE:
        SET x265_pools = ""
```

#### 3.9 总帧数

```pseudocode
FUNCTION calc_frame_count(ffprobe_csv, is_vob, is_mov):
    # 尝试多个 CSV 列找到有效帧数
    IF is_vob:
        candidates = [ffprobe_csv.J]
    ELIF is_mov:
        candidates = [ffprobe_csv.J]
    ELSE:
        candidates = [ffprobe_csv.I] + [ffprobe_csv.col FOR col IN AA..AJ]

    FOR value IN candidates:
        IF value AND PARSE_INT(value) > 0:
            frame_count = PARSE_INT(value)
            BREAK

    SET x264_frames = "--frames {frame_count}"
    SET x265_frames = "--frames {frame_count}"
    SET svtav1_frames = "-n {frame_count}"
```

#### 3.10 RAW 管道 CSP

```pseudocode
FUNCTION calc_raw_csp(pix_fmt):
    # 参考 common/color-space.md
    depth = PARSE_BITDEPTH(pix_fmt)         # 8/10/12
    chroma_format = PARSE_CHROMA(pix_fmt)   # i420/i422/i444/i400

    SET x264_rawcsp = "--input-csp {chroma_format} --input-depth {depth}"
    SET x265_rawcsp = "--input-csp {chroma_format} --input-depth {depth}"

    # SVT-AV1 使用数字枚举
    svt_color_map = {i400:0, i420:1, i422:2, i444:3}
    SET svtav1_rawcsp = "--color-format {svt_color_map[chroma_format]} --input-depth {depth}"
```

### 第四步：输出文件名（交互式）

```pseudocode
FUNCTION get_output_filename(platform, source_path):
    default_name = GET_FILENAME_WITHOUT_EXT(source_path)

    # 如果是占位符脚本或源不存在
    IF is_placeholder(default_name) OR NOT FILE_EXISTS(source_path):
        default_name = "Encode_{TIMESTAMP}"

    ASK_USER "指定压制结果的文件名——[a：从文件拷贝 | b：手写 | Enter：{default_name}]"

    SWITCH user_choice:
        "a":
            ASK_USER "请提供一个文件，将使用其文件名"
            filename = GET_FILENAME_WITHOUT_EXT(user_input)
        "b":
            ASK_USER "请输入文件名"
            filename = user_input
        DEFAULT:
            filename = default_name

    # 验证文件名合法性（参见 common/platform.md validate_filename）
    IF NOT CALL validate_filename(platform, filename):
        REPORT "⚠️ 文件名包含非法字符，已自动修正"
        filename = SANITIZE_FILENAME(platform, filename)

    RETURN filename
```

### 第五步：选择编码器基础预设（交互式）

```pseudocode
FUNCTION select_encoder_presets():
    # --- x264 ---
    ASK_USER "选择 x264 自定义预设——[a：通用 | b：剪辑素材 | q：忽略]"
    IF user_choice != "q":
        x264_base = GET_PRESET("x264", user_choice)  # 参考 common/presets.md

        # FGO 支持（可选）
        ASK_USER "少数修改版 x264 支持 --fgo，输入 'y' 启用，Enter 禁用"
        IF user_choice == "y":
            x264_base += " --fgo {10 if preset==a else 15}"

    # --- x265 ---
    ASK_USER "选择 x265 自定义预设——[a：通用 | b：录像 | c：剪辑素材 | d：动漫 | e：穷举法 | q：忽略]"
    IF user_choice != "q":
        x265_base = GET_PRESET("x265", user_choice)

    # --- SVT-AV1 ---
    ASK_USER "选择 SVT-AV1 自定义预设——[a：画质优先 | b：压缩优先 | c：速度优先 | q：忽略]"
    IF user_choice != "q":
        svtav1_base = GET_PRESET("svtav1", user_choice)

        # DLF 支持（可选，非速度预设）
        IF user_choice != "c":
            ASK_USER "少数修改版 SVT-AV1 支持 --enable-dlf 2，输入 'y' 启用，Enter 使用常规"
            IF user_choice == "y":
                svtav1_base = REPLACE(svtav1_base, "--enable-dlf 1", "--enable-dlf 2")
```

### 第六步：生成 IO 参数

```pseudocode
FUNCTION generate_io_params(platform, source_path, output_dir, output_name):
    # 上游导入参数
    ffmpeg_input = '-i QUOTE(source_path)'
    vspipe_input = 'QUOTE(script_path)'
    avs2yuv_input = 'QUOTE(script_path)'
    avs2pipemod_input = 'QUOTE(script_path)'
    svfi_input = '--input QUOTE(source_path)'

    # 下游导入参数（管道输入，含隔行参数）
    interlace_x264 = IF is_interlaced: ("--tff" IF is_tff ELSE "--bff") ELSE ""
    interlace_x265 = IF is_interlaced: ("--interlace 1" IF is_tff ELSE "--interlace 2") ELSE ""

    x264_input = '- {interlace_x264}'
    x265_input = '--input - {interlace_x265}'
    svtav1_input = '-i -'

    # 下游导出参数
    x264_output = '--output QUOTE(JOIN(output_dir, output_name + ".mp4"))'
    x265_output = '--output QUOTE(JOIN(output_dir, output_name + ".hevc"))'
    svtav1_output = '-b QUOTE(JOIN(output_dir, output_name + ".ivf"))'
```

### 第七步：拼接最终参数

```pseudocode
FUNCTION assemble_final_params(platform):
    # 参数变量名（跨平台通用，语法由脚本格式决定）
    params = {
        ffmpeg_params:      "{ffmpeg_fps} {ffmpeg_input} {ffmpeg_csp} -v warning",
        vspipe_params:      "{vspipe_input}",
        avs2yuv_params:     "{avs2yuv_input} {avs2yuv_csp}",
        avs2pipemod_params: "{avs2pipemod_input} {avs2pipemod_dll}",
        svfi_params:        "{svfi_input} {svfi_config}",

        x264_params: "{x264_keyint} {x264_sei} {x264_base} {x264_output} {x264_input}",
        x265_params: "{x265_keyint} {x265_sei} {x265_rc_lookahead} {x265_merange} {x265_subme} {x265_pme} {x265_pools} {x265_base} {x265_input} {x265_output}",
        svtav1_params: "{svtav1_keyint} {svtav1_sei} {svtav1_base} {svtav1_input} {svtav1_output}"
    }

    # RAW 管道追加参数（上游为 RAW 源时）
    IF source_csv.UpstreamCode == "e":
        x264_params = "{x264_fps} {x264_rawcsp} {x264_res} {x264_frames} " + x264_params
        x265_params = "{x265_fps} {x265_rawcsp} {x265_res} {x265_frames} " + x265_params
        svtav1_params = "{svtav1_fps} {svtav1_rawcsp} {svtav1_res} {svtav1_frames} " + svtav1_params

    RETURN params
```

### 第八步：注入到模板

```pseudocode
FUNCTION inject_params_into_template(platform, params):
    # 请求用户选择模板脚本
    IF platform == "Windows":
        ASK_USER "请选择 encode_single.bat 模板文件路径"
    ELSE:
        ASK_USER "请选择 encode_single.sh 模板文件路径"

    template_content = READ_FILE(template_path)

    # 生成参数注入块
    comment = PLATFORM_COMMENT(platform)  # "REM" or "#"
    timestamp = CURRENT_TIMESTAMP()

    IF platform == "Windows":
        set_prefix = "set "
    ELSE:
        set_prefix = ""

    params_block = """
{comment} ========================================================
{comment} [自动注入] 详细编码参数（{timestamp}）
{comment} ========================================================
{set_prefix}ffmpeg_params={params.ffmpeg_params}
{set_prefix}vspipe_params={params.vspipe_params}
{set_prefix}avs2yuv_params={params.avs2yuv_params}
{set_prefix}avs2pipemod_params={params.avs2pipemod_params}
{set_prefix}svfi_params={params.svfi_params}

{set_prefix}x264_params={params.x264_params}

{set_prefix}x265_params={params.x265_params}

{set_prefix}svtav1_params={params.svtav1_params}

{comment} ========================================================
{comment} [自动注入] RAW 管道辅助参数（手动添加）
{comment} ========================================================
{comment} x264_appendix={x264_raw_appendix}
{comment} x265_appendix={x265_raw_appendix}
{comment} svtav1_appendix={svtav1_raw_appendix}

"""

    # 查找注入锚点
    IF template_content MATCHES "参数示例" OR template_content MATCHES "Parameter examples":
        new_content = REGEX_REPLACE(template_content, anchor_pattern, params_block)
    ELSE:
        REPORT "⚠️ 未找到参数占位符，将在文件头部追加参数"
        new_content = INSERT_AFTER_LINE(template_content, line=3, params_block)

    # 保存最终文件
    IF platform == "Windows":
        final_name = "encode_task_final.bat"
    ELSE:
        final_name = "encode_task_final.sh"

    final_path = JOIN(DIRNAME(template_path), final_name)
    CALL write_output_file(platform, final_path, new_content)
```

### 第九步：保存与验证

```pseudocode
FUNCTION save_and_report(platform, final_path):
    # 验证文件
    IF platform == "Windows":
        VERIFY file_has_crlf(final_path)
    ELSE:
        VERIFY file_is_executable(final_path)

    REPORT "✅ 任务脚本生成成功！"
    REPORT ""
    REPORT "使用说明："

    IF platform == "Windows":
        REPORT "1. 直接运行 encode_task_final.bat 以开始编码"
    ELSE:
        REPORT "1. 运行 ./encode_task_final.sh 以开始编码"

    REPORT "2. 只要编码工具不变，就可以保留管线模板脚本，"
    REPORT "   以便下次编码跳过步骤 2，直接在本步骤导入"
    REPORT "3. 你可以手动更改管线模板中的命令来切换上下游编码工具链"
```

---

## 输出

| 平台 | 文件 | 用途 |
|------|------|------|
| Windows | `encode_task_final.bat` | 完整的编码任务批处理 |
| macOS | `encode_task_final.sh` | 完整的编码任务脚本 |
| Linux | `encode_task_final.sh` | 完整的编码任务脚本 |

## 依赖

- Skill 2 的管线脚本
- Skill 3 的 CSV 文件
