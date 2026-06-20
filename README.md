# Supabase Keep Alive (保活脚本)

通过 GitHub Actions 定时向 Supabase 数据库发送心跳请求，防止 x-pages 项目因长期无访问而被自动暂停。

## 🚀 工作原理

Supabase 官方政策规定，免费版项目如果连续 **7 天**没有任何 API 请求或数据库活动，将会被自动暂停。

本项目利用 **GitHub Actions** 的定时任务（Cron Job），每 **12 小时**通过 `curl` 自动调用一次 Supabase 的 REST API，更新数据库中 `keep_alive` 表的最新时间戳，从而实现永久保活。

---

## 🛠️ 配置与使用步骤

### 1. 在 Supabase 中创建数据库表

登录 Supabase 后台，打开 **SQL Editor**，运行以下 SQL 语句来初始化心跳表：

```sql
create table if not exists public.keep_alive (
    id bigint primary key generated always as identity,
    last_ping timestamp with time zone default timezone('utc'::text, now()) not null
);

-- 插入一条默认值数据（不显式指定 id）
insert into public.keep_alive default values;
```

---

## 🛑 避坑指南 (Troubleshooting)

在搭建本项目标保活机制时，经历了一连串因安全策略、平台机制和语法限制导致的“连环坑”，在此记录以备后查：

### 1. PostgreSQL 身份列限制 (Identity Column)
* **问题**：初始化 SQL 时报错 `cannot insert a non-DEFAULT value into column "id"`。
* **原因**：在 Postgres 中使用 `GENERATED ALWAYS AS IDENTITY` 定义的主键非常严格，默认不允许在 `INSERT` 中手动指定 `id` 值。
* **解决**：改用 `DEFAULT VALUES` 让数据库自动生成主键，或者在 SQL 中加入 `OVERRIDING SYSTEM VALUE` 强行覆盖。

### 2. GitHub Actions 的“大迟到”机制 (Cron Delay)
* **问题**：设定的北京时间 `11:10` 到了，GitHub Action 并没有准时触发。
* **原因**：GitHub Actions 的 `schedule` 定时任务不是强定时的。由于全球免费任务排队严重，通常会有 **5 到 30 分钟的随机延迟**。
* **解决**：测试时无需死等，应配置 `workflow_dispatch` 允许**手动触发**来验证脚本正确性。对于保活任务，迟到不影响效果。

### 3. Supabase REST API 的“假函数”限制
* **问题**：直接发送 JSON `{"last_ping": "now()"}` 导致数据死活不更新。
* **原因**：Supabase 的 REST API（PostgREST）比较严格，传入 `"now()"` 时会将其当作合法的字符串丢弃或报错，而不会在数据库端执行 `now()` 函数。
* **解决**：在发送端（GitHub Actions 内部）利用 Linux 命令先生成好标准的 ISO 8601 时间戳（如 `date -u +"%Y-%m-%dT%H:%M:%SZ"`）再作为变量传过去。

### 4. Supabase 的“静默拒绝”：行级安全策略 (RLS)
* **问题**：GitHub Actions 显示绿勾（成功），但 Supabase 里的时间死活不刷新，且无报错。
* **原因**：Supabase 默认创建的表都会自动开启 **RLS（Row Level Security）**。在没有配置具体 Policy 前，它会严格禁止通过 `anon`（公开密钥）进行任何修改。为了安全，它有时会选择“静默拒绝”（返回空结果），导致表面上看起来成功了实际上没改掉。
* **解决**：对于这种纯粹用来刷活跃度、不涉及敏感数据的心跳表，直接通过 SQL 运行 `alter table public.keep_alive disable row level security;` 关闭 RLS 是最快、最省心的解法。
