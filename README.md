# lnmp搭建wordpress详细教程：从选服务器到 HTTPS，8 个步骤跑通并解决 502 报错

搜「lnmp搭建wordpress详细教程」的人，通常已经翻过几篇命令清单，发现照着敲完，首页还是打不开。问题很少出在某一项软件没装上，而是 Nginx、PHP-FPM、MySQL 三者的连接参数没对齐。

这篇按实际部署顺序拆成 8 步：选机器、装 Nginx、装 MySQL、装 PHP、写站点配置、传 WordPress、配 HTTPS、做加固。每步都写清楚验证方法，最后单独整理 502、403、数据库连接失败这几个高频报错的排查顺序。

如果你只是想要一个能写文章的站点，不想碰编译和配置文件，也可以走另一条路：云服务器的应用镜像。下面速览里给了入口，教程部分仍然完整保留，按需取用。

> **选购速览**：个人博客或刚起步的站点，腾讯云轻量应用服务器 2核2G 档就够跑 WordPress，控制台里直接选 LNMP 或 WordPress 应用镜像，开机后按提示完成初始化即可，省掉编译和改配置的时间。👉 [查看轻量应用服务器当前活动价](https://bit.ly/yunhuodong)

## 先选对机器：LNMP 到底吃多少资源

WordPress 本身不重，真正占内存的是 MySQL 和 PHP-FPM 进程。个人博客 2核2G 能跑，装了 WooCommerce 或插件超过 20 个，建议直接上 2核4G，否则 MySQL 容易因为内存不足被系统杀掉，表现就是站点突然报“建立数据库连接时出错”。

腾讯云轻量应用服务器按 CPU、内存、带宽、系统盘组合成固定档位，套餐里含每月流量包，超出后按量计费。下面是常见的几档配置，价格随活动波动，首年优惠价和续费价通常不一样，下单前把计费周期看清楚。

| 配置档位 | vCPU / 内存 | 带宽 | 系统盘 | 适合什么站点 | 价格与购买 |
| --- | --- | --- | --- | --- | --- |
| 入门型 | 2核 2G | 3Mbps | 40GB SSD | 个人博客、刚起步 | 活动页实时价  [查看](https://bit.ly/yunhuodong) |
| 轻量型 | 2核 2G | 4Mbps | 60GB SSD | 图文博客、日 PV 几百 | 活动页实时价  [查看](https://bit.ly/yunhuodong) |
| 标准型 | 2核 4G | 5Mbps | 80GB SSD | 插件较多、带商城功能 | 活动页实时价  [查看](https://bit.ly/yunhuodong) |
| 进阶型 | 2核 4G | 6Mbps | 100GB SSD | 多站点、日 PV 几千 | 活动页实时价  [查看](https://bit.ly/yunhuodong) |
| 高性能型 | 4核 8G | 10Mbps | 160GB SSD | 企业站、多用户后台 | 活动页实时价  [查看](https://bit.ly/yunhuodong) |

轻量应用服务器适合单机部署，自带防火墙和快照，控制台里点几下就能完成备份。如果后面要做负载均衡、多台机器横向扩展，再考虑 CVM 云服务器，个人站长用轻量就够。

## 第 1 步：系统初始化和安全组

系统选 Ubuntu 22.04 LTS 或 Debian 12，软件源里的 Nginx、PHP、MySQL 版本都比较新，包管理命令简单。习惯 RHEL 系的可以用 AlmaLinux 或 CentOS Stream，但要注意 SELinux 会拦截 Nginx 读取网站目录，需要额外处理。

开机后先做三件事：

bash
# 更新系统
apt update && apt upgrade -y

# 设置时区，避免文章发布时间错乱
timedatectl set-timezone Asia/Shanghai

# 确认时间同步正常
timedatectl status


然后在云控制台的安全组或防火墙里放行 22、80、443 三个端口。80 和 443 不放行，后面 Nginx 装得再对，外网也访问不到，这一步漏掉的人不少。

如果用 RHEL 系系统，执行：

bash
setenforce 0
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config


## 第 2 步：安装 Nginx

新手直接用包管理器装，版本稳定、升级方便。需要第三方模块（比如某些缓存模块）才考虑编译安装，那是另一个工作量级。

bash
apt install nginx -y
systemctl enable --now nginx
nginx -v


浏览器访问服务器公网 IP，看到 “Welcome to nginx!” 页面就说明 Web 服务正常。如果打不开，先查安全组，再查 `systemctl status nginx`。

Nginx 的默认站点目录在 `/var/www/html`，配置文件在 `/etc/nginx/sites-available/`，后面第 5 步会新建一个 WordPress 专用配置。

## 第 3 步：安装 MySQL 8

Ubuntu 22.04 源里是 MySQL 8.0，直接装：

bash
apt install mysql-server -y
systemctl enable --now mysql
mysql --version


装完跑一遍安全初始化，设置 root 密码、删除匿名用户、禁止 root 远程登录：

bash
mysql_secure_installation


这里有一个容易踩的坑：MySQL 8 默认使用 `caching_sha2_password` 认证插件。PHP 7.4 之后的 mysqlnd 驱动能正常处理，但如果你的 PHP 版本更老，或者用了某些第三方数据库客户端，就会报认证失败。遇到这种情况，可以给 WordPress 专用账号单独改成兼容模式：

sql
ALTER USER 'wpuser'@'localhost' IDENTIFIED WITH mysql_native_password BY '你的强密码';


不建议为了省事让 WordPress 直接用 root 连数据库。单独建账号、只授权业务库，后面出问题也好排查。

## 第 4 步：安装 PHP 8.1 和扩展

WordPress 官方推荐 PHP 8.1 及以上。Ubuntu 22.04 默认源里就是 8.1，直接装：

bash
apt install php8.1-fpm php8.1-mysql php8.1-gd php8.1-curl \
php8.1-mbstring php8.1-xml php8.1-zip php8.1-intl php8.1-redis -y
systemctl enable --now php8.1-fpm


`php8.1-fpm` 是 FastCGI 进程管理器，Nginx 本身不会执行 PHP，要把 `.php` 请求转给它处理。`php8.1-mysql` 负责连数据库，`gd` 处理图片缩略图，`mbstring` 和 `intl` 处理中文和多语言，缺一个都可能让后台某功能白屏。

接着改 PHP 的上传限制。WordPress 后台传主题、传插件、传媒体文件都受这个限制：

bash
nano /etc/php/8.1/fpm/php.ini


找到并改成：

ini
upload_max_filesize = 64M
post_max_size = 64M
memory_limit = 256M
max_execution_time = 300


改完重启 PHP-FPM：`systemctl restart php8.1-fpm`。

注意，Nginx 那边还有一个 `client_max_body_size`，默认只有 1M，不同步改的话，PHP 放行了 Nginx 还是会返回 413。这个参数在第 5 步的配置里一起写。

## 第 5 步：写 Nginx 站点配置

新建一个配置文件：

bash
nano /etc/nginx/sites-available/wordpress


内容如下，把 `example.com` 换成你的域名或公网 IP：

nginx
server {
    listen 80;
    server_name example.com www.example.com;
    root /var/www/wordpress;
    index index.php index.html;

    client_max_body_size 64m;

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.1-fpm.sock;
    }

    location ~ /\.ht {
        deny all;
    }
}


几个关键点：

`try_files` 那一行负责把固定链接（伪静态）请求转给 `index.php`，少了它，WordPress 后台把固定链接改成“文章名”形式后，内页全部 404。

`fastcgi_pass` 的 socket 路径必须和实际安装的 PHP 版本一致。装的是 8.1 就写 `php8.1-fpm.sock`，写成 8.2 会直接 502。不确定的话用 `ls /run/php/` 看一眼。

启用配置并重载：

bash
ln -s /etc/nginx/sites-available/wordpress /etc/nginx/sites-enabled/
rm -f /etc/nginx/sites-enabled/default
nginx -t
systemctl reload nginx


`nginx -t` 显示 syntax is ok 再重载，配置写错时它会直接告诉你哪一行有问题。

## 第 6 步：建库、传 WordPress、跑安装

先建数据库和专用账号：

sql
CREATE DATABASE wordpress DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'wpuser'@'localhost' IDENTIFIED BY '换成强密码';
GRANT ALL PRIVILEGES ON wordpress.* TO 'wpuser'@'localhost';
FLUSH PRIVILEGES;
EXIT;


字符集用 `utf8mb4`，不要用 `utf8`。后者在 MySQL 里只支持三字节，存 emoji 或部分生僻字会截断。

下载并解压 WordPress 中文版：

bash
cd /tmp
wget https://cn.wordpress.org/latest-zh_CN.tar.gz
tar -xzf latest-zh_CN.tar.gz
mv wordpress /var/www/


设置目录权限。Nginx 和 PHP-FPM 在 Ubuntu 下都以 `www-data` 身份运行：

bash
chown -R www-data:www-data /var/www/wordpress
find /var/www/wordpress -type d -exec chmod 755 {} \;
find /var/www/wordpress -type f -exec chmod 644 {} \;


浏览器打开域名或公网 IP，会进入 WordPress 安装向导，填数据库名、用户名、密码、表前缀，然后设置站点标题和管理员账号。表前缀不要用默认的 `wp_`，改一个别的，能挡掉一部分针对默认前缀的扫描请求。

到这一步站点就能用了。如果你想跳过前面这些步骤，腾讯云轻量应用服务器的 WordPress 应用镜像已经把 Nginx、MySQL、PHP 和 WordPress 全部配好，开机后按控制台提示设置管理员账号即可。👉 [用应用镜像开通一台 WordPress 服务器](https://bit.ly/yunhuodong)

## 第 7 步：绑定域名和 HTTPS

域名解析到服务器公网 IP，A 记录指向即可，生效通常几分钟到几小时。解析生效前，先把 Nginx 配置里的 `server_name` 改成真实域名。

HTTPS 用 Let's Encrypt 的免费证书，certbot 会自动改好 Nginx 配置：

bash
apt install certbot python3-certbot-nginx -y
certbot --nginx -d example.com -d www.example.com


按提示输入邮箱、同意协议，certbot 会自动申请证书、配置 443 端口、加一条 80 跳 443 的规则。证书 90 天有效，certbot 装完会自动创建续期定时任务，用下面这条命令测试续期是否正常：

bash
certbot renew --dry-run


装完 HTTPS 后，去 WordPress 后台把站点地址改成 `https://` 开头，否则后台可能出现混合内容警告。

## 第 8 步：安全加固和备份

最小化的加固清单：

- 防火墙只放行 22、80、443，MySQL 的 3306 不要对公网开放
- WordPress 后台限制登录尝试次数，装一个限制登录插件
- `wp-config.php` 里加 `define('DISALLOW_FILE_EDIT', true);`，禁止后台直接编辑主题和插件文件
- 定期更新 WordPress 核心、主题和插件，旧版本插件是入侵重灾区
- 开启云服务器快照，每周至少一次

备份分两层：数据库用 `mysqldump` 导出，网站文件打包压缩；整机层面用云控制台的快照功能。快照恢复快，但要花钱占存储，数据库导出文件小、方便迁移，两个都留着最稳。

## 几个高频报错怎么查

**502 Bad Gateway**：Nginx 连不上 PHP-FPM。按顺序查三处——`systemctl status php8.1-fpm` 看进程在不在；`ls /run/php/` 看 socket 文件名和配置里写的是否一致；`/var/log/nginx/error.log` 看具体报错。九成是 socket 路径写错或 PHP-FPM 没启动。

**403 Forbidden**：Nginx 没有权限读取网站目录。检查 `root` 路径是否正确、目录属主是不是 `www-data`、`index` 指令里有没有 `index.php`。RHEL 系系统还要看 SELinux 是否拦截。

**建立数据库连接时出错**：先确认 MySQL 服务在运行，再用 `mysql -u wpuser -p` 手动登录测试账号密码。如果报认证插件错误，就是前面说的 `caching_sha2_password` 问题。也可能是 `wp-config.php` 里的数据库主机写成了公网 IP，本地连接应该用 `localhost`。

**上传主题或插件失败**：三处限制要同时放开——`php.ini` 里的 `upload_max_filesize` 和 `post_max_size`、Nginx 的 `client_max_body_size`、以及 WordPress 自身的文件权限。

**后台白屏或某功能报错**：多半是 PHP 扩展缺失。用 `php -m` 列出已装扩展，对照 WordPress 官方要求逐项核对，`gd`、`mbstring`、`curl` 是最常缺的三个。

## 手动装还是用镜像：按你的情况选

只想尽快有一个能写文章的站点，不打算折腾服务器配置，选轻量应用服务器的 WordPress 或 LNMP 应用镜像，2核2G 档起步，开机十分钟内能进后台。套餐里的月流量包对个人博客通常够用，超出后按量计费，下单前把首年价和续费价都看清楚。

想搞明白 Nginx、PHP-FPM、MySQL 之间怎么协作，或者后续要做缓存、调优、多站点，按上面 8 步手动装一遍，遇到问题知道从哪查起，这套经验后面迁移服务器、排查故障都用得上。

站点已经有稳定流量、装了商城或会员功能，2核4G 起步更稳妥，MySQL 和 PHP-FPM 同时跑起来内存占用不低。预算允许的话直接上 4核8G 档，省得后面因为内存不足频繁重启服务。

买之前最后确认两件事：系统盘容量够不够放网站文件和备份，月流量包能不能覆盖你的访问量。这两项超了，账单会比套餐价高不少。👉 [对比腾讯云轻量应用服务器各档配置与当前价格](https://bit.ly/yunhuodong)
