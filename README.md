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
