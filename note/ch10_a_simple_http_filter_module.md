# 一个简单的 http过滤模块 #

	```
	// ngx_http_core_module.h
	typedef enum {
    	NGX_HTTP_POST_READ_PHASE = 0,

    	NGX_HTTP_SERVER_REWRITE_PHASE,

    	NGX_HTTP_FIND_CONFIG_PHASE,
    	NGX_HTTP_REWRITE_PHASE,
    	NGX_HTTP_POST_REWRITE_PHASE,

    	NGX_HTTP_PREACCESS_PHASE,

    	NGX_HTTP_ACCESS_PHASE,
    	NGX_HTTP_POST_ACCESS_PHASE,

    	NGX_HTTP_TRY_FILES_PHASE,
    	NGX_HTTP_CONTENT_PHASE,

    	NGX_HTTP_LOG_PHASE
	} ngx_http_phases;
	
	```

	如果希望多个 HTTP 模块共同处理一个请求，则多半由 subrequest 功能来完成，即将原始请求分为多个子请求，每个子请求再由一个 HTTP 模块在 NGX_HTTP_CONTENT_PHASE 阶段处理。

	过滤模块仅处理服务器发往客户端的 HTTP 响应。

	在编译 nginx 源代码时，已经定义了一个由所有 HTTP 过滤模块组成的单链表:链表的每一个元素都是一个独立的 C 源代码文件，这个 C 源码文件会通过两个static 静态指针（分别用于处理 HTTP 头部 和 HTTP 包体）再指向下一个文件中的过滤方法。在 HTTP 框架中定义了两个指针，指向整个链表的第一个元素。

	typedef ngx_int_t (*ngx_http_output_header_filter_pt)(ngx_http_request_t *r);
	typedef ngx_int_t (*ngx_http_output_body_filter_pt)(ngx_http_request_t *r, ngx_chain_t *chain);