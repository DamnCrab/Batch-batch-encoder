# Skill 1: 运行环境检测

## 技能描述

检查用户系统是否具备运行视频编码工作流的条件，包括 Shell 环境、文件系统权限、平台特有设置和硬件信息。

## 触发条件

用户提出以下意图之一：
- 检查系统环境是否兼容
- 运行环境检测
- 查看硬件信息
- 管理 UAC 设置（Windows）

## 前置步骤

```pseudocode
# 首先检测平台（参见 common/platform.md）
platform = CALL detect_platform()
```

---

## 对话流程

### 第一步：Shell 环境检查

```pseudocode
FUNCTION check_shell_environment(platform):
    IF platform == "Windows":
        EXECUTE "$PSVersionTable.PSVersion"
        IF version >= 5.1:
            REPORT "✅ PowerShell {version} 符合要求"
        ELSE:
            REPORT "⚠️ PowerShell 版本过低，建议升级到 5.1+"

    ELIF platform == "macOS":
        EXECUTE "zsh --version"
        EXECUTE "sw_vers"  # macOS 版本
        REPORT "✅ macOS {version}, Shell: zsh {zsh_version}"

    ELIF platform == "Linux":
        EXECUTE "bash --version | head -1"
        EXECUTE "cat /etc/os-release | head -2"
        REPORT "✅ Linux ({distro}), Shell: bash {bash_version}"
```

### 第二步：文件系统权限检查

```pseudocode
FUNCTION check_filesystem_permissions(platform):
    work_dir = CALL get_work_directory(platform)
    
    IF platform == "Windows":
        EXECUTE "Get-Acl -Path $env:USERPROFILE"
        # 解析 ACL，检查当前用户权限
        # 检查 C:\ 根目录读写权限
        # 检查 %USERPROFILE% 读写权限

    ELIF platform IN ["macOS", "Linux"]:
        EXECUTE "test -w '$HOME' && echo 'HOME writable' || echo 'HOME not writable'"
        EXECUTE "test -w '/tmp' && echo 'tmp writable' || echo 'tmp not writable'"
        # 检查 $HOME 读写权限
        # 检查 /tmp 读写权限

    # 确保工作目录存在
    CALL ensure_work_directory(platform)
    REPORT permissions_result
```

### 第三步：平台特有设置（条件性）

```pseudocode
FUNCTION check_platform_specific_settings(platform):
    IF platform == "Windows":
        CALL manage_uac()  # 仅 Windows
    ELIF platform == "macOS":
        # 检查 Gatekeeper 状态（可选）
        EXECUTE "spctl --status"
        REPORT "Gatekeeper 状态: {status}"
        REPORT "提示: 如编码器被阻止运行，使用 xattr -d com.apple.quarantine {path}"
    ELIF platform == "Linux":
        # 检查 SELinux/AppArmor 状态（可选）
        EXECUTE "getenforce 2>/dev/null || echo 'SELinux not installed'"
        REPORT "安全模块状态: {status}"
```

#### Windows UAC 管理子流程

```pseudocode
FUNCTION manage_uac():
    # 仅 Windows 平台执行
    EXECUTE "Get-ItemProperty -Path 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System'"
    DISPLAY current_uac_status

    ASK_USER "选择 UAC 操作选项：
        A: 禁用 UAC（每次运行脚本不再弹出警告）
        B: 不更改（继续检测）
        C: 恢复 UAC（公用电脑建议）
        Q: 退出"

    IF user_choice == "A":
        REQUIRE admin_privileges
        ASK_USER "输入 CONFIRM 确认"
        BACKUP uac_settings TO "{TEMP}/UAC_Backup_{timestamp}.json"
        SET registry:
            ConsentPromptBehaviorAdmin = 0
            ConsentPromptBehaviorUser = 0
            PromptOnSecureDesktop = 0
            EnableLUA = 0
        REPORT "需要重启系统使更改生效"

    ELIF user_choice == "C":
        REQUIRE admin_privileges
        SET registry:
            ConsentPromptBehaviorAdmin = 5
            ConsentPromptBehaviorUser = 3
            PromptOnSecureDesktop = 1
            EnableLUA = 1
        REPORT "需要重启系统使更改生效"
```

### 第四步：显示硬件信息

```pseudocode
FUNCTION display_hardware_info(platform):
    # --- 操作系统 ---
    IF platform == "Windows":
        EXECUTE "Get-CimInstance Win32_OperatingSystem | Select Caption, BuildNumber"
    ELIF platform == "macOS":
        EXECUTE "sw_vers"
        EXECUTE "uname -m"  # 架构: arm64 / x86_64
    ELIF platform == "Linux":
        EXECUTE "cat /etc/os-release | grep PRETTY_NAME"
        EXECUTE "uname -m"

    # --- 处理器 ---
    IF platform == "Windows":
        EXECUTE "Get-CimInstance Win32_Processor | Select Name, NumberOfCores, NumberOfLogicalProcessors, MaxClockSpeed"
    ELIF platform == "macOS":
        EXECUTE "sysctl -n machdep.cpu.brand_string"
        EXECUTE "sysctl -n hw.physicalcpu"
        EXECUTE "sysctl -n hw.logicalcpu"
    ELIF platform == "Linux":
        EXECUTE "lscpu | grep -E 'Model name|Core|Thread|Socket|NUMA'"

    # --- 内存 ---
    IF platform == "Windows":
        EXECUTE "Get-CimInstance Win32_PhysicalMemory | Select Capacity, Speed, Manufacturer"
        EXECUTE "(Get-CimInstance Win32_PhysicalMemory | Measure Capacity -Sum).Sum / 1GB"
    ELIF platform == "macOS":
        EXECUTE "sysctl -n hw.memsize | awk '{print $0/1073741824 \" GB\"}'"
    ELIF platform == "Linux":
        EXECUTE "free -h | grep Mem"
        EXECUTE "dmidecode --type memory 2>/dev/null | grep -E 'Size|Speed|Manufacturer' || echo '需 root 权限查看详情'"

    # --- NUMA 信息（编码相关） ---
    numa_count = CALL get_numa_node_count(platform)
    IF numa_count > 1:
        REPORT "⚠️ 检测到 {numa_count} 个 NUMA 节点，x265 编码时可指定 --pools 参数"

    REPORT formatted_hardware_summary
```

### 第五步：总结

```pseudocode
FUNCTION summarize(platform):
    REPORT "=== 环境检测报告 ==="
    REPORT "平台: {platform}"
    REPORT "Shell 环境: {✅/⚠️ result}"
    REPORT "文件权限:   {✅/⚠️ result}"
    IF platform == "Windows":
        REPORT "UAC 状态:   {✅/⚠️ result}"
    REPORT "硬件信息:   {summary}"
    REPORT ""
    REPORT "如果所有项均为 ✅，可继续运行 Skill 2（编码管线配置）。"
```

---

## 输出

此技能不产生文件输出，仅提供系统状态报告。

## 注意事项

- UAC 管理仅在 Windows 平台可用，其他平台自动跳过
- macOS 上 Gatekeeper 可能阻止非签名编码器运行，需 xattr 清除
- Linux 上 SELinux/AppArmor 可能影响文件访问
- 硬件详情查询在 Linux 上可能需要 root 权限
