# Configure the Load Balancer

We are going to use HAProxy for our load balancer.

#### Install HAProxy

Installation is simple:

```bash
{

sudo apt update
sudo apt install -y haproxy

}
```

#### Configure HAProxy

```bash
{

cat <<EOF | sudo tee /etc/haproxy/haproxy.cfg
global
    log /dev/log	local0
    log /dev/log	local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin
    stats timeout 30s
    user haproxy
    group haproxy
    daemon
    
    # Default SSL material locations
    ca-base /etc/ssl/certs
    crt-base /etc/ssl/private
    
    # Default ciphers to use on SSL-enabled listening sockets.
    # For more information, see ciphers(1SSL). This list is from:
    #  https://hynek.me/articles/hardening-your-web-servers-ssl-ciphers/
    ssl-default-bind-ciphers ECDH+AESGCM:DH+AESGCM:ECDH+AES256:DH+AES256:ECDH+AES128:DH+AES:ECDH+3DES:DH+3DES:RSA+AESGCM:RSA+AES:RSA+3DES:!aNULL:!MD5:!DSS
    ssl-default-bind-options no-sslv3

defaults
    log global
    mode tcp
    option tcplog
    option dontlognull
        timeout connect 5000
        timeout client  50000
        timeout server  50000
    errorfile 400 /etc/haproxy/errors/400.http
    errorfile 403 /etc/haproxy/errors/403.http
    errorfile 408 /etc/haproxy/errors/408.http
    errorfile 500 /etc/haproxy/errors/500.http
    errorfile 502 /etc/haproxy/errors/502.http
    errorfile 503 /etc/haproxy/errors/503.http
    errorfile 504 /etc/haproxy/errors/504.http

frontend k8s
    bind 10.0.0.30:6443
    default_backend k8s_backend

backend k8s_backend
    balance roundrobin
    mode tcp
    server controller-0 10.0.0.10:6443 check inter 1000
    server controller-1 10.0.0.11:6443 check inter 1000
    server controller-2 10.0.0.12:6443 check inter 1000
EOF

}
```

Now we restart: `sudo service haproxy status`

And test that it is running fine: `sudo service haproxy status`

We can test healthz through the load balancer as well: `curl -k -i https://10.0.0.30:6443/healthz`

```bash
HTTP/2 200 
content-type: text/plain; charset=utf-8
x-content-type-options: nosniff
content-length: 2
date: Wed, 19 Feb 2020 02:26:39 GMT
```

Next: [Configure the load balancer](docs/06-config-lb.md)

