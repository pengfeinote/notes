## nginx mirror介绍

流量复制工具一般分为两类，应用层复制工具和网络层复制工具。应用层复制工具一般使用简单，但是会一定程度挤占线上资源(连接数，内存等)；而基于网络栈的流量复制工具，直接从链路层抓包，对应用层影响较小，但实现也相对复杂。

常用的实时流量复制工具：nginx mirror, goreplay, tcpcopy

* nginx mirror: 应用层复制；支持流量放大；不支持录制回放；
* goreplay:应用层复制；支持流量放大；不支持录制回放；
* tcpcopy:网络层复制；支持流量放大；支持录制与回放；

### mirror用法

nginx mirror在nginx 1.13.4中引入，通过在后台创建原始请求的子请求来达到流量复制的目的，子请求的响应将被nginx server忽略。

**配置示例**

```java
location / {
    mirror /mirror;   --每一条mirror配置项对应用户的一个请求副本，可以通过配置都个mirror来达到"流量放大的效果"
    proxy_pass http://backend;   --主服务
}

location = /mirror {
    internal; --internal指该location只能被"内部"请求调用，外部请求访问将返回"404 (Not Found)"
    proxy_pass http://test_backend$request_uri; --mirror服务
}
```

**开启request body复制**

可以通过mirror\_request\_body off | on选项控制mirror是否复制请求体，该选项与其他nginx选项proxy\_request\_buffering、fastcgi\_request\_buffering、scgi\_request\_buffering和uwsgi\_request\_buffering冲突，一旦mirror\_request\_body设为on，请求体将被自动缓存。

```java
location / {
    mirror /mirror;
    mirror_request_body off; --关闭请求体复制
    proxy_pass http://backend;
}

location = /mirror {
    internal;
    proxy_pass http://log_backend;
    proxy_pass_request_body off;  --是否传递请求体
    proxy_set_header Content-Length ""; --Content-Length置空
    proxy_set_header X-Original-URI $request_uri; --指定源请求
}
```

**借助split_clients模块做部分引流**

mirror模块儿本身不支持部分引流的配置，可以通过split_clients模块实现部分引流。

```java
split_clients $remote_addr $mirror_backend {
    50% test_backend; -- 设置$mirror_backend变量的值
    *   "";	
}

...

location / {
    mirror /mirror;   
    proxy_pass http://backend;   
}

location = /mirror {
    internal; 
    if ($mirror_backend = "") {
            return 400; --值为空则直接返回
    }
    proxy_pass http://test_backend$request_uri; 
}

```

### 原理

nginx在主请求的PRECONTENT阶段创建【后台子请求】，然后在同一台虚拟机上将后台子请求转发至目标系统。

1. 判断当前请求非主请求，或mirror指令未配置，则不处理

```java
# ngx_http_mirror_handler

if (r != r->main) {
    return NGX_DECLINED;
}

mlcf = ngx_http_get_module_loc_conf(r, ngx_http_mirror_module);

if (mlcf->mirror == NULL) {
    return NGX_DECLINED;
}
```

2. 如果需要连同请求包体一起复制，那么在创建「后台子请求」之前，Nginx 需要接收完整请求包体。

	* ngx\_http\_read\_client\_request\_body读取请求体，它会开启一个新的异步流程。同时，nginx可以暂停主请求的处理流程，等到请求体接收完，通过ngx\_http\_mirror\_body\_handler恢复主请求的处理流程。
	* 如果不需要复制请求体，会通过ngx\_http\_mirror\_handler\_internal创建子请求，并恢复主请求流程。

```java
# ngx_http_mirror_handler

if (mlcf->request_body) {
    ...
    rc = ngx_http_read_client_request_body(r, ngx_http_mirror_body_handler);
    ...
    ngx_http_finalize_request(r, NGX_DONE);
    return NGX_DONE;
}

return ngx_http_mirror_handler_internal(r);
```

3. 请求体接收完成后，nginx调用函数ngx\_http\_mirror\_body\_handler创建子请求，并恢复主请求流程

	* 模块通过设置主请求的 r->preserve_body = 1 防止主请求处理完成后删除请求包体所在的临时文 件，避免还未完成的「后台子请求」无请求包体可用。
	* 函数 ngx\_http\_mirror\_handler 的返回值 NGX\_DONE，会让主请求再被再次调度时（由下面的函 数 ngx\_http\_core\_run\_phases 触发），仍旧从 PRECONTENT 阶段恢复处理流程。

```java
# ngx_http_mirror_body_handler

ctx->status = ngx_http_mirror_handler_internal(r);

r->preserve_body = 1;

r->write_event_handler = ngx_http_core_run_phases;
ngx_http_core_run_phases(r);
```

4. 创建子请求

	* 读取每个mirror配置，使用ngx\_http\_subrequest创建子请求
	* 设置 sr->header_only = 1;nginx一旦接收完响应包头就会关闭上下游链接

```java
# ngx_http_mirror_handler_internal

mlcf = ngx_http_get_module_loc_conf(r, ngx_http_mirror_module);

name = mlcf->mirror->elts;

for (i = 0; i < mlcf->mirror->nelts; i++) {
    if (ngx_http_subrequest(r, &name[i], &r->args, &sr, NULL,
                            NGX_HTTP_SUBREQUEST_BACKGROUND)
        != NGX_OK)
    {
        return NGX_HTTP_INTERNAL_SERVER_ERROR;
    }

    sr->header_only = 1;
    sr->method = r->method;
    sr->method_name = r->method_name;
}
```

### 注意事项

* nignx mirror在应用层进行流量复制，因此会不可避免的占用nginx资源；nginx创建的子请求会保持对主请求的引用，可能会造成主请求占用的内存不能被及时回收，会一定程度上影响nginx的吞吐率。
* nginx接收完子请求的响应头就会关闭连接，可能不会接收完整的响应数据，造成mirror服务的业务异常。
* 在流量、负载比较高的系统中，还需慎用nginx mirror
* 在做流量镜像时，需要评估镜像服务对其依赖的中间件（db, redis, mq等）和第三方服务产生的影响。可以通过mirror或mock降低对依赖服务的影响。