# kingshard

### 启动服务
```
1. git clone github.com/flike/kingshard $GOPATH/src/github.com/flike/kingshard
2. cd $GOPATH/src/github.com/flike/kingshard
3. source ./dev.sh
4. make
5. 设置自配置文件ks.yaml
6. 运行kingshard
./bin/kingshard -config=etc/ks.yaml
```

#### 分表规则
```
子表格式：table_name_%4d，子表下标由4位数组成，如table_name_0000, table_name_0001
所有未分表的sql将会发到默认节点
```

### 创建相关库表
`test_hash_0000` ~ `test_hash_0003`
CREATE TABLE `test_hash_0000` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
  `name` varchar(20) NOT NULL DEFAULT '',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=3 DEFAULT CHARSET=utf8mb4;

`test_range_0000` ~ `test_range_0003`
CREATE TABLE `test_range_0001` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
  `name` varchar(20) NOT NULL DEFAULT '',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=10000 DEFAULT CHARSET=utf8mb4;

### 数据插入
```
连接kingshard 9696端口，在终端执行以下sql语句：
insert into test_hash(id,name) values(5,'a')
insert into test_hash(id,name) values(7,'a')
insert into test_hash(id,name) values(9,'a')
insert into test_range(id,name) values(8,'a')
insert into test_range(id,name) values(20008,'a')

此时在kingsard服务终端可以看见实际执行的sql为：
127.0.0.1:53459->127.0.0.1:3306:insert  into test_hash_0001(id, name) values (5, 'a')
127.0.0.1:53459->127.0.0.1:3306:insert  into test_hash_0003(id, name) values (7, 'a')
127.0.0.1:53459->127.0.0.1:3306:insert  into test_hash_0001(id, name) values (9, 'a')
127.0.0.1:53459->127.0.0.1:3306:insert  into test_range_0000(id, name) values (8, 'a')
127.0.0.1:53459->127.0.0.1:3306:insert  into test_range_0002(id, name) values (20008, 'a')
```

### 数据查询
```
- 等值查询
输入语句：
select * from `test_hash` where id = 7

kingshard终端为：
127.0.0.1:53459->10.225.10.70:3306:select * from test_hash_0003 where id = 7

- 范围查询
输入语句：
select * from `test_hash` where id < 7

kingshard终端为：
127.0.0.1:53459->10.225.10.70:3306:select * from test_hash_0000 where id <= 7
127.0.0.1:53459->10.225.10.70:3306:select * from test_hash_0001 where id <= 7
127.0.0.1:53459->10.225.10.70:3306:select * from test_hash_0002 where id <= 7
127.0.0.1:53459->10.225.10.70:3306:select * from test_hash_0003 where id <= 7
```

### 数据更新
```
输入语句：
update test_hash set `name`='b' where id = 5 or id = 7;

kingshard终端为：
127.0.0.1:53459->127.0.0.1:3306:update test_hash_0001 set name = 'b' where id = 5 or id = 7
127.0.0.1:53459->127.0.0.1:3306:update test_hash_0003 set name = 'b' where id = 5 or id = 7

当更新的记录落在不同的子表，kingshard会以非事务的方式将更新操作发送到多个node上
```

### sum count
```
输入语句：
select count(*) from `test_hash` where id < 10

kingshard终端为：
127.0.0.1:53459->10.225.10.70:3306:select count(*) from test_hash_0000 where id < 10
127.0.0.1:53459->10.225.10.70:3306:select count(*) from test_hash_0001 where id < 10
127.0.0.1:53459->10.225.10.70:3306:select count(*) from test_hash_0002 where id < 10
127.0.0.1:53459->10.225.10.70:3306:select count(*) from test_hash_0003 where id < 10
```

### order by
```
输入语句：
select * from `test_hash` where id < 10 order by id desc

kingshard终端为：
127.0.0.1:53459->10.225.10.70:3306:select * from test_hash_0000 where id < 10 order by id desc
127.0.0.1:53459->10.225.10.70:3306:select * from test_hash_0001 where id < 10 order by id desc
127.0.0.1:53459->10.225.10.70:3306:select * from test_hash_0002 where id < 10 order by id desc
127.0.0.1:53459->10.225.10.70:3306:select * from test_hash_0003 where id < 10 order by id desc
kingshard先将sql发送到对应的node，然后将结果集在内存中排序
```

### 强制读主库
```
加 `/*master*/`

如 select /*master*/ * from test_hash
```

### 事务
```
kingshard支持在单个node上执行事务，不支持跨node的事务。
```
