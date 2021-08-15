***Debian10 部署 lnmp + redis + swoole 生产环境***
==============================================================

* ModifyTime:2020-06-15  Author:JerRie

------------

安装准备
--------------------------------------------------------------

> 更新系统

~~~bash
apt update -y && apt upgrade -y
~~~

> 安装基础工具

~~~bash
apt install -y build-essential wget curl bzip2 git screen
~~~

> 修改vi配置

~~~bash
apt-get install -y vim vim-runtime exuberant-ctags
echo "syntax on" >> ~/.vimrc && echo "set number" >> ~/.vimrc
~~~~

> 控制台着色配置

~~~bash
vi ~/.bashrc

#控制台显示样式
export PS1='\[\e[35;40m\][\u@\h \[\e[33;36m\]\W\[\e[33;40m\]]\[\e[35;40m\]❯\[\e[33;40m\]❯\[\e[32;40m\]❯ \[\e[01;37m\]'
#ls标识文件列表颜色
alias ls='ls --color=auto'

#生效配置
source ~/.bashrc
~~~

------------

安装NGINX
--------------------------------------------------------------

> NGINX 准备工作

~~~bash
#创建用户
groupadd www && useradd -s /sbin/nologin -g www www

#创建目录
mkdir -p /data/web && chown -R www:www /data/web

#安装依赖
apt install -y libpcre3-dev libssl-dev libzip-dev
~~~

> NGINX 下载源码包并解压

~~~bash
cd /usr/local/src && wget http://nginx.org/download/nginx-1.20.1.tar.gz && tar -zxvf nginx-1.20.1.tar.gz && cd nginx-1.20.1
~~~

> NGINX 编译及安装

~~~bash
./configure \
--prefix=/usr/local/nginx \
--without-http_memcached_module \
--user=www  \
--group=www \
--with-http_stub_status_module \
--with-http_ssl_module \
--with-http_gzip_static_module && make -j4 && make install
~~~

> NGINX 配置

~~~bash
vi /usr/local/nginx/conf/nginx.conf
~~~

~~~bash
1. user nobody; 除前面的#号并修改为 user www www;
2. pid logs/nginx.pid; 去除前面的#号
3. 修改格式(去除#号即可)
   log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                     '$status $body_bytes_sent "$http_referer" '
                     '"$http_user_agent" "$http_x_forwarded_for"';
4.  gzip  on; 开启资源压缩
    gzip_http_version 1.1;
    gzip_comp_level 3;
    gzip_types text/css application/javascript;
5. server_tokens off; 增加一行(隐藏版本)
6. 在http{} 节点末尾添加 “include ../vhost/*.vhost;” 使其支持vhost配置
~~~

> NGINX 并发优化

~~~bash
worker_processes 8;(跟据服务器CPU内核数量配置此参数)
worker_rlimit_nofile 65535; (增加此参数)
events {
    use epoll;
    worker_connections  65535;
}
~~~

> NGINX 配置环境变量

~~~bash
touch /etc/profile.d/nginx.sh
echo 'export PATH=$PATH:/usr/local/nginx/sbin/' > /etc/profile.d/nginx.sh
chmod 0777 /etc/profile.d/nginx.sh
source /etc/profile.d/nginx.sh
~~~

> NGINX 创建自启动脚本

~~~bash
vi /lib/systemd/system/nginx.service
~~~

~~~bash
[Unit]
Description=nginx
After=network.target

[Service]
Type=forking
PIDFile=/usr/local/nginx/logs/nginx.pid
ExecStart=/usr/local/nginx/sbin/nginx
ExecReload=/usr/local/nginx/sbin/nginx -s reload
ExecStop=/usr/local/nginx/sbin/nginx -s quit
PrivateTmp=true

[Install]
WantedBy=multi-user.target
~~~

~~~bash
systemctl enable nginx.service      #加入自启动服务
systemctl start nginx.service       #启动服务
~~~

------------

安装MARIADB
--------------------------------------------------------------

> MARIADB 安装准备

~~~bash
#添加用户组
groupadd mysql && useradd -s /sbin/nologin -g mysql mysql

#创建数据库目录
mkdir -p /usr/local/mysql && mkdir -p /data/mysql && chown -R mysql:mysql /data/mysql

#删除自带库
apt autoremove -y libmariadb*

#安装依赖
apt install -y cmake libncurses5-dev libgnutls28-dev bison
~~~

> MARIADB 下载源码包并解压

~~~bash
cd /usr/local/src && wget https://mariadb.nethub.com.hk/mariadb-10.5.9/source/mariadb-10.5.9.tar.gz && tar -zxvf mariadb-10.5.9.tar.gz && cd mariadb-10.5.9
~~~

> MARIADB 编译安装

~~~bash
cmake . -DCMAKE_INSTALL_PREFIX=/usr/local/mysql \
-DCMAKE_BUILD_TYPE=Release \
-DSYSCONFDIR=/usr/local/mysql/support-files \
-DMYSQL_DATADIR=/data/mysql \
-DMYSQL_UNIX_ADDR=/dev/shm/mysql.sock \
-DEXTRA_CHARSETS=all \
-DDEFAULT_CHARSET=utf8mb4 \
-DDEFAULT_COLLATION=utf8mb4_general_ci \
-DWITH_READLINE=1 \
-DWITH_EMBEDDED_SERVER=1 \
-DENABLED_LOCAL_INFILE=1 \
-DWITH_INNOBASE_STORAGE_ENGINE=1 \
-DWITH_ARCHIVE_STPRAGE_ENGINE=1 \
-DWITH_BLACKHOLE_STORAGE_ENGINE=1 \
-DWITHOUT_TOKUDB=1 && make -j4 && make install
~~~

> MARIADB 初始化数据库

~~~bash
cd /usr/local/mysql/ && scripts/mysql_install_db --user=mysql --datadir=/data/mysql
~~~

> MARIADB 复制MYSQL配置文件

~~~bash
cp /usr/local/mysql/support-files/wsrep.cnf /usr/local/mysql/support-files/my.cnf
~~~

> 指定mysql.server启停脚本中的
~~~bash
vi /usr/local/mysql/support-files/mysql.server
mysqld_pid_file_path='/data/mysql/mysqld.pid'
~~~

> MARIADB 创建自启动脚本

~~~bash
vi /lib/systemd/system/mysql.service
~~~

~~~bash
[Unit]
Description=MariaDB Community Server
After=network.target

[Service]
Type=forking
PermissionsStartOnly=true
PIDFile=/data/mysql/mysqld.pid
ExecStart=/usr/local/mysql/support-files/mysql.server start
ExecReload=/usr/local/mysql/support-files/mysql.server restart
ExecStop=/usr/local/mysql/support-files/mysql.server stop
PrivateTmp=true

[Install]
WantedBy=multi-user.target
~~~

~~~bash
systemctl enable mysql.service      #加入自启动服务
systemctl start mysql.service       #启动服务
~~~

> MARIADB 初始化 根据相关提示进行操作

~~~bash
./bin/mysql_secure_installation
~~~

~~~bash
Enter current password for root (enter for none):    输入当前root密码(回车)
Switch to unix_socket authentication [Y/n]           是否启用socket授权(n)
Change the root password? [Y/n]                      是否修改root用户密码?(Y)
New password:                                        输入新root密码(a123456)
Re-enter new password:                               确认输入root密码(a123456)
Remove anonymous users? [Y/n] Y                      删除匿名用户?(Y)
Disallow root login remotely? [Y/n] Y                不允许root登录远程?(Y)
Reload privilege tables now? [Y/n] Y                 现在重新加载权限表(Y)
~~~

> MARIADB 配置环境变量

~~~bash
touch /etc/profile.d/mysql.sh
echo 'export PATH=$PATH:/usr/local/mysql/bin/' > /etc/profile.d/mysql.sh
chmod 0777 /etc/profile.d/mysql.sh
source /etc/profile.d/mysql.sh
~~~

> MARIADB 创建外部可访问的管理员帐户

~~~bash
mysql -uroot -p
~~~

~~~bash
GRANT ALL PRIVILEGES ON *.* TO 'admin'@'%' IDENTIFIED BY '*********' WITH GRANT OPTION;
quit;
~~~

------------

安装PHP
--------------------------------------------------------------

> PHP 准备工作

~~~bash
#安装PHP依赖
apt install -y libxml2-dev libsqlite3-dev libcurl4-gnutls-dev libpng-dev libjpeg-dev libfreetype6-dev libonig-dev libxslt1-dev

#PKGCONFIG依赖
cd /usr/local/src && wget http://121.199.59.110/pkg-config-0.29.2.tar.gz && tar -zxvf pkg-config-0.29.2.tar.gz && cd pkg-config-0.29.2
./configure --with-internal-glib && make -j4 && make install

#编译所需要的环境变量
export PKG_CONFIG_PATH=/usr/lib/x86_64-linux-gnu/pkgconfig
~~~

> PHP 下载源码包并解压

~~~bash
cd /usr/local/src && wget http://121.199.59.110/php-7.4.16.tar.bz2 && tar jxf php-7.4.16.tar.bz2 && cd php-7.4.16
~~~

> PHP 编译及安装

~~~bash
./configure --prefix=/usr/local/php \
--with-config-file-path=/usr/local/php/etc \
--with-config-file-scan-dir=/usr/local/php/conf.d \
--enable-fpm \
--with-fpm-user=www \
--with-fpm-group=www \
--enable-mysqlnd \
--with-mysqli=mysqlnd \
--with-pdo-mysql=mysqlnd \
--with-iconv-dir \
--enable-gd \
--with-jpeg \
--with-freetype \
--with-zlib \
--enable-xml \
--disable-rpath \
--enable-bcmath \
--enable-shmop \
--enable-sysvsem \
--enable-inline-optimization \
--with-curl \
--enable-mbregex \
--enable-mbstring \
--enable-intl \
--enable-pcntl \
--enable-ftp \
--with-openssl \
--with-mhash \
--enable-pcntl \
--enable-sockets \
--with-xmlrpc \
--with-zip \
--enable-soap \
--with-gettext \
--enable-fileinfo \
--enable-opcache \
--with-xsl \
--with-pear && make -j4 && make install
~~~

> PHP 复制配置文件

~~~bash
cp php.ini-production /usr/local/php/etc/php.ini
~~~

> PHP 配制参数

~~~bash
vi /usr/local/php/etc/php.ini
~~~

~~~bash
1. 找到 memory_limit = 128M         修改为 memory_limit = 1024M
2. 找到 date.timezone =             修改为 date.timezone = PRC
3. 找到 expose_php = On             修改为 expose_php = Off
4. 找到 opcache.enable=1            修改为 opcache.enable=1
5. 找到 opcache.enable_cli=0        修改为 opcache.enable_cli=1
6. 在 Dynamic Extensions 代码块中添加 zend_extension=opcache.so
~~~

> PHP 配置php-fpm

~~~bash
cp /usr/local/php/etc/php-fpm.conf.default /usr/local/php/etc/php-fpm.conf
cp /usr/local/php/etc/php-fpm.d/www.conf.default /usr/local/php/etc/php-fpm.d/www.conf
cp sapi/fpm/init.d.php-fpm /usr/local/php/bin/php-fpm
chmod 0777 /usr/local/php/bin/php-fpm
~~~

> PHP 配置环境变量

~~~bash
touch /etc/profile.d/php.sh
echo 'export PATH=$PATH:/usr/local/php/bin/' > /etc/profile.d/php.sh
chmod 0777 /etc/profile.d/php.sh
source /etc/profile.d/php.sh
~~~

> PHP 使用SOCK方式监听

~~~bash
vi /usr/local/php/etc/php-fpm.d/www.conf
~~~

~~~bash
#指定SOCK文件位置
将 listen = 127.0.0.1:9000 修改为 listen = /dev/shm/php-fpm.sock

#监听的用户及组
listen.owner = www
listen.group = www
listen.mode = 0660
~~~

>PHP-FPM性能优化
~~~bash
#将默认动态创建子进程参数修改为静态创建方式
将 pm = dynamic 修改为 pm = static
pm.max_children = 300 (16G内存分配参考配置 说明:每个进程占用约20M~30M)
pm.max_requests = 500 (激活该参数 说明:每个工作进程处理完500个请求后自动重启,目的防止内存溢出)
rlimit_files = 65535 (最大允许读取的文件数, 对于并发情况建议设置最大值)
~~~

> PHP 创建自启动脚本

~~~bash
vi /lib/systemd/system/php-fpm.service
~~~

~~~bash
[Unit]
Description=php-fpm
After=network.target

[Service]
Type=forking
PIDFile=/usr/local/php/var/run/php-fpm.pid
ExecStart=/usr/local/php/bin/php-fpm start
ExecReload=/usr/local/php/bin/php-fpm restart
ExecStop=/usr/local/php/bin/php-fpm stop
PrivateTmp=true

[Install]
WantedBy=multi-user.target
~~~

~~~bash
systemctl enable php-fpm.service      #加入自启动服务
systemctl start php-fpm.service       #启动服务
~~~

------------

安装REDIS
--------------------------------------------------------------

> REDIS 准备工作

~~~bash
#安装依赖
apt install -y tcl
~~~

> REDIS 下载安装包并解压

~~~bash
cd /usr/local/src && wget http://121.199.59.110/redis-6.2.1.tar.gz && tar -zxvf redis-6.2.1.tar.gz && cd redis-6.2.1
~~~

> REDIS 编译及测试

~~~bash
make -j4 && make test
~~~

> REDIS 创建存储redis文件目录 将所需要的文件移到该目录下

~~~bash
mkdir -p /usr/local/redis
cp /usr/local/src/redis-6.2.1/redis.conf /usr/local/redis/
cp /usr/local/src/redis-6.2.1/src/redis-cli /usr/local/redis/
cp /usr/local/src/redis-6.2.1/src/redis-server /usr/local/redis/
~~~

> REDIS 配置参数

~~~bash
vi /usr/local/redis/redis.conf
~~~

~~~bash
找到 “daemonize no”       修改为 “daemonize yes”
找到 “maxmemory <bytes>”  修改为 “maxmemory 2147483648” （2G，跟据服务器配制适当填写）
~~~

> REDIS 创建自启动脚本

~~~bash
vi /lib/systemd/system/redis.service
~~~

~~~bash
[Unit]
Description=The redis-server Process Manager
After=syslog.target network.target
[Service]
Type=forking
PIDFile=/var/run/redis_6379.pid
ExecStart=/usr/local/redis/redis-server /usr/local/redis/redis.conf
ExecReload=/bin/kill -USR2 $MAINPID
ExecStop=/bin/kill -SIGINT $MAINPID
[Install]
WantedBy=multi-user.target
~~~

~~~bash
systemctl enable redis.service      #加入自启动服务
systemctl start redis.service       #启动服务
~~~

------------

PHP安装REDIS扩展支持
--------------------------------------------------------------

> REDIS 安装依赖

~~~bash
apt install -y autoconf
~~~

> REDIS 下载扩展包

~~~bash
cd /usr/local/src/php-7.4.16/ext && wget http://121.199.59.110/redis-5.2.2.tgz && tar -zxvf redis-5.2.2.tgz && cd redis-5.2.2
~~~

> REDIS编译安装

~~~bash
#执行扩展命令
/usr/local/php/bin/phpize

#编译及安装
./configure --with-php-config=/usr/local/php/bin/php-config && make -j4 && make install
~~~

> REDIS 修改php.ini

~~~bash
vi /usr/local/php/etc/php.ini
~~~

~~~bash
并找到 Dynamic Extensions 节点在下方增加如下（最后一行为添加项）
extension=redis.so
~~~

------------

PHP安装SWOOLE扩展支持
--------------------------------------------------------------

> SWOOLE 下载扩展包

~~~bash
cd /usr/local/src/php-7.4.16/ext && git clone https://github.com/swoole/swoole-src.git && cd swoole-src
~~~

> SWOOLE 编译安装

~~~bash
#执行扩展命令
/usr/local/php/bin/phpize

#编译及安装
./configure --with-php-config=/usr/local/php/bin/php-config && make && make install
~~~

> SWOOLE 修改php.ini

~~~bash
vi /usr/local/php/etc/php.ini
~~~

~~~bash
并找到 Dynamic Extensions 节点在下方增加如下（最后一行为添加项）
extension=swoole.so
~~~

安装NODEJS
--------------------------------------------------------------

> NODEJS 下载并解压

~~~bash
cd /usr/local/src && wget http://121.199.59.110/node-v13.11.0-linux-x64.tar.xz && tar -xvf node-v13.11.0-linux-x64.tar.xz
~~~

> NODEJS 复制到程序目录

~~~bash
mv node-v13.11.0-linux-x64 /usr/local/nodejs
~~~

> NODEJS 创建环境变量

~~~bash
touch /etc/profile.d/nodejs.sh
echo 'export PATH=$PATH:/usr/local/nodejs/bin/' > /etc/profile.d/nodejs.sh
chmod 0777 /etc/profile.d/nodejs.sh
source /etc/profile.d/nodejs.sh
~~~
