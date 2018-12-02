---
title:  "HAProxy -- Load Balancing and Subdomain/Port Redirection"
date:   2017-01-31 23:51:07 +0300
categories: proxy haproxy
---

# HAProxy -- Load Balancing and Subdomain/Port Redirection

**Jan 31, 2017**\
<sup>Last modified: **Feb 4, 2017**</sup>

[HAProxy](//haproxy.org "HAProxy") is a high performance TCP/HTTP load balancer that can also be used as reverse-proxy. I used HAProxy for load balancing multiple web servers and managing different web applications serving under the same domain with different subdomains and ports. For an introduction to load balancing concept, you can check Digital Ocean's [tutorial](https://www.digitalocean.com/community/tutorials/an-introduction-to-haproxy-and-load-balancing-concepts).

*Tested on ubuntu:16.04*

**References**:

  - https://www.digitalocean.com/community/tutorials/an-introduction-to-haproxy-and-load-balancing-concepts
  - https://cbonte.github.io/haproxy-dconv/configuration-1.4.html#4.2-balance

# Installing

I will explain how to install and configure HAProxy on an Ubuntu Server. To install HAProxy, simply use `apt-get` command:


```bash
apt-get install haproxy
```

To be able to start HAProxy with an init script, simply edit the file `/etc/default/haproxy` with your favorite editor and add the following line


```bash
ENABLED=1
```

After saving the file, please check that you get the following output once you start the init script.


```bash
service haproxy
Usage: /etc/init.d/haproxy {start|stop|reload|restart|status}
```

# Configuring

Now we are good to go for configuring HAProxy. Open a config file under the path `/etc/haproxy/haproxy.cfg` and add the following defaults. These are the basic settings and can be changed according to different purposes.


```bash
global
    log 127.0.0.1 local0 notice
    maxconn 2000
    user haproxy
    group haproxy

defaults
    log     global
    mode    http
    option  httplog
    option  dontlognull
    retries 3
    option redispatch
    timeout connect  5000
    timeout client  10000
    timeout server  10000
```

Now, let's assume that we have two web applications, one called `alpha` and another called `beta`, both serving under the same domain, which is `mydomain.com`. And let `beta` have two web servers working under two different ports. For the sake of simplicity, the applications are assumed to be working on the same host which HAProxy is working.


```bash
alpha
127.0.0.1:8000

beta_1
127.0.0.1:8080

beta_2
127.0.0.1:8081
```

We will continue adding new blocks to the config file in `/etc/haproxy/haproxy.cfg`. The first block will be a frontend block which defines all the domain and port related configurations.


```bash
frontend http-in
    bind *:80
    acl sub1 hdr_sub(host) -i alpha.mydomain.com
    acl sub2 hdr_sub(host) -i beta.mydomain.com

    use_backend alpha_backend if sub1
    use_backend beta_backend if sub2
```

Here we binded all the connections coming through port `80`. And we defined `alpha` subdomain as `sub1` and `beta` subdomain as `sub2` which will be directed to corresponding backends `alpha_backend` and `beta_backend`. All these names are free to be configured as desired. Let's continue adding backend blocks to the config file in `/etc/haproxy/haproxy.cfg`.


```bash
backend alpha_backend
    mode http
    option forwardfor
    server alpha_server 127.0.0.1:8000

backend beta_backend
    mode http
    balance roundrobin
    option httpclose
    option forwardfor
    server beta_server_1 127.0.0.1:8080 check
    server beta_server_2 127.0.0.1:8081 check
```

With this configuration, any request on `alpha.mydomain.com` on default HTTP port (`80`) will be redirected to `alpha_backend` which serves under the port `8000`. Also, any request on `beta.mydomain.com` on port `80` will be redirected to `beta_backend` which serves under the ports `8080` and `8081` using the algorithm called `roundrobin` where each server is used in turns. To see other balance options, you can check  [HAProxy Configuration Manual](//cbonte.github.io/haproxy-dconv/configuration-1.4.html#4.2-balance). To see if load balancing works, you can kill one of your `beta` servers and check that `beta.mydomain.com` is still up an running.

You can also log HAProxy messages and even do a lot more with it! HAProxy is capable of much more of great features, but even these would suffice.
