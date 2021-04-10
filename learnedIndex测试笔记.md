## LearnedIndex 测试笔记

阅读代码：

在include\nvm_alloc.h中

```c++
//#define USE_MEM
```

通过定义不同的Alloc类（负责分配和释放存储空间）决定是存在NVM还是内存中，这里如果是在DRAM中，alloc分配内存就是malloc函数，释放就是free，很简单，封装成一个简单的alloc类。主要看一下在NVM中的Alloc类：

``` c++
Alloc(const std::string &file_name, const size_t file_size) {
        pmem_file_ = file_name;
        pmem_addr_ = PmemMapFile(pmem_file_, file_size, &mapped_len_);
        current_addr = pmem_addr_;
        used_ = freed_ = 0;
        std::cout << "Map addrs:" <<  pmem_addr_ << std::endl;
        std::cout << "Current addrs:" <<  current_addr << std::endl;
    }
...
void *alloc(size_t size)
    {
        std::unique_lock<std::mutex> lock(lock_);
        void* p = current_addr;
        used_ += size;
        current_addr = (char *)(current_addr) + size;
        // std::cout << "Alloc at pos: " << p << std::endl;
        return p;
        // return malloc(size);
    }
```

这里的PmemMapFile代码如下

``` c++
static inline void *PmemMapFile(const std::string &file_name, const size_t file_size, size_t *len)
{
    int is_pmem;
    std::filesystem::remove(file_name);
    void *pmem_addr_ = pmem_map_file(file_name.c_str(), file_size,
                PMEM_FILE_CREATE | PMEM_FILE_EXCL, 0666, len, &is_pmem);
#ifdef SERVER
    assert(is_pmem == 1);
#endif
    if (pmem_addr_ == nullptr) {
        perror("BLevel::BLevel(): pmem_map_file");
        exit(1);
    }
    return pmem_addr_;
}
```

这里面主要的函数pmem_map_file是在libmem.h中定义的，完成的工作就是创建持久化内存的文件，并将文件映射，得到指向文件的指针

![image-20210117101828237](C:\Users\shameless\AppData\Roaming\Typora\typora-user-images\image-20210117101828237.png)

（目前看代码的感觉在NVM和DRAM中的区别也就是分配内存时候分配什么地址，由上述对ALLOC类的定义不同

比如在fast-fair的btree.h中

``` c++
static void alloc_memalign(void **ret, size_t alignment, size_t size) {

  // posix_memalign(ret, alignment, size);

  *ret = NVM::structure_alloc->alloc(size);	

}
```

*posix_memalign( )*

在大多数情况下，编译器和C库透明地帮你处理对齐问题。POSIX 标明了通过*malloc( )*, *calloc( )*, 和 *realloc( )* 返回的地址对于任何的C类型来说都是对齐的。在*Linux*中，这些函数返回的地址在32位系统是以8字节为边界对齐，在64位系统是以16字节为边界对齐的。有时候，对于更大的边界，例如页面，程序员需要动态的对齐。虽然动机是多种多样的，但最常见的是直接块I/O的缓存的对齐或者其它的软件对硬件的交互，因此，*POSIX 1003.1d*提供一个叫做*posix_memalign( )*的函数



cmakelist

use `cmake -DSERVER:BOOL=ON ..` when running in server



**利用YCSB测试**

在tests\ycsb.cc文件中自定义了DB类，这样可以利用YCSB测试

ParseCommandLine解析参数

- -threads n：使用n个线程执行（默认值：1）
- -db dbname: 指定要使用的数据库名称（默认值：basic）
- -P propertyfile：从给定文件加载参数属性（有一定的格式，可以使用#进行行注释，使用=号连接属性名和值）。 可以指定多个文件，并将按照指定的顺序进行处理
- -host ip：指定ip地址
- -port port：指定port端口
  更多的参数可以参考函数`ParseCommandLine`的实现，也可以添加一些参数，辅助测试，比如对于一些嵌入式数据库，往往需要指定数据库数据存放的路径。

通过定义workload参数文件得到不同的数据



目前还有的一些问题



**25.100服务器翻墙**

cd /root/fanqiang 

nohup ./clash -d . &

export https_proxy=localhost:7890



**用自己vpn翻墙**

本机：ssh -Nf -R 2345:localhost:4780 root@192.168.25.100

服务器：export https_proxy=localhost:2345



**测试过程**

nvm挂载：mount -o dax /dev/pmem0 /mnt/pmem0_wenduo

git push origin learn-index

git clone -b learn-index git@...

./build.sh server

cd build

./ycsb -db fastfair -P ../include/ycsb/workloads



**后台运行**

nohup <command>  &



**利用OpenStreetMap数据测试**

这里数据处理用了很久，下下来的数据，试了几个官方软件格式（josm，osmosis）都没法转换osm.pbf

最后用的是osmconvert转成功了，提取了亚洲数据中的200M个经纬度信息

因为ycsb不能加载数据，所以仿照ycsb流程写了一个测试程序（pgm xindex和fastfair都可以测试了，alex还有一点奇怪的小问题）



用gdb调试alex的测试，找到了问题发生的位置，在i = 138961次插入，insert函数中的第11次while循环中，insert->find_insert_position->find_insert_position->exponential_search_upper_bound->key_greater(ALEX_DATA_NODE_KEY_AT(m), key)报错AlexCompare……狠tm奇怪……



最后找到错误的地方是他要插入的那个地址不存在，因为代码实现里有一部分是为了防止一次插入影响的移位过多而进行的优化，貌似优化出来了有问题的地方…，我把这几行代码注释掉就跑通了，而且看了一下性能也没被影响，还是很高



更改了一下测试，考虑到之前的测试数据可能存在key相同情况，现在测试用经度和纬度拼接成key，但是发现这样之后变得很慢，所以还是把原来的数据处理了一下去掉了重复的经度，然后还是把经度作为Key。

![image-20210326155345833](E:\note\learnedIndex测试笔记.assets\image-20210326155345833.png)