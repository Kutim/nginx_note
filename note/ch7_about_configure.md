# 配置 #
	nginx已经为用户提供了强大的配置项解析机制。
	```
	http{
		test_str main;

		server {
			listen 80;
			test_str server80;

			location /url1{
				mytest;
				test_str loc1
			}

			location /url2{
				mytest;
				test_str loc2;
			}
		}

		server {
			listen 8080;
			test_str server8080;
			location /url3{
				mytest;
				test_str loc3;
			}
		}
	}
	```
	同时 location 也可以相互嵌套。

	nginx 的每一个 http 块、server 块、location 块都会生成独立的数据结构来存放配置项。


1. 使用配置
	处理 http 配置项的步骤：
	- 创建数据结构用于存储配置项的参数
	- 设定配置项在 nginx.conf 中出现时的限制条件与回调方法
	- 实现上步中的回调方法，或者使用 nginx 框架预设的 14 个回调方法
	- 合并不同级别的配置块中出现的同名配置项
	
	通过 ngx_http_module_t 和 ngx_command_t 

	- 保存配置参数的数据结构
		创建一个结构体，其中包含我们感兴趣的参数。（以下说明14种预设配置项的解析方法）
		```
		typedef struct{
			ngx_str_t		my_str;
			ngx_int_t		my_num;
			ngx_flag_t		my_flag;
			size_t			my_size;
			ngx_array_t*	my_str_array;
			ngx_array_t*	my_keyval;
			off_t			my_off;
			ngx_msec_t		my_msec;
			time_t			my_sec;
			ngx_bufs_t		my_bufs;
			ngx_uint_t		my_enum_seq;
			ngx_uint_t		my_bitmask;
			ngx_uint_t		my_access;
			ngx_path_t*		my_path;
		} ngx_http_mytest_conf_t;				// 使用结构体保存是因为相同的配置项是允许同时生效的
		```
		nginx 通过 ngx_http_module_t 中的回调方法管理 自定义的存储配置结构体 ngx_http_mytest_conf_t 。

		一个实现 create_loc_conf 的 ngx_http_mytest_create_loc_conf 方法：
		```
		static void * ngx_http_mytest_create_loc_conf(ngx_conf_t *cf){
		
			ngx_http_mytest_conf_t * mycf;
			
			mycf = (ngx_http_mytest_conf_t *)ngx_pcalloc(cf->pool, size(ngx_http_mytest_conf_t));
			if (mycf == NULL){
				return NULL;
			}
			mycf->test_flag = NGX_CONF_UNSET;
			mycf->test_num = NGX_CONF_UNSET;
			mycf->test_str_array = NGX_CONF_UNSET_PTR;
			mycf->test_keyval = NULL;
			mycf->test_off = NGX_CONF_UNSET;
			mycf->test_msec = NGX_CONF_UNSET_MSEC;
			mycf->test_sec = NGX_CONF_UNSET;
			mycf->test_size = NGX_CONF_UNSET_SIZE;
			
			return mycf;
		}
		```
	- 设定配置项的解析方式
		ngx_command_s 结构体
		```
		struct ngx_command_s {
    		ngx_str_t             name;
    		ngx_uint_t            type;
    		char               *(*set)(ngx_conf_t *cf, ngx_command_t *cmd, void *conf);
    		ngx_uint_t            conf;
    		ngx_uint_t            offset;
    		void                 *post;
		};
		```
		type 可以同时取多个值，各值之间用 | 符号连接

		type 类型：
		- 处理配置项时获取当前配置块的方法：NGX_DIRECT_CONF、NGX_ANY_CONF
		- 配置项可以在那些{}配置块中出现: NGX_MAIN_CONF、NGX_EVENT_CONF、NGX_MAIL_MAIN_CONF、NGX_MAIL_SRV_CONF、NGX_HTTP_MAIN_CONF、NGX_HTTP_SRV_CONF、NGX_HTTP_LOC_CONF、NGX_HTTP_UPS_CONF、NGX_HTTP_SIF_CONF、NGX_HTTP_LIF_CONF、NGX_HTTP_LMT_CONF
		- 限制配置项的参数个数：NGX_CONF_NOARGS、NGX_CONF_TAKE1、NGX_CONF_TAKE2、...、NGX_CONF_TAKE7、NGX_CONF_TAKE12、NGX_CONF_TAKE13、NGX_CONF_TAKE23、NGX_CONF_TAKE123、NGX_CONF_TAKE1234
		- 限制配置项后的参数出现的形式：NGX_CONF_ARGS_NUMBER、NGX_CONF_BLOCK、NGX_CONF_ANY、NGX_CONF_FLAG、NGX_CONF_1MORE、NGX_CONF_2MORE、NGX_CONF_MULTI
		
		可以自定义方法来处理配置项，也可以使用预设的14个配置项解析方法：
		- ngx_conf_set_flag_slot
		- ngx_conf_set_str_slot
		- ngx_conf_set_str_array_slot
		- ngx_conf_set_keyval_slot
		- ngx_conf_set_num_slot
		- ngx_conf_set_size_slot
		- ngx_conf_set_off_slot
		- ngx_conf_set_msec_slot(毫秒)
		- ngx_conf_set_sec_slot (秒)
		- ngx_conf_set_bufs_slot (gzip_buffers 4 8k)
		- ngx_conf_set_enum_slot
		- ngx_conf_set_bitmask_slot
		- ngx_conf_set_access_slot
		- ngx_conf_set_path_slot
	
		ngx_uint_t conf:
		- 指示配置项所处内存的相对偏移位置，仅在type中没有设置 NGX_DIRECT_CONF 和 NGX_MAIN_CONF 时才会生效。对于 HTTP 模块，conf是必须要设置的
		- conf 在 HTTP 模块中的取值：NGX_HTTP_MAIN_CONF_OFFSET、NGX_HTTP_SRV_CONF_OFFSET、NGX_HTTP_LOC_CONF_OFFSET
		- 对于conf 的设置是与 ngx_http_module_t 实现的回调方法相关的。

		ngx_uint_t_offset:
		- 表示当前配置项在整个存储配置项的结构体中的偏移位置        offsetof(test_stru,b) //返回 b 在结构体test_stru 中的偏移
	
		void * post
		- 如果自定义了配置项的回调方法，那么 post 指针的用途完全由用户定义。如果不使用，随意设为 NULL 即可。
		- 取值 与预设配置项解析方法有关 （ngx_conf_post_t,ngx_conf_enum_t,ngx_conf_bitmask_t）

2. 自定义配置项处理方法
	
	假如配置项名称：test_config,接收1个或者2个参数，且第1个参数类型是字符串，第2个参数必须是整型，使用结构体来存储这两个参数：
	
	```
	typedef struct{
		ngx_str_t my_config_str;
		ngx_int_t my_config_num;
	}ngx_http_mytest_conf_t;
	```
	
	1. 按照 ngx_command_s 中set 方法指针格式来定义 这个配置项处理方法
	
		```
		static char* ngx_conf_set_myconfig(ngx_conf_t *cf, ngx_command_t *cmd, void *conf);
		```

	2. 定义 ngx_command_t 结构体
		```
		static ngx_command_t ngx_http_mytest_commands[]={
		...
			{ ngx_string("test_myconfig"),
				NGX_HTTP_LOC_CONF | NGX_CONF_TAKE12,
				ngx_conf_set_myconfig,
				NGX_HTTP_LOC_CONF_OFFSET,
				0,
				NULL
			},
			ngx_bull_command
		}
		```
	3. 实现 ngx_conf_set_myconfig 处理方法
	
		```
		static char* ngx_conf_set_myconfig(ngx_conf_t *cf, ngx_command_t *cmd, void *conf){
			// conf 为 HTTP 框架传给用户的在 ngx_http_mytest_create_loc_conf 回调方法中分配的结构体 ngx_http_mytest_conf_t
			ngx_http_mytest_conf_t *myconf=conf;

			// cf->args 是1个 ngx_array_t 队列，它的成员都是 ngx_str_t 结构。用value 指向 ngx_array_t 的 elts 内容，其中value[1]为第1个参数，value[2]为第2个参数
			ngx_str_t * value=cf->args->elts;

			//ngx_array_t 的nelts 表示参数的个数
			if(cf->args->nelts >1){
				// 直接赋值即可，ngx_str_t结构只是指针的传递
				mycf->myconfig_str = value[1];
			}
			if(cf->args->nelts >2){
				// 将字符串形式的第二个参数转为整型
				mycf->my_config_num=ngx_atoi(value[2].data,value[2].len);
				if(mycf->my_config_num==NGX_ERROR){
					return "invalid number";
				}
			}
			return NGX_CONF_OK;
		}                                                                                                                                                                                                                                                                              
		```
3. 合并配置项
	
	ngx_http_module_t 结构 8个 回调方法

	```
	/**
		* @param cf 提供一些基本的数据结构
		* @param prev 解析父配置块时生成的结构体
		* @param conf 保存子配置块的结构体
		* /
	merge_loc_conf(ngx_conf_t *cf , void *prev, void *conf)
	
	ngx_http_mytest_merge_loc_conf(ngx_conf_t *cf ,void * parent, void *child){
		ngx_http_mytest_conf_t *prev=(ngx_http_mytest_conf_t *)parent;
		ngx_http_mytest_conf_t *conf=(ngx_http_mytest_conf_t *)child;

		ngx_conf_merge_str_value(conf->my_str,prev->my_str,"defaultstr");
		return NGX_CONF_OK;
	}
	```
