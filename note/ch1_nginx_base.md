# nginx http 模块基础 #

1. 调用方式
	- 典型的调用方式：http框架接收完http请求头后，将请求的URI与配置文件中的所有 location 进行匹配，匹配后在根据其内的配置项选择http 模块来调用。
	- 非典型的调用方式：如ngx_http_access_module模块，根据ip地址决定某个客户端是否可以访问服务，他可以对所有请求产生作用。
2. 基本结构
	约定俗称的模块名称： ngx_http_名称_module

	基本数据结构： 

	- 整型的封装：
	
	`ngx_int_t 封装有符号整型，ngx_uint_t 封装无符号整型；`

	- ngx_str_t 数据结构：
	
	```
	typedef struct {
	    size_t      len;  //字符串的有效长度
	    u_char     *data;	//字符串起始地址，字符串未必会以'\0'结束，因此使用len 表示字符串的长度
	} ngx_str_t;
	```
   	
	- ngx_list_t 数据结构：链表容器
	
	```
	typedef struct ngx_list_part_s  ngx_list_part_t;

	struct ngx_list_part_s {	//链表的一个元素
    	void             *elts;		//起始地址
    	ngx_uint_t        nelts;	//已存储元素数
    	ngx_list_part_t  *next;		//下一个链表元素地址
	};

	typedef struct {	//链表结构
	    ngx_list_part_t  *last;		//指向最后一个元素
	    ngx_list_part_t   part;		//链表的首个元素
	    size_t            size;		//链表的每一个元素大小
	    ngx_uint_t        nalloc;	//每一个元素中能容纳的大小
	    ngx_pool_t       *pool;		//链表中的内存池对象
	} ngx_list_t;

	```
	nginx提供的list接口：
	```
	/**
		*	创建新的链表
		*	@param pool 内存池对象
		*	@param size 每个元素的大小
		*	@param n 每个链表元素能容纳元素的个数
		*	return 新创建的链表地址，失败为NULL
		*	
	**/
	ngx_list_t *ngx_list_create(ngx_pool_t *pool, ngx_uint_t n, size_t size);
	
	/**
		*	初始化一个已有的链表
		*	与ngx_list_create类似，传入的参数是ngx_list_t 链表
		*	返回NGX_OK,表示初始化成功；返回NGX_ERROR，表示失败。
	**/
	static ngx_inline ngx_int_t ngx_list_init(ngx_list_t *list, ngx_pool_t *pool, ngx_uint_t n, size_t size);
	
	/**
		*	添加新元素
		*	正常情况下返回新分配元素的首地址;返回NULL表示添加失败。
		*	使用方法，先用此返回地址，然后再赋值。
	**/
	void *ngx_list_push(ngx_list_t *list);
	```

	遍历链表，nginx没有提供相应的接口，可以使用如下方法遍历：
	```
	//part 指向链表中的每一个ngx_list_part_t 
	nxg_list_part_t * part=&testlist.part;

	//根据链表中的数据类型，把数组里的elts转化为该类型使用，此处以ngx_str_t 为例
	ngx_str_t *str=part->elts;

	//i 表示元素在每个ngx_list_part_t 数组中的序号
	for(i=0;/* void */;i++){
		if(i>=part->nelts){		//一个元素结束了
			if(part->next==NULL){
				breeak;		//最后一个链表元素，便利结束
			}
		
			//下一个ngx_list_part_t
			part=part->next;
			header=part->elts;

			//将 i 置为0
			i=0;
		}
		//此处输出遍历到的元素
		println("list element: %*s\n",str[i].len,str[i].data);
	}

	```
	- ngx_table_elt_t 数据结构
	
	```
	typedef struct{		//在Ngx_hash.h 中
		ngx_uint_t	hash;			//表明ngx_table_elt_t可以是散列表数据结构中的成员
		ngx_str_t	key;			//键
		ngx_str_t	value;			//值
		u_char		*lowcase_key;	//指向全小写的key字符串
	}ngx_table_elt_t
	```

	- ngx_buf_t 数据结构
	
	```
	// ngx_buf.h
	// 处理大数据的关键数据结构，既应用于内存数据，也应用于磁盘数据
	typedef void *            ngx_buf_tag_t;

	typedef struct ngx_buf_s  ngx_buf_t;

	struct ngx_buf_s {
    	u_char          *pos;				//从pos 这个位置开始处理内存中的数据
    	u_char          *last;				//有效内容到此为止
    	off_t            file_pos;			//将要处理的文件位置
    	off_t            file_last;			//截止的文件位置

    	u_char          *start;         /* start of buffer */
    	u_char          *end;           /* end of buffer */
    	ngx_buf_tag_t    tag;			//当前缓冲区的类型，例如由哪个模块使用就指向这个模块 ngx_module_t 变量的地址
    	ngx_file_t      *file;			//引用的文件
    	ngx_buf_t       *shadow;		//当前缓冲区的影子缓冲区，减少转发过程中的开销，这种结构过于复杂
		
		/* the buf's content is mmap()ed and must not be changed */
		unsigned         temporary:1;

    	/*
     	* the buf's content is in a memory cache or in a read only memory
     	* and must not be changed
     	*/
    	unsigned         memory:1;

    	/* the buf's content is mmap()ed and must not be changed */
    	unsigned         mmap:1;

    	unsigned         recycled:1;		//为1时表示可回收
    	unsigned         in_file:1;			//为1时表示缓冲区处理的是文件
    	unsigned         flush:1;			//为1时表示需要执行flush操作
    	unsigned         sync:1;			//是否使用同步方式，有些框架代码在sync 为1时，可能会有阻塞，视使用它的模块而定
    	unsigned         last_buf:1;		//是否是最后一块缓冲区，ngx_buf_t 可以由ngx_chain_t 链表串联起来，为1时，表示是最后一块带处理的缓冲区
    	unsigned         last_in_chain:1;

    	unsigned         last_shadow:1;
    	unsigned         temp_file:1;

    	/* STUB */ int   num;
	};
	```

	- ngx_chain_t 数据结构
	```
	// ngx_chain_t 是与 ngx_buf_t 配合使用的链表数据结构
	struct ngx_chain_s {
    	ngx_buf_t    *buf;			\\ 当前缓冲区
    	ngx_chain_t  *next;			\\ 下一个ngx_chain_t ,如果是最后一个ngx_chain_t, 需要把 next 置为NULL
	};

	```
	在向用户发送HTTP 包体时，就需要传入 ngx_chain_t 链表对象，如果是最,那么必须将next 置为NULL，否则永远不会发送成功，而且这个请求将一直不会结束