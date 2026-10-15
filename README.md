Personal cyber security study notes, Obsidian vault
（个人网络安全学习笔记，Obsidian 知识库）

[![Security Research](https://img.shields.io/badge/Focus-Security%20Research-red?style=flat-square&logo=hackaday)](https://github.com/wxwk2206/cybersec-notes/tree/main/%E5%AE%89%E5%85%A8%E5%9F%BA%E7%A1%80)
[![Java Code Audit](https://img.shields.io/badge/Core-Java%20%E4%BB%A3%E7%A0%81%E5%AE%A1%E8%AE%A1-orange?style=flat-square&logo=openjdk)](https://github.com/wxwk2206/cybersec-notes/tree/main/%E4%BB%A3%E7%A0%81%E5%AD%A6%E4%B9%A0)
[![Blog](https://img.shields.io/badge/Blog-Jekyll%20%2B%20GitHub%20Pages-blue?style=flat-square&logo=githubpages)](https://wxwk2206.github.io/)

## 🛡️ 关于这个仓库

这不是普通的"学习笔记"，而是一套**成体系的安全知识库**——从计算机网络底层到 Java Web 漏洞原理，从 Python 工具链到真实漏洞的完整研究路径，全部按「原理 → 条件 → 复现 → 修复」的研究范式沉淀。

**300+ 篇原创笔记**，持续更新中。

## 🎯 主攻方向

| 方向 | 深度 | 说明 |
| --- | --- | --- |
| 🔍 Java 代码审计 | 核心 | 从 Java Web 基础 → 安全基础 → 常见漏洞的完整审计链路 |
| 💉 Web 漏洞研究 | 核心 | XSS / CSRF / 越权(IDOR) / SQL 注入，按研究范式逐个击破 |
| 🐍 Python 安全开发 | 熟练 | POC 编写、自动化脚本、爬虫、数据分析、AI 辅助研究 |
| ☕ Java 后端工程 | 扎实 | Java 语言核心、MyBatis、Redis、Spring 全家桶 |
| 🌐 计算机基础 | 扎实 | 网络协议、Linux、CLI 工具链、虚拟化环境 |

## 🗂️ 知识库结构

```
学习笔记/
├── 代码学习/
│   ├── Java代码审计/     
│   │   ├── 一.Java Web基础
│   │   ├── 二.Java 安全基础
│   │   └── 三.Java Web常见漏洞
│   ├── Java基础及相关/   # Java+Js+Web+MySQL
│   ├── python/     
│   └── C语言/            # 底层功底
│
├── 安全基础/             ⭐ 核心主线
│   ├── 业务逻辑/         # 业务漏洞挖掘思路与方法论
│   ├── 工具/             # 抓包 / 扫描 / 调试工具指南
|   ├── 黑盒/             # 全类型漏洞学习 / vulhub复现
|   ├──信息收集/
│   └── 基础入门/         # 网安核心概念与数据包分析

├── 漏洞学习/          ⭐ 漏洞基础（原理→条件→复现→修复）
│   ├── 1.XSS/            # 反射型 / 存储型 / DOM 型 XSS 全面拆解
│   ├── 2.CSRF/           # 跨站请求伪造
│   ├── 3.越权 IDOR/      # 越权与不安全直接对象引用
│   └── 4.SQL注入/        # SQL 注入攻防
│
└── 计算机基础/          # 网络 / 系统 / CLI / 环境配置
```

## 🔬 研究方法论

每篇漏洞笔记都遵循同一个框架：
1. **原理** —— 漏洞产生的根本原因是什么
2. **条件** —— 什么场景下会被触发
3. **复现** —— 完整的复现步骤与 payload
4. **修复** —— 从代码层面如何根治

## 📬 交流联系
- **QQ**：2206382290
- **邮箱**： jakupcakalarcon@gmail.com
---
## ❗ 声明

This repository is for security learning purposes only. Any unauthorized attack behavior is prohibited.
本仓库仅用于网络安全学习，禁止用于未授权攻击行为。



