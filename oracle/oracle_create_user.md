# ORACLE 创建用户


## 以ORACL权限连接数据库

1.进入sqlpus环境，/nolog不登陆数据库服务器。

```sqlplus /nolog	```

2.以系统管理员（sysdba）的身份连接数据库(如果需要对数据库进行管理操作，需要以这种方式登陆数据库) 

```sql>conn / as sysdba;```

3.创建用户acc，密码 abc.123(因为.是特殊字符，所以要用""括起来)

```create user acc identified by "abc.123";```

4.授予用户acc，connect, resource权限

```grant connect, resource to acc;```

5.授予用户acc，全库导出和全库导入的角色权限(生产环境建议用另一个用户，如acc_bak)

```grant EXP_FULL_DATABASE，IMP_FULL_DATABASE to acc;```


## 创建表，初始化数据

1.登陆acc用户

``` conn acc/"abc.123" ```

2.导入建表sql文（假定建表sql文在/tmp下）


``` @/tmp/account.sql ```


3.导入追加数据sql文

``` @/tmp/accountInsert.sql ```

4.退出当前用户

``` quit ```

