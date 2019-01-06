# haproxy-sandbox
Learn how HAProxy works.


# Installing a load balancer on CentOS
...cuz we use CentOS on the work VMs :(


## Create some docker images to use as upstream hosts:
```
mkdir s2 s1
echo "Hello World." > s1/index.html
echo "Good evening, Pasadena." > s2/index.html

docker run -p 8000:80 -v "$(PWD)/1":/usr/share/nginx/html:ro -d nginx
docker run -p 8001:80 -v "$(PWD)/s1":/usr/share/nginx/html:ro -d nginx
```

## NGINX:
Looks cool but seems to require a $2500/year license to do what we want...

### Starting/stopping in CentOS
```
service nginx stop | start | reload
```


### Not connecting from CentOS? 
```
setsebool -P httpd_can_network_connect on
```

### Setup:
ensure no servers are already listening on port 80 (check `/etc/nginx/conf.d/default.conf`).

Copy the following to `/etc/nginx/conf.d/rev_proxy.conf`:

```
upstream myapp1 {
	# unsupported on free version :(
	# sticky cookie

	# you can use this but makes it tough to test since it
	# always routes a given IP to a specific server
	# ip_hash;
	server 192.168.0.30:8000;
	server 192.168.0.30:8001;
}

server {
	listen 80;
	#server_name revprox;

	location / {
		proxy_pass http://myapp1;
	}
}
```


## HAProxy for Centos

### Installation
Do this (but use newer version) https://upcloud.com/community/tutorials/haproxy-load-balancer-centos/

#### Others I didn't use:
https://lists.centos.org/pipermail/centos-announce/2018-June/022915.html
https://pario.no/2018/07/17/install-haproxy-1-8-on-centos-7/

### Make sure CentOS can connect to the upstream servers:
```
setsebool -P haproxy_connect_any 1
```


### Config file 
Put this in `/etc/haproxy/haproxy.cfg`
```
global
   log /dev/log local0
   log /dev/log local1 notice
   chroot /var/lib/haproxy
   stats timeout 30s
   user haproxy
   group haproxy
   daemon

defaults
   log global
   mode http
   option httplog
   option dontlognull
   timeout connect 5000
   timeout client 50000
   timeout server 50000

frontend http_front
   bind *:80
   stats uri /stats
   default_backend http_back
   stats realm Haproxy\ Statistics
   # change username and password to something harder...
	 stats auth username:password
			
backend http_back
   balance roundrobin
	 cookie SERVERID insert indirect nocache
	 server host1 192.168.0.30:8000 check cookie s1
	    #server host1 192.168.0.30:8000 check weight 2 cookie s1
	    server host2 192.168.0.30:8001 check cookie s2
```
