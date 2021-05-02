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

```
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

```conf
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

```conf
server {
  listen 80 default_server;
  root /usr/share/nginx/html;
  server_name _;
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

```conf
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

```conf
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

```conf
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

#### Configuring the host for SSL/TLS/HTTPS

### NGINX Rewrites

#### Cleaning up URLS

#### Redirecting all traffic to HTTPS

### NGINX Modules

#### Overview of NGINX modules

#### Adding functionality to NGINX with dynamic modules

### Reverse Proxy

#### Proxy vs Reverse Proxy

Proxy sits between our clients and the internet. It is an intermediate layer often used within organizations to monitor web traffic.

Reverse proxy sits between internet traffic and our servers. It is an intermediate layer often used to load balance traffic & serve content from a cache.

#### Reverse Proxy with proxy_pass

The `proxy_pass` is a single directive that can exists within `location` context with no default value. A `proxy_pass` can be either an internal URL with a protocol specified, i.e. `http` or `https`, or a UNIX-domain socket path.

We will set up a separate virtual host. Create a file named `conf.d/photos.example.com.conf` with `sudo vim` command, and paste the following below:

```conf
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