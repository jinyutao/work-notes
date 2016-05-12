# ORACLE 打补丁

**所有命令在oracle用户下执行 **

1.准备环境变量

```export PATH=$PATH:/home/oracle/app/product/11.2.0/dbhome_1/OPatch```

2.关闭侦听

```lsnrctl stop```

3.关闭数据库

``` 
sqlplus /nolog
SQL> CONNECT / AS SYSDBA
SQL> SHUTDOWN IMMEDIATE
SQL> QUIT
```

4.执行patch

```
unzip p12419378_11201_<platform>.zip
cd 12419378
opatch prereq CheckConflictAgainstOHWithDetail -phBaseDir ./
opatch apply
cd $ORACLE_HOME/rdbms/admin
sqlplus /nolog
SQL> CONNECT / AS SYSDBA
SQL> STARTUP
SQL> @catbundle.sql psu apply
SQL> QUIT

```
5.确认升级成功

```
SQL> CONNECT / AS SYSDBA
SQL> select action,comments from registry$history;
SQL> QUIT
```

5.1.应该有此次升级的记录

```opatch lsinventory ```



6.启动侦听

```lsnrctl start```

**完毕**


* **关于oracle密码过期**

```
SQL> CONNECT / AS SYSDBA
SQL> alter user <用户名> identified by <密码>
```