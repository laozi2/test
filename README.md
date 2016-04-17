Name
====

ngx_tcp_module - A tcp stream module for nginx. 

ngx_tcp_lua_module - Embed the power of Lua into Nginx Servers. Work under tcp stream mode.

This module is not distributed with the Nginx source. See [the installation instructions](#installation).

Table of Contents
=================

* [Name](#name)
* [Status](#status)
* [Version](#version)
* [Config example](#config-example)
* [Description](#description)
* [Directives](#directives)
* [Code Exapmle for ngx_tcp_lua_module](#code-exapmle-for-ngx_tcp_lua_module)
* [Nginx API for Lua](#nginx-api-for-lua)



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
5. then, ./configure and make and make isntall. here is configure example
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
and for the test.lua, see example [Code Exapmle for ngx_tcp_lua_module](#code-exapmle-for-ngx_tcp_lua_module)

[Back to TOC](#table-of-contents)

Description
=========
Based on nginx-1.4.1, refer to nginx-http-lua(https://github.com/openresty/lua-nginx-module), 
follow the principles of simple, efficient and highly extensible, the nginx-tcp module is designed as a customized stream protocol server, more than http, mail server. 
And the ngx_tcp_lua module is very useful in fast implement your own service.

1. Patch log.c, ngx_http_log_module.c, add the command `nlog`,`access_nlog` to send error_log,access_log to log server with udp.
2. tcp_module for customized stream protocol on tcp, support ssl. example, tcp/ngx_tcp_demo_module. 
3. tcp_lua module: embeds Lua code, 100% non-blocking on network traffic.
   * just like ngx_tcp_demo_module, ngx_tcp_lua module is a special module on tcp_module.
   * enriched the functions such as init_by_lua,ngx.sleep,ngx.exit, just like nginx-lua-module
   * lua lib with cosocket to deal with mysql,http server. simple load banlance and retry also supported
   * support ngx.nlog to send log to udp log server, more than ngx.log to local file.
   * support ssl cosocket with upstream
4. About installation, APIs, and examples, see tcp/doc/ for more details

[Back to TOC](#table-of-contents)


Directives
==========

#### for ngx-tcp-core-module
* [server](#server)
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
* [allow](#allow)
* [deny](#deny)
* [resolver](#resolver)
* [resolver_timeout](#resolver_timeout)

### for ngx-tcp-log-module
* [log_format](#log_format)
* [access_log](#access_log)
* [access_nlog](#access_nlog)

### for ngx-tcp-ssl-module
* [ssl](#ssl)
* [ssl_certificate](#ssl_certificate)
* [ssl_certificate_key](#ssl_certificate_key)
* [ssl_dhparam](#ssl_dhparam)
* [ssl_ecdh_curve](#ssl_ecdh_curve)
* [ssl_protocols](#ssl_protocols)
* [ssl_ciphers](#ssl_ciphers)
* [ssl_verify_client](#ssl_verify_client)
* [ssl_verify_depth](#ssl_verify_depth)
* [ssl_client_certificate](#ssl_client_certificate)
* [ssl_prefer_server_ciphers](#ssl_prefer_server_ciphers)
* [ssl_session_cache](#ssl_session_cache)
* [ssl_session_timeout](#ssl_session_timeout)
* [ssl_crl](#ssl_crl)

#### for ngx-tcp-lua-module
* [lua_package_cpath](#lua_package_cpath)
* [lua_package_path](#lua_package_path)
* [lua_code_cache](#lua_code_cache)
* [init_by_lua](#init_by_lua)
* [init_by_lua_file](#init_by_lua_file)
* [process_by_lua](#process_by_lua)
* [process_by_lua_file](#process_by_lua_file)
* [lua_socket_connect_timeout](#lua_socket_connect_timeout)
* [lua_socket_send_lowat](#lua_socket_send_lowat)
* [lua_socket_pool_size](#lua_socket_pool_size)
* [lua_check_client_abort](#lua_check_client_abort)
* [lua_shared_dict](#lua_shared_dict)
* [lua_regex_cache_max_entries](#lua_regex_cache_max_entries)
* [lua_regex_match_limit](#lua_regex_match_limit)

#### for ngx-tcp-demo-module
* [demo_echo](#demo_echo)

[Back to TOC](#table-of-contents)

listen
--------------------
**syntax:** `listen` [*ip*:]*port* [backlog=*number*] [rcvbuf=*size*] [sndbuf=*size*] [deferred] [bind] [ssl]  [so_keepalive=on|off|[*keepidle]:[keepintvl]:[keepcnt*]];

**default:** listen *:0;

**context:** server

**example:**
```nginx
    listen 127.0.0.1:110;
    listen *:110;
    listen 110;     # same as *:110
```

Sets the address and port for IP on which the server will accept requests. Only IPv4 supported now.

One server{} can have diffrent ip addresses. But one specified port only can be set in one server{}.

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

protocol_name must be defined in an implemented module, such as ngx_tcp_demo_module. One server{} can only have one specified protocol_name. 

[Back to TOC](#directives)

read_timeout
--------------------
**syntax:** `read_timeout` *time*;

**default:** 60s;

**context:** tcp,server

**example:**
```nginx
read_timeout 60s;
```

Sets the timeout for read the whole protocol data.

[Back to TOC](#directives)

send_timeout
--------------------
**syntax:** `send_timeout` *time*;

**default:** 60s;

**context:** tcp,server

**example:**
```nginx
send_timeout 60s;
```

Sets a timeout for transmitting a response to the client. The timeout is set only between two successive write operations, not for the transmission of the whole response. If the client does not receive anything within this time, the connection is closed. 

[Back to TOC](#directives)

keepalive_timeout
--------------------
**syntax:** `keepalive_timeout` *time*;

**default:** 6000s;

**context:** tcp,server

**example:**
```nginx
send_timeout 6000s;
```

Sets a timeout during which a keep-alive client connection will stay open on the server side. The zero value disables keep-alive client connections.  (TODO: enable never close client connection)

[Back to TOC](#directives)

connection_pool_size
--------------------
**syntax:** `connection_pool_size` *size*;

**default:** 0.5k;

**context:** tcp,server

**example:**
```nginx
connection_pool_size 1k;
```

Allows accurate tuning of per-connection memory allocations. This directive has minimal impact on performance and should not generally be used. 

[Back to TOC](#directives)

session_pool_size
--------------------
**syntax:** `session_pool_size` *size*;

**default:** 1k;

**context:** tcp,server

**example:**
```nginx
session_pool_size 1k;
```

Allows accurate tuning of per-session memory allocations. This directive has minimal impact on performance and should not generally be used. The optimal value is little larger than average receiving data length, so it can take full use of memory and minimum times of allocations.

[Back to TOC](#directives)

client_max_body_size
--------------------
**syntax:** `client_max_body_size` *size*;

**default:** 1k;

**context:** tcp,server

**example:**
```nginx
client_max_body_size 1k;
```

Sets the maximum allowed size of the client request body, specified by protocol.

[Back to TOC](#directives)

error_log
--------------------
**syntax:** `error_log` *file* | stderr | [debug | info | notice | warn | error | crit | alert | emerg];

**default:** error_log logs/error.log error;

**context:** main,tcp,server

**example:**
```nginx
error_log logs/error.log error;
#or in tcp{},server{} below: 
error_log logs/error_tcp.log debug_tcp; 
```

Configures logging. 'debug_tcp' can be used like 'debug_http'.

[Back to TOC](#directives)

nlog
--------------------
**syntax:** `nlog` *local_ip*:*local_port* *remote_ip*:*remote_prot*;

**default:** -;

**context:** main,tcp,server

**example:**
```nginx
error_log logs/error.log error;
nlog 0.0.0.0:5001 0.0.0.0:5151; #nlog must set after error_log.
```

Configures ip address to support logging to UDP log server.

[Back to TOC](#directives)

allow
--------------------
**syntax:** `allow` *ip* | all;

**default:** -;

**context:** tcp,server

**example:**
```nginx
allow 127.0.0.1;
allow 127.0.0.0/24;
allow all;
```

Allows access for the specified network or address. The rules are checked in sequence until the first match is found. 

[Back to TOC](#directives)

deny
--------------------
**syntax:** `deny` *ip* | all;

**default:** -;

**context:** tcp,server

**example:**
```nginx
deny 127.0.0.1;
deny 127.0.0.0/24;
deny all;
```

Denies access for the specified network or address. The rules are checked in sequence until the first match is found. 

[Back to TOC](#directives)

resolver
--------------------
**syntax:** `resolver` *address* [valid=*time*];

**default:** -;

**context:** tcp,server

**example:**
```nginx
resolver 127.0.0.1 8.8.8.8 valid=30s;;
```

Configures name servers used to resolve names of upstream servers into addresses.
By default, nginx caches answers using the TTL value of a response. An optional valid parameter allows overriding it. 

[Back to TOC](#directives)

resolver_timeout
--------------------
**syntax:** `resolver` *time*;

**default:** 30s;

**context:** tcp,server

**example:**
```nginx
resolver_timeout 10s;
```

Sets a timeout for name resolution.

[Back to TOC](#directives)

log_format
--------------------
**syntax:** `log_format` *name* *string* ...;

**default:** combined '$remote_addr $time_iso8601 $msec $request_time $connection $connection_requests $protocol';

**context:** tcp,server

**example:**
```nginx
log_format log1 '$remote_addr $time_iso8601 $msec $request_time $connection $connection_requests $bytes_sent $protocol';
```

Specifies log format.

The log format can contain common variables, and variables that exist only at the time of a log write:

$remote_addr
>    client ip address.

$time_local
>    local time in the Common Log Format, "28/Sep/1970:12:00:00 +0600".

$time_iso8601
>    local time in the ISO 8601 standard format, "1970-09-28T12:00:00+06:00".

$msec
>    time in seconds with a milliseconds resolution at the time of the log write.

$request_time
>    request processing time in seconds with a milliseconds resolution; time elapsed between the first bytes were read from the client and the log write after the last bytes were sent to the client.

$connection
>    connection serial number.

$connection_requests
>    the current number of requests made through a connection.

$bytes_sent
>    the number of bytes sent to a client.

$protocol
>    the current protocol name.

[Back to TOC](#directives)

access_log
--------------------
**syntax:** `access_log` *path* | off  [*format*];

**default:** logs/access_tcp.log combined;

**context:** tcp,server

**example:**
```nginx
access_log logs/access_tcp.log log1;
```

Sets the path, format for a buffered log write. Several logs can be specified on the same level. 

[Back to TOC](#directives)

access_nlog
--------------------
**syntax:** `access_nlog` *local_ip*:*local_port*  *remote_ip*:*remote_prot*;

**default:** -;

**context:** tcp,server

**example:**
```nginx
access_log logs/access_tcp.log log1;
access_nlog 127.0.0.1:5002 127.0.0.1:5151;#access_nlog must set after access_log.
```

Configures ip address to support logging to UDP log server.

[Back to TOC](#directives)

ssl
--------------------
refer to [http://nginx.org/en/docs/http/ngx_http_ssl_module.html](#http://nginx.org/en/docs/http/ngx_http_ssl_module.html) 

[Back to TOC](#directives)

ssl_certificate
--------------------
refer to [http://nginx.org/en/docs/http/ngx_http_ssl_module.html](#http://nginx.org/en/docs/http/ngx_http_ssl_module.html) 

[Back to TOC](#directives)

ssl_certificate_key
--------------------
refer to [http://nginx.org/en/docs/http/ngx_http_ssl_module.html](#http://nginx.org/en/docs/http/ngx_http_ssl_module.html) 

[Back to TOC](#directives)

ssl_dhparam
--------------------
refer to [http://nginx.org/en/docs/http/ngx_http_ssl_module.html](#http://nginx.org/en/docs/http/ngx_http_ssl_module.html) 

[Back to TOC](#directives)

ssl_ecdh_curve
--------------------
refer to [http://nginx.org/en/docs/http/ngx_http_ssl_module.html](#http://nginx.org/en/docs/http/ngx_http_ssl_module.html) 

[Back to TOC](#directives)

ssl_protocols
--------------------
refer to [http://nginx.org/en/docs/http/ngx_http_ssl_module.html](#http://nginx.org/en/docs/http/ngx_http_ssl_module.html) 

[Back to TOC](#directives)

ssl_ciphers
--------------------
refer to [http://nginx.org/en/docs/http/ngx_http_ssl_module.html](#http://nginx.org/en/docs/http/ngx_http_ssl_module.html) 

[Back to TOC](#directives)

ssl_verify_client
--------------------
refer to [http://nginx.org/en/docs/http/ngx_http_ssl_module.html](#http://nginx.org/en/docs/http/ngx_http_ssl_module.html) 

[Back to TOC](#directives)

ssl_verify_depth
--------------------
refer to [http://nginx.org/en/docs/http/ngx_http_ssl_module.html](#http://nginx.org/en/docs/http/ngx_http_ssl_module.html) 

[Back to TOC](#directives)

ssl_client_certificate
--------------------
refer to [http://nginx.org/en/docs/http/ngx_http_ssl_module.html](#http://nginx.org/en/docs/http/ngx_http_ssl_module.html) 

[Back to TOC](#directives)

ssl_prefer_server_ciphers
--------------------
refer to [http://nginx.org/en/docs/http/ngx_http_ssl_module.html](#http://nginx.org/en/docs/http/ngx_http_ssl_module.html) 

[Back to TOC](#directives)

ssl_session_cache
--------------------
refer to [http://nginx.org/en/docs/http/ngx_http_ssl_module.html](#http://nginx.org/en/docs/http/ngx_http_ssl_module.html) 

[Back to TOC](#directives)

ssl_session_timeout
--------------------
refer to [http://nginx.org/en/docs/http/ngx_http_ssl_module.html](#http://nginx.org/en/docs/http/ngx_http_ssl_module.html) 

[Back to TOC](#directives)

ssl_crl
--------------------
refer to [http://nginx.org/en/docs/http/ngx_http_ssl_module.html](#http://nginx.org/en/docs/http/ngx_http_ssl_module.html) 

[Back to TOC](#directives)

lua_package_cpath
--------------------
**syntax:** `lua_package_cpath` *lua-style-cpath-str*;

**default:** *The content of LUA_CPATH environment variable or Lua's compiled-in defaults.*;

**context:** tcp

**example:**
```nginx
lua_package_cpath '/bar/baz/?.so;/blah/blah/?.so;;';
```

Sets the Lua C-module search path used by scripts specified by init_by_lua[_file], protocol_by_lua[_file]. The cpath string is in standard Lua cpath form, and ;; can be used to stand for the original cpath.

[Back to TOC](#directives)

lua_package_path
--------------------
**syntax:** `lua_package_path` *lua-style-path-str*;

**default:** *The content of LUA_PATH environ variable or Lua's compiled-in defaults.*

**context:** tcp

**example:**
```nginx
lua_package_path '/foo/bar/?.lua;/blah/?.lua;;'
```

Sets the Lua module search path used by scripts specified by init_by_lua[_file], protocol_by_lua[_file]. The path string is in standard Lua path form, and ;; can be used to stand for the original search paths.

[Back to TOC](#directives)

lua_code_cache
--------------------
**syntax:** `lua_code_cache` on | off;

**default:** *The content of LUA_PATH environ variable or Lua's compiled-in defaults.*

**context:** tcp,server

**example:**
```nginx
lua_code_cache on;
```

Enables or disables the Lua code cache for Lua code in *_by_lua_file directives and Lua modules.

When turning off, every request served by ngx_lua will run in a separate Lua VM instance.

Disabling the Lua code cache is strongly discouraged for production use and should only be used during development as it has a significant negative impact on overall performance.

[Back to TOC](#directives)

init_by_lua
--------------------
**syntax:** `init_by_lua` *lua-script-str*;

**default:** -

**context:** tcp

**example:**
```nginx
init_by_lua 'a = require("a")';
```

Runs the Lua code specified by the argument <lua-script-str> on the global Lua VM level when the Nginx master process (if any) is loading the Nginx config file.

[Back to TOC](#directives)

init_by_lua_file
--------------------
**syntax:** `init_by_lua_file` *path-to-lua-script-file*;

**default:** -

**context:** tcp

**example:**
```nginx
init_by_lua_file 'conf/init_by_lua.lua';
```

Equivalent to init_by_lua, except that the file specified by <path-to-lua-script-file> contains the Lua code or Lua/LuaJIT bytecode to be executed.

[Back to TOC](#directives)

process_by_lua
--------------------
**syntax:** `process_by_lua` *lua-script-str*;

**default:** -

**context:** server

**example:**
```nginx
process_by_lua 'ngx.exit()';
```

Executes Lua code string specified in <lua-script-str> for requests of every connection. The Lua code may make API calls and is executed as a new spawned coroutine in an independent global environment (i.e. a sandbox).

[Back to TOC](#directives)

process_by_lua_file
--------------------
**syntax:** `process_by_lua_file` *path-to-lua-script-file*;

**default:** -

**context:** server

**example:**
```nginx
process_by_lua_file 'conf/test.lua';
```

Equivalent to process_by_lua, except that the file specified by <path-to-lua-script-file> contains the Lua code or Lua/LuaJIT bytecode to be executed.

[Back to TOC](#directives)

lua_socket_connect_timeout
--------------------
**syntax:** `lua_socket_connect_timeout` *time*;

**default:** 60s;

**context:** tcp,server

**example:**
```nginx
lua_socket_connect_timeout 5;
```

This directive controls the default timeout value used in tcp socket object's connect method and can be overridden by the settimeout method.

The <time> argument can be an integer, with an optional time unit, like s (second), ms (millisecond), m (minute). The default time unit is s, i.e., "second". The default setting is 60s.

[Back to TOC](#directives)

lua_socket_send_lowat
--------------------
**syntax:** `lua_socket_send_lowat` *size*;

**default:** 0;

**context:** tcp,server

**example:**
```nginx
lua_socket_send_lowat 0;
```

Controls the lowat (low water) value for the cosocket send buffer. 

[Back to TOC](#directives)

lua_socket_pool_size
--------------------
**syntax:** `lua_socket_pool_size` *size*;

**default:** 30;

**context:** tcp,server

**example:**
```nginx
lua_socket_pool_size 10;
```

Specifies the size limit (in terms of connection count) for every cosocket connection pool associated with every remote server (i.e., identified by either the host-port pair). 

When the connection pool exceeds the available size limit, the least recently used (idle) connection already in the pool will be closed to make room for the current connection.

Note that the cosocket connection pool is per nginx worker process rather than per nginx server instance, so size limit specified here also applies to every single nginx worker process.

[Back to TOC](#directives)

lua_check_client_abort
--------------------
**syntax:** `lua_check_client_abort` on|off;

**default:** off;

**context:** tcp,server

**example:**
```nginx
lua_check_client_abort on;
```

This directive controls whether to check for premature client connection abortion. 

[Back to TOC](#directives)

lua_shared_dict
--------------------
**syntax:** `lua_shared_dict` *name* *size*;

**default:** -;

**context:** tcp

**example:**
```nginx
lua_shared_dict dogs 10m;
```

Declares a shared memory zone, <name>, to serve as storage for the shm based Lua dictionary ngx.shared.<name>.

Shared memory zones are always shared by all the nginx worker processes in the current nginx server instance.

The <size> argument accepts size units such as k and m.  **At least 8k**;

[Back to TOC](#directives)

lua_shared_dict
--------------------
**syntax:** `lua_regex_cache_max_entries` *num*;

**default:** 1024;

**context:** tcp

**example:**
```nginx
lua_shared_dict dogs 10m;
```

Specifies the maximum number of entries allowed in the worker process level compiled regex cache.

The regular expressions used in ngx.re.match, ngx.re.gmatch, ngx.re.sub, and ngx.re.gsub will be cached within this cache if the regex option o (i.e., compile-once flag) is specified.

[Back to TOC](#directives)

lua_regex_match_limit
--------------------
**syntax:** `lua_regex_match_limit` *num*;

**default:** 0;

**context:** tcp

**example:**
```nginx
lua_shared_dict dogs 10m;
```

Specifies the "match limit" used by the PCRE library when executing the ngx.re API. To quote the PCRE manpage, "the limit ... has the effect of limiting the amount of backtracking that can take place."

When the limit is hit, the error string "pcre_exec() failed: -8" will be returned by the ngx.re API functions on the Lua land.

When setting the limit to 0, the default "match limit" when compiling the PCRE library is used. And this is the default value of this directive.

[Back to TOC](#directives)

demo_echo
--------------------
**syntax:** `demo_echo` *str*;

**default:** -;

**context:** server

**example:**
```nginx
demo_echo "hello world";
```

This directive is used for ngx_tcp_demo_module, which define a echo protocol. The request stream is 4 bytes head contains request data length. 
When demo_echo string set, the server will response 4 bytes head contains response data length, and demo_echo string. when this directive not set, the response body data is request body data.

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


Nginx API for Lua
==============
* [Introduction](#introduction)
* [ngx.say](#ngx.say)
* [ngx.print](#ngx.print)
* [ngx.receive](#ngx.receive)
* [ngx.wait_next_request](#ngx.wait_next_request)
* [ngx.exit](#ngx.exit)
* 
* [ngx.sleep](#ngx.sleep)
* 
* [ngx.utctime](#ngx.utctime)
* [ngx.localtime](#ngx.localtime)
* [ngx.time](#ngx.time)
* [ngx.now](#ngx.now)
* [ngx.today](#ngx.today)
* [ngx.tcp_time](#ngx.tcp_time)
* [ngx.parse_tcp_time](#ngx.parse_tcp_time)
* [ngx.update_time](#ngx.update_time)
* [ngx.start_time](#ngx.start_time)
* 
* [ngx.socket.tcp](#ngx.socket.tcp)
* [tcpsock:connect](#tcpsockconnect)
* [tcpsock:sslhandshake](#tcpsocksslhandshake)
* [tcpsock:receive](#tcpsockreceive)
* [tcpsock:receive_http](#tcpsockreceive_http)
* [tcpsock:send](#tcpsocksend)
* [tcpsock:close](#tcpsockclose)
* [tcpsock:setoption](#tcpsocksetoption)
* [tcpsock:settimeout](#tcpsocksettimeout)
* [tcpsock:getreusedtimes](#tcpsockgetreusedtimes)
* [tcpsock:setkeepalive](#tcpsocksetkeepalive)
* 
* [ngx.socket.udp](#ngx.socket.udp)
* [udpsock:setpeername](#udpsocksetpeername)
* [udpsock:send](#udpsocksend)
* [udpsock:receive](#udpsockreceive)
* [udpsock:settimeout](#udpsocksettimeout)
* [udpsock:close](#udpsockclose)
* 
* [ngx.new_ssl_ctx](#ngx.new_ssl_ctx)
* 
* [Nginx log level constants](#nginx-log-level-constants)
* [ngx.log](#ngx.log)
* [ngx.print](#ngx.print) 
* [ngx.nlog](#ngx.nlog)
* [nlog:send](#nlog:send)
* 
* [ngx.re.find](#ngx.re.find)
* [ngx.re.match](#ngx.re.match)
* [ngx.re.gmatch](#ngx.re.gmatch)
* [ngx.re.sub](#ngx.re.sub)
* [ngx.re.gsub](#ngx.re.gsub)
* 
* [ngx.decode_base64](ngx.decode_base64)
* [ngx.encode_base64](#ngx.encode_base64)
* [ngx.md5_bin](#ngx.md5_bin)
* [ngx.md5](#ngx.md5)
* [ngx.sha1_bin](#ngx.sha1_bin)
* [ngx.crc32_short](#ngx.crc32_short)
* [ngx.crc32_long](#ngx.crc32_long)
* [ngx.hmac_sha1](#ngx.hmac_sha1)
* [ngx.escape_uri](#ngx.escape_uri)
* [ngx.quote_sql_str](#ngx.quote_sql_str)
* 
* [ngx.shared.DICT](#ngx.shared.DICT)
* [ngx.shared.DICT.get](#ngx.shared.DICT.get)
* [ngx.shared.DICT.get_stale](#ngx.shared.DICT.get_stale)
* [ngx.shared.DICT.set](#ngx.shared.DICT.set)
* [ngx.shared.DICT.safe_set](#ngx.shared.DICT.safe_set)
* [ngx.shared.DICT.add](#ngx.shared.DICT.add)
* [ngx.shared.DICT.safe_add](#ngx.shared.DICT.safe_add)
* [ngx.shared.DICT.replace](#ngx.shared.DICT.replace)
* [ngx.shared.DICT.delete](#ngx.shared.DICT.delete)
* [ngx.shared.DICT.incr](#ngx.shared.DICT.incr)
* [ngx.shared.DICT.flush_all](#ngx.shared.DICT.flush_all)
* [ngx.shared.DICT.flush_expired](#ngx.shared.DICT.flush_expired)
* [ngx.shared.DICT.get_keys](#ngx.shared.DICT.get_keys)

[Back to TOC](#table-of-contents)


Introduction
------------------
The Nginx Lua API described below can only be called within the user Lua code run in the context of these configuration directives.

The API is exposed to Lua in the form of two standard packages ngx and ndk. These packages are in the default global scope within ngx_lua and are always available within ngx_lua directives.

Any way, lua exception will be thrown when something internal error occurs in APIs. Here are some common reason: out of memory, api calls in wrong context, wrong arguments.


ngx.say
------------------
**syntax:** send_bytes, err = ngx.print(...)

**context:** `process_by_lua*`

**arguments:** 
>    one or more lua variables inclue table, string, nil, number, boolean and `ngx.null`. table is array table incule other variable types.

**returns:** 
>    send_bytes: 

**example:**
```lua
local send_bytes, err = ngx.print("hello world",21,nil,true,false,{"a","b"})   
--output: hello world21niltruefalseab
```

Emits arguments concatenated to the client as response body.

[Back to TOC](#nginx-api-for-lua)

