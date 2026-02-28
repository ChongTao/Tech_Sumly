1 MongoDB的安装

2 Mongo的命令

数据库操作命令



```
./mongo
show dbs 显示所有数据列表
```

use db 使用数据库

文档操作



连接到MongoDB

```
mongosh --host <hostname> --port <port>
```

**切换到目标数据库**

```
use <database_name> 要切换的数据库
```

创建用户

```sh
db.createUser({
  user: "testuser",
  pwd: "password123",
  roles: [
    { role: "readWrite", db: "<database_name>" },
    { role: "dbAdmin", db: "<database_name>" }
  ]
})
```

验证用户

```
db.auth("testuser", "password123")
```

常见的命令

```sh
show dbs
use test
show collections

db.users.insertOne({...})
db.users.find({name:"Alice"})
db.users.updateOne({name:"Alice"}, {$set:{age:26}})
db.users.deleteOne({name:"Alice"})
```

删除用户

```sh
db.dropUser("testuser")
```



MongoDB Shell连接数据库

```
mongodb://[username:password@]host1[:port1][,...hostN[:portN]][/[defaultauthdb][?options]]
```

创建数据库

删除数据库

```sh
use myDatabase
db.dropDatabase()
```



插入数据

```sh
> db.runoob.insertOne({"name":"菜鸟教程"})
WriteResult({ "nInserted" : 1 })
> show dbs
admin   0.000GB
config  0.000GB
local   0.000GB
runoob  0.000GB
```

创建集合

```sh
use myNewDatabase
db.createCollection("myNewCollection")
```

查看数据库

```sh
show dbs
```



删除集合

```sh
db.collection.drop()
```



显示集合

```sh
show collections
```

更新集合名

```sh
db.adminCommand({
  renameCollection: "sourceDb.sourceCollection",
  to: "targetDb.targetCollection",
  dropTarget: <boolean>
})
```

插入文档

| 方法           | 用途               | 是否弃用 |
| :------------- | :----------------- | :------- |
| `insertOne()`  | 插入单个文档       | 否       |
| `insertMany()` | 插入多个文档       | 否       |
| `insert()`     | 插入单个或多个文档 | 是       |
| `save()`       | 插入或更新文档     | 是       |

删除文档

```
db.collection.deleteOne(filter, options)
```

查询文档

```
db.collection.find(query, projection)
```