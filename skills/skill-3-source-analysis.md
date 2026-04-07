# Skill 3: ffprobe 视频源分析

## 技能描述

使用 ffprobe 分析视频源文件，提取视频元数据（分辨率、帧率、色彩空间等），检测 VFR（可变帧率）和非方形像素问题，并导出 CSV 文件供后续步骤使用。

## 触发条件

用户提出以下意图之一：
- 分析视频源
- 读取视频信息
- 生成 ffprobe CSV
- 检测视频帧率/色彩空间

## 对话流程

### 第一步：选择上游程序类型

**向用户展示选项：**

```
选择先前脚本所用的管道上游程序（确认源符合程序要求）：
A: ffmpeg（任意源）
B: vspipe（.vpy 源）
C: avs2yuv（.avs 源）
D: avs2pipemod（.avs 源）
E: SVFI（.ini 源）
```

**记录上游代号：**
- A → 代号 `a`
- B → 代号 `b`
- C → 代号 `c`
- D → 代号 `d`
- E → 代号 `e`

### 第二步：获取视频源（根据上游类型分支）

#### 分支 A：ffmpeg

**询问用户：**
> 请提供要分析的视频源文件路径（如 .mp4/.mov/.mkv/.yuv/.y4m 等）

#### 分支 B/C/D：vspipe / avs2yuv / avs2pipemod（脚本上游）

1. **询问视频源文件：**
   > 请提供脚本引用的视频源文件路径（ffprobe 将分析此文件）

2. **询问脚本来源：**
   ```
   输入 'y' 导入自定义脚本
   输入 'n' 或 Enter 为视频源生成无滤镜脚本
   ```

3. **如果导入自定义脚本 (y)：**
   - 请求用户提供 .avs 或 .vpy 脚本文件路径
   - 验证扩展名匹配上游类型
   - 提醒用户自行检查脚本中的视频源路径

4. **如果生成无滤镜脚本 (n/Enter)：**
   - 提醒：AviSynth(+) 需要 LSMASHSource.dll
   - 生成 AVS 脚本：
     ```
     LWLibavVideoSource("视频路径") # 自动生成的占位脚本，按需修改
     ```
   - 生成 VPY 脚本：
     ```python
     import vapoursynth as vs
     core = vs.core
     src = core.lsmas.LWLibavSource(source=r"视频路径")
     # 自动生成无滤镜脚本：按需在此处加入滤镜、裁切、帧率调整等
     src.set_output()
     ```
   - 保存到 `%USERPROFILE%\bbenc\` 目录
   - 根据上游类型选择使用 AVS 或 VPY 脚本

#### 分支 D 额外步骤：avs2pipemod 需要 AviSynth.dll

**询问用户：**
> 请指定 AviSynth.dll 的路径
> 提示：从 AviSynth+ 仓库下载 filesonly.7z 获取 DLL
> https://github.com/AviSynth/AviSynthPlus/releases

#### 分支 E：SVFI

1. **自动搜索 SVFI 配置路径：**
   - 搜索 `{盘符}:\SteamLibrary\steamapps\common\SVFI\Configs\`

2. **请求 INI 配置文件路径**

3. **从 INI 文件解析视频源：**
   - 查找 `gui_inputs=` 行
   - 解析 JSON 字符串
   - 提取 `input_path` 和 `task_id`
   - 处理 Unicode 转义（`\uXXXX`）和双反斜杠
   - 验证视频文件存在

### 第三步：定位 ffprobe

1. **自动搜索：** 脚本目录、PATH 环境变量
2. **如果找到：** 直接使用
3. **如果未找到：** 请求用户提供路径

### 第四步：执行 ffprobe 分析

**执行 ffprobe 获取视频流详情：**

```bash
ffprobe -v quiet -hide_banner -select_streams v:0 \
  -show_entries stream=r_frame_rate,avg_frame_rate,nb_frames,duration,sample_aspect_ratio \
  -show_format -of json "{视频文件}"
```

**向用户报告：** 基础帧率、平均帧率、总帧数、时长、变宽比

### 第五步：VFR 检测

**参考 `common/fps-utils.md` 中的 VFR 检测逻辑。**

**评分并报告：**
- score = 0：✅ 确定是恒定帧率（CFR）
- score > 0：⚠️ 警告可能是 VFR，提供修复建议

**VFR 修复建议模板：**
```
渲染并编码为 FFV1 无损视频：
  ffmpeg -i "输入" -r {fps_num}/{fps_den} -c:v ffv1 -level 3 -context 1 -g 180 -c:a copy output.mkv

测量视频帧以确定 VFR：
  ffmpeg -i "输入" -vf vfrdet -an -f null -
```

### 第六步：非方形像素检测

如果 `sample_aspect_ratio ≠ 1:1`：
- ⚠️ 警告用户
- 提供修正方法

### 第七步：封装格式检测

**执行 ffprobe 检测真实封装格式：**

```bash
ffprobe -v quiet -hide_banner -show_format -of json "{视频文件}"
```

**格式映射逻辑：**

| format_name 匹配 | 格式 | 特殊条件 |
|------------------|------|---------|
| mpeg + dvd_nav/mpeg2video | VOB | DVD 视频 |
| mov\|mp4\|... + .mov扩展名 | MOV | — |
| mov\|mp4\|... | MP4 | — |
| matroska | MKV | — |
| webm | WebM | — |
| avi | AVI | — |
| hevc | HEVC裸流 | — |
| h264\|avc | AVC裸流 | — |

### 第八步：导出 CSV

**生成两个 CSV 文件到 `%USERPROFILE%\bbenc\` 目录：**

#### 视频信息 CSV（temp_v_info.csv / temp_v_info_is_mov.csv / temp_v_info_is_vob.csv）

**ffprobe 命令（根据格式不同）：**

标准格式：
```bash
ffprobe -i "{视频}" -select_streams v:0 -v error -hide_banner -show_streams \
  -show_entries stream=width,height,pix_fmt,color_space,color_transfer,color_primaries,avg_frame_rate,nb_frames,interlaced_frame,top_field_first:stream_tags=NUMBER_OF_FRAMES,NUMBER_OF_FRAMES-eng \
  -of csv
```

MOV/VOB 格式（包含 field_order）：
```bash
ffprobe -i "{视频}" -select_streams v:0 -v error -hide_banner -show_streams \
  -show_entries stream=width,height,pix_fmt,color_space,color_transfer,color_primaries,field_order,avg_frame_rate,nb_frames \
  -of csv
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

#### 源信息 CSV（temp_s_info.csv）

格式：
```
"源路径",上游代号,"Avs2PipeMod DLL路径","SVFI INI路径","SVFI TaskId"
```

## 输出

| 文件 | 位置 | 用途 |
|------|------|------|
| temp_v_info.csv | `%USERPROFILE%\bbenc\` | 视频元数据（标准格式） |
| temp_v_info_is_mov.csv | `%USERPROFILE%\bbenc\` | 视频元数据（MOV 格式） |
| temp_v_info_is_vob.csv | `%USERPROFILE%\bbenc\` | 视频元数据（VOB 格式） |
| temp_s_info.csv | `%USERPROFILE%\bbenc\` | 源文件信息 |
| blank_avs_script.avs | `%USERPROFILE%\bbenc\` | 无滤镜 AVS 脚本（可选） |
| blank_vs_script.vpy | `%USERPROFILE%\bbenc\` | 无滤镜 VPY 脚本（可选） |

## 依赖

- ffprobe 可执行文件（来自 ffmpeg 工具包）
