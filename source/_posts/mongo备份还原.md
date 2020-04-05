---
title: mongodb备份还原
date: 2020-04-03 17:13:26
tags: mongodb
categories: 数据库
---

# 备份

```
filename=`date +%Y-%m-%d-%H-%M-%S.zip`;
mongodump --host 127.0.0.1:3306 -u root -p "password" --authenticationDatabase admin -d dmce -o /data/mongo-dump/${filename}
zip -q -r ${filename}.zip /data/mongo-dump/${filename}
```

# 还原

```
mongorestore --host 127.0.0.1:3306 --drop --username root --password "password" --authenticationDatabase admin --db dmce /data/data/mongo-dump/2/dmce
```
