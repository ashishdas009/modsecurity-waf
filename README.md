# modsecurity-waf
## Overview
ModSecurity was originally designed for Apache web server. It could work with Nginx before version 3.0 but suffered from poor performance. ModSecurity 3.0 (aka **libmodsecurity**) was released in 2017. It’s a milestone release, particularly for Nginx users, as it’s the first version to work natively with Nginx. 

## Step 1: Install the Latest Version of Nginx on Debian/Ubuntu
ModSecurity integrates with Nginx as a dynamic module, which allows you to compile source code of individual modules without compiling Nginx itself.

> sudo apt update
>
> sudo apt install nginx-core nginx-common nginx nginx-full
>
> sudo nginx -V

## Step 2: Download Nginx Source Package
Although we don’t need to compile Nginx itself, we need the Nginx source code package in order to compile the ModSecurity dynamic module. Run the following command to make a nginx directory under /usr/local/src/ to store the Nginx source code package. Replace username with your real username.

> sudo chown username:username /usr/local/src/ -R 
>
> mkdir -p /usr/local/src/nginx
>
> cd /usr/local/src/nginx/
>
> sudo apt install dpkg-dev
>
> apt source nginx

## Step 3: Install libmodsecurity3
**libmodsecurity** is the ModSecurity library that actually does the HTTP filtering for your web applications. 

> git clone --depth 1 -b v3/master --single-branch https://github.com/SpiderLabs/ModSecurity /usr/local/src/ModSecurity/
>
> cd /usr/local/src/ModSecurity/

Install build dependencies.

> sudo apt install gcc make build-essential autoconf automake libtool gettext pkg-config libpcre3 libpcre3-dev libxml2 libxml2-dev libcurl4 libgeoip-dev libyajl-dev doxygen

Install required submodules.

> git submodule init
>
> git submodule update

Configure the build environment.

> ./build.sh 
>
> ./configure

Replace $ with the number of cores on the CPU
> make -j$
> sudo make install

## Step 4: Download and Compile ModSecurity v3 Nginx Connector Source Code
The **ModSecurity Nginx Connector** links **libmodsecurity** to the Nginx web server. Clone the ModSecurity v3 Nginx Connector Git repository.

> git clone --depth 1 https://github.com/SpiderLabs/ModSecurity-nginx.git /usr/local/src/ModSecurity-nginx/

Make sure you are in the Nginx source directory.

> cd /usr/local/src/nginx/nginx-1.21.0/

Install build dependencies for Nginx.

> sudo apt build-dep nginx
>
> sudo apt install uuid-dev

Next, configure the environment with the following command. We will not compile Nginx itself, but compile the **ModSecurity Nginx Connector** module only. The `--with-compat` flag will make the module binary-compatible with your existing Nginx binary.

> ./configure --with-compat --add-dynamic-module=/usr/local/src/ModSecurity-nginx

Build the **ModSecurity Nginx Connector** module.

> make modules
>
> sudo cp objs/ngx_http_modsecurity_module.so /usr/share/nginx/modules/

## Step 5: Load the ModSecurity v3 Nginx Connector Module
Edit the main Nginx configuration file.

> sudo nano /etc/nginx/nginx.conf

Add the following line at the beginning of the file.

> load_module modules/ngx_http_modsecurity_module.so;

Also, add the following two lines in the `http {...}` section, so ModSecurity will be enabled for all Nginx virtual hosts.

> modsecurity on;
>
> modsecurity_rules_file /etc/nginx/modsec/main.conf;

Save and close the file. Next, create the `/etc/nginx/modsec/` directory to store ModSecurity configuration

> sudo mkdir /etc/nginx/modsec/

Then copy the ModSecurity configuration file.

> sudo cp /usr/local/src/ModSecurity/modsecurity.conf-recommended /etc/nginx/modsec/modsecurity.conf

Edit the file.

> sudo nano /etc/nginx/modsec/modsecurity.conf

Find the following line.

> SecRuleEngine DetectionOnly

Change it to

> SecRuleEngine On

Then find the following line (line 224), which tells ModSecurity what information should be included in the audit log.

> SecAuditLogParts ABIJDEFHZ

Change it to

> SecAuditLogParts ABCEFHJKZ

Save and close the file. Next, create the `/etc/nginx/modsec/main.conf` file.

> sudo nano /etc/nginx/modsec/main.conf

Add the following line to include the `/etc/nginx/modsec/modsecurity.conf` file.

> Include /etc/nginx/modsec/modsecurity.conf

We also need to copy the Unicode mapping file.

> sudo cp /usr/local/src/ModSecurity/unicode.mapping /etc/nginx/modsec/

Then test Nginx configuration.

> sudo nginx -t

If the test is successful, restart Nginx.

> sudo systemctl restart nginx

## Step 6: Enable OWASP Core Rule Set

Download the latest OWASP CRS from GitHub.

> wget https://github.com/coreruleset/coreruleset/archive/v3.3.0.tar.gz

Extract the file.

> tar xvf v3.3.0.tar.gz

Move the directory to `/etc/nginx/modsec/`.

> sudo mv coreruleset-3.3.0/ /etc/nginx/modsec/

Rename the crs-setup.conf.example file.

> sudo mv /etc/nginx/modsec/coreruleset-3.3.0/crs-setup.conf.example /etc/nginx/modsec/coreruleset-3.3.0/crs-setup.conf

Then edit the main configuration file.

> sudo nano /etc/nginx/modsec/main.conf

Add the following two lines, which will make Nginx include the CRS config file and individual rules.

> Include /etc/nginx/modsec/coreruleset-3.3.0/crs-setup.conf
>
> Include /etc/nginx/modsec/coreruleset-3.3.0/rules/*.conf

Save and close the file. Then test Nginx configuration.

> sudo nginx -t

If the test is successful, restart Nginx.

> sudo systemctl restart nginx
