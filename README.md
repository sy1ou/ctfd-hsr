# ctfd-hsr

CTFd setup for HSR

## Requirements

Debian 12 with docker with the compose plugin.

Installing docker is out of the scope of this document.

## Nginx installation with VTS module

Install nginx and the build libraries needed to compile nginx sources.

```bash
apt install nginx
apt install make build-essential libpcre3-dev libssl-dev libxslt1-dev libgd-dev libgeoip-dev libperl-dev
```

Download nginx sources according to the version of nginx installed via the package.

```bash
cd /opt
wget https://nginx.org/download/nginx-1.22.1.tar.gz
tar xvf nginx-1.22.1.tar.gz
```

Download the last version of the Nginx virtual host traffic status module, see [here](https://github.com/vozlt/nginx-module-vts).

```bash
curl -fSL https://github.com/vozlt/nginx-module-vts/archive/refs/tags/v0.2.2.tar.gz | tar xzf - -C /tmp
```

Retrieve the build parameters for the installed version of nginx and add the VTS module to the build configuration by adding `--add-module=/path/to/nginx-module-vts`.

```bash
nginx -V |& grep 'configure arguments' | sed 's/.*prefix/--prefix/' | sed 's/--/\n--/g'

cd /opt/nginx-1.22.1

./configure \
    --prefix=/usr/share/nginx \
    --conf-path=/etc/nginx/nginx.conf \
    --http-log-path=/var/log/nginx/access.log \
    --error-log-path=stderr \
    --lock-path=/var/lock/nginx.lock \
    --pid-path=/run/nginx.pid \
    --modules-path=/usr/lib/nginx/modules \
    --http-client-body-temp-path=/var/lib/nginx/body \
    --http-fastcgi-temp-path=/var/lib/nginx/fastcgi \
    --http-proxy-temp-path=/var/lib/nginx/proxy \
    --http-scgi-temp-path=/var/lib/nginx/scgi \
    --http-uwsgi-temp-path=/var/lib/nginx/uwsgi \
    --with-compat \
    --with-debug \
    --with-pcre-jit \
    --with-http_ssl_module \
    --with-http_stub_status_module \
    --with-http_realip_module \
    --with-http_auth_request_module \
    --with-http_v2_module \
    --with-http_dav_module \
    --with-http_slice_module \
    --with-threads \
    --with-http_addition_module \
    --with-http_flv_module \
    --with-http_gunzip_module \
    --with-http_gzip_static_module \
    --with-http_mp4_module \
    --with-http_random_index_module \
    --with-http_secure_link_module \
    --with-http_sub_module \
    --with-mail_ssl_module \
    --with-stream_ssl_module \
    --with-stream_ssl_preread_module \
    --with-stream_realip_module \
    --with-http_geoip_module=dynamic \
    --with-http_image_filter_module=dynamic \
    --with-http_perl_module=dynamic \
    --with-http_xslt_module=dynamic \
    --with-mail=dynamic \
    --with-stream=dynamic \
    --with-stream_geoip_module=dynamic \
    --add-dynamic-module=/tmp/nginx-module-vts-0.2.2/
```

Compile only the modules and add the compiled version of the VTS module to the nginx modules.

```bash
make modules

mkdir -p /usr/lib/nginx/modules/
cp objs/ngx_http_vhost_traffic_status_*.so /usr/lib/nginx/modules/

# mkdir -p /usr/share/nginx/modules-available/
cat << EOF >> /etc/nginx/modules-available/90-mod-vts.conf
load_module /usr/lib/nginx/modules/ngx_http_vhost_traffic_status_module.so;
EOF
ln -s /etc/nginx/modules-available/90-mod-vts.conf /etc/nginx/modules-enabled/

nginx -t && nginx -s reload
```

### Nginx configuration

Add the `vhost_traffic_status_zone` directive in the `/etc/nginx/nginx.conf` file.

```conf
http {
    #...

    ##                                                                                                            
    # Basic Settings                                                                                              
    ##                                                                                                            
                                                                                                                      
    sendfile on;                                                                                                  
    tcp_nopush on;                                                                                                
    types_hash_max_size 2048;                                                                                     
    server_tokens off;

    #...

    ##
    # VTS Settings
    ##

    vhost_traffic_status_zone;

    #...
}
```

Create a new nginx site in `/etc/nginx/sites-available/ctfd`.

```conf
# Define connections limit zone
limit_conn_zone $binary_remote_addr zone=addr:10m;

upstream backend {
    server localhost:8000;
}

server {
    listen 80 default_server;
    listen [::]:80 default_server;

    server_name _;

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;

    server_name ctf.hacksecureims.eu;

    # Restriction
    limit_conn addr 10;
    client_max_body_size 4G;

    ssl_certificate /etc/ssl/certs/fullchain.pem;
    ssl_certificate_key /etc/ssl/private/privkey.pem;
    ssl_session_timeout 1d;
    ssl_session_cache shared:MozSSL:10m;  # about 40000 sessions
    ssl_session_tickets off;

    # modern configuration
    ssl_protocols TLSv1.3;
    ssl_prefer_server_ciphers off;

    # HSTS (ngx_http_headers_module is required) (63072000 seconds)
    add_header Strict-Transport-Security "max-age=63072000" always;

    # Handle Server Sent Events for Notifications
    location /events {
        proxy_pass http://backend;
        proxy_set_header Connection '';
        proxy_http_version 1.1;
        chunked_transfer_encoding off;
        proxy_buffering off;
        proxy_cache off;
        proxy_redirect off;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Host $server_name;
    }

    # Proxy connections to the application servers
    location / {
        proxy_pass http://backend;
        proxy_redirect off;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Host $server_name;
    }
}

server {
    listen 9113;

    location = /metrics {
        vhost_traffic_status_display;
        vhost_traffic_status_display_format prometheus;
    }
}
```

```bash
sudo mkdir -p /var/cache/nginx
sudo chown www-data /var/cache/nginx
sudo chmod 700 /var/cache/nginx
```

Activates the newly created site and deactivates the `default` site.

```bash
ln -s /etc/nginx/sites-available/ctfd /etc/nginx/sites-enabled/
unlink /etc/nginx/sites-enabled/default
```

## CTFd configuration

Fill the files [.env](.env) and [ctfd-app.env](ctfd-app.env).

Then start the containers using docker compose.

```bash
docker compose up -d
docker compose logs -f
```

## mysqld-exporter configuration

After creating the CTFd database, connect to the database container using the `root` credentials provided in the previous section.

```bash
docker exec -it ctfd-hsr-db-1 mysql -u root -p
```

Add a user to the database to permit the mysqld-exporter to retrieve informations about the health of the database engine.
Add a user to the database to enable mysqld-exporter to retrieve information on the health of the database engine.

```sql
CREATE USER 'exporter'@'%' IDENTIFIED BY 'CHANGEME' WITH MAX_USER_CONNECTIONS 3;
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'%';
```

Adapt the [mysqld-exporter/my.cnf](mysqld-exporter/my.cnf) file, replacing the password with the one defined in the previous command.

## ctfd-exporter configuration

After initializing the CTFd with the web interface, generate an administration API key by [following the instructions here](https://docs.ctfd.io/docs/api/getting-started#generating-an-admin-access-token).

Populate the [ctfd-exporter.env](ctfd-exporter.env) file with the newly generated API key.

## Nftables configuration

Install the nftables package.

```bash
apt install nftables
```

Edit the `/etc/nftables.conf` configuration file.

```conf
#!/usr/sbin/nft -f

# Disabled as it may conflict with docker rules
#flush ruleset

define net_ipv4_admin = 10.22.149.0/24

table inet firewall {

    chain inbound_ipv4 {
        # Accepting ping (icmp-echo-request) for diagnostic purposes.
        icmp type echo-request limit rate 5/second accept
    }

    chain inbound_ipv6 {
        # Accepting ping (icmpv6-echo-request) for diagnostic purposes.
        icmpv6 type echo-request limit rate 5/second accept
    }

    chain inbound {
        # By default, drop all traffic unless it meets a filter
        # criteria specified by the rules that follow below.
        type filter hook input priority filter; policy drop;

        # Allow traffic from established and related packets, drop invalid
        ct state vmap { established : accept, related : accept, invalid : drop }

        # Allow loopback traffic.
        iifname lo accept

        # Jump to chain according to layer 3 protocol using a verdict map
        meta protocol vmap { ip : jump inbound_ipv4, ip6 : jump inbound_ipv6 }

        # Allow HTTP(S) TCP/80 and TCP/443 for IPv4 and IPv6.
        tcp dport { http, https } accept

        # Allow SSH on port TCP/22 and allow metrics collection TCP/9104 TCP/9113 and TCP/9121
        # for IPv4 admin network.
        ip saddr $net_ipv4_admin tcp dport { ssh, 9104, 9113, 9121 } accept

        # Uncomment to enable logging of denied inbound traffic
        log prefix "[nftables] Inbound Denied: " counter drop
    }
}
```

Start the nftables service to apply the new rules.
Check that nothing is broken, then enable the service.

```bash
systemctl start nftables.service
systemctl enable nftables.service
```
