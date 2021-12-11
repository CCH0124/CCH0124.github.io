---
title: LNMP 與 wordpress
date: 2018-05-28
description: "Nginx + PHP-fpm + Mariadb 基本安裝"
tags: [Ubuntu, nginx, Mariadb, PHP]
draft: false
---

## Install And Setting

### Nginx install
可以透過 `apt show` 或 `apt search` 去查詢 nginx 版本

```shell
$ sudo apt install nginx -y # 安裝
```

服務使用

```shell
$ sudo systemctl start nginx.service
$ sudo systemctl stop nginx.service
$ sudo systemctl restart nginx.service
$ sudo systemctl status nginx.service
```

### Check Nginx Web Service
```shell
$ ps -aux | grep nginx
root      12900  0.0  0.0 124976  1420 ?        Ss   21:03   0:00 nginx: master process /usr/sbin/nginx -g daemon on; master_process on;
www-data  12901  0.0  0.1 125336  3184 ?        S    21:03   0:00 nginx: worker process
```
```shell
$ sudo systemctl status nginx.service
● nginx.service - A high performance web server and a reverse proxy server
   Loaded: loaded (/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
   Active: active (running) since Thu 2019-09-05 21:03:15 CST; 11min ago
 Main PID: 12900 (nginx)
    Tasks: 2
   Memory: 3.6M
      CPU: 40ms
   CGroup: /system.slice/nginx.service
           ├─12900 nginx: master process /usr/sbin/nginx -g daemon on; master_process on
           └─12901 nginx: worker process

Sep 05 21:03:15 ubuntu systemd[1]: Starting A high performance web server and a reverse proxy server...
Sep 05 21:03:15 ubuntu systemd[1]: nginx.service: Failed to read PID from file /run/nginx.pid: Invalid argument
Sep 05 21:03:15 ubuntu systemd[1]: Started A high performance web server and a reverse proxy server.
```

```shell
$ curl http://localhost 
# nginx 預設頁面
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
cch@ubuntu:~$
```

如果無法取得 nginx 預設頁面，至 `/var/log/nginx/error.log` 查看問題。適當地從 log 中找尋錯誤，也是可以解決問題的。

### Nginx Confing File
目錄：`/etc/nginx`

```shell
$ ls /etc/nginx/
conf.d        fastcgi_params  koi-win     nginx.conf    scgi_params      sites-enabled  uwsgi_params
fastcgi.conf  koi-utf         mime.types  proxy_params  sites-available  snippets       win-utf
```

其中 `nginx.conf` 設定檔，所有基本設置都在此檔案內。
`sites-enabled` 啟用站台設定，裡面有著初始站台預設設定值，如有更改檔名則 `nginx.conf` 配置需更改。
`sites-available` 可以啟用的站台

```shell
$ ls -l sites-enabled/ # 裡面是 ln 至 site-available 
total 0
lrwxrwxrwx 1 root root 34 Sep  5 21:03 default -> /etc/nginx/sites-available/default
```

查看 `nginx.conf` 預設值
```shell
user www-data; # 定義 nginx 運作使用者
worker_processes auto; # 定義 nginx 的 process，在這邊設定為 CPU 總核心數目
pid /run/nginx.pid; # process ID

# events 區塊，主要是影響 Nginx 伺服器與使用者的網路連接
# 此區塊對 Nginx 效能影響很大，但扔然需要依照實際狀況調整
events {
        worker_connections 768; # 允許每個 worker process 同時開啟的最大連接數
                                # 設定每個 worker process 都有能力同時接收網路連線
        # multi_accept on; 預設為註解，代表一次只能接收一個網路連線，建議啟用
}

http {

        ##
        # Basic Settings
        ##

        sendfile on; # 啟用高效率文件傳輸模式，如果使用 IO 負載重的建議設為 off
        # 用 sendfile 此 tcp_nopush 才有作用
        # 當接到要求後不會馬上傳送出去，等到一定量在傳輸
        # 這樣做法有助於解決網路阻塞
        # 對於大文件頗有幫助
        tcp_nopush on; 
        tcp_nodelay on; # 與 tcp_nopush 是相反的
        keepalive_timeout 65; # 最常連接時間
        types_hash_max_size 2048; # 設定越大，使用記憶體越多，但檢索速度變快
        # server_tokens off;

        # server_names_hash_bucket_size 64;
        # server_name_in_redirect off;

        include /etc/nginx/mime.types; # 定義包含文件類型對應表
        default_type application/octet-stream; # 預設文件類型

        ##
        # SSL Settings
        ##

        ssl_protocols TLSv1 TLSv1.1 TLSv1.2; # Dropping SSLv3, ref: POODLE
        ssl_prefer_server_ciphers on;

        ##
        # Logging Settings
        ##
        # 設定 log 部分
        access_log /var/log/nginx/access.log;
        error_log /var/log/nginx/error.log;

        ##
        # Gzip Settings
        ##

        gzip on; # 啟用壓縮
        gzip_disable "msie6";

        # gzip_vary on;
        # gzip_proxied any;
        # gzip_comp_level 6; # 壓縮等級
        # gzip_buffers 16 8k; # 壓縮緩衝
        # gzip_http_version 1.1; # 壓縮版本
        # gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

        ##
        # Virtual Host Configs
        ##

        include /etc/nginx/conf.d/*.conf;
        include /etc/nginx/sites-enabled/*;
}
...
```

查看 `sites-available/default`
```shell
# Default server configuration
#
server {
        # 開 port 80
        listen 80 default_server;
        listen [::]:80 default_server;

        # SSL 設定區塊，如果啟用 `http/2` 則必須成功設定 SSL 方能使用
        # SSL configuration
        # 
        # listen 443 ssl default_server;
        # listen [::]:443 ssl default_server;
        #
        # Note: You should disable gzip for SSL traffic.
        # See: https://bugs.debian.org/773332
        #
        # Read up on ssl_ciphers to ensure a secure configuration.
        # See: https://bugs.debian.org/765782
        #
        # Self signed certs generated by the ssl-cert package
        # Don't use them in a production server!
        #
        # include snippets/snakeoil.conf;

        root /var/www/html; # 設定主目錄

        # Add index.php to the list if you are using PHP
        index index.html index.htm index.nginx-debian.html;
        # 設定伺服器名稱
        server_name _;

        location / {
                # First attempt to serve request as file, then
                # as directory, then fall back to displaying a 404.
                try_files $uri $uri/ =404;
        }

        # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
        #
        #location ~ \.php$ {
        #       include snippets/fastcgi-php.conf;
        #
        #       # With php7.0-cgi alone:
        #       fastcgi_pass 127.0.0.1:9000;
        #       # With php7.0-fpm:
        #       fastcgi_pass unix:/run/php/php7.0-fpm.sock;
        #}

        # deny access to .htaccess files, if Apache's document root
        # concurs with nginx's one
        #
        #location ~ /\.ht {
        #       deny all;
        #}
}

```

但上述的預設是否可用 ? 原則上自行測試是可用，但上限對外則不適合，如服務的人多流量高，則必須適當調整 nginx 和 Linux 參數
記住有做更動須重啟服務

## PHP-fpm Install

```shell
$ sudo apt install php -y
```

設定檔 `/etc/php/7.0`

服務使用

```shell
$ sudo systemctl start php7.0-fpm.service
$ sudo systemctl stop php7.0-fpm.service
$ sudo systemctl restart php7.0-fpm.service
$ sudo systemctl status php7.0-fpm.service
```

### PHP config setting
預設即可，在依照硬體因素和網站服務設定，只需增加時間區

```shell
$ sudo vi /etc/php/7.0/fpm/php.ini
date.timezone ="Asia/Taipei"
cgi.fix_pathinfo=0 # 安全設定，與 PHP 語法 $_SERVER['PATH_INFO'] 有關
$ sudo vi /etc/php/7.0/fpm/pool.d/www.conf # 效能校條可以從此檔案，本人未詳細研究
```

## Mariadb Install
有另外寫一篇 Mariadb Install，內容有資料庫建立及刪除。

參考 [mariadb install](https://cch0124.github.io/mariadb/)


## nginx 設定 php-fpm
`Nginx` 將 HTTP request 轉給 PHP 有兩種方式，第一種使用 `IP`，第二種用 `Socket`
`Socket` 效能較優，`fastcgi_splite_path_info` 正規語法，原本預設 `nginx` 是讀不到 `path_info` 值。
第一個擷取的值會給 `$fastcgi_script_name`，第二個擷取的值會給 `$fastcgi_path_info`
假設原始請求為 `/show.php/article/0001`，透過切割後 `script_filename` : /path/to/php/show.php，`path_info`: /article/0001

```shell
$ sudo vi /etc/nginx/sites-enabled/default
...
        #pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
        #
        location ~ \.php$ {
                fastcgi_split_path_info ^(.+\.php)(/.+)$;
                include fastcgi.conf;
                try_files $uri 404;
                # With php7.0-fpm:
                fastcgi_pass unix:/run/php/php7.0-fpm.sock;
                fastcgi_index index.php; # 設定 PHP 首頁
        }
...
```

重啟服務，利用 `phpinfo` 檢查是否有透過 `nginx` 解析。

```shell
$ sudo su -c "echo \"<?php phpinfo(); ?>\" | tee /var/www/html/index.php"
<?php phpinfo(); ?>
$ curl http://localhost/index.php # 驗證
```

## nginx VirtualHost

設定檔 `/etc/nginx/default`，一個站台一個 virtualhost，用 `include` 方式匯入。

範例
```shell
server {
        listen 80;
        listen [::]:80;

        root /home/class/A;
        server_name A.osx.stu.edu.tw;

        index index.php;
}

server {
        listen 80;
        listen [::]:80;

        root /home/class/B;
        server_name B.osx.stu.edu.tw;

        index index.php;
}
```

## 實作
1. 安裝 nginx + php7 + php-fpm + mariadb
2. 安裝 phpmyadmin
- 安裝至 ~/wordpress/myadmin
3. 設定 Nginx
- 修改主目錄

第一步驟在前面教學已完成。
第二步驟
```shell
$ wget https://files.phpmyadmin.net/phpMyAdmin/4.9.0.1/phpMyAdmin-4.9.0.1-all-languages.zip
$ mkdir ~/wordpress/
$ mv phpMyAdmin-4.9.0.1-all-languages myadmin
$ mv myadmin/ wordpress/
```
第三步驟
```shell
$ sudo vim /etc/nginx/sites-enabled/default
...
#root /var/www/html;
root /home/cch/wordpress/;
...
$ sudo systemctl restart nginx.service
```

```shell
$ cd wordpress/myadmin/
~/wordpress/myadmin$ cp config.sample.inc.php config.inc.php
~/wordpress/myadmin$ vi config.inc.php
$cfg['blowfish_secret'] #用 / 搜尋 blowfish_secret，找到後隨意填入字元
```

出現了問題，有些套件沒裝到

```shell
$ sudo apt install php php-mysql php-gd php-cli php-curl php-mcrypt php-common php-fpm php-mbstring php-xml -y
```

最後瀏覽器輸入 `http://192.168.137.142/myadmin/index.php` 即可瀏覽。`192.168.137.142` 為主機 IP。


上述為 phpmyadmin 的實作，接下來是實作創建資料庫，後續是為了 wordpress 架設

1. 建立資料庫名稱為 `wordpressDB`

```shell
MariaDB [(none)]> CREATE DATABASE wordpressDB;
```

2. 帳號密碼為 `pressadm`:`pressadm`，給 wordpress 使用

```shell
MariaDB [(none)]> CREATE USER 'pressadm'@'%';
Query OK, 0 rows affected (0.05 sec)

MariaDB [(none)]> GRANT USAGE ON *.* TO 'pressadm'@'%';
Query OK, 0 rows affected (0.04 sec)

MariaDB [(none)]> use mysql;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MariaDB [mysql]> UPDATE user SET password=PASSWORD("pressadm") WHERE user='pressadm';
Query OK, 1 row affected (0.04 sec)
Rows matched: 1  Changed: 1  Warnings: 0
MariaDB [mysql]> GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, DROP, INDEX, ALTER ON `wordpressDB`.* TO 'pressadm'@'%'
    -> ;
Query OK, 0 rows affected (0.01 sec)

MariaDB [mysql]> FLUSH PRIVILEGES;
Query OK, 0 rows affected (0.04 sec)
```

上述的步驟，也可以從 phpmyadmin 使用。

安裝 wordpress 及設定。

1. 安裝

```shell
$ wget https://tw.wordpress.org/wordpress-5.2.3-zh_TW.zip
$ unzip wordpress-5.2.3-zh_TW.zip
```

因為上述有創建 wordpress 目錄因此接壓縮會到這目錄下。
設定檔部分在 phpmyadmin 時已經設定了。

瀏覽器輸入 `http://192.168.137.142/index.php`，即可進入設定畫面。如下

![](https://i.imgur.com/bPwoVa7.png)

隨然設定了，但是會出現權限問題，要如下設定

```shell
$ sudo chown -R www-data:www-data /home/cch/wordpress
```

大致上 wordpress 就建置完成了。