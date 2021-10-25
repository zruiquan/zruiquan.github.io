---
title: Phoenix函数应用
date: 2021-10-25 15:54:04
tags:
- Phoenix
categories:
- Phoenix
---

# Phoenix函数应用

* 日期转时间戳

```sql
select to_char(to_number(to_date('2020-03-01','yyyy-MM-dd','GMT+8'))/1000,'##########');
```



* 时间戳转日期

```sql
select to_char(CONVERT_TZ(to_date('1582992000','s'), 'UTC', 'Asia/Shanghai'),'yyyy-MM-dd HH:mm:ss');
```



* 根据phoenix时间查询当日零点时间戳

```sql
select to_number(to_date(to_char(CURRENT_TIME(),'yyyy-MM-dd'),'yyyy-MM-dd','GMT+8'));
```



* 根据外部时间戳字符串查询当日零点时间戳

```sql
select to_number(to_date(to_char(CONVERT_TZ(to_date('1635145201','s'), 'UTC', 'Asia/Shanghai'),'yyyy-MM-dd'),'yyyy-MM-dd','GMT+8'));
```


