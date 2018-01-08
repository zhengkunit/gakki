# gakki
gakki-EMS

个人笔记记录
脚本记录

# 备份原来的yum源文件
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup

# 因为最小化安装没有wget - -!!! 装的阿里的Centos7 Base源
curl -o /etc/yum.repos.d/Centos-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo

# epel 还是阿里的，发现国外的速度慢就换成阿里的了。
wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo

# 安装zabbix 官方yum源
rpm -Uvh http://repo.zabbix.com/zabbix/3.0/rhel/7/x86_64/zabbix-release-3.0-1.el7.noarch.rpm

# 安装webstatic的yum源,主要是为了装php, epel 默认是5.4版本的php,所以其实这步可以省略
# 如果想体验更多版本的php，可以考虑装一下。
rpm -Uvh https://mirror.webtatic.com/yum/el7/webtatic-release.rpm

# 重新构建缓存
yum clean all
yum makecache

# 安装一些环境, 数据库这边我直接用自带的 mariadb， 当成mysql用就好了。
yum -y install zlib-devel mariadb-devel glibc-devel curl-devel gcc automake mariadb libidn-devel openssl-devel net-snmp-devel rpm-devel OpenIPMI-devel httpd mariadb-server perl-DBI net-tools

# 安装php, 新版本的php最低版本要求5.4以上，所以我只是按照标准的来。
yum -y install php-gd php-mysql php-bcmath php-mbstring php-xml php
# 其实本来不想装java-gateway，不过有的人想用jmx监控，我就一股脑都装上了。
yum -y install zabbix-server-mysql zabbix-web.noarch zabbix-web-mysql.noarch zabbix-agent zabbix-sender zabbix-java-gateway
# 启动数据库
systemctl start mariadb

# 创建zabbix数据库，并设置密码为:123456
mysql << EOF
create database zabbix character set utf8;
grant all on zabbix.* to zabbix@localhost identified by '123456';
quit
EOF

# 导入zbx数据库, 如果cd 目录进不去，可以先 cd 到/usr/share/doc/
# 查看zabbix-server-msyql具体目录名，因为这个目录名是跟着版本号
# 走的，所以不能直接复制我这里的路径，比如我这次安装就是3.0.1
cd /usr/share/doc/zabbix-server-mysql-3.0.1

# 以前是导入三个文件，现在做成一个压缩包了。
zcat create.sql.gz | mysql -uroot zabbix

# 配置跟以前写的那篇部署文档差不多，无非就是配置一下访问数据库和php的一些基本参数
# 照抄就行了。
# zabbix 连接数据库部分，修改连接的数据库名和密码
sed -i '/^DBName/s/=.*$/=zabbix/' /etc/zabbix/zabbix_server.conf
sed -i '/^# DBPassword/s/.*$/DBPassword=123456/' /etc/zabbix/zabbix_server.conf

# 配置php.ini一些必要参数
sed -i 's/post_max_size = 8M/post_max_size = 32M/g' /etc/php.ini
sed -i 's/upload_max_filesize = 2M/upload_max_filesize = 50M/g' /etc/php.ini
sed -i 's/;date.timezone =/date.timezone = Asia\/Shanghai/' /etc/php.ini
sed -i 's/max_execution_time = 30/max_execution_time = 600/g' /etc/php.ini
sed -i 's/max_input_time = 60/max_input_time = 600/g' /etc/php.ini
sed -i 's/memory_limit = 128M/memory_limit = 256M/g' /etc/php.ini
setenforce 0                   # 关闭selinux，开机关闭我想这个大家都知道，不知道的自己百度吧。
systemctl disable firewalld    # 关闭开机启动防火墙
systemctl stop firewalld       # 停止防火墙
systemctl enable mariadb       # 开机启动数据库
systemctl enable zabbix-server # 开机启动zabbix 服务端
systemctl enable zabbix-agent  # 开机启动zabbix 客户端
systemctl enable httpd         # 开机启动web服务
systemctl start zabbix-server  # 启动zabbix 服务端
systemctl start zabbix-agent   # 启动zabbix 客户端
systemctl start httpd          # 启动web服务

