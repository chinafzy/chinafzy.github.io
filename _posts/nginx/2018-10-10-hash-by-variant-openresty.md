---
title: "在nginx/openresty中利用变量进行转发分片"
categories: nginx
tags: Nginx
---

## 应用场景 
在多服务器环境下，有时候会有这样的场景（例如`session sticky`）：
+ 要求不同的用户分配到不同的服务器上
+ 如果某个用户如果已经分配过服务器，以后会一直分到这个服务器 

我们这个范例，用url来作为变量，把请求分发到不同的后端服务器

使用openresty，或者nginx + lua-nginx-core

## 代码范例

```conf
server {
    listen 85;

    location / {
        content_by_lua_block {
            ngx.header['Content-Type'] = 'text/html'
            ngx.say(85 .. ' ' .. ngx.var.uri)
        }
    }
}

server {
    listen 86;

    location / {
        content_by_lua_block {
            ngx.header['Content-Type'] = 'text/html'
            ngx.say(86 .. ' ' .. ngx.var.uri)
        }
    }
}

upstream u_x {
    hash $u_hash;

    server localhost:85;
    server localhost:86;
}

server {
    listen 88;

    location / {
        set_by_lua_block $u_hash {
            return ngx.var.request_uri:match('^.*/')
        }

        proxy_pass http://u_x;
    }
}
```

## 验证步骤
+ 将配置范例加入到openresty的`conf/nginx.conf`文件中。
+ reload或重启openresty
+ 浏览器多次访问openresty的88端口，例如[http://localhost:88/](http://localhost:88/)，在path后面随意的加入内容，可以看见：
  + 不同的请求转发的端口可能不同
  + 相同的请求的转发端口永远是一个

范例里面是根据请求的uri来做hash分片的。

## 说明
+ 首先创建俩个模拟的服务url，分别监听端口85/86
+ 再创建一个`upstream`，注意`hash`指令，指向一个变量`$u_hash`
+ 在使用这个`upstream`之前，使用指令`set_by_lua_block`或者`set`来给上面的变量`$u_hash`赋值（使用`set_by_lua_block`的话，就可以做非常复杂的逻辑计算）。


## 扩展开发
上面的只是一个简单的范例，在这个思路上可以做几个实际的例子

**注意：新的范例里面，我们只要修改`set_by_lua_block`这部分即可**

### 根据url参数中的cityId把用户分配到不同机器上 
```conf
set_by_lua_block $u_hash {
    return ngx.var.arg_cityId or '010'
}
```

### 根据用户来源IP来分配
```conf
set_by_lua_block $u_hash {
    return ngx.var.remote_addr
}
```

这个的效果等同于nginx的指令 [ip_hash](http://nginx.org/en/docs/http/ngx_http_upstream_module.html#ip_hash)
```conf
upstream u_x {
    ip_hash;

    server localhost:85;
    server localhost:86;
}
```

### `session sticky`
`session stick` 的原理是：
+ nginx会根据请求的分片cookie信息来计算分片，分发给后端的服务器，
+ 如果没有这个cookie，nginx生成一个。

流程如下：
```
{% mermaid %}
graph TD
    agent[客户端]
    nginx[Nginx]
    appsvr[应用服务器]
    gen_cookie[生成分片Cookie]
    dispatch[根据cookie计算分片]

    agent-->|请求|nginx
    nginx-->|检查分片Cookie|has_cookie{有分片Cookie}
    has_cookie-->|N|gen_cookie
    has_cookie-->|Y|dispatch
    gen_cookie-->dispatch
    dispatch-->|分发|appsvr
    
{% endmermaid %}
```
范例中用来做分片cookie的name是`ngshard`
```conf
set_by_lua_block $u_hash {
    local v = ngx.var.cookie_ngshard;
    if not v then 
        v = ngx.now()
        ngx.header['Set-Cookie'] = 'ngshard='.. v .. '; path=/'
    end

    return v
}
```

用浏览器访问[http://localhost:88/](http://localhost:88/)，并且多次刷新，可以看到：
+ 访问一个固定的地址
+ http request中会增加一个cookie，类似于`ngshard=1546866932.493`

如果将这个cookie清除，然后再次访问，可以看到:
+ nginx会重新返回一个新的cookie，此时可能会变动访问的后端服务器。

