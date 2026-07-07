<div align="center">

# apiSQL Skill

**让 AI Agent 用一个 HTTP 接口操作数据库 —— 能做什么，由你的授权说了算。**

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](./LICENSE)
[![powered by apiSQL](https://img.shields.io/badge/powered%20by-apiSQL-22d3ee.svg)](https://www.apisql.cn)
[![databases](https://img.shields.io/badge/databases-13%2B-blue.svg)](./SKILL.md)
[![demo](https://img.shields.io/badge/demo-零注册可用-orange.svg)](https://open.apisql.cn)

[官网](https://www.apisql.cn) · [在线云平台](https://open.apisql.cn) · [官方文档](https://docs.apisql.cn) · [SKILL.md](./SKILL.md)

</div>

---

**apiSQL Skill** 让 AI Agent 通过一个 HTTP 接口读写多种数据库：把 SQL 语句作为文本
放进 JSON 请求体，apiSQL 执行后返回结果（查询返回行，写操作返回影响行数）。无需为每
条 SQL 预先定义 REST 路由，同时支持命名参数安全传值。

本仓库把 apiSQL 的 SUDB（Super Database API）子功能完整用法打包成一个
**Skill**（[`SKILL.md`](./SKILL.md)）：复制给 Claude、ChatGPT、Cursor 等 AI，它就能读
懂接口，用**公开演示环境**帮你查询数据库，也可以接到**你自己的数据库**上——零注册即
可体验，接生产只需替换三处配置。

## 🔐 能做什么，由授权决定

看到"AI 直接执行 SQL"先别紧张——AI 通过这个接口能触达的范围**不是"任意"**，而是被
两层授权严格限定的：

| 授权层 | 由谁控制 | 决定了什么 |
| :--- | :--- | :--- |
| **数据库账号授权** | 你给 apiSQL 配置的数据库账号 | 只读 / 表级 / 库级——账号没有的权限，AI 一定做不到 |
| **apiSQL 访问控制策略** | 你在 apiSQL 里配的接口策略 | API Key / IP 白名单 / JWT，还能限定只放行 SUDB 接口组 |

此外，apiSQL 会留存完整的**系统操作日志**，每一次调用可追溯、可审计。

> **把授权交给数据库和 apiSQL 本身。** 给 AI 一个只读账号，它就只能查；给它表级写
> 权限，它也越不出那张表。越权操作会在数据库层被直接拒绝——这正是安全边界在起作用。

配套的 [`SKILL.md`](./SKILL.md) 还为 AI 设定了默认策略：**默认只做只读查询，写操作需
用户显式请求、并先复述影响范围再执行**。

## 🚀 快速开始

### 方式一：复制提示词（最快）

把下面这段发给任意 AI，它会自己读懂用法并开始操作演示数据库：

```text
读取 https://raw.githubusercontent.com/apisql-dev/apisql-skill/main/SKILL.md
学会用法后，帮我在演示库上 SHOW TABLES，并查 user 表前 5 条。
```

### 方式二：安装为本地 Skill（Claude Code / Cursor 等）

让支持 Skill 的 AI 编程助手把本仓库的 `SKILL.md` 下载到本地技能目录，长期可用：

```bash
# 例：放入 Claude 的 skills 目录
mkdir -p ~/.claude/skills/apisql-sudb
curl -sSL https://raw.githubusercontent.com/apisql-dev/apisql-skill/main/SKILL.md \
  -o ~/.claude/skills/apisql-sudb/SKILL.md
```

### 方式三：直接 curl 体验

```bash
export APISQL_DEMO_KEY="sk-93e23231a9c8257f2c21854531f42b0a"   # 公开演示 Key

curl -sS -X POST "https://open.apisql.cn/api/demo-area/\$sudb" \
  -H "Content-Type: application/json;charset=UTF-8" \
  -H "Authorization: Bearer $APISQL_DEMO_KEY" \
  --data-binary '{"meta":{"ds":"mysql","sc":"SELECT * FROM user","pageNum":1,"pageSize":10}}'
```

> [!TIP]
> 演示环境的 API Key 是**有意公开**的，可直接使用；提供 `mysql` 与 `postgresql`
> 两个数据源，并已做严格写保护（仅 MySQL `user` 表可写，PostgreSQL 整库只读）——正
> 好演示了"授权决定权限"。完整的请求/响应结构、安全须知、更多示例见
> [`SKILL.md`](./SKILL.md)。

## 🛠️ 用在你自己的数据库上

请求结构与行为完全一致，只需替换三处：

| 替换项 | 换成 |
| :--- | :--- |
| **接口地址** | 你自己的网关地址与接口路径 |
| **数据源名 `ds`** | 你在 apiSQL 里配置的数据源名 |
| **API Key** | 你自己的 Key |

接生产前，先把授权配好（这是安全的关键）：

1. **用专用数据库账号连接**，按最小必要原则授权——只读分析就给只读账号，需要写入再
   按表级授予，不要用 root / 管理员账号。
2. **配置 apiSQL 访问控制策略**：为 SUDB 接口组绑定 API Key 或 IP 白名单；用 JWT
   可进一步限定只放行 SUDB 接口组并设置过期时间。
3. **区分环境**：先在测试库验证，再切生产。

> 搭建步骤：[open.apisql.cn](https://open.apisql.cn) 注册 → 装网关连库 → 加数据源 →
> 启用 SUDB → 建访问控制策略取 Key，详见[官方文档](https://docs.apisql.cn)。

## 🗄️ 支持的数据库

| 类型 | 数据库 |
| :--- | :--- |
| **关系型 / OLTP** | MySQL、Oracle、SQL Server、PostgreSQL、SQLite |
| **分析型 / OLAP** | Apache Doris、SelectDB、StarRocks、TiDB、华为 DWS(GaussDB) |
| **国产 / 其他** | 达梦(DM)、Trino、PrestoSQL |

> 企业版还支持**自定义 JDBC 驱动**接入：只要目标数据库提供 JDBC 驱动即可使用，例如
> OceanBase、人大金仓(KingbaseES) 等。

## 🔗 相关链接

- 🌐 官网：[www.apisql.cn](https://www.apisql.cn)
- ☁️ 在线云平台：[open.apisql.cn](https://open.apisql.cn)
- 📖 官方文档：[docs.apisql.cn](https://docs.apisql.cn)
- 🔌 MCP Server：[apisql-mcp](https://github.com/apisql-dev/apisql-mcp)

## 📄 License

[MIT](./LICENSE) © apiSQL
