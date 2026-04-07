# Skill 4: 编码任务批处理生成器

## 技能描述

读取 Skill 2 生成的管线模板和 Skill 3 生成的 CSV 元数据，计算视频编码参数（关键帧间隔、色彩空间、动态搜索范围等），注入到批处理模板中，生成最终的编码任务批处理文件。

## 触发条件

用户提出以下意图之一：
- 生成编码任务
- 计算编码参数
- 创建编码批处理
- 注入编码参数到模板

## 前置要求

- `encode_single.bat`（来自 Skill 2）
- `temp_v_info*.csv`（来自 Skill 3）
- `temp_s_info.csv`（来自 Skill 3）

## 对话流程

### 第一步：自动读取 CSV 文件

**LLM 执行：**

1. 在 `%USERPROFILE%\bbenc\` 目录查找 `temp_v_info*.csv` 文件，按**文件修改时间**（LastWriteTime）降序排列，取最新的一个
2. 读取 `temp_s_info.csv`

**CSV 列映射：**

视频信息 CSV (`ffprobeCSV`)：
```
A=标记, B=width, C=height, D=pix_fmt, E=color_space, F=color_transfer, G=color_primaries
标准格式: H=avg_frame_rate, I=nb_frames, J=interlaced_frame, K=top_field_first, AA-AJ=tag帧数
MOV/VOB:  H=field_order, I=avg_frame_rate, J=nb_frames
```

源信息 CSV (`sourceCSV`)：
```
SourcePath, UpstreamCode, Avs2PipeModDllPath, SvfiConfigInput, SvfiTaskId
```

### 第二步：隔行扫描检测

**判断文件格式：**
- 文件名含 `_vob` → VOB 格式
- 文件名含 `_mov` → MOV 格式

**解析隔行扫描信息：**

MOV/VOB 格式（从 field_order，即 H 列）：
| field_order | 隔行 | 场序 |
|-------------|------|------|
| progressive | 否 | — |
| tt / bt | 是 | TFF（上场优先） |
| bb / tb | 是 | BFF（下场优先） |
| unknown / 空 | 否 | — |

标准格式（从 interlaced_frame 和 top_field_first）：
- `interlaced_frame = 1` → 隔行扫描
- `top_field_first = 1` → TFF，`0/-1` → BFF

**隔行扫描编码器参数：**
| 编码器 | TFF | BFF |
|--------|-----|-----|
| avs2pipemod | `-y4mt` | `-y4mb` |
| x264 | `--tff` | `--bff` |
| x265 | `--interlace 1` | `--interlace 2` |
| SVT-AV1 | 不支持 | 不支持 |

### 第三步：计算编码参数

**以下参数对所有编码器均自动计算，用户无需逐一选择。**

#### 3.1 分辨率参数
- x264/x265: `--input-res {width}x{height}`
- SVT-AV1: `-w {width} -h {height}`

#### 3.2 色彩空间 SEI

参考 `common/color-space.md`，根据 CSV 中的 `color_space`、`color_transfer`、`color_primaries` 生成编码器参数。

#### 3.3 帧率参数

参考 `common/fps-utils.md`，根据 CSV 中的 `avg_frame_rate` 生成：
- ffmpeg: `-r {fps}`
- x264/x265: `--fps {fps}`
- SVT-AV1: `--fps-num {分子} --fps-denom {分母}`

#### 3.4 关键帧间隔（交互式）

**依次为 x264、x265、SVT-AV1 询问：**

```
请指定 {编码器} 的最大关键帧间隔秒数（正整数）：
1. 分辨率高于 2560x1440 则偏左选一格
2. 画面内容简单，平面居多则偏右选一格
大致范围：[低功耗/多轨剪辑：6-7 | 一般（不确定则用）：8-10 | 高：11-13+]
```

计算：
- x264/x265: `--keyint {round(fps × 秒数)}`
- SVT-AV1: `--keyint {秒数}s`

#### 3.5 RC Lookahead
- 公式: `round(fps × 1.8)`
- 必须大于 bframes 值
- 参数: `--rc-lookahead {帧数}`

#### 3.6 x265 动态搜索范围 (MERange)
参考 `common/presets.md`，根据分辨率自动选择。

#### 3.7 x265 子像素搜索 (Subme)
参考 `common/presets.md`，根据帧率自动选择。

#### 3.8 x265 并行动态搜索 (PME)
如果 CPU 核心数 > 36，添加 `--pme`。

#### 3.9 x265 NUMA 线程池
如果检测到多颗 CPU（多 NUMA 节点），询问使用哪个节点：
```
检测到 {N} 处 NUMA 节点，请指定使用一处节点（范围：0-{N-1}）
```
生成 `--pools` 参数。

#### 3.10 总帧数
- 遍历 CSV 中可能包含帧数的列（I, AA-AJ，VOB 为 J）
- 找到首个 > 0 的值
- x264/x265: `--frames {帧数}`
- SVT-AV1: `-n {帧数}`

#### 3.11 RAW 管道 CSP
参考 `common/color-space.md`，根据像素格式生成 `--input-csp` 和 `--input-depth`。

#### 3.12 avs2yuv 版本选择（仅上游为 avs2yuv 时）
```
选择使用的 avs2yuv(64).exe 类型：
[默认 Enter/a: AviSynth+ (0.30) | b: AviSynth (up to 0.26)]
```

### 第四步：输出文件名（交互式）

```
指定压制结果的文件名——[a：从文件拷贝 | b：手写 | Enter：{默认名}]
```

**默认名规则：**
- 如果源文件名有效：使用源文件名（去扩展名）
- 如果是占位符脚本或源不存在：使用 `Encode {时间戳}`

**选项 a：** 选择一个文件，使用其文件名（去扩展名）
**选项 b：** 手动输入文件名
**Enter：** 使用默认名

验证文件名合法性（不含 Windows 非法字符）。

### 第五步：选择编码器基础预设（交互式）

依次为 x264、x265、SVT-AV1 选择预设。

#### x264 预设选择
```
选择 x264 自定义预设——[a：通用 | b：剪辑素材]
指定一份 x264 自定义预设，输入 'q' 忽略（沿用编码器内置默认）：
```

如果选择了预设，额外询问 FGO：
```
少数修改版 x264 支持 --fgo（Film Grain Optimization）
输入 'y' 以启用 --fgo，或 Enter 以禁用：
```

#### x265 预设选择
```
选择 x265 自定义预设——[a：通用 | b：录像 | c：剪辑素材 | d：动漫 | e：穷举法]
指定一份 x265 自定义预设，输入 'q' 忽略：
```

#### SVT-AV1 预设选择
```
选择 SVT-AV1 自定义预设——[a：画质优先 | b：压缩优先 | c：速度优先]
指定一份 SVT-AV1 自定义预设，输入 'q' 忽略：
```

如果选择了非 b 预设，额外询问 DLF：
```
少数修改版 SVT-AV1 支持 --enable-dlf 2
输入 'y' 以启用 --enable-dlf 2，或 Enter 使用常规去块滤镜：
```

### 第六步：生成 IO 参数

**上游导入参数：**
| 上游 | 格式 |
|------|------|
| ffmpeg | `-i "源路径"` |
| vspipe | `"脚本路径"` (+ 隔行参数) |
| avs2yuv | `"脚本路径"` (+ 隔行参数) |
| avs2pipemod | `"脚本路径"` (+ 隔行参数) |
| svfi | `--input "源路径"` |

**下游导入参数（管道输入）：**
| 编码器 | 格式 |
|--------|------|
| x264 | `-` (+ 隔行参数) |
| x265 | `--input -` (+ 隔行参数) |
| SVT-AV1 | `-i -` |

**下游导出参数：**
| 编码器 | 格式 | 默认扩展名 |
|--------|------|-----------|
| x264 | `--output "路径.mp4"` | .mp4 |
| x265 | `--output "路径.hevc"` | .hevc |
| SVT-AV1 | `-b "路径.ivf"` | .ivf |

### 第七步：拼接最终参数

**最终参数字符串模板：**

```
ffmpeg_params = {FPS} {Input} {CSP} {LogLevel}
vspipe_params = {Input}
avs2yuv_params = {Input} {CSP}
avs2pipemod_params = {Input} {DLLInput}
svfi_params = {Input} {ConfigInput}

x264_params = {Keyint} {SEICSP} {BaseParam} {Output} {Input}
x265_params = {Keyint} {SEICSP} {RCLookahead} {MERange} {Subme} {PME} {Pools} {BaseParam} {Input} {Output}
svtav1_params = {Keyint} {SEICSP} {BaseParam} {Input} {Output}
```

**RAW 管道源（上游代号 e）额外追加：**
```
x264追加 = {FPS} {RAWCSP} {Resolution} {TotalFrames}
x265追加 = {FPS} {RAWCSP} {Resolution} {TotalFrames}
svtav1追加 = {FPS} {RAWCSP} {Resolution} {TotalFrames}
```

### 第八步：注入到模板

1. **请求用户选择 encode_single.bat 模板文件**
2. **读取模板内容**
3. **查找注入锚点：**
   - 英文: `REM Parameter examples`
   - 中文: `REM 参数示例`
4. **替换锚点区域为参数块：**
   ```batch
   REM ========================================================
   REM [自动注入] 详细编码参数（{时间戳}）
   REM ========================================================
   set ffmpeg_params={...}
   set vspipe_params={...}
   set avs2yuv_params={...}
   set avs2pipemod_params={...}
   set svfi_params={...}

   set x264_params={...}
   set x265_params={...}
   set svtav1_params={...}

   REM ========================================================
   REM [自动注入] RAW 管道辅助参数（手动添加）
   REM ========================================================
   REM x264_appendix={...}
   REM x265_appendix={...}
   REM svtav1_appendix={...}
   ```
5. **保存为 `encode_task_final.bat`**（与模板同目录）

### 第九步：保存与验证

1. UTF-8 BOM 编码、CRLF 换行符写入
2. 验证文件格式
3. 显示使用说明：
   ```
   1. 直接运行该 encode_task_final.bat 以开始编码。
   2. 只要编码工具不变，就可以保留 encode_single.bat，
      以便下次编码跳过步骤 2，直接在本步骤导入 encode_single.bat
   3. 你可以手动更改 encode_single.bat 中的命令来切换上下游编码工具链
   ```

## 输出

| 文件 | 位置 | 用途 |
|------|------|------|
| encode_task_final.bat | 与模板同目录 | 完整的编码任务批处理 |

## 依赖

- Skill 2 的 `encode_single.bat`
- Skill 3 的 CSV 文件
