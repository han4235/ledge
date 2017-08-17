# Ledge

An [ESI](https://www.w3.org/TR/esi-lang) capable HTTP cache for [Nginx](http://nginx.org) / [OpenResty](https://openresty.org), backed by [Redis](http://redis.io).


## Table of Contents

* [Overview](#overview)
* [Installation](#installation)
* [Nomenclature](#nomenclature)
* [Minimal configuration](#minimal-configuration)
* [Config systems](#config-systems)
* [Events system](#events-system)
* [Licence](#licence)


## Overview

Ledge aims to be an RFC compliant HTTP reverse proxy cache, providing a fast, robust and scalable alternative to Squid / Varnish etc.

Moreover, it is particularly suited to applications where the origin is expensive or distant, making it desirable to serve from cache as optimistically as possible. For example, using [ESI](#edge-side-includes-esi) to separate page
fragments where their TTL differs, serving stale content whilst [revalidating in the background](#stale--background-revalidation), [collapsing](#collapsed-forwarding) concurrent similar upstream requests, dynamically modifying the cache key specification, and [automatically revalidating](#revalidate-on-purge) content with a PURGE API.


## Installation

### 1. Download and install:

* [OpenResty](http://openresty.org/) >= 1.11.x
* [Redis](http://redis.io/download) >= 2.8.x
* [LuaRocks](https://luarocks.org/)

### 2. Install Ledge and its dependencies:

```
luarocks install ledge
```

This will install the latest stable release, and all other Lua module dependencies, which if installing manually without LuaRocks are:

* [lua-resty-http](https://github.com/pintsized/lua-resty-http)
* [lua-resty-redis-connector](https://github.com/pintsized/lua-resty-redis-connector)
* [lua-resty-qless](https://github.com/pintsized/lua-resty-qless)
* [lua-resty-cookie](https://github.com/cloudflare/lua-resty-cookie)
* [lua-ffi-zlib](https://github.com/hamishforbes/lua-ffi-zlib)
* [lua-resty-upstream](https://github.com/hamishforbes/lua-resty-upstream) *(optional, for load balancing / healthchecking upstreams)*

### 3. Review OpenResty documentation

If you are new to OpenResty, it's quite important to review the [lua-nginx-module](https://github.com/openresty/lua-nginx-module) documentation on how to run Lua code in Nginx, as the environment is unusual. Specifcally, it's useful to understand the meaning of the different Nginx phase hooks such as `init_by_lua` and `content_by_lua`, as well as how the `lua-nginx-module` locates Lua modules with the [lua_package_path](https://github.com/openresty/lua-nginx-module#lua_package_path) directive.


## Nomenclature

The central module is called `ledge`, and provides factory methods for creating `handler` instances (for handling a request) and `worker` instances (for running background tasks). The `ledge` module is also where global configuration is managed.

A `handler` is short lived. It is typically created at the beginning of the Nginx `content` phase for a request, and when its `run()` method is called, takes responsibility for processing the current request and delivering a response. When `run()` has completed, HTTP status, headers and body will have been delivered to the client.

A `worker` is long lived, and there is one per Nginx worker process. It is created when Nginx starts a worker process, and dies when the Nginx worker dies. The `worker` pops queued background jobs and processes them.

An `upstream` is the only thing which must be manually configured, and points to another HTTP host where actual content lives. Typically one would use DNS to resolve client connections to the Nginx server running Ledge, and tell Ledge where to fetch from with the `upstream` configuration. As such, Ledge isn't designed to work as a forwarding proxy.

[Redis](http://redis.io) is used for much more than cache storage. We rely heavily on its data structures to maintain cache `metadata`, as well as embedded Lua scripts for atomic task management and so on. By default, all cache body data and `metadata` will be stored in the same Redis instance. The location of cache `metadata` is global, set when Nginx starts up.

Cache body data is handled by the `storage` system, and as mentioned, by default shares the same Redis instance as the `metadata`. However, `storage` is abstracted via a driver system making it possible to store cache body data in a separate Redis instance, or a group of horizontally scalable Redis instances via a [proxy](https://github.com/twitter/twemproxy), or to roll your own `storage` driver, for example targeting PostreSQL or even simply a filesystem. It's perhaps important to consider that by default all cache storage uses Redis, and as such is bound by system memory.


## Minimal configuration

Assuming you have Redis running on `localhost:6379`, and your upstream is at `localhost:8080`.

```nginx
http {
    if_modified_since Off;
    lua_check_client_abort On;

    init_worker_by_lua_block {
        require("ledge").create_worker():run()
    }

    server {
        server_name example.com;
        listen 80;

        location / {
            content_by_lua_block {
                require("ledge").create_handler({
                    upstream_host = "127.0.0.1",
                    upstream_port = 8080,
                }):run()
            }
        }
    }
}
```


## Config systems

There are four different layers to the configuration system. Firstly there is the `metadata` location config and default `handler` config, which are global and must be set during the Nginx `init` phase. Beyond this, you can specify `handler` config on an Nginx `location` block basis, which will override any defaults given. And finally, there are some performance tuning config options for the `worker` instances.

In addition, there is an [events system](#events-system) for binding Lua functions to mid-request events, proving opportunities to dynamically alter configuration.


### Metadata config

The `ledge.configure()` method provides Ledge with Redis connection details for `metadata`. This is global and cannot be specified or adjusted outside the Nginx `init` phase.

```nginx
init_by_lua_block {
    require("ledge").configure({
        redis_connector_params = {
            url = "redis://mypassword@127.0.0.1:6380/3",
        }
        qless_db = 4,
    })
}
```

### Handler defaults

The `ledge.set_handler_defaults()` method overrides the default configuration used for all spawned request `handler` instances. This is global and cannot be specified or adjusted outside the Nginx `init` phase, but defaults can be overriden on a per `handler` basis.

```nginx
init_by_lua_block {
    require("ledge").set_handler_defaults({
        upstream_host = "127.0.0.1",
        upstream_port = 8080,
    })
}
```

### Handler instance config

Config given to `ledge.create_handler()` will be merged with the defaults, allowing certain options to be adjusted on a per Nginx `location` basis.

```nginx
server {
    server_name example.com;
    listen 80;

    location / {
        content_by_lua_block {
            require("ledge").create_handler({
                upstream_port = 8081,
            }):run()
        }
    }
}
```

### Worker config

Background job queues can be run at varying amounts of concurrency per worker. See [managing qless](#managing-qless) for more details.

```nginx
init_worker_by_lua_block {
    require("ledge").create_worker({
        interval = 1,
        gc_queue_concurrency = 1,
        purge_queue_concurrency = 2,
        revalidate_queue_concurrency = 5,
    }):run()
}
```


## Events system

Ledge makes most of its decisions based on the content it is working with. HTTP request and response headers drive the semantics for content delivery, and so rather than having countless configuration options to change this, we instead provide opportunities to alter the given semantics when necessary.

For example, if an `upstream` fails to set a long enough cache expiry, rather than inventing an option such as "extend\_ttl", we instead would `bind` to the `after_upstream_request` event, and adjust the response headers to include the ttl we're hoping for.

```lua
handler:bind("after_upstream_request", function(res)
    res.header["Cache-Control"] = "max-age=86400"
end)
```

This particular event fires after we've fetched upstream, but before Ledge makes any decisions about whether the content can be cached or not. Once we've adjustead our headers, Ledge will read them as if they came from the upstream itself.

Note that multiple functions can be bound to a single event, either globally or per handler, and they will be called in the order they were bound. There is also currently no means to inspect which functions have been bound, or to unbind them.


### Binding globally

Binding a function globally means it will fire for the given event, on all requests. This is perhaps useful if you have many different `location` blocks, but need to always perform the same logic.

```nginx
init_by_lua_block {
    require("ledge"):bind("before_serve", function(res)
        res.header["X-Foo"] = "bar"   -- always set X-Foo to bar
    end)
}
```

### Binding to handlers

More commonly, we just want to alter behaviour for a given Nginx `location`. 

```nginx
location /foo_location {
    content_by_lua_block {
        local handler = require("ledge").create_handler()
        
        handler:bind("before_serve", function(res)
            res.header["X-Foo"] = "bar"   -- only set X-Foo for this location
        end)
        
        handler:run()
    }
}
```

### Performance implications

Writing simple logic for events is not expensive at all (and in many cases will be JIT compiled). If you need to consult service endpoints during an event then obviously consider that this will affect your overall latency, and make sure you do everything in a **non-blocking** way, e.g. using [cosockets](https://github.com/openresty/lua-nginx-module#ngxsockettcp) provided by OpenResty, or a driver based upon this.

If you have lots of event handlers, consider that creating closures in Lua is relatively expensive. A good solution would be to make your own module, and pass the defined functions in.

```nginx
location /foo_location {
    content_by_lua_block {
        local handler = require("ledge").create_handler()
        handler:bind("before_serve", require("my.handler.hooks").add_foo_header)
        handler:run()
    }
}
```


## Author

James Hurst <james@pintsized.co.uk>


## Licence

This module is licensed under the 2-clause BSD license.

Copyright (c) James Hurst <james@pintsized.co.uk>

All rights reserved.

Redistribution and use in source and binary forms, with or without modification, are permitted provided that the following conditions are met:

* Redistributions of source code must retain the above copyright notice, this list of conditions and the following disclaimer.

* Redistributions in binary form must reproduce the above copyright notice, this list of conditions and the following disclaimer in the documentation and/or other materials provided with the distribution.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
