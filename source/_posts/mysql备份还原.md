---
title: mysql备份还原
date: 2020-04-03 17:13:26
tags: mysql
---

# 备份

```
filename=`date +%Y-%m-%d-%H-%M-%S`;
/usr/bin/mysqldump -ubackup -pbackup dbname --skip-lock-tables | zip > /data/mysql-dump/\${filename}.zip
```

# 还原

```
/usr/bin/mysql -ubackup -pbackup dbname < /data/mysql-dump/1.sql
```
