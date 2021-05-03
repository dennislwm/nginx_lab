# nginx

Self-hosted NGINX web server on a CentOS 7 cloud server.

<!-- TOC -->

- [nginx](#nginx)
  - [Project](#project)
    - [Setting up NGINX server](#setting-up-nginx-server)
    - [Basic Web Server Configuration](#basic-web-server-configuration)
      - [Setting up our first virtual host](#setting-up-our-first-virtual-host)
      - [Setting up our second virtual host](#setting-up-our-second-virtual-host)
      - [Enabling permissions for SELinux](#enabling-permissions-for-selinux)
      - [Basic Error Pages](#basic-error-pages)
      - [Access Control with HTTP Basic Auth](#access-control-with-http-basic-auth)
    - [Basic NGINX security](#basic-nginx-security)
      - [Generating self-signed certificates](#generating-self-signed-certificates)
      - [Configuring the host for SSL/TLS/HTTPS](#configuring-the-host-for-ssltlshttps)
    - [NGINX Rewrites](#nginx-rewrites)
      - [Cleaning up URLS](#cleaning-up-urls)
      - [Redirecting all traffic to HTTPS](#redirecting-all-traffic-to-https)
    - [NGINX Modules](#nginx-modules)
      - [Overview of NGINX modules](#overview-of-nginx-modules)
      - [Adding functionality to NGINX with dynamic modules](#adding-functionality-to-nginx-with-dynamic-modules)
    - [Reverse Proxy](#reverse-proxy)
      - [Proxy vs Reverse Proxy](#proxy-vs-reverse-proxy)
      - [Reverse Proxy with proxy_pass](#reverse-proxy-with-proxy_pass)
    - [Load Balancing](#load-balancing)
    - [Logging](#logging)
    - [Security](#security)
    - [Performance](#performance)
  - [References](#references)

<!-- /TOC -->

## Project

### Setting up NGINX server

Before we begin, we're going to need a NGINX server to work with. Create a CentOS 7 cloud server and run the following on it:

```bash
export LC_ALL="en_US.UTF-8"
export LC_CTYPE="en_US.UTF-8"
sudo yum update -y
```

Add a NGINX yum repository and create a file named `/etc/yum.repos.d/nginx.repo` with `sudo vim` command, and paste the following below:

```nginx
[nginx]
name=nginx repo
baseurl=https://nginx.org/packages/centos/7/$basearch/
gpgcheck=0
enabled=1
```

Run the following commands to install and run NGINX server:

```bash
sudo yum install -y nginx
sudo systemctl start nginx
sudo systemctl enable nginx
```

Test the server.

```bash
curl localhost
```

### Basic Web Server Configuration

Before we modify the default virtual host, you may want to run `sudo su -` as most of these commands require elevated privileges.

```bash
cd /etc/nginx
vim nginx.conf
```

Before we begin, we need to understand what a **directive** is. There are TWO (2) types of directives in NGINX:
1. **Single** - this is any keyword that begins on a single line. 
2. **Block** - this is any keyword that begins on a line followed by a block enclosed by `{ }`. 

```nginx
user  nginx;
worker_processes  1;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;

events {
    worker_connections  1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    #gzip  on;

    include /etc/nginx/conf.d/*.conf;
}
```

Below is an explanation of some of the above directives within the `http` context.
* include - takes a separate file and expands it into the config file
* default_type - if not found in the mime.types, then use default type
* log_format - alias followed by a user defined format for any http request
* access_log - the log file followed by its alias
* sendfile - enable system call sendfile to deliver content
* keepalive_timeout - number of seconds before recycling a connection back to `worker_connection` pool

An important explanation for the directive `include /etc/nginx/conf.d/*.conf`.
1. Takes all separate `conf.d/*.conf` files and expands it into the config file
2. Allows us to break up all the virtual hosts server config files.
3. We don't typically need to modify the root `nginx.conf` file as we can specify our virtual host config files within the `conf.d` directory.
4. As this `include` directive is already within a `http` context, we don't need to specify another `http` context within our virtual host config files.

*Note: We may use the `conf.d/default.conf` as a base template for our virtual config files. However, we will build up our virtual host config files line by line.*

#### Setting up our first virtual host

First, remove the `conf.d/default.conf` file with `sudo rm` command, and create a file named `conf.d/default.conf` with `sudo vim` command, and paste the following below:

```nginx
server {
  listen 80 default_server;
  root /usr/share/nginx/html;
  server_name _;
  # nameless server _
}
```

The `server` is a block directive within the `http` context that has no default value. Below is an explanation of some of the above directives within the `server` context.
* listen - a single directive that specifies which port to listen to; optional `default_server` parameter tells NGINX to use this `server` block as default for incoming requests.
* root - a single directive that specifies which root folder that contains the public web files, i.e. `/usr/share/nginx/html` is the default nginx public web folder which contains an `index.html`
* server_name - optional single directive that specifies the server name; parameter `_` has a special meaning where this `server` block has no name

*Note: You may have multiple server blocks for different servers, such as http, ftp, etc.*

Validate our virtual host configuration by typing the following command:

```bash
nginx -t
```

Reload our virtual host configuration and test the server.

```bash
systemctl reload nginx
curl localhost
```

#### Setting up our second virtual host

To help us understand the basic SELINUX security, we will set up a second virtual host. Create a file named `conf.d/example.com.conf` with `sudo vim` command, and paste the following below:

```nginx
server {
  listen 80;
  root /var/www/example.com/html;
  server_name example.com www.example.com;
}
```

Make a new directory named `/var/www/example.com/html` with `sudo mkdir -p`, create a file named `/var/www/example.com/html/index.html` with `sudo vim` command and paste the following below:

```html
<h1>Hello Example.com</h1>
```

Validate and reload our virtual host configuration files with `nginx -t` command and `systemctl reload nginx` respectively.

Test the server.

```bash
curl --header "Host: example.com" localhost
```

However, we should get a `403 Forbidden` error. The reason for this error is because of an SELinux security feature that prevents NGINX from serving a request from a folder that is not being allowed.

To find the cause of the error, take a look at the log file `/var/log/nginx/error.log` with `tail` command.

#### Enabling permissions for SELinux

First, we'll check if there are any permissions used by SELinux for our first virtual host that exists by default.

```bash
semanage fcontext -l | grep /usr/share/nginx/html
```

We should get the following output.

```
/usr/share/nginx/html(/.*)?                        all files          system_u:object_r:httpd_sys_content_t:s0
```

Next, we'll add a new permission to SELinux for our second virtual host.

```bash
semanage fcontext -a -t httpd_sys_content_t '/var/www(/.*)?'
```

Ignore the error `libsemanage.dbase_llist_query: could not query record value (No such file or directory).`

Check that the new permission is added.

```bash
semanage fcontext -l | grep /var/www
```

To actually get the files and directory that to have the proper labels for them from SELinux.

```bash
restorecon -R -v /var/www
```

Test the server again and we should see our `Hello Example.com` page.

```bash
curl --header "Host: example.com" localhost
```

#### Basic Error Pages

By default, NGINX automatically creates and serves a `404` html page if a web page isn't found. However, we should overwrite this with our own `404` page.

Open the file named `conf.d/default.conf` with `sudo vim` command, and paste the following below:

```nginx
server {
  # ...

  error_page 404              /404.html;
  error_page 500 502 503 504  /50x.html;
}
```

The `error_page` is a block directive that can exists within `http`, `server` or `location` context with no default value. For our example, we want the error page to be different for each server, hence we will put it within `server` context.

Create a file named `/usr/share/nginx/html/404.html` with `sudo vim` command, and paste the following below:

```html
<h1>404 Page Not Found</h1>
```

Validate and reload our virtual host configuration files. Then test the server.

```bash
curl localhost/fake.html
```

#### Access Control with HTTP Basic Auth

The `location` is a block directive that can exists within `server` or `location` context with no default value. A `location` can be either a specific URI or a regular expression.

For example, if someone tries to access `admin.html`, it will deny access if the user is not logged in.

Open the file named `conf.d/default.conf` with `sudo vim` command, and paste the following below:

```nginx
server {
  # ...

  location = /admin.html {
    auth_basic "Login Required";
    auth_basic_user_file /etc/nginx/.htpasswd
  }

  error_page 404              /404.html;
  error_page 500 502 503 504  /50x.html;
}
```

The `auth_basic` is a single directive that can exists within `http`, `server` or `location` with default value set to `off`. To enable it, we set the value to a `str` message that the users see if they're are not logged in.

The `auth_basic_user_file` is a single directive that is similar to `auth_basic` except that it has no default value, and we are required to specify a `str` file that keeps user names and passwords. Passwords are encrypted either with `htpasswd` utility from Apache HTTP Server distro or `openssl passwd` command.

Install Apache HTTP Server tools for Centos 7. For Ubuntu / Debian systems, use the package name `apache2-utils` instead.

```bash
sudo yum install -y httpd-tools
```

Run the `htpasswd` utility to create a new file with a new user. It will prompt you for a password.

```
sudo htpasswd -c /etc/nginx/.htpasswd [USER]
```

Validate and reload our virtual host configuration files. Then test the server.

```bash
curl localhost/admin.html
```

However, we should get a `401 Authorization Required` error. The reason for this error is because we didn't pass `curl` any authorization. 

Create a file named `/usr/share/nginx/html/admin.html` with `sudo vim` command, and paste the following below:

```html
<h1>Admin Page</h1>
```

Validate and reload our virtual host configuration files. Then test the server.

```bash
curl -u [USER]:[PASSWORD] localhost/admin.html
```

### Basic NGINX security

#### Generating self-signed certificates

Make a new directory named `/etc/nginx/ssl` with `sudo mkdir`, and we use `openssl` to create both a `private.key` and `public.pem` (or whatever name you choose):

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/nginx/ssl/private.key -out /etc/nginx/ssl/public.pem
```

Accept the default values when prompted, however if you are using Nginx on a production server then you should use a third-party signed certificate, such as Let's Encrypt.

#### Configuring the host for SSL/TLS/HTTPS

Change the default server configuration to handle HTTPS connections. For this we use the `ngx_http_ssl_module`.

Open the file named `conf.d/default.conf` with `sudo vim` command. Add a second `listen` directive and both the `private.key` and `public.pem`.

```nginx
server {
  listen 80 default_server;
  listen 443 ssl;
  server_name _;
  root /user/share/nginx/html;

  # ssl
  ssl_certificate /etc/nginx/ssl/public.pem;
  ssl_certificate_key /etc/nginx/ssl/private.key;

  # location..
  location = /admin.html {
    auth_basic "Login Required";
    auth_basic_user_file /etc/nginx/.htpasswd
  }

  # error..
  error_page 404              /404.html;
  error_page 500 502 503 504  /50x.html;
}
```
Validate and reload our virtual host configuration files with `nginx -t` command and `systemctl reload nginx` respectively.

Test the server by navigating to `https://[IP_ADDRESS]` of your NGINX cloud server. Ignore the message "Your connection is not secure" as we're using a self-signed certificate.

Alternatively, you can test your server using `curl -k https://localhost`.

### NGINX Rewrites

Rewrite rules allow us to modify the URL as it's coming in. We can either modify and then redirect it so that the URL matches what we expect or we can modify and then pass it to our `location` block directive so that it can be handled.

#### Cleaning up URLS

In this lab, we will remove the file types, e.g. `.html`, from a URL and still have the content served. This is kind of a vanity thing where we don't want the file type showing in the URL address.

For this we use the `ngx_http_rewrite_module`, specifically we will use the `rewrite` single directive. You can chain `rewrite` rules together.

Open the file named `conf.d/default.conf` with `sudo vim` command. Add a `rewrite` directive.

```nginx
server {
  # listen ...

  # ssl...

  # rewrite..
  rewrite ^(/.*)\.html(\?.*)?$ $1$2 redirect;
  # [REG_EXPR]
  #   ^ starts with
  #   (/.*) slash followed by any number of chars (group $1)
  #   \.html file type .html
  #   (\?.*)? query followed by any number of chars (optional group $2)
  #   $ ends with
  # [SUBSTITUTE]
  #   $1$2 for example /admin.html?key=value => /admin?key=value
  # [FLAG]
  #   redirect sends another inbound to our server with new URL (301 non-permanent)
  
  rewrite ^/(.*)/$ /$1 redirect;
  # [REG_EXPR]
  #   ^/ starts with /
  #   (.*) any number of chars (group $1)
  #   /$ ends with /
  # [SUBSTITUTE]
  #   /$1 for example /admin/ => /admin
  # [FLAG]
  #   redirect sends another inbound to our server with new URL (301 non-permanent)

  # location..

  # error..
}
```
Validate and reload our virtual host configuration files with `nginx -t` command and `systemctl reload nginx` respectively.

Test the server by navigating to `https://[IP_ADDRESS]` of your NGINX cloud server. You should see an error "The page isn't redirecting properly" because we didn't specify how to handle the new URLs that we rewrote.

For this we use the `ngx_http_core_module`, specifically we use the `try_files` that will allow us to take a URI and serve the files on disk. You can chain multiple files on disk until it finds a valid file.

Open the file named `conf.d/default.conf` with `sudo vim` command. Add a second `location` block directive with a `try_files` single directive.

```nginx
server {
  # listen ...

  # ssl...

  # rewrite..

  # location..
  location / {
    try_files $uri/index.html $uri.html $uri/ $uri =404;
  }
  # $uri contains your new URL that you rewrote. For example, assume $uri = /admin
  #   $uri/index.html => /admin/index.html (file 1)
  #   $uri.html => /admin.html (file 2)
  #   $uri/ => /admin/ (file 3)
  #   =404 returns error_page 404

  location = /admin {
  # remove .html above and add try_files single directive
    auth_basic "Login Required";
    auth_basic_user_file /etc/nginx/.htpasswd
    try_files $uri/index.html $uri.html $uri/ $uri =404;
  }

  # error..
}
```

Also, modify the previous `location` block by removing the `.html` from `= /admin` and add a `try_files` single directive. The `location` blocks `/` and `= /admin` will take inbound URLs `/` and `/admin` respectively and serve the same files on disk.

Validate and reload our virtual host configuration files with `nginx -t` command and `systemctl reload nginx` respectively.

Test the server by navigating to `https://[IP_ADDRESS]/admin.html` of your NGINX cloud server. You should see the new URL address as `https://[IP_ADDRESS]/admin`.

#### Redirecting all traffic to HTTPS

We can use `ngx_http_rewrite_module` to redirect all inbound requests to HTTPS, specifically we will use the `return` single directive.

Open the file named `conf.d/default.conf` with `sudo vim` command. Add a second `server` block directive to separate the ports `80` and `443`.

```nginx
server {
  # listen..
  listen 80 default_server;
  server_name _;
  return 301 https://$host$request_uri;
  # return 
  #   [CODE] sends another inbound to our server  (301 non-permanent)
  #   [URL] new HTTPS url
  #     $host in this order of precedence: host name from the request line, or host name 
  #       from the “Host” request header field, or the server name matching a request
  #     $request_uri full original request URI (with arguments)
}

server {
  # listen..
  listen 443 ssl;
  server_name _;
  root /user/share/nginx/html;

  # ssl..

  # rewrite..

  # location..

  # error..
}
```

Validate and reload our virtual host configuration files with `nginx -t` command and `systemctl reload nginx` respectively.

Test the server by navigating to `http://[IP_ADDRESS]/admin.html` of your NGINX cloud server. You should see the new URL address as `https://[IP_ADDRESS]/admin`.

### NGINX Modules

#### Overview of NGINX modules

So far in this lab we have been using Nginx built-in modules. However, we can add an external module to the folder `/etc/nginx/modules`.

From Nginx version 1.9.11, dynamic modules allow you to add functionality without recompiling Nginx. For example, `ngx_http_image_filter_module` is not built by default, but we could enable it with a configuration parameter.

Show the Nginx configuration parameters and show only modules in a line format.

```bash
nginx -V
nginx -V 2>%1 | tr -- - '\n' | grep _module
```

If we want a smaller Nginx binary executable, we could remove `mail_*` and `stream_*` modules by removing their respective configuration parameters.

#### Adding functionality to NGINX with dynamic modules

For this we use Nginx core functionality, specifically we will use the `load_module` single directive to load dynamic modules with a `.so` extension. We will add a third-party firewall module **ModSecurity** by compiling the dynamic module and then including it by using the `load_module` single directive.

Download developer tools and build packages.

```bash
yum groupinstall 'Development tools'
yum install -y \
geoip-devel \
libcurl-devel \
libxml2-devel \
libxslt-devel \
libgb-devel \
lmdb-devel \
openssl-devel \
pcre-devel \
perl-ExtUtils-Embed \
yajl-devel \
zlib-devel
```

Download and build the source code for ModSecurity.

```bash
cd /opt
git clone --depth 1 -b v3/master https://github.com/SpiderLabs/ModSecurity.git
cd ModSecurity
git submodule init
git submodule update
./build.sh
```

Check if compile successful with `echo $?` which should return a `0`.

Configure the ModSecurity with default values, then compile and install it.

```bash
./configure
make
make install
```

Download both Nginx wrapper for ModSecurity and Nginx source code.

```bash
cd ../
git clone --depth 1 https://github.com/SpiderLabs/ModSecurity-nginx.git
wget http://nginx.org/download/nginx-[VERSION].tar.gz
tar zxvf nginx-[VERSION].tar.gz
```

Configure Nginx with additional Nginx wrapper for ModSecurity, then compile the module.

```bash
cd nginx-[VERSION]
./configure --with-compat --add-dynamic-module=../ModSecurity-nginx
make modules
```

Check if compile successful for the file `./objs/ngx_http_modsecurity_module.so`. Copy this file to `/etc/nginx/modules` with `sudo cp`.

Open the file named `/etc/nginx/nginx.conf` with `sudo vim` command. Add a `load_module` single directive before both the `events` and `http` block directives.

```nginx
# ...

# Load ModSecurity dynamic module
load_module /etc/nginx/modules/ngx_http_modsecurity_module.so;

# events..

# http..
```

Validate with `nginx -t` command. Then, copy a default ModSecurity configuration file to a new folder.

```bash
mkdir /etc/nginx/modsecurity
cp /opt/ModSecurity/modsecurity.conf-recommended /etc/nginx/modsecurity/modsecurity.conf
cp unicode.mapping /etc/nginx/modsecurity/unicode.mapping
```

Open the file named `/etc/nginx/modsecurity/modsecurity.conf` with `sudo vim` command. Find the line below and change it as follows:

```nginx
SecAuditLog /var/log/nginx/modsec_audit.log
```

Open the file named `conf.d/default.conf` with `sudo vim` command. Add both a `modsecurity` and `modsecurity_rules_file` single directives under `server` block directive that listens on port `443`.

```nginx
server {
  # listen..
  listen 443 ssl;
  server_name _;
  root /user/share/nginx/html;

  # dynamic module
  modsecurity on;
  modsecurity_rules_file /etc/nginx/modsecurity/modsecurity.conf;
  
  # ssl...

  # rewrite..

  # location..

  # error..
}
```

Validate and reload our virtual host configuration files with `nginx -t` command and `systemctl reload nginx` respectively.

The final `conf.d/default.conf` file.

```nginx
server {
  # listen..
  listen 80 default_server;
  server_name _;
  return 301 https://$host$request_uri;
  # return 
  #   [CODE] sends another inbound to our server  (301 non-permanent)
  #   [URL] new HTTPS url
  #     $host in this order of precedence: host name from the request line, or host name 
  #       from the “Host” request header field, or the server name matching a request
  #     $request_uri full original request URI (with arguments)
}

server {
  # listen
  listen 443 ssl;
  server_name _;
  root /user/share/nginx/html;

  # dynamic module
  modsecurity on;
  modsecurity_rules_file /etc/nginx/modsecurity/modsecurity.conf;
  
  # ssl
  ssl_certificate /etc/nginx/ssl/public.pem;
  ssl_certificate_key /etc/nginx/ssl/private.key;

  # rewrite
  rewrite ^(/.*)\.html(\?.*)?$ $1$2 redirect;
  # [REG_EXPR]
  #   ^ starts with
  #   (/.*) slash followed by any number of chars (group $1)
  #   \.html file type .html
  #   (\?.*)? query followed by any number of chars (optional group $2)
  #   $ ends with
  # [SUBSTITUTE]
  #   $1$2 for example /admin.html?key=value => /admin?key=value
  # [FLAG]
  #   redirect sends another inbound to our server with new URL (301 non-permanent)
  
  rewrite ^/(.*)/$ /$1 redirect;
  # [REG_EXPR]
  #   ^/ starts with /
  #   (.*) any number of chars (group $1)
  #   /$ ends with /
  # [SUBSTITUTE]
  #   /$1 for example /admin/ => /admin
  # [FLAG]
  #   redirect sends another inbound to our server with new URL (301 non-permanent)

  # location
  location / {
    try_files $uri/index.html $uri.html $uri/ $uri =404;
  }
  # $uri contains your new URL that you rewrote. For example, assume $uri = /admin
  #   $uri/index.html => /admin/index.html (file 1)
  #   $uri.html => /admin.html (file 2)
  #   $uri/ => /admin/ (file 3)
  #   =404 returns error_page 404

  location = /admin {
  # remove .html above and add try_files single directive
    auth_basic "Login Required";
    auth_basic_user_file /etc/nginx/.htpasswd
    try_files $uri/index.html $uri.html $uri/ $uri =404;
  }

  # error
  error_page 404              /404.html;
  error_page 500 502 503 504  /50x.html;
}
```

### Reverse Proxy

#### Proxy vs Reverse Proxy

Proxy sits between our clients and the internet. It is an intermediate layer often used within organizations to monitor web traffic.

Reverse proxy sits between internet traffic and our servers. It is an intermediate layer often used to load balance traffic & serve content from a cache.

#### Reverse Proxy with proxy_pass

The `proxy_pass` is a single directive that can exists within `location` context with no default value. A `proxy_pass` can be either an internal URL with a protocol specified, i.e. `http` or `https`, or a UNIX-domain socket path.

We will set up a separate virtual host. Create a file named `conf.d/photos.example.com.conf` with `sudo vim` command, and paste the following below:

```nginx
server {
  listen 80;
  server_name photos.example.com;

  location / {
    proxy_pass http://127.0.0.1:3000;
    proxy_http_version 1.1;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Real-IP $remote_addr;
  }
}
```

The `proxy_http_version` is a single directive within the same context as above with a default value of 1.0. However, version 1.1 is recommended for use with `keepalive` connections.

The `proxy_set_header` is a single directive within the same context as above. It allows appending fields to the request header before passing it to the virtual host. A `proxy_set_header` contains both a field and a value.

NGINX provides a `$proxy_add_x_forwarded_for` variable to automatically append its internal IP to any incoming X-Forwarded-For headers that starts with the user `$remote_addr`  thus creating a load balancer, while X-Real-IP contains the user `$remote_addr` only.

Validate and reload our virtual host configuration files. Then test the server.

```bash
curl --header "Host: photos.example.com" localhost
```

However, we should get a `502 Bad Gateway` error. The reason for this error is because we didn't enable the network upstream permission within SELinux.

We can view the NGINX error log, and set upstream permission for network.

```bash
tail /var/log/nginx/error.log
setsebool -P httpd_can_network_connect 1
```

To navigate to `photos.example.com` from our workstation, we need to add an entry in `/etc/hosts` file in our workstation that resolves to the IP_ADDRESS of our NGINX cloud server.


### Load Balancing

### Logging

### Security

### Performance

---
## References

* [Install NGINX](https://www.nginx.com/resources/wiki/start/topics/tutorials/install)

* [nginx documentation](http://nginx.org/en/docs)