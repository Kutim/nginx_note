# http 模块的数据结构 #

1. 定义http模块
	ngx_module_t ngx_http_XXX_module;

	```
	// 其数据结构
	typedef struct ngx_module_s      ngx_module_t;
	...
	struct ngx_module_s {
    	ngx_uint_t            ctx_index;				// 表示当前模块在这类模块中的序号，由管理这类模块的一个nginx核心模块设置
    	ngx_uint_t            index;					// 当前模块在ngx_modules 数组中的序号。nginx 启动时根据ngx_modules 数组设置各模块的index值

    	char                 *name;

		// spare系列的保留变量，暂未使用
    	ngx_uint_t            spare0;
    	ngx_uint_t            spare1;
		
		ngx_uint_t            version;					//模块的版本，便于将来的扩展。默认为1
    	const char           *signature;
		
    	void                 *ctx;						//指向一类模块的上下文结构体，指向特定类型模块的公共接口。例如，http模块中，指向ngx_http_module_t 结构体
    	ngx_command_t        *commands;					// 处理nginx.conf 中的配置项
    	ngx_uint_t            type;						// 模块类型，与ctx指针是紧密相关的。（NGX_HTTP_MODULE、NGX_CORE_MODULE、NGX_CONF_MODULE、NGX_EVENT_MODULE、NGX_MAIL_MODULE,也可以自定义模块）
		
		//以下7个函数指针表示有7个执行点会分别调用这7个方法，如果不需要，设为NULL
    	ngx_int_t           (*init_master)(ngx_log_t *log);			// 字面上理解应当在 master 进程启动时回调，但目前为止，框架从来不会调用，因此，可以将其设为NULL
		
		ngx_int_t           (*init_module)(ngx_cycle_t *cycle);		// 在初始化所有模块时被调用。在m/w模式下，这个阶段将在启动worker 子进程前完成
		
    	ngx_int_t           (*init_process)(ngx_cycle_t *cycle);	// 在正常服务前被调用。在m/w 模式下，多个worker 子进程已产生，每个worker 进程的初始化过程会调用所有模块的init_process 函数
    	ngx_int_t           (*init_thread)(ngx_cycle_t *cycle);		// nginx 暂不支持多线程，可以设为NULL
    	void                (*exit_thread)(ngx_cycle_t *cycle);		// 同样设为NULL
    	void                (*exit_process)(ngx_cycle_t *cycle);	// 服务停止前调用，在m/w模式下，worker进程会在退出前调用它
		
    	void                (*exit_master)(ngx_cycle_t *cycle);		// 在master 进程退出前被调用
		
		// 保留字段，目前没有使用，可以使用 NGX_MODULE_V1_PADDING 宏来填充
    	uintptr_t             spare_hook0;
    	uintptr_t             spare_hook1;
    	uintptr_t             spare_hook2;
    	uintptr_t             spare_hook3;
    	uintptr_t             spare_hook4;
    	uintptr_t             spare_hook5;
    	uintptr_t             spare_hook6;
    	uintptr_t             spare_hook7;
	};

	```
	
	init_module、init_process、exit_process、exit_master,调用他们的是nginx 的框架代码。也就是与 http 框架无关。即使 没有开启 http{...} 配置项，这些方法仍然会被调用。因此，在开发 http 模块时都把它们设为 NULL。在nginx 不作为 Web 服务器时，不会执行 http 模块的任何代码。

	在定义http 模块时，最重要的是设置 ctx 和 commands 这两个成员。
	
	http 框架在读取、重载配置文件是定义了由 ngx_http_module_t 接口描述的 8个阶段：
	```
	//ngx_http_config.h
	typedef struct {
    	ngx_int_t   (*preconfiguration)(ngx_conf_t *cf);							// 解析配置文件前调用
    	ngx_int_t   (*postconfiguration)(ngx_conf_t *cf);							// 完成配置文件的解析后调用
		
    	void       *(*create_main_conf)(ngx_conf_t *cf);							// 存储 main 级别（http{...}）的配置项时，此方法创建存储全局配置项的结构体
    	char       *(*init_main_conf)(ngx_conf_t *cf, void *conf);					// 初始化 main 级别配置项
		
    	void       *(*create_srv_conf)(ngx_conf_t *cf);								// 存储 srv 级别 （server{...}）的配置项时，此方法创建 srv 级别配置项的结构体
    	char       *(*merge_srv_conf)(ngx_conf_t *cf, void *prev, void *conf);		// 合并 main 和 srv 级别下的同名配置项
		
    	void       *(*create_loc_conf)(ngx_conf_t *cf);								// 存储 loc 级别（location{...}）的配置项时，此方法创建 loc 级别配置项的结构体
    	char       *(*merge_loc_conf)(ngx_conf_t *cf, void *prev, void *conf);		// 合并 srv 和 loc 级别下的同名配置项 
	} ngx_http_module_t;
	```
	
	以上方法的实际顺序可能是这样（与 nginx.conf 配置项有关）：

	```
	create_main_conf
	create_srv_conf
	create_loc_conf
	preconfiguration
	init_main_conf
	merge_srv_conf
	merge_loc_conf
	postconfiguration
	```
	
	commands数组用于定义模块的配置文件参数，每一个元素都是 ngx_command_t 类型，数组结尾用 ngx_null_command 表示。在解析配置项时，遍历所有模块，每一个模块，遍历 commands 数组。
	
	```
	//ngx_core.h
	typedef struct ngx_command_s     ngx_command_t;

	// ngx_conf_file.h
	struct ngx_command_s {
		ngx_str_t             name;														// 配置项名称，
    	ngx_uint_t            type;														// 配置项的类型，指配置项出现的位置。例如，在 server{} 或 location{} 中，以及可以携带的参数个数
    	char               *(*set)(ngx_conf_t *cf, ngx_command_t *cmd, void *conf);		// 处理配置项参数的方法
    	ngx_uint_t            conf;														// 配置文件中的偏移量
    	ngx_uint_t            offset;													// 常用于使用预设的解析方法解析配置项，是一个优秀的设计。需要与 conf 配合使用
    	void                 *post;														// 配置项读取后的处理方法，必须是 ngx_conf_post_t 结构的指针
	};
	
	#define ngx_null_command  { ngx_null_string, 0, NULL, 0, 0, NULL }
	```

