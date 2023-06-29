# PowerShell + Bastion + VSCode 连接GCR

## 0 创建密钥并上传，选择自己的GCR机器，启动ssh服务
> #### 0.1.1 打开windows powershell
> #### 0.1.2 输入ssh
>#### 0.1.3 输入ssh-keygen -t ed25519，然后一直print enter键，如果有 overwrite（yes/no）的选项，直接选yes就行。建议过程中不要修改生成密钥的位置。

> #### 0.2.1 进入C:\Users\XXX\.ssh目录，会有一个id_ed25519.pub的文件，用记事本打开，复制里面的内容。
> #### 0.2.2 Log into the [GCR Public Key manager](https://aka.ms/gcrssh/)
> #### 0.2.3 粘贴公钥，然后点击右边的Add Pubkey。注意粘贴公钥时，要粘贴包括fareast\alias@...之类的完整内容。
> #### 0.2.4 在[GCR | T&R页面](http://gcr-reservations/default.aspx)里面申请服务器，记住所申请服务器friendly name后四位

> #### 0.3 以管理员模式运行windows powershell，然后输入下面两行代码：
> ````bash
> Set-Service -Name ssh-agent -StartupType Manual 
> Start-Service ssh-agent
> ````
> #### 0.4 可以尝试使用[页面](https://dev.azure.com/msresearch/GCR/_wiki/wikis/GCR.wiki/6627/GCR-Bastion?anchor=using-the-web-portal)中Using The Web Portal部分的方法，用网页端连接自己申请的服务器，确保机子是可用的。

## 1 下载 AZ CLI

#### 1.1 两种安装方式:

- 手动安装:

[Install the Azure CLI for Windows | Microsoft Learn](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli-windows?tabs=azure-cli)

- 自动安装(Powershell) :

```bash
$ProgressPreference = 'SilentlyContinue'; Invoke-WebRequest -Uri https://aka.ms/installazurecliwindows -OutFile .\AzureCLI.msi; Start-Process msiexec.exe -Wait -ArgumentList '/I AzureCLI.msi /quiet'; rm .\AzureCLI.msi
```



#### 1.2 安装 azure cli ssh extension (Powershell)

```bash
az extension add --name ssh
```





## 2 建立Tunnel

#### 2.1 新建文件 *gdl.ps1*

> 找个地方, 复制上面[链接](https://dev.azure.com/msresearch/GCR/_wiki/wikis/GCR.wiki/6657/GCR-Bastion-Auto-Connect-script-with-Powershell)里面文末的代码, 更改代码中第九行的 alias, 保存

#### 2.2  登录 Azure (使用Powershell)

```bash
az login
```

#### 2.3 执行 gdl.ps1 (进入gdl.ps1所在文件夹)

```bash
.\gdl.ps1 -tunnel -num xxxx
```

> **xxxx**代表GCR资源的后四位, 如你的资源为: GCRAZGDL2323 即为 -num 2323
>
> 可能出现报错, Power shell默认阻止脚本执行, 输入下面的指令更改Powershell执行策略
>
> ```bash
> Set-ExecutionPolicy -ExecutionPolicy RemoteSigned
> ```
>
> 参考: [关于执行策略 - PowerShell | Microsoft Learn](https://learn.microsoft.com/zh-cn/powershell/module/microsoft.powershell.core/about/about_execution_policies?view=powershell-7.3)



## 3 建立SSH代理

**新打开**一个Powershell窗口, 键入指令:

```bash
ssh -N -L 2222:127.0.0.1:22 FAREAST.alias@127.0.0.1 
```

> - 可以简单理解为将本机2222端口转发至22端口代理
>
> -N  不要求分配shell，只作为转发
>
> -L   正向代理, 本地转发：`ssh -L LocalPort:remoteHost:remotePort sshHost`
> 比如(`ssh -L 4444:google.com:80`如果`http://localhost:4444`在浏览器中打开，则实际上会看到Google的页面)



## 4 修改 .ssh/config.json

在.ssh/config中添加:

```json
Host tunnel
    HostName 127.0.0.1
    Port 2222
    User FAREAST.alias
    StrictHostKeyChecking=No
    UserKnownHostsFile=\\.\NUL
```



## 5 连接

~~之前是如何连接GCR的, 现在就把tunnel当成GCR, 直接连接Tunnel~~

打开VSCode，安装remote-ssh，依次点击左侧边栏 远程资源管理器→SSH→tunnel，连接到tunnel

使用过程中要保持之前打开的两个powershell为开启状态

---

> *免责声明:*
>
> 笔者在刚连通GCR的第一时间写下本篇教程, 极可能有很多"多余"甚至"错误"的步骤, 如有任何问题以及想讨论的内容, 欢迎交流:
>
> Teams: Erhan Zhang



## 6 使用bat文件同时打开两个shell
bat文件需放在gdl.ps1同目录中，内容如下
```bash
start /min powershell -NoExit -ExecutionPolicy Bypass -command .\gdl.ps1 -tunnel -num xxxx
:try_ssh
timeout /T 10 /NOBREAK
ssh -N -L 2222:127.0.0.1:22 FAREAST.alias@127.0.0.1
goto try_ssh
```
要求已完成步骤1-4，能够用步骤5连接上GCR的前提下，方可采用该方法。
> 第1行使用新的powershell执行gdl.ps1脚本建立tunnel
> 
> 第3行设置恰当延时，等待tunnel建立，延时时间可依据个人电脑反应时间进行调整
> 
> 第4行建立ssh代理
> 
> 第2、5行通过自动跳转，使得因网络波动而掉线时能够自动重连



## 7 winSCP传输文件的注意事项
> 依据步骤2打开tunnel，依据[链接](https://dev.azure.com/msresearch/GCR/_wiki/wikis/GCR.wiki/6684/Copy-file-to-your-GCR-sandbox-from-your-local-computer-using-WinSCP)中的Connect WinSCP to the running Bastion Tunnel部分的流程配置WinSCP，该部分之前的配置即为步骤2建立tunnel的内容
> 
> 若已打开SSH代理，需要将端口改为2222
> 
>- 当然个人还是推荐使用azcopy

## 8 一些可能出现的问题的大致原因（猜测）
#### 1）22端口不可用
>- 检查ssh服务是否开启
>
>- 检查22端口是否已被占用

#### 2）连接后要求输入密码
>- 检查在生成密钥时，是否设置了密码
>
>- 检查-tunnel -num xxxx输入的后四位是否为本人申请的机子
>
>- 检查当前是否为租借机子的时间段内，推荐在收到activated邮件的半小时后再尝试连接。
>
> 如果以上情况都不是，报IT或者重启电脑试试？

#### 3）global protect(笔者的版本号为6.0.0-262)连接VPN时，同时打开azure storage和tunnel，用azure storage下载大文件，会冲突导致蓝屏？
> 某天蓝屏了三次发出的疑问
> 有可能是 win 11开多用户的时候，开着global protect 容易造成两个用户的联网权限打架。建议登录公司用户时将个人用户退出登录/注销，避免权限或者内存问题。

---

> 在Erhan的工作的基础上做些补充，如有纰漏请批评指正。目前俩人都离职了，有兴趣的同事可以继续维护~
> 
> Teams: Xinwei Hong


