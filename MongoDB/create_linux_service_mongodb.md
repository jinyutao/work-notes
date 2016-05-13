# MongoDB 安装后以服务方式启动

脚本
## 安装软件源:

```
#!/bin/bash

# 安装软件源
echo '[mongodb-org-3.0]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/$releasever/mongodb-org/3.0/x86_64/
gpgcheck=0
enabled=1' > /etc/yum.repos.d/mongodb-org-3.0.repo 
```

## mongodb

```
# 安装mongodb
yum install -y mongodb-org
service mongod stop
chkconfig mongod off
```
## 配置[配置服务器]:
```
#基于“默认配置”配置[配置服务器]，配置文件参考MongoDB 配置
echo 'config mongodcfg'
cp /etc/init.d/mongod /etc/init.d/mongodcfg 
sed -i 's/\/etc\/mongod.conf/\/home\/mdata\/mongocfg\/mongod_conf.conf/g' /etc/init.d/mongodcfg
sed -i 's/\/var\/lock\/subsys\/mongod/\/var\/lock\/subsys\/mongodcfg/g' /etc/init.d/mongodcfg
sed -i 's/mongod: /mongodcfg: /g' /etc/init.d/mongodcfg
sed -i 's/    status $mongod/#   status $mongod\n    pidofproc -p "${PIDFILEPATH}" ${mongod}/g' /etc/init.d/mongodcfg
chkconfig --add mongodcfg
chkconfig mongodcfg on
```
## 配置[路由服务器]:
```
#基于“默认配置”配置[路由服务器]，配置文件参考MongoDB 配置
echo 'config mongos'
cp /etc/init.d/mongod /etc/init.d/mongos
sed -i 's/\/usr\/bin\/mongod/\/usr\/bin\/mongos/g' /etc/init.d/mongos
sed -i 's/\/etc\/mongod.conf/\/home\/mdata\/mongos\/mongos.conf/g' /etc/init.d/mongos
sed -i 's/\/var\/lock\/subsys\/mongod/\/var\/lock\/subsys\/mongos/g' /etc/init.d/mongos
sed -i 's/mongod: /mongos: /g' /etc/init.d/mongos 
sed -i 's/    status $mongod/#   status $mongod\n    pidofproc -p "${PIDFILEPATH}" ${mongod}/g' /etc/init.d/mongos
chkconfig --add mongos
chkconfig mongos on
```
## 配置[数据服务器]:
```
#修改“默认配置”配置[数据服务器]，配置文件参考MongoDB 配置
echo 'config mongod rs1'
cp /etc/init.d/mongod /etc/init.d/mongodrs1 
sed -i 's/\/etc\/mongod.conf/\/home\/mdata\/rs1\/rs1.conf/g' /etc/init.d/mongodrs1
sed -i 's/\/var\/lock\/subsys\/mongod/\/var\/lock\/subsys\/mongodrs1/g' /etc/init.d/mongodrs1
sed -i 's/mongod: /mongodrs1: /g' /etc/init.d/mongodrs1
sed -i 's/    status $mongod/#   status $mongod\n    pidofproc -p "${PIDFILEPATH}" ${mongod}/g' /etc/init.d/mongodrs1
chkconfig --add mongodrs1
chkconfig mongodrs1 on
```
```
echo 'config end'
```