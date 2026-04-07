# 平台抽象层（Platform Abstraction Layer）

本文件定义了 LLM 在不同操作系统/基座平台上执行 Skill 时的约束规则。
LLM 必须在**每次会话开始时**检测平台，并在后续所有代码生成中遵循对应规则。

---

## 1. 平台检测（DETECT_PLATFORM）

```pseudocode
FUNCTION detect_platform():
    ASK_USER "请确认你的操作系统：
        A: Windows
        B: macOS
        C: Linux
        D: 自动检测"

    IF user_choice == "D":
        EXECUTE shell_command to detect OS
        # Windows:  检测 $env:OS 或 systeminfo
        # macOS:    检测 uname -s == "Darwin"
        # Linux:    检测 uname -s == "Linux"

    SET session.platform = detected_or_selected_platform
    SET session.shell = PLATFORM_SHELL_MAP[session.platform]
    SET session.script_ext = PLATFORM_SCRIPT_MAP[session.platform]
    SET session.path_sep = PLATFORM_PATH_SEP[session.platform]
    SET session.quote_char = PLATFORM_QUOTE[session.platform]

    RETURN session.platform
```

---

## 2. 平台常量映射表

### 2.1 Shell 与脚本格式

| 平台 | 默认 Shell | 脚本扩展名 | 脚本首行 | 编码 |
|------|-----------|-----------|---------|------|
| Windows | PowerShell / cmd.exe | `.bat` / `.ps1` | `@echo off` / 无 | UTF-8 BOM + CRLF |
| macOS | zsh | `.sh` | `#!/usr/bin/env zsh` | UTF-8 (无BOM) + LF |
| Linux | bash | `.sh` | `#!/usr/bin/env bash` | UTF-8 (无BOM) + LF |

### 2.2 路径规范

| 平台 | 路径分隔符 | 引用方式 | 用户主目录 | 临时目录 |
|------|-----------|---------|-----------|---------|
| Windows | `\` | `"path"` | `%USERPROFILE%` / `$env:USERPROFILE` | `%TEMP%` |
| macOS | `/` | `"path"` 或 `'path'` | `$HOME` | `$TMPDIR` 或 `/tmp` |
| Linux | `/` | `"path"` 或 `'path'` | `$HOME` | `/tmp` |

### 2.3 可执行文件名称

| 工具 | Windows | macOS / Linux |
|------|---------|---------------|
| ffmpeg | `ffmpeg.exe` | `ffmpeg` |
| ffprobe | `ffprobe.exe` | `ffprobe` |
| x264 | `x264.exe` | `x264` |
| x265 | `x265.exe` | `x265` |
| SVT-AV1 | `SvtAv1EncApp.exe` | `SvtAv1EncApp` |
| vspipe | `vspipe.exe` | `vspipe` |
| avs2yuv | `avs2yuv.exe` | `avs2yuv`（Linux 极少见） |
| avs2pipemod | `avs2pipemod.exe` | `avs2pipemod`（Linux 极少见） |
| SVFI | `one_line_shot_args.exe` | 不支持 |

### 2.4 工具搜索路径（自动发现）

```pseudocode
FUNCTION get_search_paths(platform, tool_name):
    common_paths = [working_directory, PATH_environment_variable]

    IF platform == "Windows":
        SWITCH tool_name:
            "vspipe":       APPEND "C:\Program Files\VapourSynth\core\"
            "svfi":
                # 遍历所有盘符 C-Z 查找 Steam 安装
                FOR drive IN ['C'..'Z']:
                    APPEND "{drive}:\SteamLibrary\steamapps\common\SVFI\"
            "ffmpeg":       APPEND "C:\ffmpeg\bin\", "C:\Program Files\ffmpeg\bin\"
        
    ELIF platform == "macOS":
        SWITCH tool_name:
            "ffmpeg":       APPEND "/opt/homebrew/bin/", "/usr/local/bin/"
            "vspipe":       APPEND "/opt/homebrew/lib/vapoursynth/"
            "x264","x265":  APPEND "/opt/homebrew/bin/"
            "SvtAv1EncApp": APPEND "/opt/homebrew/bin/", "/usr/local/bin/"
        # macOS 特有：检测 Homebrew 安装路径

    ELIF platform == "Linux":
        SWITCH tool_name:
            "ffmpeg":       APPEND "/usr/bin/", "/usr/local/bin/", "/snap/bin/"
            "vspipe":       APPEND "/usr/lib/vapoursynth/", "/usr/local/lib/vapoursynth/"
            "x264","x265":  APPEND "/usr/bin/", "/usr/local/bin/"
            "SvtAv1EncApp": APPEND "/usr/bin/", "/usr/local/bin/"
        # Linux 特有：检测包管理器安装

    RETURN common_paths
```

### 2.5 平台特有功能可用性

| 功能 | Windows | macOS | Linux |
|------|---------|-------|-------|
| UAC 管理 | ✅ 可用 | ❌ 不适用 | ❌ 不适用 |
| 注册表操作 | ✅ 可用 | ❌ 不适用 | ❌ 不适用 |
| WMI/CIM 硬件查询 | ✅ 可用 | ❌ 不适用 | ❌ 不适用 |
| sysctl 硬件查询 | ❌ 不适用 | ✅ 可用 | ❌ 不适用 |
| /proc 文件系统 | ❌ 不适用 | ❌ 不适用 | ✅ 可用 |
| AviSynth / avs2pipemod | ✅ 原生支持 | ⚠️ Wine | ⚠️ Wine |
| SVFI | ✅ 仅 Windows | ❌ 不支持 | ❌ 不支持 |
| 文件对话框 (GUI) | ✅ WinForms | ⚠️ 需 osascript 或 zenity | ⚠️ 需 zenity/kdialog |

---

## 3. 代码生成规则（CODE_GENERATION_RULES）

LLM 在将伪代码转换为实际可执行代码时，**必须**遵守以下规则：

### 3.1 脚本文件生成

```pseudocode
FUNCTION generate_script(platform, content):
    IF platform == "Windows":
        script = "@echo off\nchcp 65001 >nul\nsetlocal\n\n"
        script += content
        script += "\nendlocal\ncmd /k"
        WRITE script TO file WITH encoding="UTF-8 BOM", line_ending="CRLF"
        SET file_extension = ".bat"

    ELIF platform == "macOS":
        script = "#!/usr/bin/env zsh\nset -euo pipefail\n\n"
        script += content
        WRITE script TO file WITH encoding="UTF-8", line_ending="LF"
        EXECUTE "chmod +x {file_path}"
        SET file_extension = ".sh"

    ELIF platform == "Linux":
        script = "#!/usr/bin/env bash\nset -euo pipefail\n\n"
        script += content
        WRITE script TO file WITH encoding="UTF-8", line_ending="LF"
        EXECUTE "chmod +x {file_path}"
        SET file_extension = ".sh"
```

### 3.2 管道命令生成

```pseudocode
FUNCTION generate_pipe_command(platform, upstream_cmd, downstream_cmd):
    # 管道符 "|" 在所有平台通用
    # 区别在于路径引用和变量语法

    IF platform == "Windows":
        # 使用 %variable% (cmd) 或 $variable (PowerShell)
        RETURN '{upstream_cmd} | {downstream_cmd}'

    ELIF platform IN ["macOS", "Linux"]:
        # 使用 $variable 或 ${variable}
        RETURN '{upstream_cmd} | {downstream_cmd}'
```

### 3.3 变量定义语法

| 用途 | Windows (cmd) | Windows (PS) | macOS/Linux (sh) |
|------|--------------|-------------|-----------------|
| 定义变量 | `set VAR=value` | `$VAR = "value"` | `VAR="value"` |
| 引用变量 | `%VAR%` | `$VAR` | `$VAR` 或 `${VAR}` |
| 环境变量 | `%PATH%` | `$env:PATH` | `$PATH` |

### 3.4 系统信息查询

```pseudocode
FUNCTION get_cpu_info(platform):
    IF platform == "Windows":
        EXECUTE "Get-CimInstance Win32_Processor | Select Name, NumberOfCores, NumberOfLogicalProcessors"
    ELIF platform == "macOS":
        EXECUTE "sysctl -n machdep.cpu.brand_string"
        EXECUTE "sysctl -n hw.physicalcpu"
        EXECUTE "sysctl -n hw.logicalcpu"
    ELIF platform == "Linux":
        EXECUTE "cat /proc/cpuinfo | grep 'model name' | head -1"
        EXECUTE "nproc --all"
        EXECUTE "lscpu | grep 'NUMA node(s)'"

FUNCTION get_memory_info(platform):
    IF platform == "Windows":
        EXECUTE "Get-CimInstance Win32_PhysicalMemory | Measure-Object -Property Capacity -Sum"
    ELIF platform == "macOS":
        EXECUTE "sysctl -n hw.memsize"
    ELIF platform == "Linux":
        EXECUTE "free -b | grep Mem | awk '{print $2}'"

FUNCTION get_numa_node_count(platform):
    IF platform == "Windows":
        EXECUTE "(Get-CimInstance Win32_Processor | Measure-Object).Count"
    ELIF platform == "Linux":
        EXECUTE "lscpu | grep 'NUMA node(s)' | awk '{print $NF}'"
    ELIF platform == "macOS":
        RETURN 1  # macOS 通常为单 NUMA
```

### 3.5 文件权限检查

```pseudocode
FUNCTION check_write_permission(platform, path):
    IF platform == "Windows":
        EXECUTE "Get-Acl -Path '{path}'"
        # 解析 ACL 结果
    ELIF platform IN ["macOS", "Linux"]:
        EXECUTE "test -w '{path}' && echo 'writable' || echo 'not writable'"
        # 或 EXECUTE "ls -la '{path}'"
```

### 3.6 查找可执行文件

```pseudocode
FUNCTION find_executable(platform, exe_name):
    IF platform == "Windows":
        EXECUTE "Get-Command {exe_name} -ErrorAction SilentlyContinue | Select-Object Source"
        # 或 EXECUTE "where.exe {exe_name}"
    ELIF platform IN ["macOS", "Linux"]:
        EXECUTE "which {exe_name}"
        # 或 EXECUTE "command -v {exe_name}"
```

### 3.7 文件名合法性验证

```pseudocode
FUNCTION validate_filename(platform, filename):
    IF platform == "Windows":
        illegal_chars = ['<', '>', ':', '"', '/', '\\', '|', '?', '*']
        illegal_names = ['CON', 'PRN', 'AUX', 'NUL', 'COM1'-'COM9', 'LPT1'-'LPT9']
    ELIF platform IN ["macOS", "Linux"]:
        illegal_chars = ['/', '\0']
        # macOS 额外禁止 ':'（HFS+ 兼容）

    FOR each char IN illegal_chars:
        IF char IN filename:
            RETURN false
    RETURN true
```

---

## 4. 工作目录规范

```pseudocode
FUNCTION get_work_directory(platform):
    IF platform == "Windows":
        RETURN Join-Path $env:USERPROFILE "bbenc"
    ELIF platform IN ["macOS", "Linux"]:
        RETURN "$HOME/bbenc"

FUNCTION ensure_work_directory(platform):
    dir = get_work_directory(platform)
    IF NOT exists(dir):
        IF platform == "Windows":
            EXECUTE "New-Item -ItemType Directory -Path '{dir}' -Force"
        ELSE:
            EXECUTE "mkdir -p '{dir}'"
```

---

## 5. 平台不兼容时的降级策略

当某功能在目标平台不可用时，LLM 应：

1. **明确告知用户** 该功能在当前平台不可用
2. **提供替代方案**（如有）
3. **跳过该步骤** 而非报错退出

### 降级表

| 功能 | Windows 降级 | macOS 降级 | Linux 降级 |
|------|-------------|-----------|-----------|
| UAC 管理 | 正常执行 | 跳过，提示"macOS 无 UAC 概念" | 跳过，提示"Linux 无 UAC 概念" |
| AviSynth 工具 | 正常执行 | 跳过，建议用 VapourSynth | 跳过，建议用 VapourSynth |
| SVFI | 正常执行 | 跳过，提示"SVFI 仅支持 Windows" | 跳过，提示"SVFI 仅支持 Windows" |
| 文件选择对话框 | WinForms | 请求用户直接输入路径 | 请求用户直接输入路径 |
| 注册表操作 | 正常执行 | 不适用 | 不适用 |
| .bat 脚本 | 正常执行 | 生成 .sh 脚本 | 生成 .sh 脚本 |
