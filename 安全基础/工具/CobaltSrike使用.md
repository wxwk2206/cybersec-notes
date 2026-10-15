## server端部署
加权chmod a+x teamserver
 chmod a+x TeamServerImage

![Pasted image 20260703173653](../../PNG/Pasted%20image%2020260703173653.png)

sudo ./teamserver    （IP）（密码）
![Pasted image 20260703173704](../../PNG/Pasted%20image%2020260703173704.png)

## Client端
启动client端，填写IP，密码
![Pasted image 20260703173708](../../PNG/Pasted%20image%2020260703173708.png)


# Cobalt Strike Beacon右键菜单功能说明

## 一级菜单
1. **Interact**
打开该beacon的交互控制台，弹出beacon命令行窗口，可以手动输入beacon命令（shell、execute-assembly等），就是图中下方event log这个beacon终端窗口。

2. **Sleep...**
设置Beacon休眠时间（回连间隔）+抖动jitter。默认60秒，sleep越大，beacon回连越慢，流量越少越隐蔽；抖动用来随机化回连时间，规避特征检测。实验环境经常设置`sleep 0`实现几乎实时通信。

3. **Note...**
给当前beacon会话添加备注标签，比如标记机器用途、权限、内网信息，方便多会话管理。

4. **Access**
查看当前beacon会话的权限信息：当前用户名、权限、是否高权限、UAC状态等。

5. **Explore（探测/探索）**
信息收集、人机交互功能集合，就是截图展开的子菜单。
- **Browser Pivot**：浏览器劫持，注入目标机器浏览器进程，劫持浏览器cookie、会话，以受害者身份访问网页，不用重新登录。
- **Desktop (VNC)**：VNC远程桌面，在目标机器后台启动VNC服务，图形化操作对方桌面；CS会在teamserver本地做端口转发，如图日志`started port forward on 6654`就是VNC转发端口。
- **File Browser**：图形化文件浏览器，可视化浏览、上传、下载、删除目标主机文件，不用手动执行`cd、ls、upload、download`命令。
- **Net View**：执行windows `net view`，查询局域网内工作组、域内在线主机列表，做内网发现。
- **Port Scan**：内网端口扫描，通过当前beacon会话，扫描内网其他机器端口，属于pivot扫描。
- **Process List**：列出目标主机全部进程，PID、进程名、父进程、权限，用于找可注入进程、杀软排查。
- **Screenshot**：对目标主机屏幕截图，抓取当前桌面画面，回传到CS。

6. **Pivoting（代理/隧道，内网穿透）**
内网横向代理功能：
- **SOCKS Server**：启动socks4a代理，teamserver开启端口，流量经过被控主机代理出去，proxychains可以配合扫描内网。
- **Listener**：端口转发，反向转发，把目标机器端口映射到teamserver。
- **Deploy VPN**：部署VPN隧道，把teamserver接入目标内网，虚拟网卡。

7. **Session（会话管理）**
- **Spawn**：派生新beacon会话，生成新的beacon进程，可选择监听器；也可以派生给msf拿到meterpreter。
- **Exit**：让beacon进程优雅退出，会话消失。
- **Remove**：仅仅在CS界面删除会话记录，**不会让目标beacon退出**，beacon还会继续回连。
- **Note/Sleep**：和上面一级菜单功能一致。
