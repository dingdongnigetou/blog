# why request time in access.log always 0.000

## Question
有时候我们会发现nginx的access.log上面会有很多request_time为0.000的日志, 速度真的有如此的快么？

我做了一个测试，用curl去请求在中国去请求新加坡nginx上的一个40KB的文件，本地看到的时间消耗大概是2s左右（不包含连接时间），
但是服务器上的日志依旧是0.000，反复测试都是如此，难道nginx有bug？？

## Search
在[openresty的谷歌讨论组](https://groups.google.com/g/openresty/c/t3PXLGmZR00)找到一点答案，
据说关闭keepalive并且开启linger_close就能统计到精确的时间。

我做了下尝试，设置`keepalive_time 0;linger_close on;`,但发现依旧是0.000。

查看nginx的文档，发现linger_close有三个选项，其中`always`貌似比较符合，它的定义是
`
The value “always” will cause nginx to unconditionally wait for and process additional client data.
`, 也就是无条件的等待，而`on`则是有一定条件的。

最后的设置是`keepalive_time 0;linger_close always;`生效了，日志显示出了真实的时间。

但为什么会这样呢？

## More
我们找到了关闭连接的一段代码(src/http/ngx_http_request.c=>ngx_http_finalize_connection)
```c
    if (!ngx_terminate
         && !ngx_exiting
         && r->keepalive
         && clcf->keepalive_timeout > 0)
    {
        ngx_http_set_keepalive(r);
        return;
    }

    if (clcf->lingering_close == NGX_HTTP_LINGERING_ALWAYS
        || (clcf->lingering_close == NGX_HTTP_LINGERING_ON
            && (r->lingering_close
                || r->header_in->pos < r->header_in->last
                || r->connection->read->ready)))
    {
        ngx_http_set_lingering_close(r->connection);
        return;
    }
```
这就跟我们的配置对应上了，如果设置了keepalive，将会进入keepalive的处理逻辑，只有关闭keepalive以及打开linger_close时才有机会进入
到linger的处理逻辑，值得注意的是如果是`on`，那么会判断是否还有client-body还没读完，没读完才会进入linger逻辑。

所以默认情况下，没有触发linger逻辑，nginx就会直接close不等待，立即记录日志，忽略了数据发送到client的网络时间，意味着request_time此时
仅仅是nginx的内部处理时间，只有触发linger逻辑，nginx才会等待数据的发送完毕，此时才是完整的耗时。

## 关于linger
那么linger_close其实就是延时关闭，属于tcp的底层算法（阻塞），但是nginx为了不阻塞自己实现了一套，如果打开这个选项，那么服务端在
关闭连接时，会等待发送缓冲区的数据全部发到客户端才会关闭连接。

当然这里是有个超时时间的，超过时会无条件关闭并且发送RST给到客户端，
所以有人也会利用这个特性，将超时设置为0，也就是立即发送RST给客户端断开连接而不走正常的四次挥手，这样的好处就是避免了服务端的TIME_WAIT产生，
缺点当然就是对客户端不友好不兼容比如浏览器这种。




