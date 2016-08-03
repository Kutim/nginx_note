# 将自己的 http 模块编译进 nginx #


	nginx提供一种简单的方式：所有源代码放到一个目录下，同时用 config文件通知 nginx 如何编译本模块。
	在configure 脚本执行时，加入参数 --add-module=PATH ，就可以加入自定义模块。

	在不能满足需求时，可以修改configure 脚本之后生成的 objs/Makefile 和 objs/ngx_modules.c 文件。

	
1. config 文件的写法
	最简单的形式（定义以下3个变量）：
	- ngx_addon_name: 仅在configure 执行时使用，一般设置为模块名称。
	- HTTP_MODULES: 保存所有HTTP 模块名称，每个HTTP 模块由空格符相连。切记，不要覆盖此变量
	- NGX_ADDON_SRCS: 指定新模块的源代码，多个值之间以空格符相连。可以使用 $ngx_addon_dir , 其值为 configure时 -add-module=PATH的 PATH值

	```
	ngx_addon_name=ngx_http_concat_module
	HTTP_MODULES="$HTTP_MODULES ngx_http_concat_module"
	NGX_ADDON_SRCS="$NGX_ADDON_SRCS $ngx_addon_dir/ngx_http_concat_module.c"

	```

2. 利用configure 脚本将定制的模块加入到 nginx 中
	```
	. auto/modules
	...
	.auto/make
	```

3. 可以直接修改Makefile 文件（慎用，可能导致nginx 不正常）
	
	在configure 之后，对生成的objs/ngx_modules.c 和 objs/Makefile 文件直接进行修改

	```
	//ngx_modules.c
	extern ngx_module_t ngx_http_XXX_module;		//第三方模块的声明
	
	ngx_module_t *ngx_moodules[]={					//合适的地方将 模块 加入 ngx_modules 数组
	...
	&ngx_http_XXX_module,,
	...
	}
	```

	修改Makefile 文件
	```
	//Makefile 增加编译代码的部分
	objs/addon/src/ngx_http_lua_worker.o:	$(ADDON_DEPS) \
		/root/lua-nginx-module/src/ngx_http_lua_worker.c
		$(CC) -c $(CFLAGS)  $(ALL_INCS) \
			-o objs/addon/src/ngx_http_lua_worker.o \
			/root/lua-nginx-module/src/ngx_http_lua_worker.c

	//Makefile 增加链接部分
	//参照 $(LINK) 前后其他模块
	```