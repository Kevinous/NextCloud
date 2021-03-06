## NextCloud搭建测试

### Dependencies
首先按照官方文档先把依赖加上。

```shell
sudo apt-get install apache2 mariadb-server libapache2-mod-php7.2
sudo apt-get install php7.2-gd php7.2-json php7.2-mysql php7.2-curl php7.2-mbstring
sudo apt-get install php7.2-intl php-imagick php7.2-xml php7.2-zip
```
官方是没有sudo的，但会导致锁文件打不开权限不够的问题。

Apache是一个web服务器网页端服务，libapache2-mod-php7.2提供了必须的php扩展。

接着下载NextCloud源码，直接从官网[Link](https://nextcloud.com/install/)获取，选择Download for server->Archive File。会下载一个名为nextcloud-x.x.x.zip或者其他格式的压缩包。官方文档中说到需要下载对应校验和文件比如.md5或者.sha256，但是我没找到在哪里下载，并且似乎也并不需要（我是在windows里解压的）。PGP签名也是同样没有遇到，不清楚为什么。

之后解压/提取出来会有一个/nextcloud目录，按照文档把这个目录拷到 `/var/www`下。
```shell
sudo cp -r nextcloud /var/www
sudo chown -R www-data:www-data /var/www/nextcloud/
```
这里还需要将nextcloud的所有者切换到www-data，文档中没有说
### Apache
之后开始配置apache。

由于之前已经下载好了这里可以直接设置。

此时从浏览器进入页面可以看到it works!

可以改动绑定的端口，`Listen 80`可以改为自己想用的端口，内容如下
```shell
sudo vi /etc/apache2/ports.conf
########################################content is as follows
# If you just change the port or add more ports here, you will likely also
# have to change the VirtualHost statement in
# /etc/apache2/sites-enabled/000-default.conf

Listen 80

<IfModule ssl_module>
        Listen 443
</IfModule>

<IfModule mod_gnutls.c>
        Listen 443
</IfModule>
# vim: syntax=apache ts=4 sw=4 sts=4 sr noet                                       
```

接着配置虚拟主机。这里说一下sites-enabled和sites-available的关系。/etc/apache2/apache.conf是真正的配置文件，Apache启动时会看这个文件，其他的配置文件通过引用的方式包含进来。sites-enabled是apache.conf里出现的一个目录，它里面存放的都是链接文件，链接指向/etc/apache2/sites-available。因此可以说真正的配置文件在sites-available中。但是使用sites-enabled的好处在于可以比较简单地添加删除某个配置好的主机，只需要增删链接文件而不用直接对配置文件进行操作。

```shell
cd /etc/apache2/sites-available
sudo vi nextcloud.conf
```

写入以下内容（来自NextCloud官方文档中）
```html
Alias /nextcloud "/var/www/nextcloud/"

<Directory /var/www/nextcloud/>
  Options +FollowSymlinks
  AllowOverride All

 <IfModule mod_dav.c>
  Dav off
 </IfModule>

 SetEnv HOME /var/www/nextcloud
 SetEnv HTTP_HOME /var/www/nextcloud

</Directory>
```
之后将其链接到sites-enabled中
```shell
sudo ln -s /etc/apache2/sites-available/nextcloud.conf /etc/apache2/sites-enabled/nextcloud.conf
```

此时启动127.0.0.1:80可以看到nextcloud的页面。


### MariaDB
MariaDB是兼容MYSQL的，之前通过官方文档的下载依赖已经下载好了MariaDB。

启动并且配置MariaDB
```shell
sudo systemctl start mysql  
#启动
sudo systemctl status mysql
#查看状态
sudo mysql_secure_installation
#执行初始化安全脚本，默认root密码为空，按照脚本里的提示进行初始化
```

之后创建NextCloud创建数据库和用户
创建数据库nextcloud，用户名nextcloud，密码123456
```mysql
$ sudo mysql -u root -p
MariaDB [(none)]> CREATE DATABASE nextcloud;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nextcloud.* TO 'nextcloud'@'localhost' IDENTIFIED BY 'XXXXXXXX';
MariaDB [(none)]> FLUSH PRIVILEGES;
MariaDB [(none)]> \q
```
其中
`grant 权限1,权限2,…权限n on 数据库名称.表名称 to 用户名@用户地址 identified by ‘连接口令’;`
权限1,权限2,…权限n代表select,insert,update,delete,create,drop,index,alter,grant,references,reload,shutdown,process,file等14个权限。
当权限1,权限2,…权限n被all privileges或者all代替，表示赋予用户全部权限。
当数据库名称.表名称被*.*代替，表示赋予用户操作服务器上所有数据库所有表的权限。
用户地址可以是localhost，也可以是ip地址、机器名字、域名。也可以用’%'表示从任何地址连接。
‘连接口令’不能为空，否则创建失败。

#### 目前使用Apacha成功运行了Nextcloud，接下来到nginx部分

### Nginx

首先安装nginx并设置自启动

```shell
sudo apt install nginx
sudo systemctl enable nginx
sudo systemctl start nginx
```

(可以使用`systemctl status nginx`查看状态）

并且因为需要使用php扩展来支持并且优化php，我们也需要开启php-fpm

```shell
sudo apt install php-fpm
sudo systemctl enable php-fpm
sudo systemctl start php-fpm
```

同时因为现在开始使用nginx代理，我们将之前的Apache取消开机自启动

```shell
sudo systemctl disable apache2.service
sudo systemctl stop apache2.service
```


接着创建一个Nginx Server Block

```shell
cd /etc/nginx/sites-available
sudo vi nextcloud
```
接着将[此处](https://www.linuxbabe.com/ubuntu/install-nextcloud-ubuntu-18-04-nginx-lemp)的Step 3处代码复制进去

然后进行软连接
```shell
ln -s ../sites-enabled/nextcloud nextcloud
```

接着测试一下语法是否有问题
```shell
sudo nginx -t
```

如果没有问题会反馈success，接下来让nginx重新加载

```shell
sudo systemctl reload nginx
```
_需要注意的是，千万记住要改动trusted_domin部分，详见[此处]()_
此时使用http服务的部署在nginx上的nextcloud就完成了。

### 反向代理

反向代理，即由某个服务器代理多个服务器，使得这几个服务器对客户端隐藏，起到资源均衡的作用。使用nginx代理即是使用proxy_pass来进行定向

这里为了体现出代理的作用我先改动`sites-available/nextcloud`文件，将`listen 80`改为`listen 8080`。此时再访问192.168.192.131(本地ip)出现404，访问192.168.192.131:8080出现nextcloud登录界面。

```shell
cd /etc/nginx/sites-available
sudo vi proxy
##
#paste these
#
#upstream myNextcloud {
#   	server 192.168.192.131:8080; 
#}
#server {
#       listen 80;
#       server_name _;
#       location / {
#         proxy_pass http://myNextcloud;
#         index index.html index.htm;
#         proxy_set_header Host $host;
#	}
#}
###end of content
ln -s proxy ../sites-enabled/proxy
```
_这里的proxy_set_header Host是必须的，否则会出现不信任问题———因为发给真实服务器的http请求的HOST字段还是反向代理服务器的HOST，因此没有经过设置trusted_domin会被拒绝，加上这一句header后即可解决_

-------

#### proxy_set_header
Host的含义是表明请求的主机名，因为nginx作为反向代理使用，而如果后端真是的服务器设置有类似防盗链或者根据http请求头中的host字段来进行路由或判断功能的话，如果反向代理层的nginx不重写请求头中的host字段，将会导致请求失败【默认反向代理服务器会向后端真实服务器发送请求，并且请求头中的host字段应为proxy_pass指令设置的服务器】。

同理，X_Forward_For字段表示该条http请求是有谁发起的？如果反向代理服务器不重写该请求头的话，那么后端真实服务器在处理时会认为所有的请求都来在反向代理服务器，如果后端有防攻击策略的话，那么机器就被封掉了。因此，在配置用作反向代理的nginx中一般会增加两条配置，修改http的请求头：

proxy_set_header Host $http_host;
proxy_set_header X-Forward-For $remote_addr;

这里的$http_host和$remote_addr都是nginx的导出变量，可以再配置文件中直接使用。如果Host请求头部没有出现在请求头中，则$http_host值为空，但是$host值为主域名。因此，一般而言，会用$host代替$http_host变量，从而避免http请求中丢失Host头部的情况下Host不被重写的失误。
X-Forwarded-For:简称XFF头，它代表客户端，也就是HTTP的请求端真实的IP，只有在通过了HTTP 代理或者负载均衡服务器时才会添加该项。 它不是RFC中定义的标准请求头信息，在squid缓存代理服务器开发文档中可以找到该项的详细介绍。标准格式如下：X-Forwarded-For: client1, proxy1, proxy2。

--------

输入内容后
```shell
sudo nginx -t  #测试语法
sudo nginx -s reload  #重新加载配置
```
此时访问localhost可以登录到nextcloud。
