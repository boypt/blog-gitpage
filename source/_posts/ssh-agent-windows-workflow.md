---
title: Windows下较完善的ssh-agent Keyring工作流程
date: 2022-08-12 13:41:01
tags:
	- windows
	- ssh
	- ssh-agent
categories:
	- linux
---

Win下有很多设计良好的终端软件，但是又有很多使用场景需要联合各种SSH终端使用，如同时用xshell进入主机终端、mysql客户端用plink建立隧道、winscp打开文件管理窗口、wsl2内rsync同步配置文件同时git推送到维护仓库……

这些软件都有各自的密码密钥管理系统，普通方式是生成一个key，然后转换成各个格式配置到这些终端平台的配置。矛盾的点就是，如果为了安全，key文件都得使用密码保护，那每次连接主机都要解锁一次key，非常繁琐；如果不使用密码保护key，那到处摆放的key文件似乎是其他程序随手可得，有着很大的隐患。

解决办法就是配置使用key agent，让程序通过agent协议使用一个密钥代理，代理程序独自维护密码保护的key密钥，才能无痛地从不同终端程序连接相同的主机。

<!-- more -->

配置ssh-agent后的最大妙用是使用git更加方便：在多台vps上使用同一个git仓库，传统方式就需要各台机上分别生成key并添加的git仓库的认证列表。配置agent后，agent可以从vps上的程序转发到工作这台机上的agent，从而完成转发认证（agent forward），git仓库上只需存在一个key。你登出后，其他人就不能再推到你的git仓库了。

这里重点推荐的是[WinCryptSSHAgent](https://github.com/buptczq/WinCryptSSHAgent)项目，其实现了一个ssh-agent程序，但是同时兼容pagent, xagent, hyper-v agent……基本常见的SSH软件都能兼容。

其实作者是为了使用Yubikey而造的万能兼容层，但是用来管理文件密钥也是非常方便的。[作者是v2ex网友 swchzq](https://www.v2ex.com/t/565640)。

WinCryptSSHAgent的使用很简单，就单个exe文件，运行后就呆在托盘区，没有需要安装，没有需要配置的地方。（hyper-v agent需要点一次确认安装），而是要配置其他那些ssh终端程序。

## 添加密钥

首先是要把现有的key添加到WinCryptSSHAgent。跟pagent、xagent等程序的设计不同，WinCryptSSHAgent自己不负责维护key的存放，而仅仅作为一个服务端，密钥的来源一个是操作系统内的密钥Store，另外就是通过agent协议添加key。

如果是像作者的需求，使用Yubikey直接插USB，那这个key就自动加载进去操作系统，程序就会自动找到了，不需要特殊操作。

而传统的文件key，则使用`ssh-add`命令，下面是指使用win10自带的openssh套件的`ssh-add`程序。不过需要先配置好WinSSH的环境才能使用`ssh-add`命令进行交互。见下一节中配置WinSSH部分。


## 配置各种终端

重点还是要让各种终端程序来使用这个agent，正如上述配置ssh-add命令的工作模式，终端都需要一个环境变量来找到agent的通信位置。

### 配置WinSSH

WinSSH是微软维护的openssh分支，已经进入Win10的可选组件。很可能已经默认安装。

首先配置系统环境变量，以便全局使用ssh-agent。正如右键点击托盘的图标，WinSSH的弹窗值。

![](sysenv.png)

设置后重新打开一个cmd窗口，使用`ssh-add -l`命令，应该就能列出WinCryptSSHAgent内列出的一个系统自带key了。

要添加自己的key，只需要`ssh-add my_id_ed25519`，输入保护密码，就会看到弹窗说密钥添加成功。

这时候再使用ssh命令直接连接服务器，就会使用这个key了。

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

P系列的工具使用的是共享内存技术，不需要配置，只要运行他们就自动找到agent，赞。

于是WinSCP、Putty、HeidiSQL、sqlyog这些使用putty系列的工具就自动支持了agent。


## 开机启动WinCryptSSHAgent

WinCryptSSHAgent单独exe可以简单地扔到`shell:startup`启动目录开机自动启动而不用理会。

### 使用AutoHotkey来控制添加key

我习惯使用AutoHotkey来启动一些小工具，于是把启动WinCryptSSHAgent和添加key的逻辑都整合到ahk脚本中。

```
RunWaitOne(command) {
  shell := ComObjCreate("WScript.Shell")
  exec := shell.Exec(ComSpec " /C " command)
  return exec.StdOut.ReadAll()
}

;SSH KEYAGENT
>!k::
  Process,Exist,WinCryptSSHAgent.exe
  If (ErrorLevel = 0) {
    Run, WinCryptSSHAgent.exe
    Sleep, 2000
  }
  keys := RunWaitOne("ssh-add -l")
  if !InStr(keys, "me@mylocation") {
    Run, %ComSpec% /C ssh-add %USERPROFILE%\.ssh\id_ed25519
  }
  Return
```

如此配置后，我按下"右Alt+k"，就会开启WinCryptSSHAgent，并出现一个输入key解锁密码终端窗口。

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

