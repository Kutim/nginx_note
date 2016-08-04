# 处理用户请求 #

	handler 成员的原型：
	typedef ngx_int_t (*ngx_http_handler_pt)(ngx_http_request_t *r);

1. 处理方法的返回值
	```
	// ngx_http_request.h
		#define NGX_HTTP_CONTINUE                  100
		#define NGX_HTTP_SWITCHING_PROTOCOLS       101
		#define NGX_HTTP_PROCESSING                102
		
		#define NGX_HTTP_OK                        200
		#define NGX_HTTP_CREATED                   201
		#define NGX_HTTP_ACCEPTED                  202
		#define NGX_HTTP_NO_CONTENT                204
		#define NGX_HTTP_PARTIAL_CONTENT           206
		
		#define NGX_HTTP_SPECIAL_RESPONSE          300
		#define NGX_HTTP_MOVED_PERMANENTLY         301
		#define NGX_HTTP_MOVED_TEMPORARILY         302
		#define NGX_HTTP_SEE_OTHER                 303
		#define NGX_HTTP_NOT_MODIFIED              304
		#define NGX_HTTP_TEMPORARY_REDIRECT        307
		
		#define NGX_HTTP_BAD_REQUEST               400
		#define NGX_HTTP_UNAUTHORIZED              401
		#define NGX_HTTP_FORBIDDEN                 403
		#define NGX_HTTP_NOT_FOUND                 404
		#define NGX_HTTP_NOT_ALLOWED               405
		#define NGX_HTTP_REQUEST_TIME_OUT          408
		#define NGX_HTTP_CONFLICT                  409
		#define NGX_HTTP_LENGTH_REQUIRED           411
		#define NGX_HTTP_PRECONDITION_FAILED       412
		#define NGX_HTTP_REQUEST_ENTITY_TOO_LARGE  413
		#define NGX_HTTP_REQUEST_URI_TOO_LARGE     414
		#define NGX_HTTP_UNSUPPORTED_MEDIA_TYPE    415
		#define NGX_HTTP_RANGE_NOT_SATISFIABLE     416
		#define NGX_HTTP_MISDIRECTED_REQUEST       421
		...
	```
	```
	// ngx_core.h
	#define  NGX_OK          0				// 表示成功。nginx将会继续执行该请求的后续动作
	#define  NGX_ERROR      -1				// 表示错误。这时会调用 ngx_http_terminate_request 终止请求。如果还有POST子请求，那么将会在执行完 POST 请求后再终止本次请求
	#define  NGX_AGAIN      -2
	#define  NGX_BUSY       -3
	#define  NGX_DONE       -4				// 到此为止，同时 http 框架将暂时不再继续执行这个请求的后续部分。（有点像 Linux 中中断的上下两个部分）
	#define  NGX_DECLINED   -5				// 继续在 NGX_HTTP_CONTENT_PHASE 阶段寻找下一个对该请求感兴趣的 HTTP 模块来再次处理这个请求
	#define  NGX_ABORT      -6
	```

2. 获取URI 和参数
	请求的所有信息都可以在传入的 ngx_http_request_t 类型参数 r 中取得。
	```
	// ngx_http.h
	typedef struct ngx_http_request_s     ngx_http_request_t;

	// ngx_http_request.h
	struct ngx_http_request_s {
    	uint32_t                          signature;         /* "HTTP" */
		
    	ngx_connection_t                 *connection;
		
    	void                            **ctx;
		void                            **main_conf;
    	void                            **srv_conf;
    	void                            **loc_conf;

    	ngx_http_event_handler_pt         read_event_handler;
    	ngx_http_event_handler_pt         write_event_handler;

		#if (NGX_HTTP_CACHE)
    		ngx_http_cache_t                 *cache;
		#endif

    	ngx_http_upstream_t              *upstream;
    	ngx_array_t                      *upstream_states;
                                         /* of ngx_http_upstream_state_t */
		
    	ngx_pool_t                       *pool;
    	ngx_buf_t                        *header_in;
		
    	ngx_http_headers_in_t             headers_in;
    	ngx_http_headers_out_t            headers_out;
		
    	ngx_http_request_body_t          *request_body;
		
    	time_t                            lingering_time;
    	time_t                            start_sec;
    	ngx_msec_t                        start_msec;
		
    	ngx_uint_t                        method;
    	ngx_uint_t                        http_version;
		
    	ngx_str_t                         request_line;
    	ngx_str_t                         uri;
    	ngx_str_t                         args;
    	ngx_str_t                         exten;
    	ngx_str_t                         unparsed_uri;
		
    	ngx_str_t                         method_name;
    	ngx_str_t                         http_protocol;
		
    	ngx_chain_t                      *out;
    	ngx_http_request_t               *main;
    	ngx_http_request_t               *parent;
    	ngx_http_postponed_request_t     *postponed;
    	ngx_http_post_subrequest_t       *post_subrequest;
    	ngx_http_posted_request_t        *posted_requests;
		
    	ngx_int_t                         phase_handler;
    	ngx_http_handler_pt               content_handler;
    	ngx_uint_t                        access_code;
		
    	ngx_http_variable_value_t        *variables;
		
		#if (NGX_PCRE)
			ngx_uint_t                        ncaptures;
    		int                              *captures;
    		u_char                           *captures_data;
		#endif
		
    	size_t                            limit_rate;
    	size_t                            limit_rate_after;
		
    	/* used to learn the Apache compatible response length without a header */
    	size_t                            header_size;
		
    	off_t                             request_length;
		
    	ngx_uint_t                        err_status;
		
    	ngx_http_connection_t            *http_connection;
		#if (NGX_HTTP_V2)
    		ngx_http_v2_stream_t             *stream;
		#endif
		
    	ngx_http_log_handler_pt           log_handler;
		
    	ngx_http_cleanup_t               *cleanup;
		
    	unsigned                          count:16;
    	unsigned                          subrequests:8;
    	unsigned                          blocked:8;
		
    	unsigned                          aio:1;
		
    	unsigned                          http_state:4;
		
    	/* URI with "/." and on Win32 with "//" */
    	unsigned                          complex_uri:1;
		
    	/* URI with "%" */
    	unsigned                          quoted_uri:1;
		
    	/* URI with "+" */
    	unsigned                          plus_in_uri:1;
		
    	/* URI with " " */
    	unsigned                          space_in_uri:1;
		
    	unsigned                          invalid_header:1;
		
    	unsigned                          add_uri_to_alias:1;
    	unsigned                          valid_location:1;
    	unsigned                          valid_unparsed_uri:1;
    	unsigned                          uri_changed:1;
    	unsigned                          uri_changes:4;
		
    	unsigned                          request_body_in_single_buf:1;
    	unsigned                          request_body_in_file_only:1;
    	unsigned                          request_body_in_persistent_file:1;
    	unsigned                          request_body_in_clean_file:1;
    	unsigned                          request_body_file_group_access:1;
    	unsigned                          request_body_file_log_level:3;
    	unsigned                          request_body_no_buffering:1;
		
    	unsigned                          subrequest_in_memory:1;
    	unsigned                          waited:1;
		
		#if (NGX_HTTP_CACHE)
    		unsigned                          cached:1;
		#endif
		
		#if (NGX_HTTP_GZIP)
    		unsigned                          gzip_tested:1;
    		unsigned                          gzip_ok:1;
    		unsigned                          gzip_vary:1;
		#endif
		
    	unsigned                          proxy:1;
    	unsigned                          bypass_cache:1;
    	unsigned                          no_cache:1;

    	/*
     	* instead of using the request context data in
     	* ngx_http_limit_conn_module and ngx_http_limit_req_module
     	* we use the single bits in the request structure
     	*/
    	unsigned                          limit_conn_set:1;
    	unsigned                          limit_req_set:1;

		#if 0
    		unsigned                          cacheable:1;
		#endif
		
    	unsigned                          pipeline:1;
    	unsigned                          chunked:1;
    	unsigned                          header_only:1;
    	unsigned                          keepalive:1;
    	unsigned                          lingering_close:1;
    	unsigned                          discard_body:1;
    	unsigned                          reading_body:1;
    	unsigned                          internal:1;
    	unsigned                          error_page:1;
    	unsigned                          filter_finalize:1;
    	unsigned                          post_action:1;
    	unsigned                          request_complete:1;
    	unsigned                          request_output:1;
    	unsigned                          header_sent:1;
    	unsigned                          expect_tested:1;
    	unsigned                          root_tested:1;
    	unsigned                          done:1;
    	unsigned                          logged:1;
		
    	unsigned                          buffered:4;
		
    	unsigned                          main_filter_need_in_memory:1;
    	unsigned                          filter_need_in_memory:1;
    	unsigned                          filter_need_temporary:1;
    	unsigned                          allow_ranges:1;
    	unsigned                          subrequest_ranges:1;
    	unsigned                          single_range:1;
    	unsigned                          disable_not_modified:1;
		
		#if (NGX_STAT_STUB)
    		unsigned                          stat_reading:1;
    		unsigned                          stat_writing:1;
		#endif

    	/* used to parse HTTP headers */

    	ngx_uint_t                        state;
		
    	ngx_uint_t                        header_hash;
    	ngx_uint_t                        lowcase_index;
    	u_char                            lowcase_header[NGX_HTTP_LC_HEADER_LEN];
		
    	u_char                           *header_name_start;
    	u_char                           *header_name_end;
    	u_char                           *header_start;
    	u_char                           *header_end;
		
    	/*
     	* a memory that can be reused after parsing a request line
     	* via ngx_http_ephemeral_t
     	*/
		
    	u_char                           *uri_start;
    	u_char                           *uri_end;
    	u_char                           *uri_ext;
    	u_char                           *args_start;
    	u_char                           *request_start;
    	u_char                           *request_end;
    	u_char                           *method_end;
    	u_char                           *schema_start;
    	u_char                           *schema_end;
    	u_char                           *host_start;
    	u_char                           *host_end;
    	u_char                           *port_start;
    	u_char                           *port_end;
		
    	unsigned                          http_minor:16;
    	unsigned                          http_major:16;
	};
	```
	
	- 方法名
		method:速度最快的方法是与 nginx 中定义的宏进行比较
		```
		// ngx_http_request.h
		#define NGX_HTTP_UNKNOWN                   0x0001
		#define NGX_HTTP_GET                       0x0002
		#define NGX_HTTP_HEAD                      0x0004
		#define NGX_HTTP_POST                      0x0008
		#define NGX_HTTP_PUT                       0x0010
		#define NGX_HTTP_DELETE                    0x0020
		#define NGX_HTTP_MKCOL                     0x0040
		#define NGX_HTTP_COPY                      0x0080
		#define NGX_HTTP_MOVE                      0x0100
		#define NGX_HTTP_OPTIONS                   0x0200
		#define NGX_HTTP_PROPFIND                  0x0400
		#define NGX_HTTP_PROPPATCH                 0x0800
		#define NGX_HTTP_LOCK                      0x1000
		#define NGX_HTTP_UNLOCK                    0x2000
		#define NGX_HTTP_PATCH                     0x4000
		#define NGX_HTTP_TRACE                     0x8000
		```
		- method_name: 取得用户请求中的方法名字符串。
		- request_start + method_end 获取方法名。（method_end 指向方法名的最后一个字符）
	- URI
		- uri 成员指向用户请求中的 URI。
		- uri_start + uri_end (uri_end 指向 URI 结束后的下一个地址，这种方式最常用)
		- extern （ngx_str_t 类型） 成员指向用户请求的文件扩展名。
		- uri_ext 指针指向的地址与 extern.data 相同
		- unparsed_uri 没有进行 URL 解码的原始请求
	- URL 参数
		- arg 指向用户请求中的 URL 参数
		- args_start 指向 URL 参数的起始地址，配合 uri_end 使用也可以获得 URL 参数
	- 协议版本
		- http_protocal 指向用户请求中 HTTP 的起始地址
		- http_version nginx 解析过的协议版本
	- 请求行： 使用 request_start 和 request_end 

3. 获取  http 头部
	
	- ngx_buf_t * header_in	: 指向 nginx 收到的未经解析的 http 头部
	- ngx_http_headers_in_t headers_in	： 存储已经解析过的 http 头部
		```
		// ngx_http_request.h
		typedef struct {
    		ngx_list_t                        headers;						// 所有解析过的 http 头部都在 headers 链表中，链表的每一个元素都是 ngx_table_elt_t 成员
			
			// 以下成员为 规范中定义的 http 头部，实际都指向 headers 链表中的相应成员。为 NULL 时，表示没有解析到相应的 HTTP 头部
    		ngx_table_elt_t                  *host;
    		ngx_table_elt_t                  *connection;
    		ngx_table_elt_t                  *if_modified_since;
    		ngx_table_elt_t                  *if_unmodified_since;
    		ngx_table_elt_t                  *if_match;
    		ngx_table_elt_t                  *if_none_match;
    		ngx_table_elt_t                  *user_agent;
    		ngx_table_elt_t                  *referer;
    		ngx_table_elt_t                  *content_length;
    		ngx_table_elt_t                  *content_range;
    		ngx_table_elt_t                  *content_type;
			
    		ngx_table_elt_t                  *range;
    		ngx_table_elt_t                  *if_range;
			
    		ngx_table_elt_t                  *transfer_encoding;
    		ngx_table_elt_t                  *expect;
    		ngx_table_elt_t                  *upgrade;
			
			#if (NGX_HTTP_GZIP)
    			ngx_table_elt_t                  *accept_encoding;
    			ngx_table_elt_t                  *via;
			#endif
			
    		ngx_table_elt_t                  *authorization;
			
    		ngx_table_elt_t                  *keep_alive;
			
			#if (NGX_HTTP_X_FORWARDED_FOR)
    			ngx_array_t                       x_forwarded_for;
			#endif
			
			#if (NGX_HTTP_REALIP)
    			ngx_table_elt_t                  *x_real_ip;
			#endif
			
			#if (NGX_HTTP_HEADERS)
    			ngx_table_elt_t                  *accept;
    			ngx_table_elt_t                  *accept_language;
			#endif
			
			#if (NGX_HTTP_DAV)
    			ngx_table_elt_t                  *depth;
    			ngx_table_elt_t                  *destination;
    			ngx_table_elt_t                  *overwrite;
    			ngx_table_elt_t                  *date;
			#endif
			
    		ngx_str_t                         user;
    		ngx_str_t                         passwd;
			
    		ngx_array_t                       cookies;
			
    		ngx_str_t                         server;
    		off_t                             content_length_n;				// 根据 ngx_tables_elt_t * content_length 计算出的 HTTP 包体大小
    		time_t                            keep_alive_n;
			
    		unsigned                          connection_type:2;			// HTTP 链接类型，取值范围：0、NGX_HTTP_CONNECTION_CLOSE 或者 NGX_HTTP_CONNECTION_KEEP_ALIVE

			// 以下 7 个标志位是 HTTP 框架根据浏览器传过来的 “useragent” 头部，判断浏览器的类型，为1 表示是相应的浏览器发送过来的
    		unsigned                          chunked:1;
    		unsigned                          msie:1;
    		unsigned                          msie6:1;
    		unsigned                          opera:1;
    		unsigned                          gecko:1;
    		unsigned                          chrome:1;
			unsigned                          safari:1;
    		unsigned                          konqueror:1;
		} ngx_http_headers_in_t;
		```
		对于常见的头部，直接获取 由 HTTP 框架解析过的成员即可，而对于不常见的 HTTP 头部，需要遍历 r->headsers_in.headers 链表才能获得。

4. 获取 HTTP 包体
	
	包体长度可能非常大，HTTP 框架提供了一种方法来异步地接受包体：
	```
	// ngx_http.h
	
	// 调用此方法只是说明要求 nginx 开始接收请求的包体，并不表示是否已经接受完，当包体接收完成后，post_handler 指向的回调方法会被调用
	ngx_int_t ngx_http_read_client_request_body(ngx_http_request_t *r, ngx_http_client_body_handler_pt post_handler);
	
	// 包体接收完毕后回调方法的原型
	// ngx_http_request.h
	typedef void (*ngx_http_client_body_handler_pt)(ngx_http_request_t *r);
	// 由于其返回类型是 void ,因此，在处理完时必须要主动调用 ngx_http_finalize_request 方法来结束请求
	```
	
	如果不想处理请求中的包体，可以调用 ngx_http_discard_request_body 方法将接受自客户端的 HTTP 包体丢弃掉