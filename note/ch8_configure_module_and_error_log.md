# http 配置模型 和 error 日志用法 #
	
	当nginx 检测到 http{...} 这个配置项时，HTTP配置模型就启动了，这时会首先建立一个 ngx_http_conf_ctx_t 结构

	```
	typedef struct{
		/* 指针数组，数组中的每个元素指向所有 HTTP 模块 create_main_conf 方法产生的结构体 */
		void ** main_conf;

		/* 指针数组，数组中的每个元素指向所有 HTTP 模块 create_srv_conf 方法产生的结构体 */
		void ** srv_conf;
		
		/* 指针数组，数组中的每个元素指向所有 HTTP 模块 create_loc_conf 方法产生的结构体 */
		void ** loc_conf;
	}ngx_http_conf_ctx_t;
	```
1. 解析 HTTP 配置的流程
	1. 发现配置文件中含有http{} 关键字时，HTTP 框架开始启动；
	2. HTTP框架初始化所有HTTP模块的序列号，并创建3个数组存储所有 HTTP 模块的 create_main_conf、create_srv_conf、create_loc_conf 方法返回的指针，并保存到ngx_http_conf_ctx_t 结构中；
	3. 调用每个HTTP 模块的 create_main_conf、create_srv_conf、create_loc_conf 方法，返回地址依次保存到 ngx_http_conf_ctx_t 结构体的3 个数组中；
	4. 调用每个 HTTP 模块的 preconfiguration 方法；
	5. 如果preconfiguration 返回失败，nginx 进程将会停止；
	6. HTTP 框架开始循环解析 nginx.conf 中的 http{...}里面的所有配置项；
	7. 检测到一个配置项时，遍历所有的 HTTP 模块，检查是否有此配置项；
	8. 如果有 HTTP 模块对此配置项感兴趣，调用 ngx_command_t 结构中的 set 方法来处理；如果处理失败，nginx 进程会停止
	9. 如果发现 server{...} 配置项，就会调用 ngx_http_core_module 模块来处理。因为 ngx_http_core_module 模块明确表示希望处理server{} 块下的配置项
	10. ngx_http_core_module 模块在解析 server{...}之前，也会建立 ngx_http_conf_ctx_t 结构 （srv、loc）

2. HTTP配置模型的内存布局

	每个http{...}、server{...}、location{...} 都有一个ngx_http_conf_ctx_t 结构

	nginx 设计优秀的地方：比如一个配置项只针对于 server 起作用，如果希望一个配置对多个 server 起作用，就可以写在 http{...}下面。

3. 合并配置项
	
	与紧挨的外层合并

4. 预设配置项处理方法的工作原理

	自定义的配置项处理方法读取参数：ngx_str_t * value=cf->args-elts ; 就可以获取参数，接下来把参数赋值到 ngx_http_mytest_conf_t 结构体的相应成员中。

	预设的配置项处理方法：由于 ngx_command_t 结构体的 offset 成员已经进行了正确的设置。nginx 配置项解析模块在调用ngx_command_t 结构体的set 回调方法时，会同时把 offset 偏移位置传进来。
	```
	char *ngx_conf_set_num_slot(ngx_conf_t *cf, ngx_command_t *cmd, void *conf){
    	char  *p = conf;						// 存储参数的结构体的地址

    	ngx_int_t        *np;
    	ngx_str_t        *value;
    	ngx_conf_post_t  *post;

    	np = (ngx_int_t *) (p + cmd->offset);	// 根据ngx_command_t 中的 offset 偏移量，找到结构体中的成员，对于此方法，其类型必为 ngx_int_t

    	if (*np != NGX_CONF_UNSET) {			// 这里说明了 为什么要把 ngx_conf_set_num_slot 方法解析的成员在create_loc_conf 等方法中初始化为 NGX_CONF_UNSET
        	return "is duplicate";
    	}

    	value = cf->args->elts;					// value将指向配置项的参数
    	*np = ngx_atoi(value[1].data, value[1].len);	//将字符串的参数转化为整形，
    	if (*np == NGX_ERROR) {
        	return "invalid number";
    	}

    	if (cmd->post) {				// 如果 ngx_command_t 中的 post 已经实现，还需要调用 post->post_handler 方法
        	post = cmd->post;
        	return post->post_handler(cf, post, np);
    	}

    	return NGX_CONF_OK;
	}
	```

5. error日志的用法
	
	错误日志模块(ngx_errlog_module)、访问日志模块（ngx_http_log_module）

	日志模块对于支持可变参数平台提供的3个接口：
	```
	// ngx_log.h
	#define ngx_log_error(level, log, args...)                                    \
    	if ((log)->log_level >= level) ngx_log_error_core(level, log, args)

	#define ngx_log_debug(level, log, args...)                                    \
    	if ((log)->log_level & level)                                             \
        	ngx_log_error_core(NGX_LOG_DEBUG, log, args)

	// ngx_log.c
	void ngx_log_error_core(ngx_uint_t level, ngx_log_t *log, ngx_err_t err,const char *fmt, ...)

	```
	
	1. level 参数：
	
	ngx_log_error日志的level参数取值范围：
	NGX_LOG_STDERR(0)、NGX_LOG_EMERG(1)、NGX_LOG_ALERT(2)、NGX_LOG_CRIT、NGX_LOG_ERR、NGX_LOG_WARN、NGX_LOG_NOTICE、NGX_LOG_INFO、NGX_LOG_DEBUG

	ngx_log_debug日志的level参数的取值范围：
	NGX_LOG_DEBUG_CORE、NGX_LOG_DEBUG_ALLOC、NGX_LOG_DEBUG_MUTEX、NGX_LOG_DEBUG_EVENT、NGX_LOG_DEBUG_HTTP、NGX_LOG_DEBUG_MAIL、NGX_LOG_DEBUG_MYSQL

	2. log 参数：
	
	在使用过程中，ngx_http_request_t结构中的connection 成员就有一个 ngx_log_t 类型的 log 成员。
	
	在读取配置阶段，ngx_conf_t 结构也有log 成员可以用来记录日志。
	
	3. err参数
	
	错误码，一般是执行系统调用失败后取得的errno 参数。

	4. fmt 参数
	
	可变参数，与printf 类似
	
	打印日志的 格式 可以详细参见 《nginx模块开发与架构解析》 中 p153

	nginx提供的不支持可变参数的调试日志接口：
	ngx_log_debug0
	ngx_log_debug1
	...
	ngx_log_debug8

6. 请求的上下文
	
	这里上下文的含义是指：HTTP框架为每个HTTP 请求所准备的结构体。

	一个 HTTP 请求对应与每一个 HTTP模块都可以有一个独立的上下文结构体。

	- 使用 HTTP 上下文。
		ngx_http_get_module_ctx 和 ngx_http_set_ctx 宏


