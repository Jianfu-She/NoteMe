# Centos7下搭建JavaWeb环境

在阿里云上租了服务器，顺便记录下Centos7.2搭建JavaWeb环境的全过程。

## 1 工具
[Xshell](https://www.netsarang.com/products/xsh_overview.html)
[Xftp](https://www.netsarang.com/products/xfp_overview.html)

## 2 连接到阿里云服务器 
通过Xshell连接到阿里云服务器，具体不表。

## 3 上传安装文件 
通过Xftp上传相关文件，包括[JDK](http://www.oracle.com/technetwork/java/javase/downloads/index.html)，[tomcat](http://tomcat.apache.org/)，以及测试用的war包、sql文件。

## 4 安装JDK
1）在“/”目录下新建好软件安装的目录，这里把tomcat安装的目录也提前建好。
```shell
cd /
cd usr
mkdir java 
cd java 
mkdir tomcat 
```
2）将JDK解压到相应目录
```shell
rpm -ivh jdk-8u101-linux-x64.rpm
# 或者 tar zxvf jdk-8u101-linux-x64.gz -C /usr/java
```
3）配置环境变量，打开文件
```shell
vi /etc/profile
```
4）在其末尾添加如下内容
```shell
JAVA_HOME=/usr/java/jdk1.8.0_101  
JRE_HOME=/usr/java/jdk1.8.0_101/jre  
PATH=$PATH:$JAVA_HOME/bin:$JRE_HOME/bin  
CLASSPATH=:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JRE_HOME/lib  
export JAVA_HOME JRE_HOME PATH CLASSPATH  
```
5）编辑保存后执行如下命令,使保存的环境变量生效
```shell
source /etc/profile
```
6）执行如下命令如果显示对应的jdk版本就表明安装配置成功了
```shell
java -version
```
7）可以用以下命令查看环境变量的配置
```shell
echo $PATH 
```

## 5 安装tomcat 
1）将tomcat解压到对应目录
```shell
tar zxvf apache-tomcat-8.0.36.tar.gz -C /usr/java/tomcat
```
2）然后进入到tomcat的bin目录下，编辑setclasspath.sh文件，在末尾添加如下内容，即可完成tomcat环境变量的配置
```shell
export JAVA_HOME=/usr/java/jdk1.8.0_101   
export JRE_HOME=/usr/java/jdk1.8.0_101/jre 
```
3）然后通过cd命令进入tomcat的bin目录，执行
```shell
./startup.sh 
```
出现‘Tomcat started’说明启动成功。 <br/>
4）此时在浏览器输入`ip:8080`会出现Apache的tomcat页面。 <br/>
5）设置tomcat开机自启动 <br/>
- 为tomcat增加启动参数 <br/>
  在/usr/java/tomcat/apache-tomcat-8.0.36/bin下增加setenv.sh配置，catalina.sh启动的时候会调用，同时配置Java内存参数。 <br/>
```shell
cd /usr/java/tomcat/apache-tomcat-8.0.36/bin
vi setenv.sh 

#add tomcat pid  
CATALINA_PID="$CATALINA_BASE/tomcat.pid"  
#add java opts  
JAVA_OPTS="-server -XX:PermSize=256M -XX:MaxPermSize=1024m -Xms512M -Xmx1024M -XX:MaxNewSize=256m" 
```

- 增加tomcat.service服务脚本 
  在/usr/lib/systemd/system目录下增加tomcat.service，目录必须是绝对目录。
```shell  
cd /usr/lib/systemd/system
vi tomcat.service

[Unit]
Description=Tomcat
After=syslog.target network.target remote-fs.target nss-lookup.target

[Service]
Type=forking
PIDFile=/usr/java/tomcat/apache-tomcat-8.0.36/tomcat.pid
ExecStart=/usr/java/tomcat/apache-tomcat-8.0.36/bin/startup.sh
ExecReload=/usr/bin/kill -s HUP $MAINPID
ExecStop=/usr/bin/kill -s QUIT $MAINPID
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```
  **[Unit]**配置了服务的描述，规定了在network启动之后执行 <br/>
  Description:描述服务 <br/>
  After:描述服务类别 <br/>
  **[Service]**服务运行参数的设置 <br/>
  Type=forking是后台运行的形式 <br/>
  ExecStart为服务的具体运行命令 <br/>
  ExecReload为重启命令 <br/>
  ExecStop为停止命令 <br/>
  PrivateTmp=True表示给服务分配独立的临时空间 <br/>
  注意：[Service]的启动、重启、停止命令全部要求使用绝对路径 <br/>
  **[Install]**服务安装的相关设置，可设置为多用户 <br/>

- 设置tomcat开机启动 <br/>
  启用tomcat开机启动： <br/>
  `systemctl enable tomcat` <br/>
  查看tomcat是否开机启动： <br/>
  `systemctl is-enabled tomcat` <br/>
  启动tomcat： <br/>
  `systemctl start tomcat` <br/>
  停止tomcat： <br/>
  `systemctl stop tomcat` <br/>
  重启tomcat： <br/>
  `systemctl restart tomcat` <br/>
  因为配置pid，在启动的时候会再tomcat根目录生成tomcat.pid文件，停止之后删除。同时tomcat在启动时候，执行start不会启动两个tomcat，保证始终只有一个tomcat服务在运行。多个tomcat可以配置在多个目录下，互不影响。<br/>

关于Tomcat开机启动的配置还有[另外一种方案](http://jingyan.baidu.com/article/6525d4b1382f0aac7d2e9421.html) <br/>

## 6 下载并安装MySQL 
1）下载并安装MySQL
```shell
rpm -ivh http://dev.mysql.com/get/mysql57-community-release-el7-8.noarch.rpm  
```
```shell
yum install mysql-server -y  
```
2）设置mysqld服务开机自启动
```shell
systemctl enable mysqld.service  
```
3）开启 mysqld 服务
```shell
systemctl start mysqld.service 
```
4）此时，如果直接执行`mysql -uroot -p`回车，会报如下错误： <br/>
<font color="red">ERROR 1045 (28000): Access denied for user 'root'@'localhost' (using password: NO)</font> <br/>
查看官方文档，找其原因：mysql 5.7 在安装过程中自动生成了一个默认的root密码（5.7以下版本默认root密码为空） <br/>
5）解决方案，用如下命令获取root的默认密码（红色部分） 
```shell
grep 'temporary password' /var/log/mysqld.log 
```
2016-06-26T12:45:43.799230Z 1 [Note] A temporary password is generated for root@localhost:<font color="red">jyki7m+>RD_*</font> <br/>
6）重复步骤4，输入该默认密码，成功登入mysql，此时执行命令`show databases`，会报如下错误：<br/>
<font color="red">ERROR 1820 (HY000): You must reset your password using ALTER USER statement before executing this statement.</font> <br/>
7）根据该错误提示，必须修改密码
```shell
mysql> SET PASSWORD = PASSWORD('new password'); 
```
注意：新密码必须 **大小写+符号** 全部包含，不然会提示密码不符合规则 <br/>

## 7 部署war包
通过Xftp将war包上传到/usr/java/tomcat/apache-tomcat-8.0.36/webapps下，然后进入到tomcat的bin目录执行如下命令启动tomat
```shell
./startup.sh 
```
启动tomcat成功后，即可访问部署的项目。

## 8 把sql文件导入数据库 
1）在命令行输入用户名及密码，进入数据库
```shell 
mysql -uroot -p 
```
2）新建数据库，名称和项目里数据库的名称要一致
```shell 
create database abc;
use abc;
```
3）导入sql文件到数据库，从本地上传sql文件到云服务器后，执行
```shell
source /usr/java/storage.sql;
```
即可完成sql文件的导入 <br/>
此时此刻，部署到云服务的项目就可以正常工作啦！

## 9 关于防火墙 
安装好tomcat后可能仍无法正常访问，问题出在防火墙上，相应的端口被防火墙屏蔽掉了。CentOS7关于防火墙的命令和以前的版本有较大区别，特此记录：<br/>
查看已经开放的端口：<br/>
```shell
firewall-cmd --list-ports
```
开启端口：<br/>
```shell
firewall-cmd --zone=public --add-port=80/tcp --permanent
```
命令含义：<br/>
–zone #作用域 <br/>
–add-port=80/tcp #添加端口，格式为：端口/通讯协议 <br/>
–permanent #永久生效，没有此参数重启后失效 <br/>
重启防火墙： <br/>
```shell
firewall-cmd --reload #重启firewall
systemctl stop firewalld.service #停止firewall
systemctl disable firewalld.service #禁止firewall开机启动
```

## 10 修改tomcat访问端口为80端口 
```shell 
cd /usr/java/tomcat/apache-tomcat-8.0.36/conf
vi server.xml 
```
找到下面这段配置： <br/>
```shell
<Connector port="8080" protocol="HTTP/1.1"
               connectionTimeout="20000"
               redirectPort="8443" />
```
将端口为80端口，保存退出，重启使用80端口访问：
```shell
cd ../bin
./shutdown.sh
./startup.sh
```

## 11 FTP服务器的安装配置 
并非搭建JavaWeb环境必需，按需使用。 <br/>
1）在安装前查看是否已安装vsftpd <br/>
```shell
rpm -q vsftpd
```
2）如果没有，用yum安装 <br/>
```shell
yum -y install vsftpd
```
3）查看配置文件所在路径 <br/>
```shell
rpm -qc vsftpd
# 或者 whereis vsftpd
```
4）启动vsftpd服务 <br/>
```shell
systemctl start vsftpd.service 
```
安装完默认情况下是开启匿名登录的，对应的是`/var/ftp`目录，这时只要服务启动了，就可以连上FTP了。 <br/>
然而，一般情况下我们都是按分配的用户去访问各自的目录，vsftpd的用户分为系统用户和虚拟用户，系统用户也就是系统实际存在的Linux用户，而虚拟用户则是存在于配置文件里面的。系统用户方式比较简单，创建系统用户，确保用户能对FTP目录进行读写就可以了。有的系统已默认创建了名为ftp的用户了。以下是创建普通Linux用户。 <br/>
5）创建系统用户
```shell
useradd -g root -M -d /var/www/html -s /sbin/nologin ftpuser
passwd ftpuser
输入密码 <br/>
```
把`/var/www/html`的所有权给ftpuser.root
```shell
chown -R ftpuser.root /var/www/html
```
这样就可以通过ftpuser用户连接FTP了。 <br/>
<font color="red">下面重点讲一下如何创建虚拟用户。</font> <br/>
6）备份vsftpd原有配置文件
```shell
cd /etc/vsftpd/
cp vsftpd.conf vsftpd.conf.origin
```
7）创建密码明文文件
```shell
vi /etc/vsftpd/vftpuser.txt 
ftpuser
p@ssw0rd
```
8）根据明文创建密码DB文件
```shell
db_load -T -t hash -f /etc/vsftpd/vftpuser.txt /etc/vsftpd/vftpuser.db 
```
9）查看密码数据文件 
```shell
file /etc/vsftpd/vftpuser.db 
```
10）创建vftpd的guest账户
```shell 
useradd -d /usr/ftp -s /sbin/nologin ftpuser 
```
注意：/usr/ftp中的ftp文件夹一定不要自己手动创建 <br/>
11）指定pam文件
```shell 
vi /etc/pam.d/vsftpd
```
打开/etc/pam.d/vsftpd，将auth及account的所有配置行均注释掉，添加如下内容： <br/>
auth required pam_userdb.so db=/etc/vsftpd/vftpuser <br/>
account required pam_userdb.so db=/etc/vsftpd/vftpuser <br/>
12）修改配置文件
打开/etc/vsftpd/vsftpd.conf，将 anonymous_enable=YES改为anonymous_enable=NO，在anon_umask=022上面添加如下内容：
```shell 
virtual_use_local_privs=YES
guest_enable=YES # 设定启用虚拟用户功能
guest_username=ftpuser # 指定虚拟用户的宿主用户，CentOS中已经有内置的ftp用户了
chroot_local_user=YES
allow_writeable_chroot=YES  # 如果启用了限定用户在其主目录下需要添加这个配置
```
13）设置vsftpd开机启动
```shell
systemctl enable vsftpd 
```
14）重新启动vsftpd服务 
```shell
systemctl restart vsftpd 
```
15）配置防火墙和SELinux
```shell
firewall-cmd --permanent --zone=public --add-service=ftp 
firewall-cmd --reload

getsebool -a | grep ftp
setsebool -P ftpd_full_access on
```
16）完成 <br/>
现在可以通过Xftp的FTP连接服务器。<br/>

## 12 生产环境下的MySQL与Tomcat 
推荐两篇博文：<br/>
[CentOS7+Tomcat 生产系统部署](http://blog.csdn.net/smstong/article/details/39958675) <br/>
[ MySQL添加用户、删除用户与授权](http://blog.csdn.net/bobo_93/article/details/51737156) <br/>

## 13 挂载数据盘 
我们默认购买的Linux 阿里云服务器ECS系统盘是40GB的，对于一般的网站也足够使用。如果自己的项目数据比较大，在开始的时候可以增加数据盘，但是必须在安装系统之前对数据盘挂载到指定的目录，这样我们的项目站点才可以放在数据盘中，合理利用数据盘，即便在系统重装，不会影响到网站的数据文件。 <br/>
下面推荐两篇文章，详细介绍了如何在阿里云服务器中挂载数据盘： <br/>
[老左博客](http://www.laozuo.org/5080.html) <br/>
[阿里云linux服务器如何挂载数据盘](http://jingyan.baidu.com/article/90808022d2e9a3fd91c80fe9.html) <br/>