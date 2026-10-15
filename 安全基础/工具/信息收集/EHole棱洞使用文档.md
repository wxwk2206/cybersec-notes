# EHole（棱洞）：红队重点攻击系统指纹探测工具

> 项目地址：https://github.com/EdgeSecurityTeam/EHole
> 作者：shihuang（开源版 @shihuang）
> 当前开源版本：3.0（Usage 显示 version 2.0）
## 一、简介
EHole 是一款对资产中**重点系统指纹识别**的工具。在红队作战中，信息收集是必不可少的环节，如何从大量资产中提取有用的系统（如 OA、VPN、Weblogic...）是关键。EHole 旨在帮助红队人员在信息收集期间，快速从 C 段、大量杂乱的资产中**精准定位到易被攻击的系统**，从而实施进一步攻击。
### 3.0 版本更新内容
1. 多个 CMS 指纹识别（一个 URL 同时匹配多个 CMS 指纹时会同时显示）
2. 颜色高亮（识别到的系统以红色显示，更易区分）
3. 新增 FOFA 批量端口提取（拿到一万甚至十万 IP 时可使用 `-fall` 批量从 FOFA 提取端口信息）
4. 优化多个识别规则，更易识别重点系统
5. 新增多条指纹，目前能识别大部分常见的系统
6. 优化文件读取，更快速准确地识别内容
7. FOFA 语法搜索优化，使用 `-fofa` 参数快速从 fofa 提取资产进行识别
## 二、使用

> ⚠️ **重要**：GitHub Release 下载的 3.0 版二进制基于 cobra 框架重构，**必须使用子命令** `finger` / `fofaext`。README 里 `EHole -l url.txt` 是 v2.0 老版本用法，与当前二进制不匹配，直接执行会报 `unknown shorthand flag`。

> 配合「红队中易被攻击的一些重点系统漏洞整理」食用效果更佳：
> https://forum.ywhack.com/bountytips.php?Vulnerability

### 命令结构
```
ehole
├── finger    指纹识别模块（本地文件 / fofa / 单 URL）
├── fofaext   从 fofa API 提取资产并保存为 xlsx
└── help
```

### 1. finger 子命令（指纹识别）

```
Usage: ehole finger [flags]

Flags:
  -l, --local string   从本地文件读取资产识别（支持无协议，如 192.168.1.1:9090 | http://192.168.1.1:9090）
  -f, --fip string     从 fofa 提取资产识别，仅支持 IP / IP 段（192.168.1.1 | 192.168.1.0/24）
  -s, --fofa string    从 fofa 提取资产识别，支持 fofa 全部语法
  -u, --url string     识别单个目标
  -o, --output string  输出结果（仅支持 json / xlsx 后缀）
  -t, --thread int     指纹识别线程数（默认 100）
  -p, --proxy string   访问目标时指定代理，支持 http 和 socks5（http://127.0.0.1:8080 | socks5://127.0.0.1:8080）
```

**本地文件识别：**
```powershell
.\EHole_windows_amd64.exe finger -l url.txt
```
> 每行一个目标，可带协议也可不带（不带协议自动补 https://）。

**单个 URL 识别：**
```powershell
.\EHole_windows_amd64.exe finger -u https://www.example.com
```

**FOFA 识别**（需在 `config.ini` 配置好 FOFA 邮箱与密钥）：
```powershell
# 按 IP / IP 段识别
.\EHole_windows_amd64.exe finger -f 192.168.1.0/24

# 按 fofa 语法识别
.\EHole_windows_amd64.exe finger -s 'title="后台管理"'
```

**结果输出 / 线程 / 代理：**
```powershell
.\EHole_windows_amd64.exe finger -l url.txt -o result.json
.\EHole_windows_amd64.exe finger -l url.txt -t 200
.\EHole_windows_amd64.exe finger -l url.txt -p http://127.0.0.1:8080
```

### 2. fofaext 子命令（fofa 资产提取）
```
Usage: ehole fofaext [flags]

Flags:
  -l, --ipfile string   从文本读取大量 IP 去 fofa 搜索
  -s, --fofa string     从 fofa 提取资产，支持全部语法
  -o, --output string   输出文件名（仅支持 xlsx，默认 results.xlsx）
```

```powershell
.\EHole_windows_amd64.exe fofaext -l ips.txt -o assets.xlsx
.\EHole_windows_amd64.exe fofaext -s 'app="xxx"' -o result.xlsx
```
### 3. 使用注意
- 直接运行 `ehole`（不带子命令）没有任何动作，必须跟 `finger` 或 `fofaext`
- `--config` 参数是 cobra 模板遗留的占位（默认 `$HOME/.ehole.yaml`），实际 FOFA 配置仍读取 exe 同目录下的 `config.ini`
- 目标文件为空时程序只打印「重点资产：」而无结果，注意检查输入文件非空

## 三、finger.json 指纹文件的使用

### 1. 加载机制（源码确认）
EHole 在启动扫描时自动加载指纹文件（`module/finger/finger.go`）：
```go
err := LoadWebfingerprint(source.GetCurrentAbPathByExecutable() + "/finger.json")
if err != nil {
    color.RGBStyleFromString("237,64,35").Println("[error] fingerprint file error!!!")
    os.Exit(1)
}
```
要点：
- **文件名必须为 `finger.json`**，放在**可执行文件所在目录**（`GetCurrentAbPathByExecutable` 取运行程序目录，非当前工作目录），与 `config.ini` 同目录
- 加载失败会报 `[error] fingerprint file error!!!` 并直接退出，程序不运行
- **官方原版没有命令行参数指定指纹文件路径**，路径在源码中写死

### 2. 如何更换指纹规则
- **替换法（不改代码）**：文件名固定为 `finger.json`，但内容可随意更换。可准备多套规则（如 `finger_oa.json`、`finger_vpn.json`），用哪套就复制改名为 `finger.json` 放到 exe 目录（注意先备份原文件）
- **改源码重新编译**：编辑 `module/finger/finger.go` 中的加载路径，或自行添加 `-finger` 参数，然后 `go build`

### 3. 真实 JSON 结构
仓库内 `finger.json` 的**顶层结构是 `fingerprint` 数组**（README 示例只有单个对象，易误导）：
```json
{
    "fingerprint": [
        {
            "cms": "致远OA",
            "method": "keyword",
            "location": "body",
            "keyword": ["/seeyon/USER-DATA/IMAGES/LOGIN/login.gif"]
        },
        {
            "cms": "禅道 zentao",
            "method": "keyword",
            "location": "title",
            "keyword": ["Welcome to use zentao"]
        },
        {
            "cms": "Dreamer CMS",
            "method": "faviconhash",
            "location": "body",
            "keyword": ["-748565678"]
        }
    ]
}
```

对应代码结构：
```go
type Packjson struct {
    Fingerprint []Fingerprint
}
type Fingerprint struct {
    Cms      string
    Method   string
    Location string
    Keyword  []string
}
```

### 4. 字段说明（含与 README 的差异，以代码为准）

| 字段 | 说明 |
|---|---|
| `cms` | 系统名称（如 致远OA、禅道、Weblogic、Sangfor SSL VPN） |
| `method` | 识别方式，代码实际支持：`keyword`、`faviconhash`、`regular`（README 写的 `regula` 为笔误） |
| `location` | 匹配位置，代码实际支持：`body`、`header`、`title`（README 只写 body/header，内置指纹大量使用 `title`） |
| `keyword` | 关键字数组（favicon 图标 hash / 正则 / 关键字） |

⚠️ **注意**：
- keyword 支持多关键字匹配，需**所有关键字**都匹配上才识别。如：
```json
"keyword": ["sys/ui/extend/theme/default/style/icon.css", "sys/ui/extend/theme/default/style/profile.css"]
```
- `faviconhash` 方式仅比较 `keyword[0]` 与计算出的 favicon 哈希值
- `keyword` 方式在 `location=body` 时匹配响应正文，`location=header` 时匹配响应头，`location=title` 时匹配页面标题
## 四、使用效果

**fofa 识别示例（输出字段）：**
```
http://目标IP:端口 | 识别到的CMS | 服务器指纹 | 状态码 | 长度 | 标题/详情
```
扫描结果会区分「重点资产」板块，将识别到的重点系统（如致远 OA、Tomcat、锐捷等）单独高亮列出，并在最后显示程序运行时间。

**输出效果字段说明：**

| 字段 | 含义 |
|---|---|
| 第一个字段 | 目标 URL（协议://IP:端口） |
| 第二个字段 | 识别到的 CMS/系统名称（如 seeyon、Tomcat 默认页面、Ruijie） |
| 第三个字段 | 服务器指纹（如 Apache-Coyote/1.1、nginx、Seeyon-Server/1.0） |
| 第四个字段 | HTTP 状态码（200/400/403/404 等） |
| 第五个字段 | 响应长度 |
| 第六个字段 | 页面标题或详细系统信息（如「致远A6-V5协同管理软件 V5.6SP1」） |

## 五、参考
- 项目地址：https://github.com/EdgeSecurityTeam/EHole
- License：Apache-2.0
