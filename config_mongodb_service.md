# MongoDB 配置

## MongoDB的安装

此处略。 请参考官网文档
[http://docs.mongodb.org/manual/tutorial/install-mongodb-on-red-hat-centos-or-fedora-linux/](http://docs.mongodb.org/manual/tutorial/install-mongodb-on-red-hat-centos-or-fedora-linux/)
通常安装完毕后，将所有的服务全部停止。以便于我们自定义配置
```
killall mongod
killall mongos
```

同时，关闭MongoDB系统服务
```
centOS:
　service mongod stop
　chkconfig mongod off
ubuntu:
　待查
 ```
## 配置文件
MongoDB的实例有三个角色，
* 数据服务器: mongod [--shardsvr]
* 配置服务器: mongod [--configsvr]
* 路由服务器: mongos

**注意**

1. <>中的内容，基于实际运行环境作相应的替换
1. 并假定所有的文件存储和运行环境在/home/mongodata/下 

**建议模板如下**

* 数据服务器配置文件（我们是按照分片且每片都是复制集的方式构建的）
```
systemLog:
destination: file
path: "/home/mongodata/log/rs1.log"
logAppend: true
storage:
journal:
enabled: true
directoryPerDB: true
dbPath: "/home/mongodata/dbfiles/rs1_file"
processManagement:
pidFilePath: "/home/mongodata/pid/rs1.pid"
fork: true
net:
bindIp: <服务器的IP地址>,127.0.0.1
port: 27018      
security:
clusterAuthMode: keyFile
keyFile: "/home/mongodata/keyfile/mongodb.pem"
replication:
replSetName: <副本集名>  # 如："rs1"
oplogSizeMB: 10000
sharding:
clusterRole: shardsvr
```

* 配置服务器配置文件
```
systemLog:
destination: file
path: "/home/mongodata/log/confsvr.log"
logAppend: true
storage:
journal:
enabled: true
directoryPerDB: true
dbPath: "/home/mongodata/dbfiles/confsvr_file/"
processManagement:
pidFilePath: "/home/mongodata/pid/confsvr.pid"
fork: true
net:
bindIp: <服务器的IP地址>,127.0.0.1
port: 27019
security:
clusterAuthMode: keyFile
keyFile: "/home/mongodata/keyfile/mongodb.pem"
sharding:
clusterRole: configsvr
```

* 路由服务器配置文件 
```
systemLog:
destination: file
path: "/home/mongodata/log/mongos.log"
logAppend: true
processManagement:
pidFilePath: "/home/mongodata/pid/mongos.pid"
fork: true
net:
bindIp: 127.0.0.1  
# 如果是正式的生产环境不建议帮定<服务器的IP地址>，
# 每个需要访问mongoDB的服务器都部署属于自己的路由服务器
port: 27017
security:
clusterAuthMode: keyFile
keyFile: "/home/mongodata/keyfile/mongodb.pem"
sharding:
configDB: <配置服务器1 IP:Prot>,<配置服务器2 IP:Prot>,<配置服务器2 IP:Prot>
```

关于[security:]字段的说明：
1. 由于是生产环境，并且我们所布署的云主机是和别的公司共用的，为了防止我们的DB数据被非法镜像，我们需要对所有接入MongoDB集群的服务器做认证。认证的方式可以是私有key也可以是ssl方式。但是，ssl方式是收费版才提供的，因此我们使用了私有key的认证方式。
  1. 私有key方式仅仅是在服务器接入集群时作确认
  2. .ssl方式则是在通讯时也同时作了加密 
2. 一旦使用了[security:]字段，那意味着客户端认证模式也被打开，所有通过mongo或driver登入到DB的应用端将不能匿名的操作数据库。
3. [security.keyFile]文件内容在所有的服务器上必须一致

```
推荐:
-- 文件内容采base64编码，一共100个字符
openssl rand -base64 100 > /home/mongodata/keyfile/mongodb.pem

-- 仅DB实例用户可读写
chmod 600 /home/mongodata/keyfile/mongodb.pem
```

## 启动

数据服务器:

```mongod -f <数据服务器配置文件>```

配置服务器:

```mongod -f <配置服务器配置文件>```

路由服务器:

```mongos -f <路由服务器配置文件>```

配置成OS服务的方法

```
# 复制服务脚本文件
cp /etc/init.d/mongod /etc/init.d/{新的服务名} 
   # 新的服务名 如：cp /etc/init.d/mongod /etc/init.d/mongod_rs 
# 修改配置文件
vim /etc/init.d/{新的服务名}
   # 新的服务名 如：vim /etc/init.d/mongod_rs
# 修改文件路径 
 # CONFIGFILE="服务器配置文件路径"
# 如果配置的是路由服务器 
 # mongod=${MONGOD-/usr/bin/mongod} => mongod=${MONGOD-/usr/bin/mongos}

# 注册服务 
chkconfig --add {新的服务名} # 如：chkconfig --add mongod_rs 
# 设置开机启动
chkconfig {新的服务名} on    # 如：chkconfig mongod_rs on
# 启动服务
service {新的服务名} start   # 如：service mongod_rs start
```

**注意：路由服务器中涉及到三台配置服务器，mongoDB要求所有的配置服务器的时间是一致的。我们通常要求所有的服务器同步时间。**

## 停止

使用kill命令或
```
killall mongod
killall mongos
```

## 构建存储
生产环境下，为了以后的横向扩展，我们在初期就考虑构建分片。同时为了保证每一片数据的安全，我们的片都使用复制集来构建。

### 构建Replica Sets（复制集）
#### 在（复制集）上创建用户&分配权限 
由于我们使用了[security:]字段，因此，我们只能在本地[127.0.0.1]上登陆，远程登陆将无法操作DB

```mongo 127.0.0.1:27018/admin```

第一次登陆，且是在本地，由于DB中没有用户，所以可以做任何操作 创建用户：

```
use admin  # 切换到admin库
db.createUser({ user:"root", pwd:"root", 
roles: [{ role: "userAdminAnyDatabase", db: "admin" } , 
      { role: "dbOwner", db: "admin" }, 
      { role: "userAdmin", db: "admin" } ] } );
```

如有用户必须的登陆后，方可操作

```
db.auth("root","root");
```

修改权限

```
db.grantRolesToUser("root", 
 ["clusterAdmin",  # 此次追加项
 { role: "userAdminAnyDatabase", db: "admin" } , 
 { role: "dbOwner", db: "admin" }, 
 { role: "userAdmin", db: "admin" } ] , 
{ w: "majority" , wtimeout: 4000 })
```
关于权限的说明 参考内置权限
上述举例中，
```
{ role: "userAdminAnyDatabase", db: "admin" } 
{ role: "dbOwner", db: "admin" }
{ role: "userAdmin", db: "admin" }
```

三者等价的，都能使该用户成为本地库的超级用户（雷同于：root,sa,administrator...）。
权限 "clusterAdmin"，是赋予该用户**集群操作**。

#### 初始化Replica Sets（复制集） 

初始化复制集的 主节点
```
rs.initiate({ _id:"<复制集名>", members:[ {_id:0,host:'<服务器的IP和port>'}]});
# <复制集名>要与配置文件一致
# 关于<服务器的IP和port>，即便我们现在是在本地登陆的，<服务器的IP>应该填写真实的IP地址
```
追加其余数据服务器 从节点 在另一台服务器上，当我们成功启动了数据服务器后，继续前面的操作
```
rs.add('<另一台服务器的IP和port>');
```
追加 投票节点 通常投票节点服务器在性能上可以弱一点，当我们成功后启动，继续前面的操作
```
rs.addArb('<投票节点服务器的IP和port>');
```
确认
```
rs.status();
```

### 构建分片（sharding）
在配置服务器和路由服务器成功启动后

#### 创建用户&分配权限 

和配置复制集一样，由于我们使用了[security:]字段，因此，我们只能在本地[127.0.0.1]上登陆，远程登陆将无法操作DB
```
mongo 127.0.0.1:27017/admin
```
登陆到mongoDB中：
```
use admin
db.createUser({ user: "superman", pwd: "superman", 
    roles: [ "clusterAdmin",
             "userAdminAnyDatabase",
             "dbAdminAnyDatabase",
             "readWriteAnyDatabase" ] } )
```
关于权限定义，请参考前面的内容。 登陆后，开始后续操作
```
db.auth("superman","superman")
```
**注意**： 网上又说需要在config库中也追加相关用户，但测试下来发现无此需要。命令如下：

```
use config
db.createUser( { user: "superman",pwd: "XXXX",
    roles: ["clusterAdmin",
            "userAdminAnyDatabase",
            "dbAdminAnyDatabase",
            "readWriteAnyDatabase" ] } ) 
# 上面的命令居然失败！！ config库中没有 "clusterAdmin"权限！！
db.createUser( { user: "superman",
                  pwd: "XXXX",
                  roles: ["dbOwner","userAdmin"] } )
# 上面的命令是成功的。但感觉和集群操作无关。因为没有集群操作相关的权限
db.auth("superman","XXXX")
```

#### 初始化分片（sharding） 

切换到admin库，并登陆用户
```
use admin
db.auth("superman","superman")
```
追加片
```
sh.addShard("<副本集名/服务器的IP和port>")
# 由于数据服务器我们使用的是副本集，因此，服务器的IP和port可以是副本集中的任意一台
```
配置需要开启分片的数据库
```
sh.enableSharding("<业务数据库>");
```
数据集合开启分片
```
sh.shardCollection("<数据集合全名>",{key});
如：
sh.shardCollection("TSP20DB.fs.chunks",{ "files_id" : 1 });
```
### 集群维护教程

参考如下URL:
* [集群维护教程](http://docs.mongoing.com/manual/administration/sharded-cluster-maintenance.html)
* [保持域名不变迁移配置服务器](http://docs.mongoing.com/manual/tutorial/migrate-config-servers-with-same-hostname.html)
* [迁移配置服务器到不同的域名](http://docs.mongoing.com/manual/tutorial/migrate-config-servers-with-different-hostnames.html)
* [替换坏掉的配置服务器](http://docs.mongoing.com/manual/tutorial/replace-config-server.html)




### 性能监控

目前，我们的云平台使用的是negios系统。而mongoDB也有negios的插件[Nagios-MongoDB](https://github.com/mzupan/nagios-plugin-mongodb)

### 讨论

* 关于文件分片的片建的选择 

[对GridFS数据进行分片](http://docs.mongoing.com/manual-zh/tutorial/shard-gridfs-data.html)

* 关于mongoDB容量扩展的方法 
