# 高速缓存设计

自从我们给CPU添加了AXI总线接口并去除了指令RAM和数据RAM之后，他的运行效率就大打折扣了，运行同样的程序需要花费更多的时间周期。以我举例，此时的IPC已经降到了0.01-0.02之间。

```{admonition} 那么去除指令RAM和数据RAM是不是一种设计倒退呢？

其实不然，指令RAM和数据RAM的使用要求软件人员明确掌握物理内存的容量、起始地址，增加了软件的开发难度。目前这种硬件架构仅在那些对于成本、功耗或执行延迟的确定性极为敏感的低端嵌入式领域广泛使用。那些应用领域还有一个特点是软件规模不大、程序行为相对确定，否则没有虚拟化存储管理对于应用开发来说就是灾难。纵然指令RAM和数据RAM有这样的不足，但是性能问题也是要解决的。

```


我们的解决思路就是增加高速缓存(Cache)


## 基础知识
## 动手实践

本章节主要参考《CPU设计实战》第10章“高速缓存设计”

在本章中，我们会把Cache的设计复杂度控制在入门级水平，兼顾性能

### Cache规模的设计

解决Axi总线速度缓慢的思路——高速缓存(Cache)

#### Cache的设计规格

首先我们来明确以下与Cache相关的主要设计规格，避免因为后续的讨论过于宏观而无法具体到细节。这些设计规格包括以下方面：
1. CPU内集成一个指令Cache和一个数据Cache。避免取指和取数产生结构冒险。
2. 指令Cache和数据Cache的容量均为8KB，均为两路组相连，Cache line大小均为16B
3. 指令Cache和数据Cache采用Tag和Data同步访问的形式。
4. 指令Cache和数据Cache均采用“虚Index实Tag”(简称VIPT)的访问形式
5. 指令Cache和数据Cache均采用伪随机替换算法
6. 数据Cache采用写回写分配的策略，指令Cache不考虑写策略
7. 指令Cache和数据Cache均采用阻塞式(Blocking)设计，即一旦发生Cache Miss，则阻塞后续访问直至数据填回Cache中。
8. Cache不采用“关键字优先”技术
   
我们解释以下制定上述设计规格的初衷
* 设计一个指令Cache和一个数据Cache是为了保证流水线能够满负荷运行
* 指令Cache和数据Cache各方面规格一致，确保即使不把Cache模块写成可参数化的，也可以通过所定义的Cache模块实例化两份分别用于实现Cache和数据Cache，减轻代码开发和调试的工作量。
* 将每一路Cache的容量定义为4KB是为了在采用VIPT访问方式的同时规避Cache别名问题。
* Cache采用VIPT可以将TLB的查找和Cache的访问并行进行，从而提高CPU的频率
* Cache采用阻塞式设计，主要是因为目前我们实现的是一个静态顺序执行的流水线，即使Cache设计成非阻塞式也不会带来整体的性能提升。
* Cache不采用“关键字优先”技术，可以降低与Axi总线交互的复杂度。
  
根据上述设计规格，我们可以计算地址相关的Tag、Index和Offset位数。
* Offset: Cache行内偏移，宽度为log2(sizeof(Cache line)) = 4
* Index:  Cache组索引,宽度为log2(组数) = log2(256) = 8
* Tag:    Cache行的标签，宽度为物理地址宽度-log2(路大小) = 64 - 8 - 4 = 52
```{note}
由于我们还没实现TLB，因此目前CPU发出的地址可以认为是物理地址
```

#### Cache表的组织管理

落实到Verilog中即我们有
* 两张256项 x 52 bits的Tag表
* 两张256项 x 1 bits的V表
* 两张256项 x 1 bits的D表
* 两张256项 x 128 bits的Data表

```{note}
既然是存储信息的表，电路上肯定要用存储器件来实现，通常是用Regfile或RAM来实现，我们常用的指导思想是，容量大的表（如Bank表）用RAM实现，容量小的表（如D表）用Regfile实现。
```

我们依据在读、写操作访问Cache执行过程中所属的不同阶段，将对Cache模块进行的访问归纳为四种：Look Up、Hit Write、Replace和Refill，下面给出四种访问的定义。
* Look Up: 根据地址中的index信息读取对应的两个Cache line，为下一阶段判断是否在Cache中作准备
* Hit Write: 命中的读操作将数据返回，写操作进入Write Buffer，随后将数据写入命中的Cache line的对应位置
* Replace: 为了给Refill的数据空出位置而发起的一个读取Cache行的操作
* Refill: 将内存返回的数据(以及store miss待写入的数据)填入Replace空出的位置上

```{caution}
Tag表、V表和D表同一时刻至多接收一个读请求或写请求。
Data表情况略微有些复杂，因为来自一个读操作的LookUp和来自一个写操作的Hit Write可能同时发生。这样的例子为
```store   xxxx   load    xxxx```
当store指令写数据命中时，会将数据传给Write Buffer然后由这个Write Buffer来写入数据，同时执行下一条指令，如果下一条指令是store或load指令时，就会出现上面的情况。

至于为什么在写命中Cache和写入Cache之间引入一个Write Buffer，是出于时序方面的考虑，避免引入RAM输出端到RAM输入端的路径：Cache命中信息来自Cache RAM读出的Tag的比较结果，命中的写操作需要根据Tag的比较结果来生成Cache里的哪个路径，如果命中时直接写，就引入了Cache RAM的Tag读出到Cache RAM的Data写使能这一路径。

因此我们需要在LookUp和Data Write访问同一数据阻塞LookUp操作。
```

#### Cache模块功能边界划分

CPU流水线向Cache模块发送请求，Cache模块给CPU流水线返回数据或是写成功的响应。

axi_rw模块为内部访存总线到AXI总线的接口转换模块。这个接口转换模块对CPU内部有多个端口，对CPU外部只有一个AXI总线接口，我们把多个请求之间的仲裁、AXI协议的处理都放到这个模块中，这样Cache模块与这个AXI总线模块接口之间的功能划分依然是很简洁的，Cache模块向AXI总线接口发送请求，AXI总线接口模块返回数据或响应。

Cache模块向AXI总线发送的请求中的地址、操作类型、长度等信息一拍之内交互完毕即可。对于读操作，AXI总线接口模块每周期至多给Cache模块返回32位数据，Cache模块将返回的数据填入Cache的Bank RAM中或者直接将其返回给CPU流水线；对于写操作，Cache模块在一个周期内直接将一个Cache line的数据传给AXI总线接口模块，AXI总线接口模块内部设一个16字节的写缓存保存这些数据，然后以Burst方式发出去。

由于此时mem阶段可能会同时需要Axi总线读和写，因此将axi_rw模块对内接口调整为两组读接口和一组写接口。



### 将Cache集成到CPU中

需要修改的模块：
1. axi_rw模块。转换为两读一写
修改接口即可，去除```mem_req_i```信号，转而使用```mem_rvalid_i```和```mem_wvalid_i```信号来区分mem阶段的读请求和写请求，需要修改input信号和信号影响的组合逻辑以及状态机中状态的转换条件。
2. 增加pre-if模块，实现pre-if发送地址，if得到指令的时序
将if_stage模块中的new_pc作为inst_addr发出，
3. 在exe阶段发送访存地址，mem阶段获得结果，注意异常时需要无效信号
exe阶段需要检查mem阶段和wb阶段的exception_flag信号，如果异常则置```mem_rvalid```和```mem_wvalid```信号为无效
4. 增加cache模块，实现和cpu以及axi_rw的对接

此时axi_rw中读的数据宽度为64，写的数据宽度为128。

多次读取可以方便以后搞关键字优先

Cache和CPU流水线的交互接口
|  名称   | 位宽  |  方向  |  含义  |
|  ----  | ----  | ----  | ----  |
| valid |   1   |  IN    | 表明请求有效 |
| op    |   1   |  IN    | 表明请求类型，1为write，0为read |
| index |   8   |  IN    | 地址的index域(addr[11 : 4])|
| tag   |   52   |  IN    | 地址的tag域，从tlb（如果有的话）查询到的地址形成的tag，由于来自组合逻辑运算，故与index为同拍信号|
| offset|   4   |  IN    | 地址的offset域(addr[3 : 0])|
| wstrb |   8   |  IN    | 写字节使能信号|
| wdata |   64   |  IN    | 写数据|
| addr_ok |   1   |  OUT  | 该次请求的地址传输OK，读：地址被接收；写：地址和数据被接收|
| data_ok |   1   |  OUT    | 该次请求的数据传输OK,读：数据返回；写：数据写入完成|
| rdata |   64   |  OUT    | 读cache的结果|

Cache和AXI总线的接口
|  名称   | 位宽  |  方向  |  含义  |
|  ----  | ----  | ----  | ----  |
| r_valid |   1   |  OUT    | 读请求有效信号|
| r_addr    |   64   |  OUT    | 请求地址，1为write，0为read |
| r_type |   3   |  OUT    | 读请求类型|
| r_ready   |   1   |  OUT    | 读请求能否被接收的握手信号|
| r_valid   |   1   |  IN    | 读数据有效信号|
| r_last   |   1   |  IN    | 读数据为最后一个返回数据信号|
| w_valid|   1   |  OUT    | 写请求有效信号|
| w_strb |   8   |  OUT    | 写字节使能信号|
| w_data |   64   |  OUT    | 写数据|
| w_addr |   1   |  OUT  | 写地址|
| w_type |   3   |  OUT    | 写请求类型|
| w_ready |   1   |  IN    | 写请求能否接收的握手信号，此处要求ready信号要先于valid信号置起，valid信号看到ready后才可能置起，ready信号当AXI总线接口内部的16字节缓存为空时置起|

```{note}
将地址分为index、tag和offset传送为了后面添加tlb作准备，此时直接传入addr，后续添加tlb后需要添加tlb转换后的地址
```

#### 集成ICache

#### 集成DCache


可优化地方：
1. 读数据和tag时同时读取dirty位，然后replace时如果dirty位为0，则不用replace。
