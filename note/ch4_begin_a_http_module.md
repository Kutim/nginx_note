# 开始自定义一个 http 模块 #

	明确自定义的模块如何介入 nginx 处理用户请求的流程中。

	此处以 alibaba 的 nginx-http-concat 为例

	https://github.com/alibaba/nginx-http-concat/blob/master/ngx_http_concat_module.c

1. 定义 配置项的处理
	
	```
	static ngx_command_t  ngx_http_concat_commands[] = {

    { ngx_string("concat"),														// 名称
      NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_CONF_FLAG,		// 可以出现的位置，及参数个数
      ngx_conf_set_flag_slot,													// 处理配置项参数的方法	此处在 ngx_conf_file.c 中，https://github.com/nginx/nginx/blob/master/src/core/ngx_conf_file.c
      NGX_HTTP_LOC_CONF_OFFSET,													// 配置文件中的偏移量
      offsetof(ngx_http_concat_loc_conf_t, enable),								// 常用于使用预设的解析方法解析配置项，是一个优秀的设计。需要与 conf 配合使用
      NULL 																		// 配置项读取后的处理方法，必须是 ngx_conf_post_t 结构的指针
	},																	
	...
    { ngx_string("concat_types"),
      NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_CONF_1MORE,
      ngx_http_types_slot,
      NGX_HTTP_LOC_CONF_OFFSET,
      offsetof(ngx_http_concat_loc_conf_t, types_keys),
      &ngx_http_concat_default_types[0] },
	...
      ngx_null_command
};
	```

2. 配置项的处理方式（此处为自定义的）
	
	```
	static char * ngx_http_mytest(ngx_conf_t *cf, ngx_command_t *cmd,void *conf){
		ngx_http_core_loc_conf_t *clcf;
		
		/* 首先找到 配置项所属的配置块，clcf 看上去像是 location 块内的数据结构，其实不然，它可以是 main、srv 或者 loc 级别配置项，
			即在每个http{} 和 server{} 内也都有一个 ngx_http_core_loc_conf_t 结构体 */
		clcf = ngx_http_conf_get_module_loc_conf(cf, ngx_http_core_module);

		/* http 框架处理用户请求进行到 NGX_HTTP_CONTENT_PHASE 阶段时，如果请求的主机名、URI 与配置项所在的配置块相匹配，
			就将调用我们实现的 ngx_http_XXX_handler 方法处理这个请求 */
		clcf-hander = ngx_http_XXX_handler;
		return NGX_CONF_OK;
		}
	```

	```
	typedef enum {
    	NGX_HTTP_POST_READ_PHASE = 0,			// 接收到完整的 http 头部后处理的 http 阶段
		
    	NGX_HTTP_SERVER_REWRITE_PHASE,			// 还没有查询到 URI 匹配的 location 前，这时 rewrite 重写 URL 也作为一个独立的 http 阶段
		
    	NGX_HTTP_FIND_CONFIG_PHASE,				// 由 URI 寻找匹配的 location，通常由 ngx_http_core_module 模块实现，不建议重新定义
    	NGX_HTTP_REWRITE_PHASE,					// 重写 URL 
    	NGX_HTTP_POST_REWRITE_PHASE,			// 重写 URL 后重新调到 NGX_HTTP_FIND_CONFIG_PHASE，由 ngx_http_core_module 模块使用
		
    	NGX_HTTP_PREACCESS_PHASE,				// 处理 NGX_HTTP_ACCESS_PHASE 阶段前，http 模块可以介入的处理阶段
		
    	NGX_HTTP_ACCESS_PHASE,					// 这个阶段用于让 http 模块判断是否允许这个请求访问 nginx 服务器
    	NGX_HTTP_POST_ACCESS_PHASE,				// 当 NGX_HTTP_ACCESS_PHASE 拒绝时，这个阶段构造拒绝服务的用户响应
		
    	NGX_HTTP_TRY_FILES_PHASE,				// 为 try_files 配置项而设立，
    	NGX_HTTP_CONTENT_PHASE,					// 处理 http 请求内容的阶段
		
		NGX_HTTP_LOG_PHASE						// 处理完请求后记录日志阶段
	} ngx_http_phases;							？？？？？？？？？ 如何指定 介入的阶段 ^w^
	```

3. ngx_http_module_t 接口

	```
	static ngx_http_module_t  ngx_http_concat_module_ctx = {
    	NULL,                                /* preconfiguration */
    	ngx_http_concat_init,                /* postconfiguration */
	
    	NULL,                                /* create main configuration */
    	NULL,                                /* init main configuration */
	
    	NULL,                                /* create server configuration */
    	NULL,                                /* merge server configuration */
	
    	ngx_http_concat_create_loc_conf,     /* create location configuration */
    	ngx_http_concat_merge_loc_conf       /* merge location configuration */
	};	
	```

4. 模块定义
	```
	ngx_module_t  ngx_http_concat_module = {
    		NGX_MODULE_V1,
    		&ngx_http_concat_module_ctx,         /* module context */
    		ngx_http_concat_commands,            /* module directives */
    		NGX_HTTP_MODULE,                     /* module type */
			NULL,                                /* init master */
    		NULL,                                /* init module */
    		NULL,                                /* init process */
    		NULL,                                /* init thread */
    		NULL,                                /* exit thread */
			NULL,                                /* exit process */
    		NULL,                                /* exit master */
    		NGX_MODULE_V1_PADDING
	};

	```