---
layout:     post
title:      Kong http-log 插件记录未发送到 http server 的日志
subtitle:   Kong 插件的修改
date:       2021-09-01
author:     JerAxxxxxxx
header-img: img/background/computers_electronics_macro_chips.jpg
catalog: true
tags:
- kong
---



### 背景

在 Kong 的插件中, http-log/tcp-log 以及自己开发的 http-log-with-body 都可以记录日志, 使用方法也大同小异

通过配置 http 的 endpoint 或者 tcp server , 用来接收数据

当 http-server 配置错误, 或者 http-server 服务不健康时, 日志就会丢失, 这时就需要我们在 plugin 阶段做一些修改.

### Http-log 插件

在配置好 http 地址后, 每个请求的日志都会发送到指定好的 url 中.

这一行为发生在插件的 `log` 阶段, 插件会创建一个队列用于存放要发送的日志, `send_payload` 就会向配置好的 url 发送日志, 以下代码在官方 **http-log** 的 `handler.lua` 中

```
  local queue_id = get_queue_id(conf)
  -- 创建一个队列
  local q = queues[queue_id]
  if not q then
    -- batch_max_size <==> conf.queue_size
    local batch_max_size = conf.queue_size or 1
    local process = function(entries)
      local payload = batch_max_size == 1
                      and entries[1]
                      or  json_array_concat(entries)
      -- payload 即是日志主体
      return send_payload(self, conf, payload)
    end
    -- 省略
  end
```

通过更改 `send_payload` , 当请求失败时记录下日志主体即可,以防丢失.

```
local response_body = res:read_body()
  local success = res.status < 400
  local err_msg

  if not success then
    print(payload)
    err_msg = "request to " .. host .. ":" .. tostring(port) ..
              " returned status code " .. tostring(res.status) .. " and body " ..
              response_body
  end
```

这里的 `print` 方法会将日志主体打印到容器日志中,即可查看丢失的日志.

这里需要注意的是:

**Kong 的 http-log 插件, 如果配置好的 url 请求失败, 会在队列中重试十次, 若重试十次后, 会从队列丢弃.**

```
2021/09/01 06:09:57 [error] 23#0: *3411 [lua] batch_queue.lua:187: entry batch was already tried 10 times, dropping it, context: ngx.timer, client: 127.0.0.1, server: 0.0.0.0:8000
```


----

<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/"><img alt="知识共享许可协议" style="border-width:0" src="https://i.creativecommons.org/l/by-sa/4.0/88x31.png" /></a><br />本作品采用<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">知识共享署名-相同方式共享 4.0 国际许可协议</a>进行许可。

