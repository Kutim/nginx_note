# 访问第三方服务 #
	nginx 提供两种全异步方式：upstream 和 subrequest

1. upstream 
	
	upstream 提供8个回调方法，用户视自己需要实现其中的方法。

	在ngx_http_request_t 结构体中有一个 ngx_http_upstream_t 类型的成员 upstream。
	
	- upstream 初始状态下为NULL，可以调用 HTTP 框架提供好的 ngx_http_upstream_create 方法创建 upstream；
	- 接着设置上游服务器地址。
	- 设置 upstream 的回调方法
	- 调用 ngx_http_upstream_init 方法启动 upstream
	- create_request
	- process_header
	- finalize_request

	1. ngx_http_upstream_t 结构体
		```
		struct ngx_http_upstream_s {
    		ngx_http_upstream_handler_pt     read_event_handler;
    		ngx_http_upstream_handler_pt     write_event_handler;

    		ngx_peer_connection_t            peer;

    		ngx_event_pipe_t                *pipe;

    		ngx_chain_t                     *request_bufs;			// 决定发送什么样的请求给上游服务器，在实现 create_request 方法时需要设置它

    		ngx_output_chain_ctx_t           output;
    		ngx_chain_writer_ctx_t           writer;

    		ngx_http_upstream_conf_t        *conf;					// upstream访问时的所有限制性参数
			#if (NGX_HTTP_CACHE)
    			ngx_array_t                     *caches;
			#endif

    		ngx_http_upstream_headers_in_t   headers_in;

    		ngx_http_upstream_resolved_t    *resolved;				// 通过 resolved 可以直接指定上游服务器地址

    		ngx_buf_t                        from_client;
			
			/*  buffer 存储接收自上游服务器发来的响应内容，可以复用：
				a) process_header 方法解析上游响应的包头时，buffer中保存完整的响应包头
				b) 当 buffering 成员为 1，此时 upstream 向下游转发上游包体时，buffer 没有意义
				c) 当 buffering 为0 ，buffer 缓冲区会被用于反复接收上游的包体，进而向下游转发
				d) 当upstream 并不用于转发上游包体时，buffer 反复用于接收上游包体， HTTP 模块实现的 input_filter 需要关注它
			*/
    		ngx_buf_t                        buffer;				
    		off_t                            length;

    		ngx_chain_t                     *out_bufs;
    		ngx_chain_t                     *busy_bufs;
    		ngx_chain_t                     *free_bufs;

    		ngx_int_t                      (*input_filter_init)(void *data);
    		ngx_int_t                      (*input_filter)(void *data, ssize_t bytes);
    		void                            *input_filter_ctx;

			#if (NGX_HTTP_CACHE)
    			ngx_int_t                      (*create_key)(ngx_http_request_t *r);
			#endif
    		ngx_int_t                      (*create_request)(ngx_http_request_t *r);			// 构造发往上游服务器的请求内容
    		ngx_int_t                      (*reinit_request)(ngx_http_request_t *r);
    		ngx_int_t                      (*process_header)(ngx_http_request_t *r);			// 收到上游服务器响应后回调。如果此函数返回 NGX_AGAIN，表示没有收到完整的响应报头
    		void                           (*abort_request)(ngx_http_request_t *r);
    		void                           (*finalize_request)(ngx_http_request_t *r,
                                         ngx_int_t rc);											// 销毁 upstream 请求时调用
    		ngx_int_t                      (*rewrite_redirect)(ngx_http_request_t *r,
                                         ngx_table_elt_t *h, size_t prefix);
    		ngx_int_t                      (*rewrite_cookie)(ngx_http_request_t *r,
                                         ngx_table_elt_t *h);
    		ngx_msec_t                       timeout;

    		ngx_http_upstream_state_t       *state;

    		ngx_str_t                        method;
    		ngx_str_t                        schema;
    		ngx_str_t                        uri;

			#if (NGX_HTTP_SSL)
    			ngx_str_t                        ssl_name;
			#endif

    		ngx_http_cleanup_pt             *cleanup;

			unsigned                         store:1;
    		unsigned                         cacheable:1;
    		unsigned                         accel:1;
    		unsigned                         ssl:1;					// 是否基于 SSL 协议访问上游服务器
			#if (NGX_HTTP_CACHE)
    			unsigned                         cache_status:3;
			#endif

    		unsigned                         buffering:1;			// 向客户端转发上游服务器的包体时有用。为1 时，表示使用多个缓冲区以及磁盘文件来转发上游的响应包体。为0时，表示只使用上面的这个buffer 缓冲区来向下游转发响应包体。
    		unsigned                         keepalive:1;
    		unsigned                         upgrade:1;

    		unsigned                         request_sent:1;
    		unsigned                         request_body_sent:1;
    		unsigned                         header_sent:1;
		};

		```
		当ngx_http_request_t 结构体中的 subrequest_in_memory 标志位为1 时：由 HTTP 模块实现的 input_filter 方法处理包体；
		当subrequest_in_memory 为0 时，upstream 会转发响应包体；
		当ngx_http_upstream_conf_t 配置结构体中的buffering 标志位为 1 时：将开启更多的内存和磁盘文件缓存上游的响应包体，以为上游更快；
		当buffering 为0 时，将使用固定大小的缓冲区来转发响应包体。
	2. upstream 的限制性参数
		
		ngx_http_upstream_conf_t 

		HTTP 反向代理模块在 nginx.conf 文件中提供的配置项大都是用来设置 ngx_http_upstream_conf_t 结构体中的成员的。
		
		```
		typedef struct{
			...
			ngx_msec_t		connect_timeout; 			// 连接上游服务器的超时时间，单位为毫秒
			ngx_msec_t		send_timeout;				// 发送 TCP 包到上游服务器的超时时间，单位为毫秒
			ngx_msec_t		read_timeout;				// 接收 TCP 包的超时时间，单位为毫秒
		}ngx_http_upstream_conf_t;
		```
		上面的三个参数，默认值为0，如果不设置，将无法与上游服务器建立 TCP 连接。

	3. 设置需要访问的第三方服务器地址
		
		ngx_http_upstream_t 结构中的 resolved 成员可以直接设置上游服务器的地址。
		```
		typedef struct{
			...
			ngx_uint_t	naddrs;			// 地址个数

			struct sockaddr	* sockaddr；	// 上游服务器的地址

			socklen_t		socklen;
		}ngx_http_upstream_resolved_t;
		```
2. 使用 upstream 的示例
	
	以访问 mytest 模块的 URL 参数作为 搜索引擎的关键字，用 upstream 方式访问 google，查询 URL 里的参数。

	例如 访问 URL 是 /test?lumia

	nginx.conf中可以这样配置 location

	location /test{
		mytest;
	}

	1. upstream 的各种配置参数
		
		每一个请求都会有独立的 ngx_http_upstream_conf_t 结构体，此处简化处理，所有请求共享一个 ngx_http_upstream_conf_t 结构体，因此，这里把它放到 ngx_http_mytest_conf_t 配置结构体中

		typedef struct｛
			ngx_http_upstream_conf_t upstream;
		｝ngx_http_mytest_conf_t

		在启动 upstream 前，先将 ngx_http_mytest_conf_t下的 upstream 成员赋给 r->upstream->conf 成员。


	2. 请求上下文
		
		在解析 HTTP 响应行时，可以使用 HTTP 框架 提供的 ngx_http_status_t 结构，把 ngx_http_status_t 结构放到上下文中，在process_header 解析响应行时使用

	3. 在 create_request 方法中构造请求
		
		模仿正常的搜索请求，在内存池中申请内存

	4. 在 process_header 方法中解析包头
	
		解析 HTTP 响应行 和 HTTP 头部

	5. 在 finalize_request 方法中释放资源
	
	6. 在 ngx_http_mytest_handler 方式中启动 upstream

3. subrequest 的使用方式
	
	使用 subrequest 的方式：
	- 在 nginx.conf 文件中配置好子请求的处理方式
	- 启动 subrequest 子请求				// 调用 ngx_http_subrequest 方法建立 subrequest 子请求
	- 实现子请求执行结束时的回调方法		// nginx在子请求正常 或者 异常结束时，都会调用 ngx_http_post_subrequest_pt 回调方法
	- 实现父请求被激活时的回调方法		// 在ngx_http_post_subrequest_pt 回调方法内必须设置父请求激活后的处理方法，pr->write_event_handler= ngx_http_event_handler_pt 的实现







