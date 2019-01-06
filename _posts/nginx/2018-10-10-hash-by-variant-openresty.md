---

title: "在nginx/openresty中利用变量进行转发分片"

tags: Nginx Openresty
---

## 代码范例

```conf

server {
    listen 85;

    location / {
        content_by_lua_block {
            ngx.say(85 .. ' ' .. ngx.var.uri)
        }
    }
}

server {
    listen 86;

    location / {
        content_by_lua_block {
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
  + 不同的请求不一定转发给同一个端口
  + 相同的请求的转发端口永远是一个

范例里面是根据请求的uri来做hash分片的。

## 说明
+ 首先创建俩个模拟的服务url，分别监听端口85/86
+ 再创建一个`upstream`，注意`hash`指令，指向一个变量`$u_hash`
+ 在使用这个`upstream`之前，使用指令`set_by_lua_block`或者`set`来给上面的变量`$u_hash`赋值（使用`set_by_lua_block`的话，就可以做非常复杂的逻辑计算）。
