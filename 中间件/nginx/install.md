# nginx 安装

## 编译安装

下载nginx包：http://nginx.org/en/download.html

```bash
wget http://nginx.org/download/nginx-1.24.0.tar.gz
```

安装编译环境

```bash
yum -y install gcc gcc-c++
```

安装pcre软件包（使nginx支持http rewrite模块）

```bash
yum install -y pcre pcre-devel
```

安装openssl

```bash
yum install -y openssl openssl-devel 
```

安装依赖zlib

```bash
yum install -y zlib zlib-devel gd gd-devel
```

创建用户

```bash
useradd -M -s /sbin/nologin nginx 
```

编译安装

```bash
./configure --prefix=/usr/local/nginx \
--user=nginx \
--group=nginx \
--with-pcre \
--with-http_ssl_module \
--with-http_v2_module \
--with-http_realip_module \
--with-http_addition_module \
--with-http_sub_module \
--with-http_dav_module \
--with-http_flv_module \
--with-http_mp4_module \
--with-http_gunzip_module \
--with-http_gzip_static_module \
--with-http_random_index_module \
--with-http_secure_link_module \
--with-http_stub_status_module \
--with-http_auth_request_module \
--with-http_image_filter_module \
--with-http_slice_module \
--with-mail \
--with-threads \
--with-file-aio \
--with-stream \
--with-mail_ssl_module \
--with-stream_ssl_module 
    
make && make install
cd /usr/local/nginx/sbin
./nginx              # 启动Nginx
./nginx -t           # 验证配置文件是正确
./nginx -s reload    # 重启Nginx
./nginx -s stop      # 停止Nginx
./nginx -v            # 查看是否安装成功
```

配置systemd启动

```bash
cat /usr/lib/systemd/system/nginx.service
[Unit]
Description=The nginx HTTP and reverse proxy server
After=network.target remote-fs.target nss-lookup.target
# After=network.target

[Service]
Type=forking
PIDFile=/usr/local/nginx/logs/nginx.pid
# Nginx will fail to start if /run/nginx.pid already exists but has the wrong
# SELinux context. This might happen when running `nginx -t` from the cmdline.
# https://bugzilla.redhat.com/show_bug.cgi?id=1268621
ExecStartPre=/usr/bin/rm -f /usr/local/nginx/logs/nginx.pid
ExecStartPre=/usr/local/nginx/sbin/nginx -t
ExecStart=/usr/local/nginx/sbin/nginx
ExecReload=/bin/kill -s HUP $MAINPID
KillSignal=SIGQUIT
TimeoutStopSec=5
KillMode=process
PrivateTmp=true

[Install]
WantedBy=multi-user.target

systemctl daemon-reload
```

