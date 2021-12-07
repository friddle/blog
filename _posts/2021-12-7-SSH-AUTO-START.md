---
title: 如何在Windows上开机自动启动脚本和SSH反向隧道
categories:
  - Tech
tags:
  - Windows SSH TAG
date: 2021-12-07 02:56 +0000
---

经常有这种需求。开机自动启动一个ssh反向隧道。能把本地的3389/22本地端口映射到服务器上

## 安装原生Windows版本的SSH
```powershell
Get-WindowsCapability -Online | Where-Object Name -like 'OpenSSH*'
Add-WindowsCapability -Online -Name OpenSSH.Client~~~~0.0.1.0
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
```

## 启动SSH服务
Start-Service sshd

## 设置自动启动
Set-Service -Name sshd -StartupType 'Automatic'
## 添加防火墙
```powershell
if (!(Get-NetFirewallRule -Name "OpenSSH-Server-In-TCP" -ErrorAction SilentlyContinue | Select-Object Name, Enabled)) {
    Write-Output "Firewall Rule 'OpenSSH-Server-In-TCP' does not exist, creating it..."
    New-NetFirewallRule -Name 'OpenSSH-Server-In-TCP' -DisplayName 'OpenSSH Server (sshd)' -Enabled True -Direction Inbound -Protocol TCP -Action Allow -LocalPort 22
} else {
    Write-Output "Firewall rule 'OpenSSH-Server-In-TCP' has been created and exists."
}
# 设置默认的shell
New-ItemProperty -Path "HKLM:\SOFTWARE\OpenSSH" -Name DefaultShell -Value "C:\Program Files\Git\bin\bash.exe" -PropertyType String -Force
```

## 添加开机自动启动脚本  
一般在这个目录:
`C:\ProgramData\Microsoft\Windows\Start Menu\Programs\StartUp` 新建 autostart.vbs
内容为

```vbs
Dim objShell
Set objShell = WScript.CreateObject ("WScript.shell")
Set outVar=objShell.exec("cmd.exe /C ""c:\Program Files\Git\bin\sh.exe"" --login -i -c c:/Users/user/xxx/auto_start.sh -R")
Set objShell=Nothing
```
既可执行`c:/Users/user/xxx/auto_start.sh`的脚本.这里面是用Git的bash来执行脚本。大家也可以换成自己喜欢的sh执行

## 添加反向隧道的命令
即在auto_start.sh中添加如下命令
```bash
   ssh -CNR 222:localhost:22 friddle@xxxx.com
   ssh -CNR 3389:localhost:3389 friddle@xxx.com
```

ssh -CNR 远程端口:localhost:本地端口 用户@中转服务器   
3389是Windows Remote登录的默认端口   
作为这些步骤恭喜你可以在服务器上远程登录本地了   

当然远程登录需要你在windows中开启远程登录
