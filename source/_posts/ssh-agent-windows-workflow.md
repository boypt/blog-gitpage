---
title: Windows下较完善的ssh-agent工作流程:WinCryptSSHAgent
date: 2022-08-12 13:41:01
tags:
	- windows
	- ssh
	- ssh-agent
categories:
	- linux
---

Win下有很多设计良好的终端软件，但是又有很多使用场景需要联合各种SSH终端使用，如同时用xshell进入主机终端、mysql客户端用plink建立隧道、winscp打开文件管理窗口、wsl2内rsync同步配置文件，在不同的虚拟机、vps环境内使用github的仓库……

这些软件都有各自的密码密钥管理系统，普通使用方式是生成一个个人key，然后转换成各个格式配置到这些终端平台的配置。

矛盾在于，如果为了安全，key文件都得使用密码保护，那每次连接主机都要解锁一次key，非常繁琐；如果不使用密码保护key，那到处摆放的key文件似乎是其他程序随手可得，有着很大的隐患。

解决办法就是配置使用ssh agent，key文件只需让agent单份处理。其他程序通过协议与agent进行通信认证。

<!-- more -->
这里重点推荐的是[WinCryptSSHAgent](https://github.com/buptczq/WinCryptSSHAgent)项目，其实现了一个ssh-agent程序，但是同时兼容pagent, xagent, hyper-v agent……基本常见的SSH软件都能兼容。

作者是为了使用Yubikey而造的万能兼容层，但是用来管理文件密钥也是非常方便的。[作者是v2ex网友 swchzq](https://www.v2ex.com/t/565640)。

WinCryptSSHAgent的使用很简单，单个exe，运行后就呆在托盘区，没有需要安装、配置的地方。（hyper-v agent需要点一次确认安装），而是要配置其他那些ssh终端程序。

## 添加密钥

首先是要把现有的key添加到WinCryptSSHAgent。跟pagent、xagent等程序的设计不同，WinCryptSSHAgent自己不负责维护key的存放，而仅仅作为一个服务端，密钥的来源一个是操作系统内的密钥Store，另外就是通过agent协议添加key。

### 操作系统证书

打开`certmgr.msc`内的“个人”类别证书就是可以被WinCryptSSHAgent使用的认证证书（注意需要有私钥）。

![](oscert.png)

如果是像作者的需求，使用Yubikey直接插USB，那这个key就自动加载进去操作系统，程序就会自动找到了，不需要特殊操作。（也许一些支持通用协议的U盾设备也可以，未尝试）。

但也可以自己生成自签证书导入到“个人”证书。这个过程需要用到openssl。

```bash
# 生成密钥对
openssl req -subj '/CN=MyGenKey/O=SELF/C=CN' -new -newkey rsa:2048 -days 3650 -nodes -x509 -keyout my.key -out my.cer

# 转换成Win下可导入的pfx格式
openssl pkcs12 -inkey my.key -in my.cer -export -out mygenkey.pfx
```

输出的`mygenkey.pfx`在Win下双击即可进行导入。导入过程中有个挺有趣的选项，不可更改的：

![](oskeyconfirm.png)

勾上后每次使用该密钥都会操作系统的弹窗弹窗，来确认当前密钥的使用。

这种方式只能使用rsa密钥。

### OpenSSH证书

OpenSSH证书用的是另外一个体系，跟操作系统的证书管理无关。

WinCryptSSHAgent跟Openssh的ssh-agent服务兼容，使用命名管道跟ssh工具通信，使用`ssh-add`命令进行密钥的维护。

如无意外只需要启动WinCryptSSHAgent后就能使用Win10自带的`ssh-add`命令，不需特殊配置。但有可能出现错误或者冲突，常见是因为Win10自带的OpenSSH-Agent服务正在运行中，需要停止，见下一节WinSSH环境的配置描述。

使用OpenSSH证书好处是能够使用方便的ed25519证书、兼容已有的证书。而WinCryptSSHAgent则会在右下角弹窗通知每次密钥的使用。比起默认的openssh-agent静默服务，也提供了一定的安全性改善。

## 配置各种终端

要让各种终端程序来使用这个agent，如上述配置`ssh-add`命令的工作模式，终端程序需要找到agent的通信位置。

### 配置WinSSH

WinSSH是微软维护的openssh分支，已经进入Win10的可选组件，很可能已经默认安装。它会自动找到设置的命名管道跟agent进行通信。

对于WinSSH重点不是要配置，而且确认Windows自带的openssh-agent服务要停下来，避免跟WinCryptSSHAgent冲突或者混淆了。有些情况下WinCryptSSHAgent[启动即报错命名管道权限问题](https://github.com/buptczq/WinCryptSSHAgent/issues/29)，就是这个原因。

在PowerShell(管理员权限)下运行以下命令确认ssh-agent服务停止，不自动启动。

```powershell
Get-Service ssh-agent | Stop-Service
Get-Service ssh-agent | Set-Service -StartupType Manual
```

可以用`ssh-add -l`确认跟WinCryptSSHAgent的通信是否正常。
```powershell
ssh-add -l

ssh-add # 会添加%USERPROFILE%\.ssh\目录内的所有key
```

### 配置WSL2

WSL2内其实是个虚拟机环境，程序建立了个代理隧道，而WSL2内则打开一个socat进行管道转发实现的。

具体代码还是右键托盘图标，点WSL2弹窗的代码，粘贴到`~/.bashrc`（或者你用其他shell的话对应的启动配置）。

### 配置XShell

XShell不用配置变量，只需要打个勾。

从File （文件）- Default Session Properties （默认会话属性），点开Connection - SSH，勾上图中选项。

![](xshell.png)

这样以后新建的会话都会尝试联系agent。

已创建的保存会话，需要在会话管理器内设置它们各自的属性（Tips：可以全选然后点属性按钮，一次过更改所有所选会话）。

### Putty/Plink

P系列的工具使用的是共享内存技术，不需要配置，只要运行就自动找到agent。

于是WinSCP、Putty、HeidiSQL、sqlyog这些使用putty系列的工具就自动支持了agent。

## 开机启动WinCryptSSHAgent

WinCryptSSHAgent单独exe可以简单地扔到`shell:startup`启动目录开机自动启动而不用理会。

### 使用AutoHotkey来控制添加key

我习惯使用AutoHotkey来启动一些小工具，于是把启动WinCryptSSHAgent和添加key的逻辑都整合到ahk脚本中。

```
RunWaitOne(command) {
    shell := ComObjCreate("WScript.Shell")
    exec := shell.Exec(ComSpec " /C " command)
    return exec.StdOut.ReadAll() . exec.StdErr.ReadAll()
}

;SSH KEYAGENT
>!k::
	Process,Exist,WinCryptSSHAgent.exe
	If (ErrorLevel = 0) {
		Run, WinCryptSSHAgent.exe
		MsgBox,, Running, Run WinCryptSSHAgent, 1
	}
	keys := RunWaitOne("ssh-add -l")
	if InStr(keys, "Error") {
		MsgBox, Failed to run WinCryptSSHAgent:`n`n%keys%
		return
	}
	Run, ssh-add.exe
	return
```

如此配置后，按下"右Alt+k"，就会开启WinCryptSSHAgent，并出现一个输入key解锁密码终端窗口。

### 使用WinSSH/OpenSSH的自带功能添加key

OpenSSH的配置文件`%USERPROFILE%\.ssh\config`（或wsl内`~/.ssh/config`）可以加上这么一段：

```
Host *
  HostkeyAlgorithms +ssh-rsa
  StrictHostKeyChecking no
  AddKeysToAgent yes
```

重点是`AddKeysToAgent yes`这句，如果agent里面并没有key，ssh会读取`.ssh/id_xxx`，如果解锁认证成功，就顺便通过agent协议把这个key添加到agent。

如果主要使用VSCode的SSH Remote功能，那首次连接时候解锁一次密钥，密钥就呆在agent中了，非常方便。
