## PMDK

Persistent Memory Development Kit是一系列为了方便非易失性存储开发的函数库和工具，pmemobj是其中一个函数库。



writer.c 和 reader.c编译

gcc reader.c -o reader -lpmemobj -lpmem -pthread

## pmemobj

### part 0

可以假设现在有两个小任务

- 从标准输入读入一个字符串并写到存储器
- 从存储器读入一个字符串输出到标准输出

一种方法是在第一个应用程序中将缓冲区及其长度写入文件，在第二个应用程序中仅在缓冲区完成时才读取缓冲区。

旨在提供一个持久指针，和原子操作它们的方法，

### part 1 访问持久内存

pmempbj可以管理内存池

pmemobj_create（）	创建内存池，要传入layout（一个字符串，是对这个内存池的标识符）

pmemobj_open（）	打开指定layout的内存池

pmemobj_close（）	当内存池不再需要的时候，用于释放



**持久内存指针**，持久内存指针和易失性内存上指针类似，区别是易失性内存指针只是一个对象在内存地址空间的起始地址，而持久内存指针额外包括一个pool id因为一个应用中可能有多个资源池需要进行标识。

```c++
typedef struct pmemoid {
	uint64_t pool_uuid_lo;
	uint64_t off;
} PMEMoid;
```

而通过pool id和offset即可唯一定位，（void *)((uint64_t)pool + oid.off)

这也是pmemobj_direct函数做的事情。

**root object**

是一个明确知道的可以从内存池中开始搜索的位置，是联系所有存储结构的一个”锚“



**安全地存储数据**

 这里也是对非易失性内存和易失性内存不一样的地方，在非易失性内存需要考虑如果突然掉电等导致不正常崩溃，会出写一致性问题，所以如果和在易失性内存上一样：

```c++
void set_name(const char *my_name) {
memcpy(root->name, my_name, strlen(my_name));
}
```

如果在memcpy执行过程中可能崩溃会出现崩溃写问题，而如果在memcpy之后崩溃，考虑到CPU的乱序执行，也可能会出现问题。

解决方法如下：

```c++
void set_name(const char *my_name) {
root->length = strlen(my_name);
pmemobj_persist(&root->length, sizeof (root->length));
pmemobj_memcpy_persist(root->name, my_name, root->length);
}
```

其实就是memcpy之前添加了一步存储长度，所以在读的时候可以确认是否正确，这里的pmemobj_persist是可以保证原子性的，因为8字节的内存可以原子写入。（这里是原理性的实现，实际上pmemobj提供了更方便的写法如事务），这也是非易失性内存编程的要点。



### part 2 事务

**生命周期**

![image-20210325140702522](E:\note\PMEM笔记.assets\image-20210325140702522.png)

pmemobj_tx_process可以替代其他函数推进事务进度，可以在不知道当前状态情况下调用。

pmemobj提供了一系列宏定义简化事务过程

```c++
/* TX_STAGE_NONE */

TX_BEGIN(pop) {
	/* TX_STAGE_WORK */
} TX_ONCOMMIT {
	/* TX_STAGE_ONCOMMIT */
} TX_ONABORT {
	/* TX_STAGE_ONABORT */
} TX_FINALLY {
	/* TX_STAGE_FINALLY */
} TX_END

/* TX_STAGE_NONE */
```

除了TX_BEGIN和TX_END都是可选的，同时可以嵌套事务，当一个嵌套事务中止，整个事务都会中止。

关于TX_FINALLY为什么必要

```c++
void do_work() {
	struct my_task *task = malloc(sizeof *task);
	if (task == NULL) return;

	TX_BEGIN(pop) {
		/* important work */
		pmemobj_tx_abort(-1);
	} TX_END

	free(task);
}

...
TX_BEGIN(pop)
	do_work();
TX_END
```

这里的free不会被调用，因为事务采用setjmp和longjmp，dowork中的TX_END后会跳回到外部的事务，应该修改为：

```c++
void do_work() {
	volatile struct my_task *task = NULL;

	TX_BEGIN(pop) {
		task = malloc(sizeof *task);
		if (task == NULL) pmemobj_tx_abort(ENOMEM);

		/* important work */
		pmemobj_tx_abort(-1);
	} TX_FINALLY {
		free(task);
	} TX_END
}
```

这样可以保证free被执行



**事务操作**

三种不同的事务操作，allocation，free，set

set：

pmemobj_tx_add_range

pmemobj_tx_add_range_direct

逻辑是，对当前的这部分空间做一个snapshot存入undo log，然后对其更改，如果出现错误就自动回滚。

eg:

```c++
struct vector {
	int x;
	int y;
	int z;
}

PMEMoid root = pmemobj_root(pop, sizeof (struct vector));

/*
struct vector *vectorp = pmemobj_direct(root);
TX_BEGIN(pop) {
	pmemobj_tx_add_range(root, offsetof(struct vector, x), sizeof(int));
	vectorp->x = 5;

	pmemobj_tx_add_range(root, offsetof(struct vector, y), sizeof(int));
	vectorp->y = 10;

	pmemobj_tx_add_range(root, offsetof(struct vector, z), sizeof(int));
	vectorp->z = 15;
} TX_END
*/

struct vector *vectorp = pmemobj_direct(root);
TX_BEGIN(pop) {
	pmemobj_tx_add_range(root, 0, sizeof (struct vector));
	vectorp->x = 5;
	vectorp->y = 10;
	vectorp->z = 15;
} TX_END
```

可以看到上面都是对root操作的，而pmemobj_tx_add_range_direct就是具体对其中的元素做修改

eg:

```c++
struct vector *vectorp = pmemobj_direct(root);
int *to_modify = &vectorp->x;
TX_BEGIN(pop) {
	pmemobj_tx_add_range_direct(to_modify, sizeof (int));
	*to_modify = 5;
} TX_END
```



TX_ONCOMMIT:在commit时执行

TX_ONABORT:在abort时执行



### part 3 类型

layout 声明

```c++
POBJ_LAYOUT_BEGIN(string_store);
POBJ_LAYOUT_ROOT(string_store, struct my_root);
POBJ_LAYOUT_END(string_store);

#define	MAX_BUF_LEN 10
struct my_root {
	char buf[MAX_BUF_LEN];
};

...
    
pmemobj_create(path, POBJ_LAYOUT_NAME(string_store), PMEMOBJ_MIN_POOL, 0666);
...
pmemobj_open(path, POBJ_LAYOUT_NAME(string_store));
```



有类型的持久化指针

用PMEMoid指针类似于用void *指针操作，并不安全，所以有了有类型的持久化指针

TOID宏定义：

```c++
#define	TOID(type)\
union _toid_##type##_toid

#define	TOID_DECLARE(type)\
TOID(type)\
{\
	PMEMoid oid;\
	type *_type;\
}

//eg:

TOID_DECLARE(struct car);
...

TOID(struct car) car1;
TOID(struct car) car2;
...
D_RW(car1)->velocity = 2 * D_RO(car2)->velocity;

```

关于这里为什么类型名要加上前缀和后缀，这里巧妙的解决了如果类型是一个两个单词的情况，比如struct car，union car，enum car，这种情况加上前缀和后缀会变成：

_toid_struct car_toid，这里中间有一个空格，但可以通过定义这用几个空的define解决：

```c++
#define _toid_struct
#define _toid_union
#define _toid_enum
```

所以这里的union就会变成car_toid。



TOID宏可以对指针更改类型，解引用时候通过D_RO和D_RW宏读写

```c++
TOID(struct my_root) root;

if (D_RO(root)->buf[0] != 0)
	D_RW(root)->buf[0] = 0;
```

TOID宏封装了对持久性内存对象的访问细节，使得用户使用更为简便。在PMEMoid指针形式下，需要先获得对象的实际内存地址，然后执行内存读写，并手动完成持久化操作。在TOID类型下，其使用方式如下：



### part 4 事务性分配内存



### part 5 原子动态分配内存

事务性的分配内存虽然方便写

