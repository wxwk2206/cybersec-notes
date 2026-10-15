## 一、基础系统信息收集
```
systeminfo
输出系统完整信息：OS版本、补丁号、主机名、域/工作组、网卡配置、系统安装时间、热修复补丁等

ver
快速查看Windows内核版本号

hostname
查看当前主机名称

set
列出全部系统环境变量，包含用户名、系统路径、临时目录、用户目录等敏感信息

wmic os get caption,version,buildnumber,installdate
结构化输出系统名称、版本、内部版本号、安装时间，便于脚本化提取
```

## 二、网络拓扑与存活探测
```
ipconfig /all
查看所有网卡详细配置：IP地址、子网掩码、网关、DNS、MAC地址、DHCP状态，可发现多网卡/多网段

route print
打印系统路由表，可发现内网其他网段、静态路由、出口网关与VPN路由

arp -a
查看本地ARP缓存表，获取同网段已通信主机的IP与MAC对应关系，快速发现存活主机

netstat -ano
列出所有TCP/UDP网络连接、监听端口、外部地址与对应进程PID，用于发现开放服务与异常外联

netstat -anob
【需管理员权限】在netstat -ano基础上，额外显示端口对应的进程名称与可执行文件路径

ping -n 1 -w 500 192.168.1.1
向目标IP发送1个ICMP包，超时500ms，用于快速探测主机存活（可配合for循环批量扫网段）

tracert -d 目标IP
路由追踪，不解析域名，查看数据包经过的路由节点，判断内网层级与拓扑

nslookup 域名
对指定域名进行DNS解析，可用于探测内网DNS服务器、域控制器地址
```

## 三、用户与权限信息排查
```
whoami /all
查看当前用户名、所属用户组、安全标识符(SID)、拥有的系统权限等完整信息

whoami /priv
列出当前用户持有的系统权限，重点关注SeDebugPrivilege、SeImpersonatePrivilege等高价值提权权限

net user⭐
列出本地所有用户账户

net user 用户名
查看指定用户的详细信息：账户状态、上次登录时间、所属组、密码过期时间等

net localgroup
列出本地所有用户组

net localgroup administrators
查看本地管理员组的成员列表，排查高权限账户

net accounts
查看本地账户策略：密码长度、锁定阈值、过期时间等安全配置
```

## 四、进程、服务与启动项
```
tasklist
列出当前所有运行进程的名称、PID、会话、内存占用

tasklist /svc
显示每个进程对应的系统服务，用于定位svchost等宿主进程承载的具体服务

tasklist /m
列出进程加载的所有DLL模块，可用于排查DLL注入等异常

net start
列出所有已启动的系统服务名称

sc query type= service state= all
列出所有服务的详细状态：服务名、显示名、运行状态、启动类型等

wmic process get name,processid,executablepath,commandline
结构化输出进程名称、PID、可执行文件路径、启动命令行，可发现可疑进程的隐藏路径与参数

reg query "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run"
查询系统级开机启动项；将HKLM替换为HKCU可查询当前用户的用户级启动项
```

## 五、共享资源与网络连接
```
net share
查看本地开启的所有共享文件夹，包括系统默认共享（C$、D$、ADMIN$、IPC$）及共享路径

net use
查看当前已建立的所有网络共享连接、映射的网络驱动器与凭证

net view \\主机名/IP⭐
查看指定远程主机上开放的共享资源列表

net view
查看同一工作组/域内的在线主机列表（依赖Computer Browser服务）

net use Z: \\目标IP\共享名 "密码" /user:"用户名"
【授权场景下使用】将远程共享文件夹映射为本地Z盘，用于文件上传下载
```

## 六、域环境信息收集（仅域内主机可用）
```
net config workstation⭐
查看工作站信息，包含所属域名、登录的域控制器名称

net view /domain⭐
列出当前域内的所有计算机主机

net time /domain
查看域内时间服务器（通常为主域控制器）的时间，同时可定位域控IP

nltest /dclist:域名
列出指定域内所有域控制器列表

net group "Domain Admins" /domain
查看域管理员组的成员列表（普通域用户权限即可执行）

net group "Domain Users" /domain
查看域内所有域用户列表
```

## 七、文件与注册表快速检索
```
dir /s /b C:\*.txt
递归列出C盘下所有txt文件；/s=递归子目录，/b=仅输出完整路径；可替换为.doc/.xls/.conf等后缀

dir /a:h /s /b 目录路径
查看指定目录下的所有隐藏文件与文件夹

reg query "HKLM\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters"
查询TCP/IP核心配置参数，包含主机名、域名、DNS服务器等
```