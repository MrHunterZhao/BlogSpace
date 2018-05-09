---
title: 通过Nginx内嵌Lua脚本实现资源访问权限控制
date: 2017-10-15 02:19:27
categories: Lua
tags:
    - Nginx
    - Lua
    - 权限
---
![access-control-by-lua](/images/post/2017/10/15/access_control_by_lua.jpg)

有些敏感的静态资源（比如身份证照片，企业执照等）是不能裸奔在互联网上的，被爬虫抓取后果比较严重，所以要加入一定的访问权限，满足某种规则后才可以访问。

最常用的做法是访问者发起请求时首先对资源名使用私钥进行MD5作为signature，再将signature作为参数传入，服务端接到请求后按照同样规则进行验签，signature匹配则视为合法请求，否则deny返回403。

<!-- more -->

```lua
--
-- Author: Hunter Zhao
-- Mail: zhaohevip@gmail.com
-- Date: 13/10/2017
-- Time: 10:25
-- Description：敏感资源访问签名认证
--

-- =============================================

-- the secret key
SECRET_KEY = "iamasecretkey"

--[[
  获取资源名
  e.g. 访问路径 /image/some_pic_name.jpg   返回 -> some_pic_name.jpg
]]
function getResourceName()

  local url = ngx.var.uri
  local len = string.len(url)
  local _url = string.reverse(url)
  local pos = string.find(_url, "/")
  -- start position index
  local start_pos = len - pos + 1
  -- get substring from the index
  local img_name = string.sub(url, start_pos + 1, len)

  return img_name
end


--[[
  验证签名合法性
]]
function validateSignature()

  local sign = ngx.var.arg_sign

  -- exit if signature is blank
  if sign == nil or sign == "" then
    ngx.exit(401)
    return false
  end

  -- get request resource name
  local img_name = getResourceName()
  -- sign the specified string via MD5 algorithm
  local token = ngx.md5(img_name..SECRET_KEY)

  -- if signature doesn't equal the token
  if sign ~= token then
    ngx.exit(403)
    return false
  end

  -- if everything is OK
  return true
end

-- =============================================

-- return the result of access interceptor
return validateSignature()

```

Nginx配置
```bash
server {
    listen 80 default;
    server_name localhost;    
    root  /path/to/root;

    location /image/ {
        default_type 'text/html';
        access_by_lua_file "/path/to/resource_access_control.lua"; #通过lua来控制访问权限
    }
}
```
