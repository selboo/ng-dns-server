# ng-dns-server

dns server

## install openresty

```
# wget https://openresty.org/download/openresty-1.15.8.1.tar.gz
# ./configure --prefix=/usr/local/openresty-dns/
# gmake -j
# gmake install
```

## install lua-resty-dns-server

https://github.com/vislee/lua-resty-dns-server

```
opm get vislee/lua-resty-dns-server
```

## install lua-resty-mlcache

https://github.com/thibaultcha/lua-resty-mlcache

```
opm get thibaultcha/lua-resty-mlcache
```

## install openresty-dns

```
wget https://raw.githubusercontent.com/selboo/openresty-dns/master/53.lua \
 -O /usr/local/openresty-dns/nginx/conf/53.lua
```

## install redis

```
# yum install redis -y
```

## config nginx.conf

```
#user  nobody;
worker_processes auto;

error_log  logs/error.log ;
pid        logs/nginx.pid;

events {
    worker_connections  1024;
}


stream {

    lua_package_path '/usr/local/openresty-dns/lualib/?.lua;/usr/local/openresty-dns/nginx/conf/?.lua;;';
    lua_package_cpath '/usr/local/openresty-dns/lualib/?.so;;';

    lua_shared_dict QUERYCACHE 32m;

    init_by_lua_block {
        local mlcache = require "resty.mlcache"

        local cache, err = mlcache.new("my_cache", "QUERYCACHE", {
            lru_size = 100000,
            ttl      = 10,
            neg_ttl  = 10,
        })

        _G.cache = cache

        local DNSTYPES = {}

        DNSTYPES[1]   = "A"
        DNSTYPES[2]   = "NS"
        DNSTYPES[5]   = "CNAME"
        DNSTYPES[6]   = "SOA"
        DNSTYPES[12]  = "PTR"
        DNSTYPES[15]  = "MX"
        DNSTYPES[16]  = "TXT"
        DNSTYPES[28]  = "AAAA"
        DNSTYPES[33]  = "SRV"
        DNSTYPES[99]  = "SPF"
        DNSTYPES[255] = "ANY"

        _G.DNSTYPES = DNSTYPES
    }

    server {
        listen 53 udp ;
        content_by_lua_file conf/53.lua;
    }


}
```

## start redis openresty-dns

```
# systemctl restart redis
# /usr/local/openresty-dns/nginx/sbin/nginx
```

## dns type

#### A

```
## tld|sub|view|type   value|ttl   set
# redis-cli
127.0.0.1:6379> sadd aikaiyuan.com|lb|*|A 220.181.136.165|3600 220.181.136.166|3600
OK
# dig @127.0.0.1 lb.aikaiyuan.com
```

#### CNAME

```
## tld|sub|view|type   value|ttl    set
# redis-cli
127.0.0.1:6379> sadd aikaiyuan.com|www|*|CNAME   aikaiyuan.appchizi.com.|3600
OK
# dig @127.0.0.1 www.aikaiyuan.com CNAME
```

#### AAAA

```
## tld|sub|view|type   value|ttl   set
# redis-cli
127.0.0.1:6379> sadd aikaiyuan.com|ipv6|*|AAAA 240c::6666|60 240c::8888|60
OK
# dig @127.0.0.1 ipv6.aikaiyuan.com AAAA
```

#### NS

```
## tld|sub|view|type   value|ttl   set
# redis-cli
127.0.0.1:6379> sadd aikaiyuan.com|ns|*|NS ns10.aikaiyuan.com.|86400
OK
# dig @127.0.0.1 ns.aikaiyuan.com NS
```

#### TXT

```
## tld|sub|view|type   value|ttl   set
# redis-cli
127.0.0.1:6379> sadd aikaiyuan.com|txt|*|TXT txt.aikaiyuan.com|1200
OK
# dig @127.0.0.1 txt.aikaiyuan.com txt
```

#### MX

```
## tld|sub|view|type   value|ttl|preference   set
# redis-cli
127.0.0.1:6379> sadd aikaiyuan.com|@|*|MX smtp1.qq.com.|720|10 smtp2.qq.com.|720|10
OK
# dig @127.0.0.1 aikaiyuan.com MX
```

#### SRV

```
## tld|sub|view|type   priority|weight|port|value|ttl
# redis-cli
127.0.0.1:6379> sadd aikaiyuan.com|srv|*|SRV 1|100|800|www.aikaiyuan.com|120
OK
# dig @127.0.0.1 srv.aikaiyuan.com SRV
```

