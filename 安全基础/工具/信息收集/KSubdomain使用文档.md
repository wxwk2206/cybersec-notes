# KSubdomain：极速无状态子域名爆破工具

> 项目地址：https://github.com/boy-hack/ksubdomain
> 原项目：knownsec/ksubdomain

## 一、简介
KSubdomain 是一款基于**无状态技术**的子域名爆破工具，采用原始套接字（Raw Socket）直接操作网络适配器，绕过系统内核协议栈，配合可靠的状态表重发机制保证结果完整性。支持 Windows、Linux、macOS，是大规模 DNS 资产探测的首选工具。
### 核心优势
- **闪电般的速度**：无状态扫描直接操作网卡，速度是 massdns 的 7 倍、dnsx 的 10 倍以上
- **极低资源消耗**：对象池 + 全局内存池，海量域名处理依然低内存占用
- **动态超时自适应**：基于 TCP RFC 6298 RTT 滑动均值（EWMA）动态调整超时，无需手动调参
- **多网卡并发发包**：支持 `--interface` 重复指定多张网卡，叠加带宽、利用多个出口 IP
- **流式 SDK**：提供 `EnumStream`/`VerifyStream` 回调 API，实时处理结果
- **跨平台支持**：Windows、Linux、macOS 完美兼容

### 性能对比（4 核 CPU、5M 带宽、10 万字典）

| 工具 | 方式 | 耗时 | 成功数 |
|---|---|---|---|
| KSubdomain | pcap 网卡发包 | ~30 秒 | 1397 |
| massdns | pcap/socket | ~3 分 29 秒 | 1396 |
| dnsx | socket | ~5 分 26 秒 | 1396 |

## 二、安装
### 1. 下载预编译二进制
前往 GitHub **Releases** 页面下载对应系统的最新版本。
**依赖安装：**
- **Windows**：安装 [Npcap](https://npcap.com/) 驱动
- **Linux**：已静态编译，通常无需操作；如有问题安装 `libpcap-dev`
- **macOS**：系统自带 libpcap，无需安装
### 2. 源码编译
```bash
git clone https://github.com/boy-hack/ksubdomain.git
cd ksubdomain
# 建议通过 ldflags 注入版本号
go build -ldflags "-X github.com/boy-hack/ksubdomain/v2/pkg/core/conf.Version=v2.x.y" \
 -o ksubdomain ./cmd/ksubdomain
```

或使用 `go install`：
```bash
go install github.com/boy-hack/ksubdomain/v2/cmd/ksubdomain@latest
```
⚠️ **注意**：需要 root / `CAP_NET_RAW` 权限运行（原始套接字抓包）。

## 三、使用说明
```
用法:
 ksubdomain [全局选项] <命令> [命令选项]

命令:
 enum, e    枚举模式：对主域名进行子域名爆破
 verify, v  验证模式：验证域名列表是否解析
 test       测试本地网卡最大发包速度
 device     查看可用网卡信息
 help, h    显示帮助

全局选项:
 --help, -h     显示帮助
 --version, -v  打印版本信息
```

### 1. 验证模式（verify）
验证提供的域名列表是否有 DNS 解析记录。
```
选项:
 --domain, -d            指定域名（可重复）
 --filename, -f          从文件读取域名列表
 --stdin                 从标准输入读取
 --bandwidth, -b         带宽限制，如 5m、10m、100m（默认 3m）
 --resolvers, -r         指定 DNS 服务器（默认使用内置）
 --output, -o            输出文件路径
 --format, -f            输出格式：txt（默认）、json、csv、jsonl
 --silent, -s            安静模式，仅输出域名（不显示 banner 和日志）
 --only-domain, --od     只输出域名，不显示 IP/记录
 --retry                 重试次数，-1 表示无限重试（默认 3）
 --timeout               单次查询超时秒数（默认 6）
 --interface, -e         指定网卡名（可重复，多网卡并发）
 --wildcard-filter       泛解析过滤：none（默认）、basic、advanced
 --predict               启用子域名预测模式
 --quiet, -q             不打印结果到屏幕（仅写文件）
 --color, -c             彩色输出
```

**示例：**
```bash
# 验证单个/多个域名
./ksubdomain v -d www.example.com -d mail.example.com

# 从文件读取，保存为 txt
./ksubdomain v -f domains.txt -o output.txt

# 管道输入，带宽 10M，静默模式对接下游工具
cat domains.txt | ./ksubdomain v --stdin -b 10m -s --od | httpx -silent

# 高级泛解析过滤，输出 JSONL
./ksubdomain v -f domains.txt --wildcard-filter advanced --format jsonl -o output.jsonl

# 多网卡并发（叠加带宽）
./ksubdomain v -f domains.txt --interface eth0 --interface eth1
```

### 2. 枚举模式（enum）

基于字典和预测算法爆破指定域名下的子域名。
```
选项:
 --domain, -d            目标主域名（可重复）
 --domain-list, --ds     批量主域名文件
 --filename, -f          字典文件路径（默认使用内置字典）
 --use-ns-records        读取域名 NS 记录并加入 DNS 解析器
 （其余选项同 verify 模式）
```

**示例：**
```bash
# 枚举单个域名（使用内置字典）
./ksubdomain e -d example.com

# 指定字典文件
./ksubdomain e -d example.com -f subdomains.txt -o result.txt

# 启用预测模式 + 高级泛解析过滤
./ksubdomain e -d example.com --predict --wildcard-filter advanced

# 批量枚举多个主域名
./ksubdomain e --domain-list roots.txt -b 10m --format jsonl -o result.jsonl

# 管道对接 httpx
./ksubdomain e -d example.com --od -s | httpx -silent
```

### 3. 测试 / 设备命令

```bash
# 测试网卡最大发包速度
./ksubdomain test

# 查看可用网卡
./ksubdomain device
```

## 四、功能特性

### 动态超时自适应
无需手动设置 `--timeout`，引擎自动基于实测 RTT（指数加权移动平均，EWMA α=0.125）动态调整超时上下界，有效减少漏报同时避免不必要等待。

### 多网卡并发
重复指定 `--interface` 即可使用多张网卡同时发包，多个 goroutine 共享同一 `domainChan` 竞争消费，天然实现负载均衡：
```bash
./ksubdomain e -d example.com --interface eth0 --interface eth1 -b 20m
```

### 预测模式
启用 `--predict` 后，工具会根据已发现的子域名预测可能存在的相关子域名，提升发现率。

### 泛解析过滤
- `none`：不过滤（默认）
- `basic`：基础过滤，剔除明显泛解析 IP
- `advanced`：高级过滤，综合多维度判断

### 输出格式

| 格式 | 说明 | 适用场景 |
|---|---|---|
| `txt` | `域名 => 记录`，实时输出 | 默认，人工查看 |
| `json` | 完整 JSON，完成后写入 | 程序解析 |
| `csv` | CSV 表格，完成后写入 | 数据分析 |
| `jsonl` | 每行一条 JSON，实时输出 | 流式处理、管道对接 |

## 五、工具联动示例
```bash
# 联动 httpx 探测存活 Web
./ksubdomain e -d example.com -s --od | httpx -silent

# 联动 nuclei 漏洞扫描
./ksubdomain e -d example.com -s --od | nuclei -l /dev/stdin

# JSONL 流式处理，只取 A 记录
./ksubdomain e -d example.com --format jsonl | jq -r 'select(.type=="A") | .domain'

# 多域名批量 + 去重
./ksubdomain e --domain-list roots.txt -s --od | sort -u > all_subs.txt
```

## 六、平台注意事项

- **macOS**：BPF 缓冲区较小，建议限制带宽
  ```bash
  sudo ./ksubdomain e -d example.com -b 5m
  ```
- **WSL/WSL2**：通常使用 eth0
  ```bash
  ./ksubdomain e -d example.com --interface eth0
  ```
- **Windows**：需先安装 Npcap，以管理员权限运行
  ```
  .\ksubdomain.exe enum -d example.com
  ```

## 七、Go SDK 快速上手
```bash
go get github.com/boy-hack/ksubdomain/v2/sdk
```

```go
import "github.com/boy-hack/ksubdomain/v2/sdk"

scanner := sdk.NewScanner(&sdk.Config{
    Bandwidth:      "5m",
    Retry:          3,
    WildcardFilter: "advanced",
})

// 阻塞式，收集所有结果
results, err := scanner.Enum("example.com")

// 流式回调，实时处理
err = scanner.EnumStream(ctx, "example.com", func(r sdk.Result) {
    fmt.Printf("%s => %v\n", r.Domain, r.Records)
})
```


