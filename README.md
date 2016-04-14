Name
====

A tcp stream module for nginx. 

ngx_tcp_lua_module - Embed the power of Lua into Nginx Servers.

*This module is not distributed with the Nginx source.* See [the installation instructions](#installation).

Table of Contents
=================

* [Name](#name)
* [Status](#status)
* [Version](#version)
* [Config example](#config-example)
* [Description](#description)
* [Directives](#directives)
* [Code Exapmle for ngx_tcp_lua_module](#code-exapmle-for-ngx_tcp_lua_module)
* [API for ngx_tcp_lua_module](#api-for-ngx_tcp_lua_module)



Status
======

Production ready.

Version
=======

This document describes ngx_lua [v0.10.2](https://github.com/openresty/lua-nginx-module/tags) released on 8 March 2016.


Installation
==========
1. Download the nginx-1.4.1 [HERE](http://nginx.org/download/nginx-1.4.1.tar.gz) 
2. unzip, cd nginx-1.4.1,  copy auto.patch,src_core.patch,src_http.patch into current directory
3. patch -p1 < auto.patch 
   patch -p1 < src_core.patch
   patch -p1 < src_http.patch   #optional, for nginx access_nlog
4. copy tcp/ into current src/
5. configure example
```bash
    # yum -y install  -y pcre* openssl*
    # for pcre, such as ngx.gmatch etc, --with-pcre=PATH/pcre-8.36 --with-pcre-jit
    # if use openssl, then need --with-http_ssl_module
    #
    export LUAJIT_LIB=/usr/local/lib
    export LUAJIT_INC=/usr/local/include/luajit-2.0
    ./configure --prefix=/usr/local/nginx_tcp \
			--with-debug \
			--with-pcre=/root/ngx_tcp_compile/softwares/pcre-8.36 \
			--with-pcre-jit \
			--without-http_gzip_module \
			--with-http_stub_status_module \
			--with-http_ssl_module \
			--with-tcp \
			--with-tcp_ssl_module \
			--with-openssl=/opt/openssl-1.0.1e \
			--with-openssl-opt=-g \
			--add-module=src/tcp/ngx_tcp_log_module \
			--add-module=src/tcp/ngx_tcp_demo_module \
			--add-module=src/tcp/ngx_tcp_lua_module
```

6. Build the source with ngx_tcp_lua_module:
```bash
    wget http://luajit.org/download/LuaJIT-2.0.0.tar.gz
    tar -xvfz LuaJIT-2.0.0.tar.gz
    cd LuaJIT-2.0.0
    make &&  make install

    # tell nginx's build system where to find luajit:
    export LUAJIT_LIB=/usr/local/lib
    export LUAJIT_INC=/usr/local/include/luajit-2.0

    # or tell where to find Lua
    #export LUA_LIB=/path/to/lua/lib
    #export LUA_INC=/path/to/lua/include
```


Config example
==========

#### nginx tcp core module, for tcp stream server
```nginx
    tcp {
        #connection_pool_size 1k;   #main/srv/take one/default 0.5k
        session_pool_size 1k;  #main/srv/take one/default 1k
        client_max_body_size 1k;   #main/srv/take one/default 1k;

        read_timeout 60s;    #main/srv/take one/default 60s
        #send_timeout 60s;    #main/srv/take one/default 60s
        #keepalive_timeout 60; #main/srv/take one/no set,no keepalive_timeout 

        #error_log  logs/error_tcp.log debug_tcp;  #main/srv/take one more/default null
        error_log logs/error_tcp.log info;

        log_format token '$remote_addr $time_iso8601 $msec $request_time $connection $connection_requests $bytes_sent $protocol';
        #default log_format combined '$remote_addr $time_iso8601 $msec $request_time $connection $connection_requests $protocol';

        server {
            listen 6666;

            protocol demo;

            #access_log off;
            access_log logs/access_tcp.log token;  #default access_log logs/access_tcp.log;
            access_nlog 0.0.0.0:5002 0.0.0.0:5151;

            allow 127.0.0.1;
            deny all;
        }
        
        #server {
        #    listen 6433;
        #
        #    protocol demo;
        #
        #    #access_log off;
        #    access_log logs/access_tcp.log demo;  #default access_log logs/access_tcp.log;
        #    access_nlog 0.0.0.0:5002 0.0.0.0:5151;
        #
        #    ssl                  on;
        #    ssl_certificate      xxx-chain.pem;
        #    ssl_certificate_key  xxx-key.pem;
        #    ssl_session_timeout  5m;
        #    ssl_protocols  SSLv2 SSLv3 TLSv1;
        #    ssl_ciphers  HIGH:!aNULL:!MD5;
        #    ssl_prefer_server_ciphers   on;
        #}
    
    }

```

####  for lua module
```nginx

    tcp {
        lua_package_path '/usr/local/nginx_tcp/conf/?.lua;/usr/local/nginx_tcp/conf/lua_module/?.lua;;';
        lua_package_cpath '/usr/local/nginx_tcp/conf/lua_module/?.so;;';

        lua_shared_dict db_lock 100m;

        init_by_lua_file 'conf/init_by_lua.lua';

        server {
            listen 6666;
            
            protocol tcp_lua;
            process_by_lua_file 'conf/test.lua';
    }
```

[Back to TOC](#table-of-contents)

Description
=========
Based on nginx-1.4.1, refer to nginx-http-lua(https://github.com/openresty/lua-nginx-module), 
based on principles of simple, efficient and highly extensible, the nginx-tcp module is designed as a customized stream protocol server, more than http, mail server.
And the ngx_tcp_lua module is very useful in fast implement your own service.

1. Patch log.c, ngx_http_log_module, add the command `nlog`,`access_nlog` to send error_log,access_log to log server.
2. tcp_module for customized stream protocol on tcp, support ssl.
    example, tcp/ngx_tcp_demo_module. 
3. tcp_lua module: embeds Lua code, 100% non-blocking on network traffic.
    3.1 just like ngx_tcp_demo_module, ngx_tcp_lua module is a special module on tcp_module.
    3.2 enriched the functions such as init_by_lua,ngx.sleep,ngx.exit, just like nginx-lua-module
    3.3 lua lib with cosocket to deal with mysql,http server. simple load banlance and retry also supported
    3.4 support ngx.nlog to send log to udp log server, more than ngx.log to local file.
    3.5 support ssl cosocket with upstream
4. About installation, APIs, and examples, see tcp/doc/ for more details

[Back to TOC](#table-of-contents)


Directives 
==========

#### for core config
* [listen](#listen)
* [protocol](#protocol)
* [read_timeout](#read_timeout)
* [send_timeout](#send_timeout)
* [keepalive_timeout](#keepalive_timeout)
* [connection_pool_size](#connection_pool_size)
* [session_pool_size](#session_pool_size)
* [client_max_body_size](#client_max_body_size)
* [error_log](#error_log)
* [nlog](#nlog)
* [log_format](#log_format)
* [access_log](#access_log)
* [access_nlog](#access_nlog)
* [allow](#allow)
* [deny](#deny)
* [resolver](#resolver)
* [resolver_timeout](#resolver_timeout)

#### for lua module
* [lua_package_cpath](#lua_package_cpath)
* [lua_package_path](#lua_package_path)
* [lua_code_cache](#lua_code_cache)
* [init_by_lua](#init_by_lua)
* [init_by_lua_file](#init_by_lua_file)
* [process_by_lua](#process_by_lua)
* [process_by_lua_file](#process_by_lua_file)
* [lua_socket_connect_timeout](#lua_socket_connect_timeout)
* [lua_socket_pool_size](#lua_socket_pool_size)
* [lua_check_client_abort](#lua_check_client_abort)
* [lua_shared_dict](#lua_shared_dict)

[Back to TOC](#table-of-contents)

listen
--------------------
**syntax:** `listen` [*ip*:]*port* [backlog=*number*] [rcvbuf=*size*] [sndbuf=*size*] [deferred] [bind] [so_keepalive=on|off|[*keepidle]:[keepintvl]:[keepcnt*]];

**default:** listen *:0;

**context:** server

**example:**
```nginx
    listen 127.0.0.1:110;
    listen *:110;
    listen 110;     # same as *:110
```

Sets the address and port for IP on which the server will accept requests. Only IPv4 supported now.

One server{} can have diffrent ip addresses. But one specified port only can be set in on server{}.

[Back to TOC](#directives)

protocol
--------------------
**syntax:** `protocol` *protocol_name*;

**default:** -;

**context:** server

**example:**
```nginx
    protocol demo;
```

protocol_name must be defined in implemented module, such as ngx_tcp_demo_module. One server{} can only have one specified protocol_name. 

[Back to TOC](#directives)


Code Exapmle for ngx_tcp_lua_module
===========

```lua
    local client_sock = ngx.socket.tcp()
    client_sock:settimeout(5000,1000,3000)

    local data = "hello world 1234567890"
    local data_len = string.len(data)
    local data_len_h = netutil.htonl(data_len)
    local req_data = netutil.packint32(data_len_h) .. data


    local upstream_test = function(u)
        local ret,err = u:connect("127.0.0.1",8000,"127.0.0.1_8000_1");
        local reuse = u:getreusedtimes()
        ngx.log(ngx.INFO,"connect : "..tostring(ret).." "..tostring(err).." "..tostring(reuse))
        if not ret then 
             return
        end

        ret,err = u:send(req_data)
        ngx.log(ngx.INFO,"send : "..tostring(ret).." "..tostring(err))
        if not ret then
            return
        end

        local data,err = u:receive(4,nil)
        if use_log then ngx.log(ngx.INFO,"receive : "..tostring(data).." "..tostring(err)) end
        if not data then
            return
        end

        local totoal_len = netutil.unpackint32(string.sub(data,1,4))
        ngx.log(ngx.INFO,"totoal_len : "..tostring(totoal_len))

        local data,err = u:receive(totoal_len - 4,nil)
        ngx.log(ngx.INFO,"receive again: ["..tostring(data).."] "..tostring(err))
        if not data then
             return
        end

        if totoal_len - 4 ~= #data then
             ngx.log(ngx.INFO,"receive len not match")
             return
        end
        u:setkeepalive()
    end


    local test_shm = function()
         local key = "x"
         local value = "hello world"
         local dogs = ngx.shared.db_lock
         local shm_value = dogs:get(key)

         if not shm_value then
             local succ, err, forcible = dogs:set(key,value,10000)
             ngx.log(ngx.INFO,tostring(succ)..","..tostring(err)..","..tostring(forcible))
         end

         local shm_value = dogs:get(key)
         ngx.log(ngx.INFO,tostring(shm_value))
    end


    while true do
         local data,r2,r3 = ngx.receive(10,6)

         ngx.say("receive ret "..tostring(data).." "..tostring(r2).." "..tostring(r3) .. ","..collectgarbage("count"))
         if not data then
              ngx.say("exit")
              ngx.exit()
         end

       --ngx.sleep(5)

       --upstream_test(client_sock)

       test_shm()

       --collectgarbage()

        ngx.wait_next_request()
    end

```

[Back to TOC](#table-of-contents)
