# Puppet的安装和配置

## Puppet简介

Puppet基于ruby语言开发的自动化系统配置工具，可以C/S模式或独立运行，支持对所有UNIX及类UNIX系统的配置管理，最新版本也开始支持对Windows操作系统有限的一些管理。Puppet适用于服务器管的整个过程 比如初始安装、配置更新以及系统下线。

## Puppet安装

Puppet的安装方式支持源码安装、yum安装以及ruby的gem安装。官网推荐使用yum来安装puppet，方面以后的升级、管理、维护。Centos可以采用yum来安装，但是Centos的默认源中没有puppet包，因此需要先安装epel包。

### Master的安装
```
CentOS6:
# 更新源
rpm -ivh https://yum.puppetlabs.com/puppetlabs-release-el-6.noarch.rpm
# 安装软件包
yum install puppet-server
# puppet更新到最新
puppet resource package puppet-server ensure=lates
# 开始服务
service puppetmaster start
# 开机启动
chkconfig puppetmaster on
```
### Agent的安装
```
CentOS6:
# 更新源
rpm -ivh https://yum.puppetlabs.com/puppetlabs-release-el-6.noarch.rpm
# 安装软件包
yum install puppet
# puppet更新到最新
puppet resource package puppet ensure=latest
# 开始puppet 服务（Agent角色）
service puppet start
# 开机启动
chkconfig puppet on
```
Master的安装后，Agent的组件也自动安装完毕

## Puppet的简单配置
### Master的配置

puppet主目录下的文件和作用：
```
ls -1 /etc/puppet/
auth.conf               #定义puppet master的acl文件
fileserver.conf         #定义puppet master文件服务器的配置文件
manifests/              #puppet脚本主文件目录，site.pp文件必须存在
modules/                #puppet模块目录
puppet.conf             #puppet主配置文件
ssl/                    #存放ssl证书的目录
```
刚开始的话，puppet.conf不需要配置就可以满足。需要更改hosts文件，注意hosts要和主机名对应。
```
vim /etc/hosts 
 或 echo '192.168.30.204 testaa' >> /etc/hosts
添加如下内容：
192.168.30.204 testaa
192.168.30.31  trac-svr
```
根据实际情况加，这里是一个master(testaa)，一个agent(trac-svr)。

### Agent的配置

Agent的配置主要是更改agent上的/etc/puppet/puppet.conf文件的[agent]部分。
在agent上/etc/puppet/puppet.conf 添加如下配置
```
vim /etc/puppet/puppet.conf
server = testaa        #master服务器的地址
runinterval = 60       #每隔多久的时间进行自动更新,时间单位为秒
#listen = true         #客户端作为一个服务进行监听，允许远程master触发puppet Agent的节点配置,
                     #不建议使用该功能,是因为远程master触发的话，某些命令会受限（如：exec）
```
### puppet使用

puppet客户端发送证书
```
#客户端第一次启动向服务端发送证书，要求签好之后才能通信
puppet agent --no-daemonize --onetime --verbose --debug --server=puppet_server(服务端主机名)
正常提示信息:
info: Creating a new SSL key for client.puppet.com info: Caching certificate for ca
info: Creating a new SSL certificate request for node.innofidei.com info: Certificate Request fingerprint (md5): 74:34:A9:DC:F6:52:B4:96:D1:FF:D3:68:F6:E5:7B:DE
Exiting; no certificate found and waitforcert is disabled
```
puppet服务端签证书
```
puppet cert list --all                        # 查看所有客户端的请求(有+号的代表已经签好证书可以通信，没有加号的代表尚未签好证书)
puppet cert --sign puppet_client(客户端主机名) # 这条命令加客户端主机名就能签字，自此可以通信
或 puppet cert --sign --all                   # 或者签发所有未签名的agent 端
正常提示信息:
notice: Signed certificate request for trac-svr
notice: Removing file Puppet::SSL::CertificateRequest client.puppet.com at '/var/lib/puppet/ssl/ca/requests/trac-svr.pem'
```
## 参考URL

官网：

* [安装前环境准备pre_install.html](http://docs.puppetlabs.com/guides/install_puppet/pre_install.html)
* [安装install_el.html](http://docs.puppetlabs.com/guides/install_puppet/install_el.html)
* [安装完成后的后续工作-签名post_install.html](http://docs.puppetlabs.com/guides/install_puppet/post_install.html)

其他：

- [百度文库：Puppet学习之puppet的安装和配置](http://wenku.baidu.com/link?url=cv3S65fjIIVpXGwcSwynrqqc7GYIvJENQb5q9jzL3lqAezYbntnQaJZCQNRuCO8pcP59RRfYFmjHER6ycR4t8KIXeuPJ3otid7ZU2IM1MQS)
- [Puppet中文wiki](http://puppet.wikidot.com/file)
- [puppet报告系统Dashboard部署及配置详解](http://dreamfire.blog.51cto.com/418026/1322552/)
- [Puppet使用方法总结](http://dongxicheng.org/cluster-managemant/puppet/)

### Mcollective 相关

通过MCollective更加安全地实现puppet的推送更新功能

#### agent端安装命令
```
yum install -y mcollective  mcollective-common mcollective-puppet-agent mcollective-puppet-common
vim /etc/mcollective/server.cfg 
修改：
plugin.activemq.pool.1.host = tsp-rls-jobs2
plugin.activemq.pool.1.port = 61613
/etc/rc.d/init.d/mcollective start
chkconfig mcollective on
```
#### master端安装命令
```
安装ActiveMQ：
yum install tanukiwrapper activemq activemq-info-provider 
vim /etc/activemq/activemq.xml
修改users配置信息：
<simpleAuthenticationPlugin>
  <users>
  <authenticationUser username="mcollective" password="secret" groups="mcollective,admins,everyone"/>  #配置通信的账号及密码
  </users>
</simpleAuthenticationPlugin>

/etc/rc.d/init.d/activemq start
netstat -nlatp | grep 61613  #查看监听端口
chkconfig activemq on

安装MCollective：
yum install mcollective-common  mcollective-client mcollective-puppet-client mcollective-puppet-common #依赖包rubygem-stomp  
vim /etc/mcollective/client.cfg
修改host port:
plugin.stomp.host = 192.168.100.110  #Middleware地址
plugin.stomp.port = 61613  #Middleware监听端口
```
