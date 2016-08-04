# 发送响应 #
	
	Http 响应主要包括响应行、响应头部、包体三个部分。
	发送响应时需要执行发送 HTTP 头部（发送 HTTP 头部时也会发送响应行）、发送 HTTP 包体。


1. 发送 http 头部
	
	HTTP 框架提供的发送 HTTP 头部的方法：
	```
	// ngx_http.h
	ngx_int_t ngx_http_send_header(ngx_http_request_t *r);
	```
	
	- 设置响应中的HTTP 头部(ngx_http_request_t 中的 headers_out)
		只要指定 headers_out 中的成员，就可以在调用 ngx_http_send_header 时正确地把 HTTP 头部发出。
		
		```
		// ngx_http_request.h
		typedef struct {
    		ngx_list_t                        headers;					// 待发送的 HTTP 头部链表，与 headers_in 中的 headers 成员类似
			
    		ngx_uint_t                        status;					// 响应中的状态值
    		ngx_str_t                         status_line;				// 响应的状态行
			
			// 以下成员设置后， ngx_http_header_filter_module 过滤模块可以把它们加到发送的网络包中
    		ngx_table_elt_t                  *server;
    		ngx_table_elt_t                  *date;
    		ngx_table_elt_t                  *content_length;
    		ngx_table_elt_t                  *content_encoding;
    		ngx_table_elt_t                  *location;
    		ngx_table_elt_t                  *refresh;
    		ngx_table_elt_t                  *last_modified;
    		ngx_table_elt_t                  *content_range;
    		ngx_table_elt_t                  *accept_ranges;
    		ngx_table_elt_t                  *www_authenticate;
    		ngx_table_elt_t                  *expires;
    		ngx_table_elt_t                  *etag;
			
    		ngx_str_t                        *override_charset;
			
    		size_t                            content_type_len;		// 可以调用 ngx_http_set_content_type(r) 设置 Conten-Type 头部，这个方法会根据 URI 中的文件扩展名并对应着 mine.type 来设置 Conten-Type 值
    		ngx_str_t                         content_type;
    		ngx_str_t                         charset;
    		u_char                           *content_type_lowcase;
    		ngx_uint_t                        content_type_hash;
			
    		ngx_array_t                       cache_control;
			
    		off_t                             content_length_n;		// 这里指定过后，不用再次到 ngx_table_elt_t * content_length 中设置响应长度
    		off_t                             content_offset;
    		time_t                            date_time;
    		time_t                            last_modified_time;
		} ngx_http_headers_out_t;
		```
		添加自定义的 HTTP 头部时，使用 ngx_list_push (先返回地址，再设置值)
	
2. 将内存字符串作为包体发送
	
	调用 ngx_http_output_filter 方法即可向客户端发送 HTTP 响应包体：
	```
	// ngx_http_core_module.c
	// ngx_chain_t 是对 ngx_buf_t 缓冲区的封装，把内存中的内容放入这个结构中，就可以发送包体，由于 nginx 是全异步服务器，因此，尽量将ngx_buf_t 中 pos 指针指向从内从池里分配的内存
	ngx_int_t ngx_http_output_filter(ngx_http_request_t *r, ngx_chain_t *in){
    	ngx_int_t          rc;
    	ngx_connection_t  *c;
		
    	c = r->connection;
		
    	ngx_log_debug2(NGX_LOG_DEBUG_HTTP, c->log, 0,
                   "http output filter \"%V?%V\"", &r->uri, &r->args);

    	rc = ngx_http_top_body_filter(r, in);

    	if (rc == NGX_ERROR) {
        	/* NGX_ERROR may be returned by any filter */
        	c->error = 1;
    	}
		
    	return rc;
	}
	```
	
	宏 ngx_string 可以很方便的初始化 ngx_str_t, 例如，ngx_str_t type = ngx_string("text/plain")

3. 内存分配的函数
	
	ngx_palloc(ngx_pool_t * pool ,size_t size)
	ngx_pcalloc(ngx_pool_t * pool , size_t size)			// 申请到的内存块全部置为 0
	
	ngx_buf_t *b = ngx_create_temp_buf(r->pool,128)			// 包含了其他成员的设置

4. 将磁盘文件作为包体发送
	
	和发送内存中的数据具有相同的接口，关键在于 ngx_buf_t 缓冲区（in_file 标志位）
	```
	// ngx_core.h
	typedef struct ngx_file_s        ngx_file_t;
	
	// ngx_file.h
	struct ngx_file_s {
    	ngx_fd_t                   fd;					// 文件句柄描述符
    	ngx_str_t                  name;				// 文件名称
    	ngx_file_info_t            info;				// 文件大小等资源信息，实际就是 Linux 系统定义的 stat 结构
		
    	off_t                      offset;				// 当前处理的位置，一般不用设置，nginx 框架会根据当前发送状态设置
    	off_t                      sys_offset;			// 当前文件系统偏移量，一般不用设置，同样由 nginx 框架设置
		
    	ngx_log_t                 *log;					// 日志对象，相关的日志会输出到 log 指定的日志文件中
			
		#if (NGX_THREADS)
    		ngx_int_t                (*thread_handler)(ngx_thread_task_t *task,
      	    	                                     ngx_file_t *file);
    		void                      *thread_ctx;
    		ngx_thread_task_t         *thread_task;
		#endif
		
		#if (NGX_HAVE_FILE_AIO)
    		ngx_event_aio_t           *aio;
		#endif

    	unsigned                   valid_info:1;		// 目前未使用
    	unsigned                   directio:1;			// 与配置文件中的 directio 配置项相对应，在发送大文件时可以设为 1
	};
	```
	使用宏打开文件：
	```
	// 
	#define ngx_open_file(name, mode, create, access)	open((const char *) name, mode|create, access)
	```
	
	```
	ngx_buf_t * b;
	b=ngx_palloc(r->pool,sizeof(ngx_buf_t));

	u_char * filename = (u_char *)"/tmp/test.txt";
	b->in_file = 1;
	b->file = ngx_pcalloc(r->pool, sizeof(ngx_file_t));
	b->file->fd = ngx_open_file(filename, NGX_FILE_RDONLY | NGX_FILE_NONBLOCK, NGX_FILE_OPEN, 0);
	b->file->log = r->connection->log;
	b->file->name.data = filename;
	b->file->name.len = size(filename)-1
	if (b->file->fd <= 0){
		return NGX_HTTP_NOT_FOUND;
	}
	
	// ngx_files.h
	typedef struct stat              ngx_file_info_t;
	
	// ngx_files.h
	#define ngx_file_info(file, sb)  stat((const char *) file, sb)
	
	//因此，获取文件信息时可以先这样写
	if (ngx_file_info(filename,&b->file->info) == NGX_FILE_ERROR
		return NGX_HTTP_INTERNAL_SERVER_ERROR;
	}

	// 之后必须要设置 Content-length 头部
	r->headers_out.content_length_n = b->file->info.st_size;

	// 还需要设置 ngx_buf_t 缓冲区的 file_pos 和 file_last
	b->file_pos = 0;
	b->file_last = b->file->info.st_size;
	```

5. 清理文件句柄
	必须要求 HTTP 框架在响应发送完毕后关闭已经打开的文件句柄，否则，会出现句柄泄露问题。
	- 使用 ngx_pool_cleanup_t 结构体，并将 nginx 提供的 ngx_pool_cleanup_file 函数设置到 它的 handler 回调方法中
	- 其他方式（请求结束时回调各个 HTTP 模块的 cleanup 方法）
	
	```
	// ngx_palloc.h
	typedef struct ngx_pool_cleanup_s  ngx_pool_cleanup_t;
	struct ngx_pool_cleanup_s {
    	ngx_pool_cleanup_pt   handler;		// 执行实际清理资源工作的回调方法
    	void                 *data;			// 回调方法需要的参数
    	ngx_pool_cleanup_t   *next;			// 下一个 ngx_pool_cleanup_t 清理对象，如果没有，需置为 NULL
	};

	// ngx_palloc.c
	void ngx_pool_cleanup_file(void *data){
    	ngx_pool_cleanup_file_t  *c = data;
    	ngx_log_debug1(NGX_LOG_DEBUG_ALLOC, c->log, 0, "file cleanup: fd:%d",
                   c->fd);
		
    	if (ngx_close_file(c->fd) == NGX_FILE_ERROR) {
        	ngx_log_error(NGX_LOG_ALERT, c->log, ngx_errno,
                      ngx_close_file_n " \"%s\" failed", c->name);
    	}
	}

	typedef struct {
    	ngx_fd_t              fd;				// 文件句柄
    	u_char               *name;				// 文件名称
    	ngx_log_t            *log;				// 日志对象
	} ngx_pool_cleanup_file_t;

	```

	清理文件句柄的完整代码如下：
	```
	ngx_pool_cleanup_t * cln = ngx_pool_cleanup_add(r->pool,sizeof(ngx_pool_cleanup_file_t));
	if(cln == NULL)	{ // 分配失败
		return NGX_ERROR;
	}
	
	cln-> handler = ngx_pool_cleanup_file;
	ngx_pool_cleanup_file_t *clnf = cln->data;

	clnf-> fd = b->file->fd;
	clnf-> name = b->file->name.data;
	clnf-> log =r->pool->log;
	```

6. 多线程下载和断点续传
	RFC2616 规范中 range 协议
	nginx 支持range 协议
	http_range_header_filter 模块就是用来处理 HTTP 请求头部 range 部分的。

	对我们来说，只要在发送前设置 ngx_http_request_t 的成员 allow_ranges 变量为1即可。