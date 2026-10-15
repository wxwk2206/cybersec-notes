# Kscan：轻量化全方位扫描器

> 项目地址：https://github.com/lcvvvv/kscan
> 当前版本：v1.87

## 一、简介
Kscan 是一款纯 Go 开发的全方位扫描器，具备**端口扫描、协议检测、指纹识别、暴力破解**等功能。支持协议 1200+、协议指纹 10000+、应用指纹 20000+、暴力破解协议 10 余种。
### 设计特点
- **多格式输入**：无需在使用前对扫描对象分类（区分 IP 还是 URL），所有条目均能正常输入和识别。若是 URL 地址会保留路径进行检测；若只是 `IP:PORT` 则会优先对该端口进行协议识别。支持三种输入方式：`-t/--target`、`-f/--fofa`、`--spy`。
- **注重准确性**：不是根据端口号与常见协议进行比对来确认端口协议，而是追求高准确性的协议识别，为后续应用层识别提供良好条件。
- **以端口为单位输出**：不以模块化方式做功能堆叠，而是以端口为单位输出资产信息。例如端口协议为 HTTP 则自动进行后续指纹识别、标题获取；端口协议为 RPC 则尝试获取主机名等。

### 扫描流程
扫描任务输入（file、fofa、ip、spy）→ 主机存活性检测（ICMP、TCP）→ 端口存活性检测（TCP、UDP）→ TCP 协议识别（NMAP + 自定义指纹库，攻击包探针 149 个、指纹规则 11891 条、可检测协议 1347 种）→ 应用层指纹识别（HTTP 抓框架/源码/组件/标题，RPC 获取主机名，应用层指纹 20000+）→ 分发器 → 暴力破解模块（hydra，支持 RDP、Telnet、FTP 等 10 余种协议）/ 漏洞利用模块（仍在开发中）

## 二、免责声明
- 本工具仅面向**合法授权的企业安全建设行为与个人学习行为**，如需要测试本工具的可用性，请自行搭建靶机环境。
- 在使用本工具进行检测时，应确保该行为符合当地法律法规，并且已经取得了足够的授权。**请勿对非授权目标进行扫描。**

## 三、编译安装
官方提供编译手册（见项目 README 中的「编译手册」章节），可直接下载 Release 预编译版本使用。

## 四、使用方法

### 命令总览
```

usage: kscan [-h,--help,--fofa-syntax] (-t,--target,-f,--fofa,--spy) [-p,--port|--top] [-o,--output] [-oJ] [--proxy] [--threads] [--path] [--host] [--timeout] [-Pn] [-Cn] [-sV] [--check] [--encoding] [--hydra] [hydra options] [fofa options]

```

### 三种输入目标方式
**1. `-t/--target`：指定探测对象**（可添加 `--check` 参数，只对指定的目标端口进行指纹识别；否则对目标进行端口扫描和指纹识别）
```
IP地址：114.114.114.114
IP地址段：114.114.114.114/24（不建议子网掩码小于 12）
IP地址段：114.114.114.114-115.115.115.115
URL地址：https://www.baidu.com
文件地址：file:/tmp/target.txt（新版本 -t 可直接输入文件路径，无需 file: 前缀）
```

**2. `--spy`：网段探测模式**（自动探测主机可达的内网网段；可添加 `--scan` 参数对存活 C 段进行端口扫描和指纹识别，否则只检测存活的网段）
```
[空]：检测本机 IP 地址，对本机 IP 所在 B 段进行探测
[all]：对所有私网地址（192.168/172.32/10 等）进行探测
192 / 10 / 172：对对应网段进行探测
指定IP地址：对指定 IP 地址所在 B 段进行探测
```

**3. `-f/--fofa`：FOFA 检索**（可添加 `--check` 对检索结果进行存活性验证，添加 `--scan` 对检索结果进行端口扫描和指纹识别，否则只返回 fofa 检索结果）
```
fofa搜索关键字：将直接返回 fofa 搜索结果
```

### 参数详解
```
optional arguments:
 -h , --help          显示帮助信息
 -f , --fofa          从 fofa 获取检测对象，需提前配置环境变量:FOFA_EMAIL、FOFA_KEY
 -t , --target        指定探测对象（IP/网段/URL/文件）
 --spy                网段探测模式，可接收参数为：(空)、192、10、172、all、指定IP地址
 --check              针对目标地址做指纹识别，仅不进行端口探测
 --scan               针对 --fofa、--spy 提供的目标对象，进行端口扫描和指纹识别
 -p , --port          扫描指定端口，默认扫描 TOP400，支持：80,8080,8088-8090
 -eP, --excluded-port 跳过扫描指定的端口，支持：80,8080,8088-8090
 -o , --output        将扫描结果保存到文件
 -oJ                  将扫描结果使用 json 格式保存到文件
 -Pn                  使用后不进行智能存活性探测（默认开启智能存活性探测，提高效率）
 -Cn                  使用后控制台输出结果不带颜色
 -Dn                  使用后关闭 CDN 识别功能
 -sV                  使用后对所有端口进行全探针探测，此参数极度影响效率，慎用！
 --top                扫描经过筛选处理的常见端口 TopX，最高支持 1000 个，默认为 TOP400
 --proxy              设置代理 (socks5|socks4|https|http)://IP:Port
 --threads            线程参数，默认线程 100，最大值为 2048
 --path               指定请求访问的目录，只支持单个目录
 --host               指定所有请求的头部 Host 值
 --timeout            设置超时时间
 --encoding           设置终端输出编码，可指定为：gb2312、utf-8
 --match              对资产返回 banner 进行检索，存在关键字的才会显示，否则不显示
 --hydra              自动化爆破，默认会开启全部协议

hydra options:
 --hydra-user         自定义 hydra 爆破用户名: username or user1,user2 or file:username.txt
 --hydra-pass         自定义 hydra 爆破密码: password or pass1,pass2 or file:password.txt
                      （若密码中存在逗号，使用 \, 进行转义，其他符号无需转义）
 --hydra-update       自定义用户名、密码模式；携带此参数为新增模式，将用户名和密码补充在默认字典后面，
                      否则将替换默认字典
 --hydra-mod          指定自动化暴力破解模块: rdp or rdp,ssh,smb

fofa options:
 --fofa-syntax        获取 fofa 搜索语法说明
 --fofa-size          设置 fofa 返回条目数，默认 100 条
 --fofa-fix-keyword   修饰 keyword，该参数中的 {} 最终会替换成 -f 参数的值
```

### 暴力破解支持协议
ssh、rdp、ftp、smb、mysql、mssql、oracle、postgresql、mongodb、redis

## 五、典型使用示例
```bash
# 端口扫描 + 指纹识别（默认 TOP400 端口）
kscan -t 192.168.21.1/24

# 指定端口扫描
kscan -t 192.168.21.1/24 -p 80,443,8080-8090

# 存活网段探测（探测本机所在 B 段网关）
kscan --spy

# 指定网段探测
kscan --spy 172

# fofa 检索（需配置 FOFA_EMAIL、FOFA_KEY 环境变量）
kscan -f "title:\"后台管理\"" --fofa-size 15

# fofa 检索结果 + 存活性检测及指纹识别
kscan -f "title:\"后台管理\"" --check

# fofa 检索结果 + 端口扫描
kscan -f "title:\"后台管理\"" --scan

# 开启暴力破解（全部协议）
kscan -t 127.0.0.1 --hydra

# 指定爆破协议与字典
kscan -t 127.0.0.1 --hydra --hydra-mod ssh,rdp --hydra-user admin --hydra-pass file:pass.txt

# 指定输出文件
kscan -t 192.168.21.1/24 -o result.txt

# JSON 格式输出
kscan -t 192.168.21.1/24 -oJ result.json
```

> 💡 **Tips**：
> - 使用 `-f/--fofa` 参数时，若搜索关键字中含有 `&` 或 `"` 等符号，应使用单引号将所有参数包裹，或使用 `\` 进行转义。
> - Windows 系统下可新增环境变量 `KSCAN_COLOR=TRUE` 开启颜色显示。


