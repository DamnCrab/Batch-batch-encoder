# 编码器预设与工具链参考数据

## 工具链组合（Pipe Presets）

共 15 种上游×下游组合：

| ID | Preset 名称 | 上游 (Upstream) | 下游 (Downstream) | 管道格式 |
|----|------------|----------------|-------------------|---------|
| 1  | ffmpeg_x264 | ffmpeg | x264 | Y4M |
| 2  | ffmpeg_x265 | ffmpeg | x265 | Y4M |
| 3  | ffmpeg_svtav1 | ffmpeg | svtav1 | Y4M |
| 4  | vspipe_x264 | vspipe | x264 | Y4M |
| 5  | vspipe_x265 | vspipe | x265 | Y4M |
| 6  | vspipe_svtav1 | vspipe | svtav1 | Y4M |
| 7  | avs2yuv_x264 | avs2yuv | x264 | Y4M |
| 8  | avs2yuv_x265 | avs2yuv | x265 | Y4M |
| 9  | avs2yuv_svtav1 | avs2yuv | svtav1 | Y4M |
| 10 | avs2pipemod_x264 | avs2pipemod | x264 | Y4M |
| 11 | avs2pipemod_x265 | avs2pipemod | x265 | Y4M |
| 12 | avs2pipemod_svtav1 | avs2pipemod | svtav1 | Y4M |
| 13 | svfi_x264 | svfi | x264 | Y4M |
| 14 | svfi_x265 | svfi | x265 | Y4M |
| 15 | svfi_svtav1 | svfi | svtav1 | Y4M |

## 管道命令模板

### 下游管道参数

```
Y4M 管道:
  x264:   --demuxer y4m
  x265:   --y4m
  svtav1: (无需额外参数)

RAW 管道:
  x264:   --demuxer raw
  x265:   (无需额外参数)
  svtav1: (无需额外参数)
```

### 上游命令模板

```
ffmpeg:      "{upstream}" %ffmpeg_params% -f yuv4mpegpipe -an -strict unofficial - | "{downstream}" {pipe_arg} %{encoder}_params%
vspipe:      "{upstream}" %vspipe_params% {y4m_arg} - | "{downstream}" {pipe_arg} %{encoder}_params%
avs2yuv:     "{upstream}" %avs2yuv_params% - | "{downstream}" {pipe_arg} %{encoder}_params%
avs2pipemod: "{upstream}" %avs2pipemod_params% -y4mp | "{downstream}" {pipe_arg} %{encoder}_params%
svfi:        "{upstream}" %svfi_params% --pipe-out | "{downstream}" {pipe_arg} %{encoder}_params%
```

## x264 编码器预设

### a: 通用 (General Purpose)
```
--bframes 14 --b-adapt 2 --me umh --subme 9 --merange 48 --no-fast-pskip --direct auto --weightb --min-keyint 5 --ref 3 --crf 18 --chroma-qp-offset -2 --aq-mode 3 --aq-strength 0.7 --trellis 2 --deblock 0:0 --psy-rd 0.77:0.22
```
> 隔行扫描源额外添加 `--weightp 0`

### b: 剪辑素材 (Stock Footage)
```
--partitions all --bframes 12 --b-adapt 2 --me esa --merange 48 --no-fast-pskip --direct auto --weightb --min-keyint 1 --ref 3 --crf 16 --tune grain --trellis 2
```

### 可选: --fgo 参数
部分修改版（Mod）x264 支持 `--fgo`（Film Grain Optimization）:
- 通用预设: `--fgo 10`
- 素材预设: `--fgo 15`
- 检测方法: `x264.exe --fullhelp | findstr fgo`

## x265 编码器预设

### a: 通用 (General Purpose)
```
--high-tier --preset slow --me umh --weightb --aq-mode 4 --bframes 5 --ref 3
```

### b: 录像 (Movie)
```
--high-tier --ctu 64 --tu-intra-depth 4 --tu-inter-depth 4 --limit-tu 1 --rect --tskip --tskip-fast --me star --weightb --ref 4 --max-merge 5 --no-open-gop --min-keyint 3 --fades --bframes 8 --b-adapt 2 --b-intra --crf 21.8 --crqpoffs -3 --ipratio 1.2 --pbratio 1.5 --rdoq-level 2 --aq-mode 4 --aq-strength 1.1 --qg-size 8 --rd 5 --limit-refs 0 --rskip 0 --deblock 0:-1 --limit-sao --sao-non-deblock --selective-sao 3
```

### c: 剪辑素材 (Stock Footage)
```
--high-tier --ctu 32 --tskip --me star --max-merge 5 --early-skip --b-intra --no-open-gop --min-keyint 1 --ref 3 --fades --bframes 7 --b-adapt 2 --crf 17 --crqpoffs -3 --cbqpoffs -2 --rd 3 --limit-modes --limit-refs 1 --rskip 1 --splitrd-skip --deblock -1:-1 --tune grain
```

### d: 动漫 (Anime)
```
--high-tier --tu-intra-depth 4 --tu-inter-depth 4 --max-tu-size 16 --tskip --tskip-fast --me umh --weightb --max-merge 5 --early-skip --ref 3 --no-open-gop --min-keyint 5 --fades --bframes 16 --b-adapt 2 --bframe-bias 20 --constrained-intra --b-intra --crf 22 --crqpoffs -4 --cbqpoffs -2 --ipratio 1.6 --pbratio 1.3 --cu-lossless --psy-rdoq 2.3 --rdoq-level 2 --hevc-aq --aq-strength 0.9 --qg-size 8 --rd 3 --limit-modes --limit-refs 1 --rskip 1 --rect --amp --psy-rd 1.5 --splitrd-skip --rdpenalty 2 --deblock -1:0 --limit-sao --sao-non-deblock
```

### e: 穷举法 (Exhaustive)
```
--high-tier --tu-intra-depth 4 --tu-inter-depth 4 --max-tu-size 4 --limit-tu 1 --rect --amp --tskip --me star --weightb --max-merge 5 --ref 3 --no-open-gop --min-keyint 1 --fades --bframes 16 --b-adapt 2 --b-intra --crf 18.1 --crqpoffs -5 --cbqpoffs -2 --ipratio 1.67 --pbratio 1.33 --cu-lossless --psy-rdoq 2.5 --rdoq-level 2 --hevc-aq --aq-strength 1.4 --qg-size 8 --rd 5 --limit-refs 0 --rskip 2 --rskip-edge-threshold 3 --no-cutree --psy-rd 1.5 --rdpenalty 2 --deblock -2:-2 --limit-sao --sao-non-deblock --selective-sao 1
```

## SVT-AV1 编码器预设

### a: 画质优先 (Quality)
```
--preset 2 --scd 1 --enable-tf 2 --tf-strength 2 --crf 30 --enable-qm 1 --enable-variance-boost 1 --variance-boost-curve 2 --variance-boost-strength 2 --variance-octile 2 --sharpness 6 --progress 1 --enable-dlf {1或2}
```

### b: 压缩优先 (Compression)
```
--preset 2 --scd 1 --enable-tf 2 --tf-strength 2 --crf 30 --sharpness 4 --progress 1 --enable-dlf {1或2}
```

### c: 速度优先 (Speed)
```
--preset 2 --scd 1 --scm 0 --enable-tf 2 --tf-strength 2 --crf 30 --tune 0 --enable-variance-boost 1 --variance-boost-curve 2 --variance-boost-strength 2 --variance-octile 2 --sharpness 4 --progress 1
```
> 速度预设不支持 `--enable-dlf 2`

### 可选: --enable-dlf 2
部分修改版 SVT-AV1（如 SVT-AV1-Essential）支持高精度去块滤镜:
- 检测方法: `SvtAv1EncApp.exe --help | findstr enable-dlf`

## x265 动态搜索范围 (MERange)

根据视频分辨率自动计算：

| 分辨率 | MERange |
|--------|---------|
| ≥ 3840×2160 | 56 |
| ≥ 2560×1440 | 52 |
| ≥ 1920×1080 | 48 |
| ≥ 1280×720 | 40 |
| < 1280×720 | 36 |

## x265 子像素搜索 (Subme)

根据帧率自动计算：

| 帧率 | Subme |
|------|-------|
| < 25 fps | 3 |
| 25–48 fps | 4 |
| 49–60 fps | 5 |
| > 60 fps | 6 |

## 关键帧间隔 (Keyint) 计算

- 公式: `keyint = round(fps × 用户指定秒数)`
- 用户选择范围建议:
  - 低功耗/多轨剪辑: 6–7 秒
  - 一般（默认）: 8–10 秒
  - 高: 11–13+ 秒
- 高分辨率（>2560×1440）建议偏小
- 画面简单/平面居多建议偏大
- SVT-AV1 使用 `--keyint {秒}s` 格式
- x264/x265 使用 `--keyint {帧数}` 格式

## 率控制前瞻 (RC Lookahead)

- 公式: `frames = round(fps × 1.8)`
- 必须大于 `--bframes` 值
- 参数: `--rc-lookahead {frames}`
