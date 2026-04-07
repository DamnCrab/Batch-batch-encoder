# 色彩空间参考数据

## ffmpeg 像素格式 (pix_fmt)

支持的像素格式列表:

| 格式 | 色度采样 | 位深 |
|------|---------|------|
| yuv420p | 4:2:0 | 8 |
| yuv420p10le | 4:2:0 | 10 |
| yuv420p12le | 4:2:0 | 12 |
| yuv422p | 4:2:2 | 8 |
| yuv422p10le | 4:2:2 | 10 |
| yuv422p12le | 4:2:2 | 12 |
| yuv444p | 4:4:4 | 8 |
| yuv444p10le | 4:4:4 | 10 |
| yuv444p12le | 4:4:4 | 12 |
| gray | 灰度 | 8 |
| gray10le | 灰度 | 10 |
| gray12le | 灰度 | 12 |
| nv12 | 4:2:0 (2平面) | 8 |
| nv16 | 4:2:2 (2平面) | 8 |

## RAW 管道 CSP 映射

### 编码器输入 (x264/x265)

| pix_fmt 前缀 | --input-csp | --input-depth |
|-------------|-------------|---------------|
| yuv420 | i420 | 从格式解析 |
| yuv422 | i422 | 从格式解析 |
| yuv444 | i444 | 从格式解析 |
| gray / yuv400 | i400 | 从格式解析 |
| nv12 | i420 | 8 |
| nv16 | i422 | 8 |

### SVT-AV1 输入

| pix_fmt 前缀 | --color-format | --input-depth |
|-------------|---------------|---------------|
| yuv420 / nv12 | 1 | 从格式解析 |
| yuv422 / nv16 | 2 | 从格式解析 |
| yuv444 | 3 | 从格式解析 |
| gray / i400 | 0 | 从格式解析 |

### avs2yuv 输入

- AviSynth+ (0.30): `-depth {位深}` (无 -csp 参数)
- AviSynth (≤0.26): `-csp {格式} -depth {位深}`

## 位深解析规则

从像素格式字符串中提取位深:
- 匹配末尾数字+le/be: `yuv420p10le` → 10
- 无后缀数字: 默认 8 bit
- 仅支持 8、10、12 bit

## ColorMatrix / Transfer / Primaries SEI 映射

### x264 (AVC) 参数名

| 元数据类型 | 参数 | 特殊处理 |
|-----------|------|---------|
| ColorMatrix | `--colormatrix` | "unknown" → "undef"，"bt2020nc" → "undef" |
| Transfer | `--transfer` | "unknown" → "undef" |
| Primaries | `--colorprim` | "unknown"/"unspec" → "undef" |

### x265 (HEVC) 参数名

| 元数据类型 | 参数 | 特殊处理 |
|-----------|------|---------|
| ColorMatrix | `--colormatrix` | "bt2020nc" → "unknown" |
| Transfer | `--transfer` | 直接传递 |
| Primaries | `--colorprim` | "unknown"/"unspec" → "unknown" |

### SVT-AV1 参数名（使用数字枚举）

**ColorMatrix (`--matrix-coefficients`)**

| 值 | 数字 |
|----|------|
| identity | 0 |
| bt709 | 1 |
| unspec | 2 |
| fcc | 4 |
| bt470bg | 5 |
| bt601 | 6 |
| smpte240m | 7 |
| ycgco | 8 |
| bt2020-ncl | 9 |
| bt2020-cl | 10 |
| smpte2085 | 11 |
| chroma-ncl | 12 |
| chroma-cl | 13 |
| ictcp | 14 |

**Transfer (`--transfer-characteristics`)**

| 值 | 数字 |
|----|------|
| bt709 | 1 |
| unspec | 2 |
| bt470m | 4 |
| bt470bg | 5 |
| bt601 | 6 |
| smpte240m | 7 |
| linear | 8 |
| log100 | 9 |
| log100-sqrt10 | 10 |
| iec61966-2-4 | 11 |
| iec61966-2-1 | 13 |
| bt2020-10 | 14 |
| bt2020-12 | 15 |
| smpte2084 | 16 |
| smpte428 | 17 |
| hlg | 18 |

**Primaries (`--color-primaries`)**

| 值 | 数字 |
|----|------|
| bt709 | 1 |
| unspec / unknown | 2 |
| bt470m | 4 |
| bt470bg | 5 |
| bt601 | 6 |
| smpte240m | 7 |
| film | 8 |
| bt2020 | 9 |
| xyz | 10 |
| smpte431 | 11 |
| smpte432 | 12 |
| ebu3213 | 22 |
