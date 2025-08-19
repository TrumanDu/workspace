# hive 笔记

## hive 锁

查看所有锁

```
-- 查看所有表的锁状态
SHOW LOCKS;

-- 查看特定表的锁
SHOW LOCKS <table_name>;

-- 查看扩展信息
SHOW LOCKS <table_name> EXTENDED;
```

手动解锁

```
unlock table  my_hive_table;
```
