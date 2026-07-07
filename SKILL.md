---
name: apisql-sudb
description:
  通过 apiSQL 的 SUDB（Super Database API），让 AI 用一个 HTTP 接口操作数据库——
  能做什么，由你的授权说了算。权限 = 数据库账号授权 × apiSQL 访问控制策略。AI
  只能在你为它配置的账号权限和接口策略范围内读写数据（查询 / 增删改查 / 聚合 /
  多表 JOIN 等）。默认只做只读查询，写入需用户显式请求。适用于想"用一个接口跑
  SQL"、想体验 apiSQL、想让 AI 在受控范围内读写数据库的场景。触发词：SUDB、
  apiSQL、Super Database API、一个接口跑 SQL、数据库转 API、让 AI 连数据库。支持
  MySQL、Oracle、SQL Server、PostgreSQL、SQLite、Apache
  Doris、SelectDB、StarRocks、TiDB、华为 DWS(GaussDB)、达梦(DM)、Trino、PrestoSQL
  等数据库，企业版还支持自定义 JDBC 驱动接入（如 OceanBase、人大金仓等）。
---

# apiSQL SUDB（Super Database API）技能

SUDB 让 AI 用**一个 HTTP 接口**操作数据库：把 SQL 语句作为文本放进 JSON
请求体，apiSQL 执行后返回结果（查询返回行，写操作返回影响行数）。无需为每条 SQL
预先定义 REST 路由，同时支持命名参数安全传值。

**核心原则——能做什么，由授权决定。** AI 通过这个接口能触达的范围，不是"任意"，
而是被两层授权严格限定的：

1. **数据库账号授权**：apiSQL 连接数据库用的是你指定的数据库账号。你给这个账号授
   什么权限（只读 / 表级 / 库级），AI 就只能做到这一步。越权操作会在数据库层被直接
   拒绝。
2. **apiSQL 访问控制策略**：接口本身还有一层认证与授权（API Key / IP 白名单 /
   JWT），JWT 甚至能限定只放行 SUDB 接口组。

把授权交给数据库和 apiSQL 本身，AI 就只是"在你划定的圈里干活"。这是使用本技能的
安全基线，请始终遵守。

## 支持的数据库

SUDB 底层可对接多种数据库，切换只需改请求体里的 `ds`（数据源名）字段：

MySQL、Oracle、SQL Server、PostgreSQL、SQLite、Apache
Doris、SelectDB、StarRocks、TiDB、华为 DWS(GaussDB)、达梦(DM)、Trino、PrestoSQL 等。

此外，**企业版支持自定义 JDBC 驱动**：只要目标数据库提供 JDBC 驱动，即可接入使用，
例如 OceanBase、人大金仓(KingbaseES) 等国产及各类数据库。

## ⚠️ 安全须知（务必先读）

这是一个"AI 可在授权范围内操作数据库"的能力，请遵守以下默认策略：

1. **默认只做只读查询（SELECT / SHOW / DESC / 聚合 / JOIN）**，除非用户在本次对话
   中明确要求写入。不要主动执行写操作。
2. 用户传入的值一律走 `params` 的 `:命名参数`，**不要拼进 SQL 字符串**（防注入）。
3. 涉及 **INSERT / UPDATE / DELETE / DDL** 时，先向用户复述将影响的表和大致范围，
   得到确认后再执行；带 WHERE 的写操作要提示条件是否符合预期。
4. AI 的实际权限由数据库账号和 apiSQL 策略决定。若返回 `command denied` /
   `read-only transaction` / `permission denied`，说明该操作超出了授权范围——**这是
   预期的保护，不要尝试绕过**，应如实告诉用户"当前账号无此权限"。
5. 用户在**自己的数据库**上使用时，强烈建议从**测试库**或**只读账号**起步，按最小
   必要原则授权到表级，不要用管理员账号连接。

## 公开演示接口

apiSQL 提供了一个开箱即用的公开演示环境，零注册、零配置即可体验：

- 地址：`https://open.apisql.cn/api/demo-area/$sudb`
- 方法：`POST`
- 请求头：`Content-Type: application/json;charset=UTF-8`
- 鉴权头：`Authorization: Bearer <APIKEY>`

演示用 API Key 是公开的（这是有意开放的演示项目）：
`sk-93e23231a9c8257f2c21854531f42b0a`

建议存入环境变量，curl 时用 `$APISQL_DEMO_KEY`
引用，不要在交给用户的脚本里硬编码：

```bash
export APISQL_DEMO_KEY="sk-93e23231a9c8257f2c21854531f42b0a"
```

演示环境目前提供 **MySQL**（`ds=mysql`）与
**PostgreSQL**（`ds=postgresql`）两个数据源，并已通过账号权限与只读事务做了严格的写
保护——正好演示了"授权决定权限"这一点：

- MySQL 通过账号权限控制，仅 `user` 表可增删改，其余表只读。
- PostgreSQL 通过只读事务控制，整库只读（可查询、可用 PostGIS 空间函数，不可写）。

越权操作会在数据库层被拒绝，返回 `command denied` / `read-only transaction` /
`permission denied`——这正是安全边界在起作用。

## 请求体结构

```json
{
  "meta": {
    "ds": "mysql", // 数据源：mysql / postgresql / oracle / ...
    "sc": "SELECT * FROM user", // 要执行的 SQL
    "pageNum": 1, // 可选：SELECT 分页页码
    "pageSize": 10 // 可选：每页条数
  },
  "params": {
    // 可选：为 sc 中的 :命名参数 传值（防注入）
    "myId": 1001
  }
}
```

## 响应结构

查询（SELECT / SHOW / DESC）：

```json
{
  "meta": {
    "pageMode": true,
    "totalCount": 20,
    "offset": 0,
    "limit": 5,
    "thisCount": 5
  },
  "rows": [{ "id": 1, "name": "张三", "age": 30 }]
}
```

写操作（INSERT / UPDATE / DELETE）：

```json
{
  "info": { "affectedRows": 1, "insertId": 8001, "changedRows": 0 },
  "meta": { "affectedRows": 1 },
  "rows": []
}
```

错误（SQL 错误或权限拒绝，从数据库层原样透传）：

```json
{
  "code": "API_GATEWAY_SERVER_ERROR",
  "message": "DELETE command denied to user 'demo_user1'@'...' for table 'area'"
}
```

## 演示库已知事实（实测）

- MySQL 8.0.34，库名 `demodb1`，14 张表（含 `user` 用户表、`area`
  全国行政区划 4.4 万行、 `dw_*` 数仓维度/事实表、`ecom_*` 电商、`ods_*`
  分层等）。仅 `user` 表可写。
- PostgreSQL 13.1，库名 `demodb1`，装有 PostGIS 3.0.2 / pgRouting 3.1.0 /
  pg_cron / hstore，整库为只读事务（可查询、可用 PostGIS 空间函数，不可写）。
- `SHOW TABLES`、`DESC`、`information_schema`、多表 JOIN、聚合、PostGIS 函数（如
  `ST_Distance`、`ST_MakePoint`）均可执行。

## 可直接运行的示例

查询（分页）：

```bash
curl -sS -X POST "https://open.apisql.cn/api/demo-area/\$sudb" \
  -H "Content-Type: application/json;charset=UTF-8" \
  -H "Authorization: Bearer $APISQL_DEMO_KEY" \
  --data-binary '{"meta":{"ds":"mysql","sc":"SELECT * FROM user","pageNum":1,"pageSize":10}}'
```

命名参数条件查询（推荐，防注入）：

```bash
curl -sS -X POST "https://open.apisql.cn/api/demo-area/\$sudb" \
  -H "Content-Type: application/json;charset=UTF-8" \
  -H "Authorization: Bearer $APISQL_DEMO_KEY" \
  --data-binary '{"meta":{"ds":"mysql","sc":"SELECT id,name,age FROM user WHERE age > :minAge ORDER BY age DESC","pageNum":1,"pageSize":5},"params":{"minAge":30}}'
```

切换到 PostgreSQL + PostGIS 空间计算：

```bash
curl -sS -X POST "https://open.apisql.cn/api/demo-area/\$sudb" \
  -H "Content-Type: application/json;charset=UTF-8" \
  -H "Authorization: Bearer $APISQL_DEMO_KEY" \
  --data-binary '{"meta":{"ds":"postgresql","sc":"SELECT name, ROUND((ST_Distance(ST_MakePoint(lon::float8,lat::float8)::geography, ST_MakePoint(113.665412,34.757975)::geography)/1000)::numeric,0) AS km FROM area WHERE level=1 AND lon IS NOT NULL ORDER BY km LIMIT 5"}}'
```

写入（演示库仅 `user` 表可写；写操作应先经用户确认，测试后请清理）：

```bash
curl -sS -X POST "https://open.apisql.cn/api/demo-area/\$sudb" \
  -H "Content-Type: application/json;charset=UTF-8" \
  -H "Authorization: Bearer $APISQL_DEMO_KEY" \
  --data-binary '{"meta":{"ds":"mysql","sc":"INSERT INTO user(id,name,age) VALUE(:myId,:myName,:myAge)"},"params":{"myId":8001,"myName":"小明","myAge":18}}'
```

## 用在用户自己的 apiSQL 上

请求结构和行为完全一致，只需替换三处：

- **接口地址**：换成用户自己的网关地址与接口路径
- **数据源名 `ds`**：换成用户在 apiSQL 里配置的数据源名
- **API Key**：换成用户自己的 Key

**给 AI 用之前，先把授权配好（关键）：**

1. **用专用数据库账号连接**，按最小必要原则授权——只读分析场景就给只读账号，需要
   写入才按表级授予写权限，不要用 root / 管理员账号。
2. **配置 apiSQL 访问控制策略**：为 SUDB 接口组绑定 API Key 或 IP 白名单；用 JWT
   可进一步限定只放行 SUDB 接口组、并设置过期时间。
3. **区分环境**：先在测试库 / 测试数据源验证，再切生产。

搭建步骤（[open.apisql.cn](https://open.apisql.cn) 注册 → 装网关连库 → 加数据源 →
启用 SUDB → 建访问控制策略取 Key），详见[官方文档](https://docs.apisql.cn)。

了解更多：[apiSQL 官网](https://www.apisql.cn) ·
[在线云平台](https://open.apisql.cn) · [官方文档](https://docs.apisql.cn)
