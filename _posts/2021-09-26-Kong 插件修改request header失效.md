---
layout:     post
title:      Kong 插件修改request header失效
subtitle:   记录一次 lua 赋值错误导致 Kong 插件失效
date:       2021-09-26
author:     JerAxxxxxxx
header-img: img/background/Veni_vidi_venice.jpg
catalog: true
tags:
- kong
---


### 插件背景

在使用 Kong 的过程中, 因业务需要, 开发了一个插件, 主要功能为从 redis 中获取 token, 修改请求的 request header. 将 OAuth2 所需的 token 加在 request header 的 `Authorization` 字段中

### 错误现象

在测试过程中, 发现有的请求会出现 token 不正确的情况, 通过排查记录的日志, 并没有找到明显的异常.

在 Postman 调用过程中，5 次可能只有 1、2 次调用成功，其他情况都是 token 不正确，直接返回 401 状态码。

在修改 header 的代码中, `handler.lua` 主要的逻辑代码如下

```lua
local BasePlugin = require "kong.plugins.base_plugin"

local get_headers = kong.request.get_headers
local set_headers = kong.service.request.set_headers
-- 省略其他包导入

function TokenHandler:access(conf)
    -- 省略从 redis 获取 token
    -- token_value 为从 redis 获取的 value
    local headers = get_headers()
    headers["authorization"] = "Bearer  " .. token_value
    set_headers(headers)
end
```

这一部分在测试过程中并没有发现异常，每次请求的 header 的值都被顺利更改。但在实际联调过程中，发现部分请求不生效，header 的值并没有改变。并且在 Kong 的请求容器日志中，并没有发现异常请求和正常请求大的区别，怀疑是 header 更改未生效。在确认修改和获取 header 时的 api 都为 Kong 原生的 api，感觉可能在写法上。



在抱着试一试的心态，将以上代码改成如下时，经过多次测试，问题消失。

```lua
function TokenHandler:access(conf)
    -- 省略从 redis 获取 token
    -- token_value 为从 redis 获取的 value
    local headers = get_headers()
    local wms_token = "Bearer  " .. token_value
	headers["authorization"] = wms_token
    set_headers(headers)
end
```



### 问题原因

问题根本原因时 lua 赋值时的语法问题。这里底层原因暂未了解。做一个记录避免下次遇到时无从下手。



----

<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/"><img alt="知识共享许可协议" style="border-width:0" src="https://i.creativecommons.org/l/by-sa/4.0/88x31.png" /></a><br />本作品采用<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">知识共享署名-相同方式共享 4.0 国际许可协议</a>进行许可。

