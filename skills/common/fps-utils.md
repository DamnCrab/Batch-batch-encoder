# 帧率处理参考数据

## 帧率字符串格式

ffprobe 返回的帧率格式:
- 分数格式: `24000/1001`、`30000/1001`、`60000/1001`
- 整数格式: `24`、`25`、`30`、`60`

## 帧率解析规则

1. 如果是分数 `num/den`，计算 `num ÷ den` 得到浮点值
2. 如果是整数，直接使用
3. 如果是小数字符串，直接转换

## 常用帧率预设

| 序号 | 帧率 | 分数格式 |
|------|------|---------|
| 1 | 23.976 | 24000/1001 |
| 2 | 24 | 24/1 |
| 3 | 25 | 25/1 |
| 4 | 29.97 | 30000/1001 |
| 5 | 30 | 30/1 |
| 6 | 48 | 48/1 |
| 7 | 50 | 50/1 |
| 8 | 59.94 | 60000/1001 |
| 9 | 60 | 60/1 |
| 10 | 120 | 120/1 |
| 11 | 144 | 144/1 |

## 编码器帧率参数格式

### ffmpeg
```
-r {fps_string}
```

### x264 / x265
```
--fps {fps_string}
```

### SVT-AV1
- 分数格式: `--fps-num {分子} --fps-denom {分母}`
- 常用小数映射:
  - 23.976 → `--fps-num 24000 --fps-denom 1001`
  - 29.97 → `--fps-num 30000 --fps-denom 1001`
  - 59.94 → `--fps-num 60000 --fps-denom 1001`
- 其他: `--fps {整数}`

## VFR (可变帧率) 检测逻辑

评分系统（score > 0 则警告）:

| 检测项 | 分数 | 条件 |
|--------|------|------|
| 基础帧率 ≠ 平均帧率 | +1 | `abs(r_fps - a_fps) / max(r_fps, a_fps) > 1e-9` |
| 估计帧率 ≠ 平均帧率 | +2 | `abs(e_fps - a_fps) / max(e_fps, a_fps) > 1e-9`，其中 `e_fps = nb_frames / duration` |
| 特殊 r_frame_rate | +3 | `r_frame_rate == "90000/1"` |
| 大分母且接近质数 | +2 | 平均帧率分母 > 50000 且整除数 ≤ 5 |
| 大分母但非质数 | +1 | 平均帧率分母 > 50000 但整除数 > 5 |

判定等级:
- score ≥ 5: 确定是 VFR
- score ≥ 4: 大概率是 VFR
- score ≥ 2: 应该是 VFR
- score > 0: 有迹象是 VFR
- score = 0: 确定是 CFR

VFR 修复建议:
```
ffmpeg -i "input" -r {r_fps_num}/{r_fps_den} -c:v ffv1 -level 3 -context 1 -g 180 -c:a copy output.mkv
```

VFR 测量命令:
```
ffmpeg -i "input" -vf vfrdet -an -f null -
```

## 非方形像素 (SAR) 检测

如果 `sample_aspect_ratio ≠ 1:1`，警告用户并提供修正方法:
```
ffmpeg -i "input" -c copy -aspect {SAR} output.mkv
```
