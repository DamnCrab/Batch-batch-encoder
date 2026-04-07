# Skill 1: 运行环境检测

## 技能描述

检查用户系统是否具备运行视频编码工作流的条件，包括 PowerShell 版本、文件系统权限、UAC 设置和硬件信息。

## 触发条件

用户提出以下意图之一：
- 检查系统环境是否兼容
- 运行环境检测
- 查看硬件信息
- 管理 UAC 设置

## 对话流程

### 第一步：PowerShell 版本检查

**LLM 执行：**
```powershell
$PSVersionTable.PSVersion
```

**判断逻辑：**
- 版本 ≥ 5.1：✅ 告知用户版本符合要求
- 版本 < 5.1：⚠️ 警告某些功能可能无法正常工作

### 第二步：文件系统权限检查

**LLM 执行：**
```powershell
# 检查 C:\ 根目录权限
$acl = Get-Acl -Path "C:\"
$currentUser = [Security.Principal.WindowsIdentity]::GetCurrent()
$userName = $currentUser.Name

foreach ($access in $acl.Access) {
    if ($access.IdentityReference -eq $userName -or
        $access.IdentityReference -eq "BUILTIN\Users" -or
        $access.IdentityReference -eq "NT AUTHORITY\Authenticated Users") {
        Write-Host "权限: $($access.FileSystemRights)"
    }
}
```

**向用户报告：**
- 是否拥有 C:\ 的读写权限
- 是否拥有 `%USERPROFILE%` 的读写权限
- 是否拥有完全控制权限

### 第三步：UAC 管理（交互式）

**向用户展示当前 UAC 状态后，提供选项：**

```
选择 UAC 操作选项：
A: 禁用 UAC（每次运行脚本不再弹出警告）
B: 不更改（继续检测）
C: 恢复 UAC（公用电脑建议）
Q: 退出
```

**用户选择后的处理逻辑：**

#### 选项 A：禁用 UAC
1. 检查是否有管理员权限
2. 如果没有，询问用户是否请求提升权限
3. 要求输入 `CONFIRM` 确认
4. 备份当前 UAC 设置到 `%TEMP%\UAC_Backup_{时间戳}.json`
5. 设置注册表值：
   ```
   ConsentPromptBehaviorAdmin = 0
   ConsentPromptBehaviorUser = 0
   PromptOnSecureDesktop = 0
   EnableLUA = 0
   ```
6. 提示需要重启系统

#### 选项 B：跳过
直接继续

#### 选项 C：恢复 UAC
1. 检查管理员权限
2. 恢复默认值：
   ```
   ConsentPromptBehaviorAdmin = 5
   ConsentPromptBehaviorUser = 3
   PromptOnSecureDesktop = 1
   EnableLUA = 1
   ```
3. 提示需要重启系统

**读取 UAC 的注册表路径：**
```
HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System
```

### 第四步：显示硬件信息

**LLM 执行以下 WMI/CIM 查询并整理输出：**

```powershell
# 操作系统
$os = Get-CimInstance -ClassName Win32_OperatingSystem
Write-Host "操作系统: $($os.Caption) (Build $($os.BuildNumber))"

# 主板
$baseboard = Get-CimInstance -ClassName Win32_Baseboard
# 显示: 名称、厂商、型号、序列号、版本

# BIOS
$bios = Get-CimInstance -ClassName Win32_BIOS
# 显示: 名称、版本、发布日期

# 处理器
$processors = Get-CimInstance -ClassName Win32_Processor
# 显示: 名称、插槽、当前/最大频率、核心/线程数、L2/L3缓存、当前负载

# 内存
$memoryModules = Get-CimInstance -ClassName Win32_PhysicalMemory
# 显示: 每个模块的容量、厂商、型号、速度、序列号，以及总内存
```

### 第五步：总结

向用户展示完整的检测结果摘要：
- ✅/⚠️ PowerShell 版本
- ✅/⚠️ 文件系统权限
- ✅/⚠️ UAC 状态
- 硬件信息概要

如果 UAC 已禁用，建议处理完毕后重新启用。

## 输出

此技能不产生文件输出，仅提供系统状态报告。

## 注意事项

- UAC 修改需要管理员权限
- UAC 修改后需要重启系统才能生效
- 备份文件保存在 `%TEMP%` 目录下
