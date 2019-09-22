# 《STL源码剖析》阅读摘要

## 第一章

介绍一些STL中特定的组态和设定

## 第二章 空间配置器（allocator）

第一级、第二级空间配置器

STL设计的**空间配置器** std::alloc

[STL的**内存池**实现（供参考）](https://www.cnblogs.com/nzhl/p/5753728.html)

自由链表中的节点大小都是 8bytes 的倍数，如果用户申请的节点不是 8 的整数倍，那么就将所需内存向上调整到 8 的倍数。为什么是 8 的倍数？原因是，在 **64位** 系统下，指针大小为 8 个字节，如果节点大小为 4 个字节的话，那么在 64 位系统下，**该节点将不足以容纳一个指针，而导致该链表无法维护**；

三个基本处理工具：uninitialized_copy、uninitialized_fill、uninitialized_fill_n



## 第三章 迭代器与traits编程技法

**迭代器粘合了 算法与容器**

这一部分大量使用了**偏特化**，例如，当调用一个泛型算法时，根据输入类型，使用模板函数+模板类（参数类型推断机制，返回一个临时**对象**）推断输入的迭代器的类型（这非常重要）。然后以类型（临时对象）为**参数**去调用被封装的函数，这个时候根据参数的类型的不同，会匹配不同的特化版本，保证最优效率。

3.6节介绍了如何推断输入类型的完整代码，用到了“萃取器”traits（模板类），对迭代器对象、指针、指向const的指针有多个偏特化版本。

> 举个例子具体怎么做：
>
> 假设我们用struct定义了好几种迭代器类型，例如
>
> ```C++
> struct A {};
> struct B :public A {};
> struct C :public A {}; `    //等等，都是空的，没成员。
>```
>
> 定义了`traits`，它将不同迭代器的类型重新统一命名:
>
> ```C++
> template <class Iterator>
> struct iterator_traits {
> 	typedef typename Iterator::iterator_category iterator_category;
> };
> ```
>
> 然后定义某种迭代器, 例如：
>
> ```C++
> struct __XXX__iterator {
> 	typedef C iterator_category; //定义这个迭代器属于C类
> };
> ```
>
> 这样，在某个封装函数种用到迭代器时，先使用traits类去推断（取得）迭代器类型，以类型为参数调用被封装的函数。
>
> ```C++
> template<class Iterator>
> void fun(Iterator& iter) {
> 	typedef typename iterator_traits<Iterator>::iterator_category category;
> 	fun_(iter, category());        //A(), B()或者C()
> }
> 
> template<class A_iter>
> void fun_(A_iter& i, A) {                     //函数多了一个tag，A类的某个对象
> 	std::cout << "I am A" << std::endl;
> }
> 
> template<class B_iter>
> void fun_(B_iter& i, B) {
> 	std::cout << "I am B" << std::endl;
> }
> 
> template<class C_iter>
> void fun_(C_iter& i, C) {
> 	std::cout << "I am C" << std::endl;
> }
> //测试函数
> int main()
> {
> 	__XXX__iterator X_iter;
> 	fun(X_iter);                          //输出 "I am C"
> }
> 
> //决定迭代器类型的函数
> template <class Iterator>
> inline typename iterator_traits<Iterator>::iterator_category
> iterator_category(const Iterator&){
> 	typedef typename iterator_traits<Iterator>::iterator_category category;
> 	return category();               //返回一个临时对象，在这里其实就是C(),空的没成员
> }
> ```



## 第四章 序列容器

array, vector, heap, list, deque(双端队列), stack, queue

#### 1.vector

讲解如何控制大小，空间配置策略，重新配置使得数据移动

vector的迭代器为 **普通指针**， vector空间不足时用空间配置器扩容2倍、不够就扩容需要的大小n

vector扩容是在另外一片空间，然后将原来的内容**拷贝**过来，然后再构造新的内容。

vector 删除erase(iter) 返回的是 **删除位置的迭代器**

插入insert的时候根据插入位置会有不同的策略，插入位置为指示节点的前方（插入元素a,b,指示点p）;



`void push_back(const T& x)`参数是**const引用（很多地方的参数都是const引用）**，还有空间就调用`construct(x)`同样参数为常引用，`construct(x)`会调用默认的`placement new`，在内存上调用x的**拷贝构造函数。**没空间的话调用插入函数`insert_aux(end(), x)`，引起扩容然后插入。

`void insert_aux(iterator pos, const T& x)`，这个函数负责插入，首先检查是否有空间，有的话，将pos位置之后的全部元素后移，并在原pos位置赋值新元素。不够的话，先发生扩容，扩容后要将原来的元素全部用`uninitialized_copy()`拷贝过来，**拷贝到构造这个过程如果失败的话会发生回滚**(异常处理)。



#### 2.list

list时一个环状双向链表，**不能使用普通指针为迭代器**

迭代器可以++, --   但是不能+1, +2 这种运算。

```c++
template<class T>
struct __list_node{
    typedef void* void_pointer;
    void_pointer prev;                     //void* 其实可为__list_node<T>*
    void_pointer prev;
    T data;
}
```

根据前一章的内容，需要为迭代器设计iterator_category（迭代器类型），value_type，pointer, reference等等

```C++
typedef __list_node<T>* link_type;
link_type node；                         //指向节点的指针
```

由于是一个环状的链表，所以class list**只用一个node**指针就可以表示整个环（这个Node为`end()`，Node的Next就是头节点，如果发现指回了自己那就是空list）

list的insert在指定位置插入，erase删除指定位置，返回删除结点的后一个

操作：push_front, push_back, erase, pop_front, pop_back, clear, remove, unique, splice, merge, reverse, sort

clear:**清空所有节点，注意是从begin开始清空**

`remove(const T& value)`：从头开始遍历，找到值为value的就调用`erase`

unique:移除连续相同元素

**transfer(iter p, iter first, iter last)**      把[first, last)中的元素接到p之前的位置(顶替p的位置)

splice（将所指元素移到指定），merge（合并两个有序链表，归并），reverse（反转），sort（快排，找指定位置）都通过transfer来执行，其中sort是list特有的sort。



#### 3.deque

deque是双向开口的线性空间，（vector单向开口）。可以在头尾插入和删除，且都是常数级操作。

deque是用动态的**分段连续空间合成**的，在空间不足时，可以随时增加一段新的空间并连接起来，不会出现vector中需要重新配置、复制旧元素的情况，因此也不需要预留(capacity)。deque支持随机访问，为了同时实现这些功能，它的迭代器并**不是普通指针**，整体实现也相当复杂。

##### 中控器

deque使用一块map（连续空间，不是容器map）作为主控，其中每个元素都是指针，指向另一块连续的线性空间，称为**缓冲区**（这才是存储元素的地方）。deque通过中控器表现出表面上的连续，并提供随机访问接口。

```C++
template <class T, class Alloc == alloc, size_t Bufsize = 0>
class deque
{
public:
    typedef T value_type;
    typedef value_type* pointer;
protected:
    typedef pointer* map_pointer;
    
    map_pointer map; //T**，指向中控器map，连续空间；元素为指针（节点），指向缓冲区
    size_type map_size; //map容纳的 指针数量
}
```

##### 迭代器

STL自定义迭代器的类型，迭代器保存了`T* cur(迭代器指向的元素)，T* first(缓冲头), T* last(缓冲尾)`以及一个**指向中控器map的指针**。在操作碰触边界时，要利用这些指针来跳转缓冲区（set_node）。

##### deque数据结构

duque维护了两个迭代器，一个指向头(start)， 一个指向尾(finish)，还保存了中控器map的大小。

```C++
protected:
	iterator start;
	iterator finish;
```

deque在构造时候，**保有一个空的缓冲区**（无元素），然后再调用函数、根据元素数量来扩展。在扩展时，会将有数据缓冲区**指针**保持在最中央，使得两头可以扩充的缓冲区一样大（调整map_size，保证至少一个）。然后**为现用的节点配置缓冲区**，这些缓冲区就是deque的可用空间（最后一个可能留有余裕）。

**添加元素**，添加操作分为push_front、push_back。缓冲区有多的就直接添加，没有就要配置新的缓冲区，在配置缓冲区前，如果中控map节点不够就要重新配置map（reserve_map_at_front()，reserve_map_at_back()）。
发生了节点不足的情况时，分头节点不足和尾节点不足两种情况执行，首先**检查map整体上是不是还有充足的节点空间**（例如，尾部插入太多，头部为空），如果是这种情况，就调整节点的位置到map中央；如果不是，就配置新空间，给新map使用，将原节点指针拷贝过来，然后，更新`map_pointer map, map_size`，最后两种情况都需要更新`start, finish`两个迭代器。

**删除元素**，pop_back，pop_front，clear，erase。erase删除元素时，重点是要**考虑哪端需要移动的元素较少**，如果前端元素较少就移动前端，反之移动后端，并对应移动`start, finish`。删除时，如果发生了跳转缓冲区，就要释放掉缓冲区，但是，清除完整个deque(clear)，还是会保有一个缓冲区不释放（策略，也是初始状态）。

**插入元素**，insert，同样要考虑哪端元素较少。



#### 4.stack与queue
缺省情况下都使用deque为底部结构，由于都以底部容器完成所有工作，被称为容器配接器。stack（栈）所有元素先进后出，queue（队列）所有元素先进先出，都不提供迭代器、遍历功能。**还可以用list作为底部容器**。



#### 5.heap
为优先队列（priority queue）服务，考虑到实现难度采用堆，复杂度在**队列和二叉排序树之间。**
heap的算法:
`make_heap(vector.begin(), vector.end()),`
`push_heap(vector.begin(), vector.end()),`
`pop_heap(vector.begin(), vector.end()),`
`sort_heap(vector.begin(), vector.end())`
迭代器的范围就是操作范围，先要是堆的顺序才能执行后三个方法，**其中要先主动`push_back`然后执行`push_heap`，执行`pop_heap`后要主动`pop_back()`。**



#### 6.priority_queue
只允许在底端加入元素，并从顶端取出元素（权值较高），利用max-heap自动排序，**不提供迭代器，只有最顶端元素才有机会被外界取用**。缺省情况下使用vector为底部容器，是一种容器适配器。
```C++
template<class T, class Sequence = vector<T>, 
		class Compare = less<typename Sequence::value_type>>
class priority_queue{
	Sequence c;
    const_reference top();
    void push(const type_value& x)
    {
    	c.push_back(x);
    	...
    }
    void pop()
    {
        ...
        c.pop_back();
    }
}
```



#### 7.slist

单向链表，待补充




## 第五章 关联式容器
所谓关联式容器，设计观念上类似关联式数据库（实际上简单很多）：每笔数据都有一个键值(key)和一个实值(value)。

记录几个关联容器通用的重要方法：

> **lower_bound**( begin,end,num)：从数组的begin位置到end-1位置二分查找**第一个大于或等于num**的数字，找到返回该数字的地址，不存在则返回end。通过返回的地址减去起始地址begin,得到找到数字在数组中的下标。
> **upper_bound**( begin,end,num)：从数组的begin位置到end-1位置二分查找**第一个大于num**的数字，找到返回该数字的地址，不存在则返回end。通过返回的地址减去起始地址begin,得到找到数字在数组中的下标。
>
> **equal_range**是C++ STL中的一种二分查找的算法，试图在已排序的[first,last)中寻找value，它返回一对迭代器i和j，其中i是在不破坏次序的前提下，**value可插入的第一个位置（亦即lower_bound）**，j则是在不破坏次序的前提下，**value可插入的最后一个位置（亦即upper_bound）**，因此，[i,j)内的每个元素都等同于value，而且[i,j)是[first,last)之中符合此一性质的最大子区间



#### AVL-Tree

维护平衡的方式：子树高度差不能超过1

若插入节点后影响了平衡：只需要从插入或者删除节点向上找到第一个不平衡的节点，调整它到平衡

两种调整动作：左旋（逆时针旋转），右旋（顺时针旋转）

不平衡类型：

1. 左 + 左（左子树外侧）：右旋一次
2. 左 + 右（左子树内侧）：左旋一次到情况1，然后右旋一次
3. 右 + 左（右子树内侧）：右旋一次到情况4，然后左旋一次
4. 右 + 右（右子树外侧）：左旋一次



-------------

红黑树与4种基于红黑树(RB-Tree)的关联式容器

-------------

#### RB-Tree

1. 节点不是红色就是黑色
2. 根节点为黑色
3. 如果节点为红色，子节点一定为黑色
4. 任一节点到null的任何路径，所含黑节点数必须相同

关于STL种红黑树的实现，每个节点有三个指针，除了左右外还有个父，存在一个head节点（也是`end()`所对应的节点），刚创建时，head节点的父指0，在插入元素后，指向真正的第一个元素的节点`root`，同时`root`也指向head，head的左指向最左节点，也就是`begin()`需要返回的节点，右指针指向最右节点，也就是`rbegin()`需要返回的节点。

更多待补充



#### set

操作基于RB-Tree，key值即value值


#### map

操作基于RB-Tree，key值不同，value值可以相同

map的插入操作有一些小细节，使用key值通过下标访问符[]进行操作时，会发生**插入操作**，调用了一个返回`pair<iterator, bool>`的`insert`函数，其中第一个是迭代器，第二个bool表示插入成功与否，成功时迭代器指向新元素，失败时也就是元素已经存在时，指向已经存在的元素。



#### multiset

可重复key值版本，调用RB-Tree的`insert_equal`

#### multimap

可重复key值版本，调用RB-Tree的`insert_equal`



---------------------

哈希表与4种基于哈希表的容器

-------------------------

#### hashtable

可被视为字典结构，提供常数时间的基本操作。

实现方法：使用线性容器存储元素，每一个位对应一个元素的key值（索引），并使用`hash function`（散列函数）将大数映射为小数，作为key值（索引），举例：

> 按ASCII编码把字符串编码，“jjhou”——'j', 'j', 'h', 'o', 'u' 如下：
>
> 'j' * 128^4 + 'j' * 128 ^ 3 + 'h' * 128 ^ 2 + 'o' * 128 ^ 1 + 'u' * 128 ^ 0
>
> 106 * 128^4 + 106 * 128 ^ 3 + 104 * 128 ^ 2 + 111 * 128 ^ 1 + 117 * 128 ^ 0 = 28678174709
>
> 这个数显然太大了，保存如此大的数组显然不科学。
>
> 所以使用散列函数，例如X % TableSize会得到一个范围在0~TabelSize-1之间的数，作为索引。

这样做的问题是会有不同的元素映射到相同位置（发生碰撞），解决办法：线性探测、二次探测、开链

STL采用开链，hashtabel内元素为桶(bucket)，意思每个单元可能存储的是多个**节点**，每个节点存有值和指向下一个节点的指针（像list），整个聚合在一起的（buckets）使用`vector`来完成。

实现细节：

- **迭代器内型为forward**，意思只能前进，迭代器指向一个节点，前进一个位置即到达下一个节点，如果正巧在list的尾端，则跳到下一个有效的桶(bucket)上，也就是下一个list的头部节点。
- 表格大小一般为质数，STL中预备了28个质数(大致按照翻倍顺序)。
- 插入元素时首先判断hashtabel是否需要重建，标准为新增元素后的**总元素个数与当前`bucket vector`的大小比较**，如果前者大于后者就重建。重建时要挨个对旧表的元素重哈希计算，挂到新表正确的桶里面。
- 封装了一个`bkt_num()`函数计算元素落到哪个桶，需要获得key值和桶的个数（buckets，vector的大小），哈希函数对字符串类型`const char*`进行了特定的处理。
- 删除`clear`删掉了每一个节点，但是`vector`并没有释放，复制`copy_from`先要`clear`清空所有节点，然后复制`vector`的每一个`bucket`，也就是指向节点的指针，还要对每个`bucket`内的节点挨个复制`(new node)`。

#### hash_set 与 hash_multiset

操作基于hashtable，key值即value值（multi为可重复版本）

和用RB-Tree实现的set区别在于元素无序

#### hash_map 与 hash_multimap

操作基于hashtable，key值不同，value值可以相同（multi为可重复版本）

和用RB-Tree实现的map区别在于元素无序



## 第六章 算法

sort 排序算法

STL的sort算法，数据量大时采用Quick sort， 分段递归排序。一旦分段后的数据量小于某个门槛（5-20），为避免Quick sort的递归调用带来过大的额外负担，就改用Insertion sort（插入排序）。如果递归层次过深，还会改用Heap Sort（堆排）。

Quick Sort中的分段原则，通常**取整个序列头、尾、中央三个位置的元素，以其中值作为枢轴(pivot)**。



## 第七章 仿函数



## 第八章 配接器

将一个class的接口转换为另一个class的接口，使原本因接口不兼容而不能合作的classes，可以一起运作。