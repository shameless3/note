# alex移植优化

## Alex

### 1 代码目录简介

alex_base.h					包含线性模型和模型的构建，bitmap的帮助函数和cost model相关函数，

alex_fanout_tree.h		利用fanout tree帮助alex决定在bulk load和分裂节点时的最优fanout情况和key分割模式

alex_map.h					 定义了自己的map类，和std:map有一些不同，比如这里是前向迭代器和std::map是双向迭代器，以及这里的key和payloads是分开存储的

alex_multimap.h			定义了自己的map类，和std::multimap的不同和上述相同

alex_nodes.h				  定义了alex的节点，包括模型节点（内部节点）和数据节点（叶节点）

alex.h							   实现大部分api函数如下



### 2 alex_base.h

class LinearModel：

	- expand() 扩张函数，这里就是对斜率和截距都乘上扩张倍数
	- predict()  应该是应用于内部节点的预测，因为进行了取整
	- predict_double() 应该是应用于叶节点的预测，返回double类型

class LinearModelBuilder

- add() 	添加节点
- build()   利用最小二乘法确定模型的斜率和截距

定义了一些bitmap的帮助函数

定义了cost model的一些阈值参数

定义了一些数据的累加器，即那些性能指标，如单次插入会产生多少次移动



### 3 alex_fanout_tree.h

定义了扇出树，这是为了bulk loading实现的，用于确定每个节点的最优扇出数量

这部分我论文笔记里没写，这里补充一下

![image.png](https://i.loli.net/2021/03/02/6G7VDL1dPNbBom8.png)

扇出树是一个完全二叉树，当要决定一个节点的最优扇出数量时，我们就构建一个扇出树如上图，每个节点代表当前模型节点的一个可能子节点即构建这样一个节点的成本，如第一层代表当节点为一个数据节点时的成本120，第二层代表模型节点扇出树为2时，成本为110（同层求和）。构建fanout tree的流程是从一个节点向下构建，统计每层的成本并求和，直到某一层的成本比上一层成本要大，就不向下构建了，这是因为向下继续构建确实可能有更小的成本，但越往下计算成本的成本也越大。然后从成本最低的那一层开始选择合并和分割来得到更小的成本，比如这里10分割成1和1，20和25合并成40，最后得到上图加深颜色的成本和是52的扇出方式。

***

帮助函数

- collect_used_nodes()		收集所有的标记为使用的扇出树节点并对其排序（从左向右排序）
- merge_nodes_upwards()    向上合并节点（如果这样可以减小cost的话）  

***

bulk load使用的函数

- compute_level()
- find_best_fanout_bottom_up()    找到最佳的fanout策略
- find_best_fanout_top_down()    仅供实验用
- find_best_fanout_existing_node()    对已经存在的节点确定最佳的fanout策略

### 4 alex_nodes.h

class AlexNode	内部节点和叶节点类的父类

class AlexModelNode 内部节点类	

- get_child_node()	对给出的key找到负责这个key的子节点，其实就是调用上述的模型中的predict函数（因为这里是内部节点）

- expand()	内部节点扩张就是复制，即对存在的所有子节点的指针都复制为2的指数倍，传入的参数是指数，返回扩张的倍数，也要对节点里的模型调用expand()函数
- validate_structure()    用于debug，检测结构是否正确

class AlexDataNode 数据节点类

- 构造函数和析构函数

- 一些帮助函数

迭代器

  - class Iterator	定义迭代器（要存储当前位置和bitmap中的位置）


***

cost model     成本模型
  - shifts_per_insert()	每次插入的移位次数
  - exp_search_iterations_per_opration()    每次操作的指数搜索迭代次数
  - empirical_cost()    计算开销（调用上面的两个函数）
    - 计算开销公式（后面计算cost都是这个公式）：cost = kExpSearchIterationsWeight *  search_iters+kShiftsWeight * shifts * insert_frac
  - frac_inserts()    插入操作比例
  - reset_stats()  重置参数，包括：移位次数，指数搜索迭代次数，插入次数，查询次数，num_resizes
  - compute_expected_cost()    计算节点预期的开销（上述计算的都是经验的）
    - 通过accumulator计算当前模型每个key进行一次模拟的插入（查询），即对这个key用此模型做一次预测，然后统计如果这是一次插入（查询），需要多少次移位和指数搜索再求均值计算开销
  - compute_expected_cost（） 计算根据输入的间隙数组构造一个数据节点的开销（假设现存的模型是依据间隙数组训练的）
    - build_node_implicit()  帮助函数，模拟构造数据节点过程，计算开销
  - compute_expected_cost_sampling（）取样构造，预测开销（通过逐步增大样本数量，当变化不大且样本数量>总数1/2时停止）
    - build_node_implicit_sampling（）帮助函数，同上

  - compute_expected_cost_from_existing（）从已经存在的key范围中计算开销
    - build_node_implicit_from_existing（）帮助函数，同上

***



- bulk loading和model loading
  - bulk_load()	加载块
  - bulk_load_from_existing()    
  - build_model()    构建模型
  - build_model_sampling()    采用样本构建模型
  
- 查询
  - predict_position()    给出预测的key位置
  - find_key()    找到key的位置（先预测再指数搜索）
  
- 插入和调整大小
  - significant_cost_deviation()	减策经验开销是否和预期开销差太多
  - catastrophic_cost()    如果插入的开销“灾难性”得高返回true，需要强制分裂节点，这里设置的标准是100，即一次插入需要超过100次移位操作
  - insert()    返回一个pair，第一个值代表插入结果，0代表成功，1代表经验开销和预期开销相差太多，2代表插入开销“灾难性”得高，3代表容量满了，-1代表插入的key本身就存在；第二个参数是插入的key的位置，或者是已经存在的key的位置，没插入就是-1。
  - resize()    将data node调整成目标密度（用于空隙太少或太多的时候）
  - insert_element_at()    在指定位置插入key，此函数的调用函数需保证插入的位置是空位。
  - insert_using_shifts()    指定位置插入，可以使用移位找到空位，返回真正插入的位置
  - closest_gap()    找到指定位置最近的gap的位置
  
- 删除
  - erase_one()    删除给定的key的最左边的一个值，若不存在则不删除，返回1代表删除了，反之返回0
  - erase_one_at()     删除给定位置的key，注意这里要把这里的值赋值为下一个key的值并更新bitmap设置这里是gap
  - erase()    删除给定的key的所有值，返回删除的个数（？一个key不应该只会占一个位置么，可能会存相同的key值但应该除了一个外其他都是gap），最后检查key数量是不是太少，是的话进行resize-
  - erase_range()     范围删除
  
- 统计信息
  - node_size()    node的大小
  - data_size()     存储数据的大小（key,payload,data_slots,bitmap)
  - num_packed_regions()    连续key（不包含间隙）的块数量
  
- debug    用于debug的几个函数



### 5 alex.h

定义class alex，这就是描述alex整体模型的类，提供了很多供用户调用的api如下

 ** ALEX with key type T and payload type P, combined type V=std::pair<T, P>.*

 ** Iterating through keys is done using an "Iterator".*

 ** Iterating through tree nodes is done using a "NodeIterator".*

***

 ** Core user-facing API of Alex:*

 ** - Alex()*

 ** - void bulk_load(V values[], int num_keys)*

 ** - void insert(T key, P payload)*

 ** - int erase_one(T key)*

 ** - int erase(T key)*

 ** - Iterator find(T key) // for exact match*

 ** - Iterator begin()*

 ** - Iterator end()*

 ** - Iterator lower_bound(T key)*

 ** - Iterator upper_bound(T key)*

***

 ** User-facing API of Iterator:*

 ** - void operator ++ () // post increment*

 ** - V operator \* () // does not return reference to V by default*

 ** - const T& key ()*

 ** - P& payload ()*

 ** - bool is_end()*

 ** - bool operator == (const Iterator & rhs)*

 ** - bool operator != (const Iterator & rhs)*



所以用户要使用alex的话，需要用到的几个基本操作调用api如下：

put	调用insert()

get	调用get_payload()

update 	调用get_payload得到地址后更改地址中的值

scan	找到起始key的迭代器位置，利用迭代器遍历得到范围结果



## 持久化存储到nvm

sbh学长代码更改部分:

alex_fanout_tree.h	

335行 模板中增加了Alloc，为了传递给AlexModelNode

***

alex.h	

41行增加头文件common_time.h

写入数据的地方添加NVM::Mem_persist持久化存储到内存

2321行和2335行 删掉了const

***

alex_nodes.h

写入数据的地方添加NVM::Mem_persist持久化存储到内存

***

pmreorder	











## xindex 

### 1 代码目录简介

- xindex_utiil.h	RCU读写锁相关函数实现

- helper.h	帮助函数
- xindex.h  xindex_impl.h    最顶层的接口，定义实现了供用户使用的get,put等api函数
- xindex_root.h  xindex_root_impl.h    定义实现了根节点相关，即rmi
- xindex_group.h  xindex_group_impl.h    定义实现了group相关，xindex中的group就类似alex中的节点，包含模型
- xindex_model.h  xindex_model_impl.h    定义实现了模型相关
- xindex_buffer.h  xindex_buffer_impl.h     定义实现了缓存相关

### 2 xindex_model.h  xindex_model_impl.h

class LinearModel	定义了线性模型类，包含成员函数如下：

- prepare()	传入数据，准备模型，存储数据，调用prepare_model计算出模型
- prepare()    重载上一个函数，传入参数中key可以是迭代器
- prepare_model()    
- predict()    利用模型做出预测，
- get_error_bound()
- get_error_bound()

### 3 xindex_root.h  xindex_root_impl.h



