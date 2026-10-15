# HaE 使用教程

> 项目地址：[https://github.com/overspace-labs/HaE](https://github.com/overspace-labs/HaE)
> 整理自实战使用经验，聚焦 HaE（网络版 / HaENet）在 Burp Suite 中的用法。
> 适用版本：HaE Network 3.0+（基于 Montoya API）

---

## 一、HaE 是什么

**概括**：HaE 是一套面向网络安全（数据安全）领域的「标记 + 提取」框架，用**乐高积木式**模块化设计，对 HTTP 报文（含 WebSocket）和本地文件做精细化高亮标记与信息抽取，帮你在海量流量里快速锁定高价值报文。

**比喻**：Burp 抓包把整座沙堆倒给你，HaE 是那台带彩色探照灯的淘金筛——按你写好的正则规则扫过每条报文，把"像金子"的内容用不同颜色标亮，并把关键信息抽出来摆到一张面板上。

**细节**：
- 核心机制 = **多引擎自定义正则**：
  - **DFA 引擎**（Deterministic Finite Automaton，确定性有限自动机）：对文本每个字符只扫一次，速度快、特性少。
  - **NFA 引擎**（Nondeterministic Finite Automaton，非确定性有限自动机）：反复标注/取消标注字符，速度慢但特性丰富（分组、替换、分割）。
- 匹配成功后做三件事：**颜色高亮标记**、**注释**、**信息提取**，结果集中到数据面板一键查询。
- 内置「颜色升级算法」：同一颜色重复出现会自动升一级，避免满屏同色分不出重点。

### 仓库架构（别搞错主仓）
| 仓库 | 角色 | 说明 |
|---|---|---|
| `overspace-labs/HaE` | 品牌主仓 | 只放简介与导航，源码以 submodule 挂载 |
| `overspace-labs/HaENet` | 网络版 | **Burp Suite 扩展**，处理 HTTP/WebSocket 报文 |
| `overspace-labs/HaEFile` | 文件版 | 处理本地文件 |
| `HaEValidator` | 验证器（AI+） | 对匹配结果做严重程度分级 |

> ⚠️ 你给的 `HaE/README_CN.md` 是品牌主仓，详细用法在 **HaENet** 仓库里。实际装的是 HaENet 的 Jar 包。

---

## 二、核心原理：被动分析

**关键点**：HaE Network 是**纯被动（passive）**工具，它**不会主动发送任何数据包**，只分析「流经 Burp 的流量」。

| 流量来源 | 是否会被 HaE 分析 |
|---|---|
| 浏览器通过代理的正常浏览 | ✅ |
| Repeater 手动改包 | ✅ |
| Intruder 爆破请求 | ✅ |
| Spider / Crawler 生成的请求 | ✅（但这是 Burp 发的，不是 HaE） |
| HaE 自己主动发出 | ❌ 不会 |

> 含义：HaE 的覆盖度 = 你的流量覆盖度。想挖出更多接口，要么手动多点点，要么用 Burp 的 Spider/Crawl 把站点走过一遍（Burp 发，HaE 看），或者用独立 Linkfinder 命令行工具递归抓 JS 后丢给 HaEFile。

---

## 三、安装步骤

1. ⭐ 确认 **Burp Suite ≥ 2023.12.1**（v3.0 起用 Montoya API 开发，老版本跑不起来）。
2. 下载 HaENet 的 Jar 包。
3. 在 Burp 里：`Extender（扩展）` → `Extensions（扩展）` → `Add（添加）` → `Select File（选择文件）` → `Next`。
4. ⭐ 首次加载会从 Jar 内加载**离线规则库**；想更新规则点 `Reinit` 重新初始化。

```bash
# 带 submodule 克隆主仓（仅当你要读源码时）
git clone --recurse-submodules https://github.com/overspace-labs/HaE.git
```

---

## 四、配置文件位置

| 系统 | 路径 |
|---|---|
| Windows | `%USERPROFILE%/.config/HaE/` |
| Linux / Mac | `~/.config/HaE/` |

⭐ 也可以把配置文件放在 **Jar 包同级目录的 `/.config/HaE/`** 下，方便离线携带。
- `Config.yml`：总配置
- `Rules.yml`：规则库（内置规则地址：`HaENet/.../resources/rules/Rules.yml`）

---

## 五、规则字段（9 字段）

⭐ **最关键的坑**：提取的内容必须用 `()` 包起来。例如匹配 Shiro 应用，普通写法是 `rememberMe=delete`，在 HaE 里必须写成 `(rememberMe=delete)`。

| 字段 | 含义 |
|---|---|
| **Name** | 规则名，简短概括用途 |
| **F-Regex** | 主正则，需提取部分用 `()` 包裹 |
| **S-Regex** | 二次正则，对 F-Regex 结果再提取（可留空） |
| **Format** | 格式化输出，`{0}`/`{1}` 取分组，默认 `{0}` |
| **Scope** | 作用域：请求/响应的行、头、体或完整报文 |
| **Engine** | 正则引擎：DFA（快）或 NFA（特性丰富） |
| **Color** | 命中时高亮颜色，内置升级算法防撞色 |
| **Sensitive** | 是否大小写敏感（True / False） |
| **Validator** | 外部验证器，按 high/medium/low/none 分级（Command / Timeout / Bulk 三个子项），通过 stdin/stdout 在本地跑程序，**不发包** |

### 内置规则库常用规则速查
| 规则名 | 颜色 | 检测目标 |
|---|---|---|
| Shiro | 绿 | Apache Shiro 框架（可后续测反序列化） |
| JSON Web Token | 绿 | JWT 凭证 |
| Swagger UI | 红 | 接口文档暴露（高危） |
| Druid | 橙 | 阿里 Druid 监控页未授权访问 |
| Java Deserialization | 黄 | 反序列化特征（疑似 RCE） |
| Upload Form | 黄 | 文件上传点 |
| Passwd File | 红 | Linux `/etc/passwd` 内容（任意文件读取） |
| Win.ini File | 红 | Windows `win.ini` 内容 |
| Vite DevMode | 红 | 前端 Vite 开发模式未关闭 |
| Debug Logic Parameters | 青 | 调试参数（access / adm 等） |
| URL As A Value / Linkfinder | 青 | URL 作为值 / 链接端点提取 |

---

## 六、界面说明（4 个面板）

| 面板 | 作用 |
|---|---|
| **Rules（规则管理）** | 查看/编辑/启停规则，核对 `Name ↔ Color` 映射 |
| **Config（配置管理）** | 全局配置 |
| **Databoard（数据集合）** | 所有标记/提取结果集中地，按分类分标签页 |
| **MarkInfo（数据展示）** | 单条报文命中详情，按规则名分标签页（如 `All URL (3)`、`Password Field (1)`、`Linkfinder (9)`） |

### Databoard 分类标签页
| 标签页 | 含义 |
|---|---|
| **Fingerprint（指纹）** | 识别到的技术栈/组件（Shiro、Swagger、Druid…） |
| **Maybe Vulnerability（疑似漏洞）** | 需人工验证的风险点（上传、删除、预览、反序列化入口） |
| **Basic Information（基础信息）** | 站点基础信息（根路径、域名、静态资源） |
| **Sensitive Information（敏感信息）** | 泄露的敏感内容（passwd、JWT、API Key、密码字段） |
| **其他** | 未归类结果 |

---

## 七、HTTP 历史里的颜色含义

Burp 列表里的行颜色 = **某条 HaE 规则命中后自动打上的高亮**，颜色由该规则的 `Color` 字段决定，**没有官方固定的"红=危险"含义**。

- 想知道某个颜色对应哪条规则 → 打开 HaE 的 `Rules` 面板，核对 `颜色` 列与 `Name` 列的映射。
- ⭐ 建议把真正高危的规则（Swagger UI、Passwd File、Java Deserialization、Druid）配成更扎眼的颜色，方便一眼定位。

---

## 八、实战案例：从首页挖隐藏接口

**场景**：访问 `GET /` 首页，响应看起来"什么都没有"，但 HaE 的 `Linkfinder` 标签挖出 9 条结果。

**原理**：请求虽简单，但服务器返回的 HTML/JS 源码里**硬编码**了 favicon、CSS、以及 `onlinePreview?url=`、`deleteFile?fileName=` 等接口地址。Linkfinder 规则用正则在**响应体**里把它们全部提取出来。

**提取结果示例**：

| 内容 | 类型 | 安全看点 |
|---|---|---|
| `./favicon.ico`、`image/x-icon` | 静态资源 / MIME | 无攻击面 |
| `css/loading.css`、`css/theme.css` | 静态资源 | 无攻击面 |
| `https://.../onlinePreview?url=` | 功能接口 | ⭐ 文件预览，可能存在 SSRF / 任意文件读取 |
| `https://.../deleteFile?fileName=` | 功能接口 | ⭐ 文件删除，可能存在任意文件删除 / 越权 |
| `https://.../`、`/index` | 站点根 / 入口 | 应用入口 |

**后续验证方向**（原理 → 条件 → 复现 → 修复思路）：
1. **onlinePreview?url=**：尝试 `file:///etc/passwd` 或外网地址 → 条件：参数未校验协议/白名单 → 复现：替换 url 值看是否回显文件内容 → 修复：限制协议 + 路径白名单 + 鉴权。
2. **deleteFile?fileName=**：尝试 `../` 目录遍历、未登录直接调用 → 条件：无权限校验 / 路径拼接 → 复现：传 `../../etc/passwd` 或他人文件 → 修复：鉴权 + 路径规范化 + 限定目录。

---

## 九、使用流程总结（序号步骤）

1. 安装 HaENet Jar 到 Burp（≥2023.12.1）。
2. 用浏览器 / Spider 让目标流量流经 Burp（HaE 被动分析）。
3. 在 Burp HTTP 历史里通过**颜色**快速筛选高价值报文。
4. 进 HaE 的 **MarkInfo / Databoard** 查看按规则/分类归好的命中结果。
5. 重点切到 `Maybe Vulnerability` 和 `Sensitive Information` 标签做人工验证。
6. 按需自定义规则（记得 `()` 包裹提取内容），把高危规则配醒目颜色。

---

## 十、注意事项

- ⚐ HaE 只分析已有流量，不主动探测；想扩大覆盖要配合 Burp Spider 或主动扫描器。
- ⚐ 自定义规则必须用 `()` 包裹提取内容，否则提取为空。
- ⚐ `Scope` 支持请求和响应两部分，规则可只针对请求（如 Debug 参数）或只针对响应。

