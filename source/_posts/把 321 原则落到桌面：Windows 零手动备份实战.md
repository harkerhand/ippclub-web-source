---
title: 把 321 原则落到桌面：Windows 零手动备份实战
date: 2025-09-06 18:11:36
tags:
---

## 前情提要

发生该[事故](https://huanyp.cn/2025/09/06/%E6%8A%80%E6%9C%AF/%E5%BC%80%E6%BA%90/NcatBot%20%E7%B3%BB%E5%88%97%E8%BD%AF%E4%BB%B6%E9%87%8D%E5%A4%A7%E5%AE%89%E5%85%A8%E4%BA%8B%E6%95%85%E5%9B%9E%E9%A1%BE/)后，NcatBot 为了避免被卷入有关风波，进行了一些紧急操作以避险。由于是首次遇到此类事件，应对经验不足，导致丢失了约 6 h 的工作代码，同时也切实反映出数据安全的重要性。

本文就 321 原则，提供一个 Windows 操作系统下简易但切实可行的数据安全方案。

## 321 原则：三份数据、两块介质、一处异地，少一个都可能翻车！

321 原则是一种广泛应用于数据备份和数据安全领域的最佳实践。其核心思想是：

- **3**：至少保留 3 份数据副本（包括原始数据和备份）。
- **2**：将数据存储在 2 种不同的介质上（如硬盘、移动硬盘、云存储等）。
- **1**：至少有 1 份备份存放在异地（如云端或物理隔离的地点）。

通过遵循 321 原则，即使遇到硬件故障、误操作、自然灾害或勒索软件攻击等突发事件，也能最大程度地保障数据的完整性和可恢复性。这一原则简单易行，适用于个人用户、小型团队以及企业级的数据安全需求。

## 情况简析

对于个人开发者来说，三种情况的概率排序为 误操作 >> 硬件故障 > 其他所有情况。如果是开源代码，那么数据的 safety（数据完整性与可恢复性）要求是远大于 security（数据保密性与防止未授权访问）要求的。

对于**开发者**来说，数据风险主要在于**劳动成果的损失**，基于这个原则，我设计了以下的备份方案。

## 第一道防线：VS Code 的 Local-History，秒级回血

Local-History 是 Visual Studio Code 编辑器的一个插件，它能够自动记录你对文件的每次修改，形成本地的版本历史。这是防范误操作的第一道防线，简单粗暴但非常有效。

### 插件简介

10 s 即可快速完成安装，极大提升代码安全性。在 VSCode 中搜索并安装 "Local History" 插件即可。

每当你阶段性的需要保存文件时（按下 ctrl+s），插件会自动拷贝一个带时间戳的副本存储在本地，简单粗暴。

这种方式虽然简单，但对于日常的误删、误改等操作具有极强的防护作用。尤其是进行 git 有关的操作时。

### 最佳实践

- 🚨 需要将 `.history` 加入 `.gitignore`。

- 建议 `ctrl+,` 找到 `saveDelay` 设置为 15 s 或者 30 s（连续多长时间无更改就保存），提高空间效率。


## 第二道防线：Restic 快照，防的是‘整盘蒸发’

有时候脑抽达到了一种地步————把整个项目文件夹都给扬了，这时候 Local History 就有些回天乏术了。

再或者，哪天物理机器突然抽风坏掉了，那又咋办？

我们需要一个更好的备份操作。

### 情况介绍

为了防止误删，我们需要在别处完整的建立一个备份区域。这个备份区域是不能采用 "同步" 策略的，而应该采用 "快照" 策略。

<details>
<summary>同步和快照分别是什么？</summary>

![image-20250906190851595](https://ghfast.top/https://raw.githubusercontent.com/huan-yp/image_space/master/images/image-20250906190851595.png)

![image-20250906190904014](https://ghfast.top/https://raw.githubusercontent.com/huan-yp/image_space/master/images/image-20250906190904014.png)

</details>

由于我们能够接受一定程度的数据损失（例如 1 h 的工作成果损失），所以采用时效性稍差的 "快照" 仍然可以有效的解决问题。

特别的，为了防止单盘损坏，我们更需要在与系统盘所在**物理硬盘**不同的另一个盘上做备份。这是所谓的**两种介质**。

另外，为了防止极端情况发生，我们还需要**一份异地**。这里我采用了 SMA 共享文件夹方案，这种方案仍然可以用 restic 实现。


### 操作过程

#### 设置工作区变量

🚨 首先设置工作区目录变量，方便后续使用：

```bash
# 设置工具安装目录变量
$TOOLS_DIR = "C:\tools"
# 设置本地仓库目录变量（先创建对应目录）
$BACKUP_DIR_LOCAL = "D:\restic\MyRepo"
# 设置远程备份目录变量（先创建对应目录）
$BACKUP_DIR_REMOTE = "\\192.168.1.111\share\MyRepo"
# 设置密码（请牢记）
$PASSWORD = "YourStrongPassword"
# 目标目录（写绝对路径）
$TARGET_DIR = "C:/Users/yourname/Desktop/Proj"
```

#### 下载有关软件

```bash
mkdir $TOOLS_DIR
Invoke-WebRequest -Uri https://ghfast.top/https://github.com/restic/restic/releases/download/v0.17.1/restic_0.17.1_windows_amd64.zip ` -OutFile "$TOOLS_DIR\restic.zip"
Expand-Archive "$TOOLS_DIR\restic.zip" -DestinationPath $TOOLS_DIR
Rename-Item "$TOOLS_DIR\restic_0.17.1_windows_amd64.exe" -NewName "restic.exe"
Remove-Item "$TOOLS_DIR\restic.zip"
setx PATH "%PATH%;$TOOLS_DIR"          # 永久加入 PATH
```

#### 初始化有关仓库

```bash
cd $TOOLS_DIR
$Env:RESTIC_REPOSITORY_LOCAL  = $BACKUP_DIR_LOCAL
$Env:RESTIC_PASSWORD = $PASSWORD   # 仓库密码
restic init
$Env:RESTIC_REPOSITORY_REMOTE  = $BACKUP_DIR_REMOTE
$Env:RESTIC_PASSWORD = $PASSWORD   # 仓库密码
restic init
```

#### 写入密码

```bash
cat $PASSWORD | Out-File -FilePath "$TOOLS_DIR\restic_password.txt" -Encoding ASCII
```

#### 备份脚本

将脚本内容写入 `$TOOLS_DIR` 下的 `backup.ps1`：

<details>
<summary>展开即可抄作业</summary>

```powershell
@"
param(
    [string]`$Repo,
    [string]`$Pass
)
`$Env:RESTIC_REPOSITORY = `$Repo
`$Env:RESTIC_PASSWORD   = `$Pass
# 实际备份
restic backup ``
   --tag hourly ``
   --exclude-caches ``
   --exclude "*.log" ``
   --exclude "node_modules" ``
   $TARGET_DIR
# 清理旧快照
restic forget --prune ``
   --keep-hourly 72 --keep-daily 30 --keep-weekly 16
"@ | Out-File -FilePath "$TOOLS_DIR\backup.ps1" -Encoding UTF8
```

</details>

#### 设置 Windows 定时任务

<details>
<summary>展开即可抄作业</summary>

```bash
# 1. 本地备份任务 - 每小时
$trigger = New-ScheduledTaskTrigger -Once -At (Get-Date) `
                  -RepetitionInterval (New-TimeSpan -Hours 1) `
                  -RepetitionDuration (New-TimeSpan -Days 3650)
$action  = New-ScheduledTaskAction -Execute "PowerShell.exe" `
                  -Argument "-WindowStyle Hidden -ExecutionPolicy Bypass -File $TOOLS_DIR\backup.ps1 `
                            -Repo $BACKUP_DIR_LOCAL -Pass $PASSWORD"
Register-ScheduledTask -TaskName "ResticLocal" -Trigger $trigger -Action $action `
                       -User "$env:USERNAME" -Settings (New-ScheduledTaskSettingsSet -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries)

# 2. NAS备份任务 - 每小时（错开 15 min 减轻网络突刺）
$trigger2 = New-ScheduledTaskTrigger -Once -At (Get-Date).AddMinutes(15) `
                   -RepetitionInterval (New-TimeSpan -Hours 1) `
                   -RepetitionDuration (New-TimeSpan -Days 3650)
$action2  = New-ScheduledTaskAction -Execute "PowerShell.exe" `
                   -Argument "-WindowStyle Hidden -ExecutionPolicy Bypass -File $TOOLS_DIR\backup.ps1 `
                             -Repo $BACKUP_DIR_REMOTE -Pass $PASSWORD"
Register-ScheduledTask -TaskName "ResticNas" -Trigger $trigger2 -Action $action2 `
                       -User "$env:USERNAME" -Settings (New-ScheduledTaskSettingsSet -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries -StartWhenAvailable)
```

</details>

### 再一道保险

需要保证备份程序的正确设置。

#### 试运行备份

创建一个测试文件：

```bash
New-Item -Path $TARGET_DIR\test.txt -ItemType File
```

运行备份：

- 打开 "任务计划程序"，找到 "ResticLocal" 和 "ResticNas" 任务，点击运行。

- 终端运行以下命令检查备份完整性：

```bash
echo $PASSWORD | restic snapshots -r $BACKUP_DIR_LOCAL
echo $PASSWORD | restic snapshots -r $BACKUP_DIR_REMOTE
```

- 如果均显示了一张包含了备份时间、备份目录的表，则备份成功。


#### 定期检查备份完整性

设置脚本：

<details>
<summary>展开即可抄作业</summary>

```bash
@'
param(
    [string]$Repo,
    [string]$Pass
)

$Env:RESTIC_REPOSITORY = $Repo
$Env:RESTIC_PASSWORD   = $Pass

function Write-EventLogError {
    param($Message)
    Write-Error $Message
    New-EventLog -LogName Application -Source "ResticVerify" -ErrorAction SilentlyContinue
    Write-EventLog -LogName Application -Source "ResticVerify" -EntryType Error -EventId 1 -Message $Message
}

function Show-ToastNotification {
    param(
        [string]$Title,
        [string]$Message
    )
    
    # 方法2: 系统托盘气球
    Add-Type -AssemblyName System.Windows.Forms
    $notify = New-Object System.Windows.Forms.NotifyIcon
    $notify.Icon = [System.Drawing.SystemIcons]::Information
    $notify.BalloonTipIcon = "Error"
    $notify.BalloonTipTitle = $Title
    $notify.BalloonTipText  = $Message
    $notify.Visible = $true
    $notify.ShowBalloonTip(5000)
    Start-Sleep -Seconds 6
    $notify.Dispose()
}

# 统一错误处理函数
function Invoke-ResticCheck {
    param(
        [scriptblock]$Command,
        [string]$StepName
    )
    try {
        Write-Output "执行步骤: $StepName"
        & $Command
        if ($LASTEXITCODE -ne 0) { throw "restic 返回非零退出码 $LASTEXITCODE" }
    }
    catch {
        $errMsg = "$StepName 失败: $($_.Exception.Message)"
        Show-ToastNotification -Title "备份验证失败" -Message $errMsg
        Write-EventLogError $errMsg
        exit 1
    }
}

# 1. 基础 check
Invoke-ResticCheck -Command { restic check } -StepName "restic check"

# 2. 抽检 10%
Invoke-ResticCheck -Command { restic check --read-data-subset=10% } -StepName "restic check --read-data-subset=10%"

Write-Output "TestPass"
'@ | Out-File -FilePath "$TOOLS_DIR\verify.ps1"
```

</details>

设置定时任务：

<details>
<summary>展开即可抄作业</summary>

```bash
# 本地仓库验证任务
$trigger = New-ScheduledTaskTrigger -Weekly -DaysOfWeek Sunday -At 04:30
$action  = New-ScheduledTaskAction -Execute "PowerShell.exe" `
                  -Argument "-WindowStyle Hidden -ExecutionPolicy Bypass -File $TOOLS_DIR\verify.ps1 `
                            -Repo $BACKUP_DIR_LOCAL -Pass $PASSWORD"
$settings = New-ScheduledTaskSettingsSet -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries -StartWhenAvailable
Register-ScheduledTask -TaskName "ResticVerifyLocal" -Trigger $trigger -Action $action -Settings $settings -User $env:USERNAME

# NAS仓库验证任务（错开 30 min 执行）
$trigger2 = New-ScheduledTaskTrigger -Weekly -DaysOfWeek Sunday -At 05:00
$action2  = New-ScheduledTaskAction -Execute "PowerShell.exe" `
                   -Argument "-WindowStyle Hidden -ExecutionPolicy Bypass -File $TOOLS_DIR\verify.ps1 `
                             -Repo $BACKUP_DIR_REMOTE -Pass $PASSWORD"
$settings2 = New-ScheduledTaskSettingsSet -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries -StartWhenAvailable
Register-ScheduledTask -TaskName "ResticVerifyNas" -Trigger $trigger2 -Action $action2 -Settings $settings2 -User $env:USERNAME
```

</details>

### 试运行验证

```bash
PowerShell.exe -File $TOOLS_DIR\verify.ps1 -Repo $BACKUP_DIR_LOCAL -Pass $PASSWORD
PowerShell.exe -File $TOOLS_DIR\verify.ps1 -Repo $BACKUP_DIR_REMOTE -Pass $PASSWORD
```

- 如果显示 "TestPass"，则验证成功。

## 总结

### Why 定期验证

定期验证的目的是确保备份的完整性，事实上，备份由于偶然因素损坏的概率和硬件损坏的概率大致相当，定期验证可以及时发现备份损坏的问题。

即使备份发生损坏，同时发生主数据损坏的概率极低。只需要重新修复备份即可。

### 该方案的优点

- 基本践行 321 原则：
   - Local-History 作为第一道防线，能够秒级恢复工作区数据。
   - Restic 作为第二和第三道防线，完成**两种介质**和**一份异地**备份。
- 一次配置，永久使用，备份操作无需任何手动介入。


### 该方案的不足

- 密钥相关：
   - 🚨 没有提供密钥备份方案，需要自行多端备份恢复密钥。
   - 🚨 密钥明文存储，对于勒索性丢失有一定风险。

- 🚨 需要自行配置一个支持 SMB 的 NAS，并且需要确保 NAS 的稳定性。
- 备份触发时无法做到**完全无感**，会打断工作聚焦。
- 告警机制不够完善，Windows 弹窗日志偶尔会被忽略，可以考虑发送邮件等更加完善的方式。
