这是一个Linux-like的玩具内核,截止2018年5月10日,总代码量为11000行.<br>
ＷeiOS内核除了基本的段页管理,终端交互,中断和陷阱等支撑OS运行的一般机制,还有以下亮点:<br>
&#160; &#160; &#160; &#160;1.伙伴系统和slab机制来管理内存.<br>
&#160; &#160; &#160; &#160;2.电梯算法管理磁盘IO.<br>
&#160; &#160; &#160; &#160;3.动态优先级完成进程调度<br>
&#160; &#160; &#160; &#160;4.散列表管理inode和block缓冲区.<br>
&#160; &#160; &#160; &#160;5.写时复制(COW),进程间通信(IPC).<br>
&#160; &#160; &#160; &#160;6.read,write,pipe,open等基本的系统调用.(浏览kern/syscall.c可见所有可用系统调用)<br>
运行环境:<br>
&#160; &#160; &#160; &#160;亲测在ubuntu16.04和kali,gcc版本为5.4.0时可成功编译运行,其他环境未知...<br>
编译和运行方法:<br>
&#160; &#160; &#160; &#160;编译: scripts/updata_image.sh<br>
&#160; &#160; &#160; &#160;运行: scripts/run_bochs.sh<br>
&#160; &#160; &#160; &#160;组合技能: scripts/rebootWeiOS.sh<br>
![image](https://github.com/mynamevhinf/WeiOS/blob/master/images/Selection_003.png)
<br><br>

韦清福的内核开发手册
======
引导启动程序
------
###boot.S

这个模块是关于bootloader的,我们将使用"平坦模型"的内存,在我的设计中,大部分的初始化工作都交给内核函数来进行,因此bootloader的功能很简单,大概就是以下几点:
1.解锁A20地址线,这样我们才可以利用1MB以上的地址.
2.装载临时GDT,并开启段保护模式.
3.跳转到bootmain()函数,该函数的作用是将内核从磁盘中读入到内存.
我承认,这一部分的代码是借鉴了unix的启动程序,包括用宏的方式建立GDT段描述符,当然用纯汇编语言来进行也是可以的,工作量不大,但考虑到在之后创建TSS段是在任意的某阶段进行,用C来进行更加通俗易懂,所以我们一开始就用C函数来完成这项工作.

###bootmain.c

bootmain.c中的主要函数就是bootmain(),它根据kernel.img中elf头部给出的信息,将整个kernel从硬盘中读取到起始内存为0x10000的地方.相关链接<a href="http://man7.org/linux/man-pages/man5/elf.5.html"> Linux manual page-elf(5)</a>
1.将内核启始4096字节读入指定位置,这个区域实际包含了结构elf所需的信息,因为elf格式的可执行文件中,元数据会置于文件的最开端.
2.根据elf头所提供信息将每段读入相应的内存地址.
3.跳转到内核的入口点.
至此,常规意义上的bootloader已经结束.

###entry.S

entry.S其实就是内核入口点函数所在的文件,之所以没使用grub进行多重引导,是因为我懒得看一大堆文献,去遵循它的标准,所以选择了最直接粗暴的方式来进行,入口函数完成以下工作:
1.开启扩展页用于初始的页表.
2.装载也目录表,注意:不是严格意义上的也目录表,因为使用了扩展页.装载也目录表,注意:不是严格意义上的页目录表,因为使用了扩展页.关于使用扩展页时,我们可以直接绕过页表这一层,从线性地址到物理地址的转换也变得不一样,具体可以浏览<a href="https://pdos.csail.mit.edu/6.828/2017/readings/ia32/IA32-3A.pdf"> Volume 3 of the current Intel manuals</a>
3.开启页保护.
4.跳转到main()函数.来初始化各个模块.
我们可以告别启动阶段和晦涩难懂的汇编代码了,接下来的大部分工作将由我们熟悉的C语言来完成.
在之前的代码中,我从unix中学到了很多,我在寒假时对汇编语言和x86保护模式进行了深入的学习,所以在重写的时候并没有多困难.
之所以选择使用扩展页特性来搭建初始的页目录表,是因为在之前完成相关lab的时候遇到了这样的扩展,恰好我写这个内核的目的本来就是用于探索.
而之所以选择平坦内存模型的原因,是以为我认为有了页保护模式,再使用段保护有点多余,徒增烦恼罢了,况且书写平坦模式的内核也是我落下的一个作业----详情参见\<x86从实模式到保护模式\>第16章课后作业.



头文件备忘录:
---------  
​	elf.h -- Elf格式相关的数据结构</br>
        boot.h -- 用于boot.S文件</br>
        cmos.h -- 关于读写cmos的端口和各种映射区域的偏移量,例如基础内存和扩展内存.</br>
        mem.h -- 主要是OS的内存布局,包括一些界限的变量,例如USERTOP,KERNBASE等.</br>
        mmu.h -- 存储和内存管理有关的相关宏,例如PTE_P代表页表项的存在位,以及页大小的规定等.</br>
        types.h -- 我自己定义的一些类型.</br>
        x86.h -- 一些处理in,out等与x86平台密切相关的函数.</br>
	cga.h -- 和屏幕输出有关的声明以及宏定义.</br>
	console.h -- 终端管理的数据结构体</br>
	kbd.h -- 和键盘有关的宏定义,键盘操作的端口等.</br>
	pmap.h -- 与线性地址转换,物理地址分配等有关的宏函数和定义.</br>
	proc.h -- 和进程有关的数据结构和宏.</br>
	trap.h -- 声明了一些结构体, 用于特殊的中断管理机制.</br>
	stdarg.h -- va_start(), va_end()和va_arg().</br>
​	error.h -- 记录了ＷeiOS中所有的错误信息.</br>								
​	picirq.h -- 和8259A相关的定义,函数声明.<br/>										string.h -- 处理字符的函数,块内存操作函数--例如memset(), memmove().</br>
​	buddy.h -- 伙伴系统</br>
​	slab.h -- slab机制.</br>
​	time.h -- 系统时间.</br>
​	kernel.h -- 内核数据结构,目前只有双向链表,未来可能会增加红黑树.</br>
​	monitor.h -- 用于早期内核级别的调试.</br>
​	stdio.h -- 和C语言中的类似.</br>
​	trap.h -- 中断,异常的各类函数和定义.</br>

终端驱动
-----
首先要写的就是终端的输入输出函数,这包括向屏幕(0XB8000)读写函数,以及键盘的驱动.
在我的设计中没有串口通信这个模块,因为我觉得这实在是多余,我懒得花时间去阅读天书一样的文档,况且光是键盘那一块就让我够头疼了,所以这一部分的代码其实很简单,我用结构体struct tty_struct来管理唯一的一个终端,也就是控制屏幕的"控制终端",该结构体包括三个类型为struct tty_queue的成员,以及现在还未写的另一个结构体struct termios,用于管理终端输入输出的属性--例如文本是否回显到屏幕上.现在--2018年3月30日下午2点13分,我计划让管理属性的结构提兼容POSIX的标准,但以后我怎么实现我也说不准.
从键盘中读取的生(raw)数据,也就是扫描码,直接存到 read_buf 成员变量中,根据termios成员变量规定的属性,我们选择用将读取到扫描码转译为ASCII码或者保持原样,然后我们将处理后的数据写到 write_buf 中,write_buf 中的数据将直接写向屏幕,也可以用作其他认识,取决与调用的程序.
我们先来看关于键盘的部分,这里需要粗略解释一下kbd.h头文件.

**--2018年4月22号晚,我放弃了,我也没有写struct termios结构.因我觉得太鸡肋,以后上多用户系统的时候,只需要设置echo为0,不回显敏感信息就好了.**

###kbd.h & kbd.c

关于804X键盘控制器的各个端口:
0x60(r) -- 当键盘控制器收到来自 键盘的扫描码 或者 命令响应的时候,控制器将状态寄存器的位0设置为1,同时产生中断IRQ1.也就是说,我们读取键盘的扫描码是通过这个端口进行的,当且仅当状态寄存器的位0为1的时候!!!!!!!
0x60(w) -- 向键盘发送命令或参数,或者向键盘控制器写参数.
0x64(r) -- 状态寄存器
0x64(w) -- 向键盘控制器写命令,可以携带参数,但参数必须写入0x60(r)
讲老实话,我觉得写键盘的驱动好像给屎画肖像,真的很恶心,我一直在寻求空间和时间开销的完美平衡,但我到目前为止还是没有任何解决方案,有的需要很多的分支判断语句,这样违背了CPU的架构,比如预执行,如果用多个数组,对空间的负担就太昂贵了--虽然只是几百字节,但我不允许.所以最后我随便摘抄了一段还算可用的代码,我打算等上层模块全都构建完毕,再回过头来搞这些乱七八糟的驱动,和硬件打交道是我这辈子最不想干的事情,实际上,我认为我缺乏硬件编程的天赋,--2018/03/31 00:02

###cga.h & cga.c

就是向0xB8000的显存映射的内存区域写ASCII码,由于实在太简单了,我都懒得记录.

###console.h & console.c

头文件存放struct tty_* 的结构,以管理终端I/O,这里就懒得讲了.而.c文件中,我除了实现基本的终端操作,还添加了用于格式化打印的prink函数,和真正的printf相比,功能很简陋,主要是用于debug,还是绰绰有余的.为了实现格式化打印功能,我不得不自己手写了C语言不定参数的相关函数,我全部用宏来实现,这个也不难,因为C函数调用时候遵循一定的风俗,基本上从高地址到低地址存储的内容就是: 参数n, ..., 参数1, eip, ebp, 局部变量...,所以我们根据括号里最左边的参数,即可求得全部参数的起始地址,难点在不同长度的参数中,准确地找到每一个的起始地址,否则可能会读入错误的数据.**--后来我直接built in了,没有自己写.**
我实现了prink函数,这个取名来自linux(我猜是来自linux吧,在很早以前看过一本内核的书出现了这个函数).在这个函数的实现中,我第一实现了编译原理课本上抽象的概念,使用了scanner相关的技术.这个简易版的prink支持%x,%d,%p,%o,%u和自定义颜色的文字输出,使用'\033[(0|0digit|digit digit)m'为前缀,'m'和'['之间的数字代表高八位的属性,直接写到显存映射区中.例如'\033[07m'意思就是说黑底白字.



内存管理
----
###mem.h

weiOS中进程对内存的使用遵循一定的原则.每个进程独立分配了一个内核栈,当中断发生时,中断会在当前进程的内核栈中进行处理,内核栈大小为4kb.用户态的栈大小为4kb,顶端是0xd0000000,用户的.text部分从线性地址0x08048000开始,这么设计的理由是:似乎linux的内存布局就是如此,我想直接支持Linux的程序?
然后马一下自己设计的内存布局.作为一个内核开发的新手,我并不知道怎么设计内存的布局才是合理的,高效的,所以我xjb画了十分钟,那些零散的'Invalid Memory'的区域是用来充当哨兵的,防止进程对内核态内存消耗过大,如果一旦出现这种情况,例如使用内核栈超出限制,就会直接杀死进程.
每个进程的**UTOP**以上的部分都是共享的,但是该区域就不是我们常说的内核地址空间,因为从**USTACKTOP(0xd0000000)**--也就是用户栈的顶端开始才是.我这里在UTOP和USTACKTOP之间留下了广袤的区域,是因为我暂时还没想好如何利用.--2018年4月22日我终于想好怎么用了.

###buddy.c & buddy.h

今天是2018年4月3日,消极了两天之后,我决定开始写我的迷你buddy系统了.尽管打算上多处理器,但我还是选择一个node管理整个内存区域,因为整个内存只分配了128MB.而不像真正的Linux那样,为了权衡每个cpu访问内存的时间,用了多个node结构.
因为只用了一个node结构,所以就没有必要明显地声明和初始化了,甚至我的代码中都没有struct node这个结构.取而代之,我直接使用两个struct zone结构体,一个叫做kernel_zone理论上负责内核所需的内存,包括了各种描述符结构体,以及DMA传输所需的空间,另一个叫normal_zone,负责常规的内存.

我对于zone级别的内存分配策略很简单,因为我设定kernel_zone有更加重要的用途的,可能用于DMA传输,某些中断例程申请内存,内核数据结构等等,所以在每次直接向zone结构索要内存时,首先从normal_zone结构开始,除非在flag参数中明确标记了__GFP_DMA__或者__GFP_IRQ__,在这种情况下只能向kernel_zone申请内存.而当normal_zone内存低于结构成员pages_low之后,再考虑kernel_zone申请内存,初始化的时候,我设定**normal_zone.pages_low =  0**,所以意思就是里边的内存可以用光.而kernel_zone就不一样.

因为DMA传输只能识别低于16MB物理内存的地址,所以当标记__GFP_DMA__时,我们必须向kernel_zone申请内存.如果当前内存余量和申请内存大小之差,小与当前pages_low的值,内核就会陷入panic()函数中,就是凉凉的意思.

而__GFP_IRQ__是服务于中断例程的,本着中断例程不能阻塞,睡眠的原则,我们必须在kernel_zone的**保留区**中申请内存.所谓的保留区就是pages_low以下的内存区域(注意我在normal_zone中没有保留区),也只有标记了__GFP_IRQ__才能动保留区的内存.如果保留区内存不足以满足申请的时候,内存也会凉凉.

在zone中操作内存遵循**buddy**系统的基本原则,通过**__rmqueue()**来分配内存,通过**__free_pages_bulk()**来释放内存.一般来说我们不直接调用这两个函数,而是通过封装函数来间接调用.**注意:我在这两个函数中都不改动页面的p_ref域,我将假定上层函数对其值负责.**

详细说明这两个函数有助于后续上层函数的编写.在此之前先解释一下struct zone这个精简的结构,我把伙伴系统共分为**MEMLEVEL=11**个等级的连续内存块,用一个大小为MEMLEVEL的**struct free_area**数组来管理每个大小的若干内存块,与每个等级一一对应,而等级的意思是,比如我有等级为n=2的内存块,那个该结构管理的内存块的大小为2的n次方个页面.也就是 (2^n)*PGSIZE=16384 个字节,并且是连续的.也就是说最小内存块大小为1个页面, 4KB,而最大的内存块为 4MB .每个 struct free_area 结构包含两个成员,nr_free代表该结构管理的内存块的数目,而 free_list 是一个双链表,链表中的项是该结构管理的每个内存块中,第一个物理页面对应的struct page结构.

**__rmqueue(struct zone *z, int order)**：该函数首先从内存区管理结构数组 free_area 挑选出合适的区域(大小满足且非空),然后取出首页面的结构.如果这次分配导致大的内存区域分裂,就执行相应的分裂操作.**注意:当页面结构还属于zone时候,我用首页面的 p_private 成员来标记该内存块所属的内存区域,但是经过次函数分配时,我会将此域标记为 OUT_OF_BUDDY = 13,表示不属于任何区域**.

**__free_pages_bulk(struct page *page, struct zone *z, int order)**:将一定数目的页面归还给伙伴系统,如果该内存块的伙伴内存块也属于相同的 free_area 结构中,则层层合并,直到与伙伴天各一方.对于每个被合并的伙伴,我将其首页面的 p_private 值标记为**IN_BUDDY**标记当前页面属于一个伙伴中,但不是所属内存块首页面.

在我的内核中,每个对伙伴系统的内存操作都要先经过**per_cpu_pageset**－实际上就是缓冲区,这是基于cpu会缓存最近使用的内存区域提高运行速度的假设.按字面理解,虽说是每个cpu在一个zone结构中都含有cache,但是在这里我做了精简,所有cpu共用同个zone中的两个cache,一个是**hot cache**,另外一个叫做**cold cache**.**并且我直接无视了cold cache的存在,也就是说所有的分配,释放操作都对hot cache进行,因为我觉得对于128MB内存的内核来说,不必搞得太复杂.**最后增加区域中空闲页面的数目.

**在这里,我还是将更新页面结构 p_ref 域的工作交给上层函数.**

**buffered_rmqueue(struct zone *z, int order, gfp_t gfp_flags)**:在cache中分配内存通过该函数进行,分配成功则返回起始页面对象的地址,失败则返回0.当分配1个以上(不包含1个)页面的时候,将直接从其所属的zone结构中直接分配.根据gfp_flags中指定的cache(hot还是cold),我们从相应的 per_cpu_pageset 中分配内存.同时检测当前页面数减去申请页面数的值低于 pageset->low 时,就会自动向buddy系统申请内存先填冲缓冲区,然后再执行分配.任何越过pageset->low的事件都导致返回0,意味着失败.

**free_hot_cold_page(Page page, int cold)**:把内存从上层函数归还给cache系统.默认一次释放一个页面.用cold参数指明将页面释放到哪个cache中,如果cache中的页面数量大于高水位线 pageset->high,还要调用**free_pages_bulk()**把页面从cache归还到相应的zone中.

**free_pages_bulk(struct zone *z, struct list_head *page_list, int order)**:这个函数很容易和上面的**__free_pages_bulk()**函数混淆,但是功能截然不同.前面说道,我用struct per_cpu_pageset来作为cache的描述符,该描述符中也有叫做free_list的域,作用也是用双向链表把这个cache管理的物理页面串联起来.而非下划线开头的这个函数目前只在**free_hot_cold_page()**中被调用,具体作用是将2^order个页面从代表cache的free_list中归还给所属的zone.free_pages_bulk()在一个循环中调用了__free_pages_bulk()完成具体的页面归还工作.

这里有个问题:**我们如何从物理页面描述符中得到其所属的zone呢?**

**答案是,因为我只设置了两个zone,所以我把辨别的信息存在了page描述符的flag域中,若该域的0位为1,则代表该页面属于normal_zone中,反之属于kernel_zone.**

**free_hot_page()**和**free_cold_page()**是对**free_hot_cold_page()**的不同封装,意义如字面.为了简单,所有从上层归还给cache的内存都放入**hot cache**,这是我的规定--因为我认为对于WeiOS这种粗糙的小内核,cold cache存在的意义不大.

当然我也不可能直接通过cache的函数直接和伙伴系统交互.出于保护和封装的目的,buddy系统提供一些更加抽象的接口给外界,这些接口中最核心的就是**__alloc_page()**和**__free_pages()**函数.如字面所述一个分配一个回收.

**__alloc_page(int order, gfp_t gfp_flags)**:这个函数实现了我构想的对于zone的分配策略--先常规后内核,除非特别指明.我首先就是屏蔽了GFP_COLD这个标志,之所以不彻底取消cold cache的原因是我考虑到未来的扩展.然后依据标记一一测试.如果包含GFP_DMA,那么就自动从kernel_zone中分配内存,如果越过了保留区的内存,将导致失败.然后是保留区的内存分配,只有包含了**GFP_IRQ**才可以从保留区中分配内存(只有kernel_zone有保留区),而保留区内存的分配是比较特殊,它将越过cache这个层次,直接和buddy系统对话.所以我在zone 结构中特意保留了**pages_reserved**和**reserved_pages_list**两个成员.前者记录了保留区仅存的页面数,后者则是结点为页结构的双链表.如果是常规内存的分配(没有GFP_DMA和GFP_IRＱ),就会首先从normal_zone中分配内存,如果被该区域被消耗光了,才能染指kernel_zone.在这一步中,就算kernel_zone中非保留区内存被消耗光了,也不能干涉到保留区的内存.我这里规定,只有普通的分配操作才可以休眠等待,所以最后会判断GFP_WAIT是否被设置,如果被设置则,执行休眠.并且在normal_zone的队列中休眠.

**__free_pages(Page page, int order)**:用于除保留区以外内存的释放.

**__free_page(Page page)**:封装函数,如果页面属于保留区(page->flag & RESERVED_PAGE == 1),就调用**__free_rerserved_page()**释放页面--我在这个子函数中不再判断页面是否属于保留取,所以全靠_free_page()了.否则就调用**__free_pages()**.

接下来将实现**slab**系统.

## slab.h && slab.c##

我遵循Linux2.6的原则,但做了一些简化,我只能保证最基本的功能,和体系结合的性能方面我不关心,因为还没必要认真到那个地步,主要是做了以下改变:

1.我规定一个slab只能占有一个页面.

2.遵循DMA和常规内存分配分开的原则,因此WeiOS还是有两个kmem_cache结构的数组.但是我把对象的大小作了裁剪,分为32, 64, 128, 256, 512, 1024, 2048, 4096,共8种,也就是说内核中共有16个用于一般内存管理的kmem_cache结构,**后8个负责DMA内存分配**.如果中断程序需要在保留区中分配内存,必须直接和buddy系统的cache交互,不通过slab.

3.所有的slab描述符结构都由一个单独叫做**meta_cache**的kmem_cache结构管理,也就是说它所管理的对象大小就是slab描述符--struct slab结构的大小.**而meta_cache本身的slab结构,则存在每个slab的开头.**

4.我规定对象描述符的数据结构是ushort = unsigned short.当对象为可用的时候才有意义,其存储了下一个可用对象在slab中的索引.最后一个可用对象的描述符的值为 **BUFCTL_END = 0xFFFF**表示已到尽头.

5.我认为如果把对象描述符们一致地存储到slab描述符中的做法不可取,对于32字节大小的对象来说,光是存储其对象描述符就需要花费256个字节.倘若每个slab描述符都预留这么多的空间,那么对于对象大小为4096字节的slab描述符(只需要一个对象描述符)来说,浪费的空间也太多了.所以我想出一个折中的方案:对象大于等于512字节,就把对象描述符存储在slab描述符中,反之,则将这些对象描述符存在存储对象的页面中,放置在最开始的字节里.

**为了方便,如果一个物理页面被分配给了slab系统,那么我就把其对应结构page中,置lru.prev为所属cache的描述符地址,lru.next为所属slab的描述符地址**,因为lru成员只有在伙伴系统中才做为双链表的节点,发挥实际意义.

**void kmem_cache_init(kmem_cache_t cache, uint32_t obj_size, gfp_t gfp_flags)**:根据所给出的参数初始化kmem_cache_t结构体,没啥好说的.

**struct page *kmem_get_page(kmem_cache_t *cachep, gfp_t flags)**:通过调用伙伴系统提供的接口函数**alloc_page()**为cachep所指向的结构分配一个新的物理页面,然后调用**set_page_slab()**将新分配页面的flag域置PAGE_IN_SLAB,该函数一般用于**cache_grow()**创建一个新的slab.元缓冲区和一般缓冲区共用这个函数.

**void kmem_free_page(void *vaddr)**:接受虚拟地址参数,将该虚拟地址所对应的物理页归还给伙伴系统.同时调用**clear_page_slab()**清空PAGE_IN_SLAB.元缓冲区和一般缓冲区共用这个函数.

**struct slab *alloc_slab_desc(kmem_cache_t *m_cache_ptr)**:

**void destroy_slab_desc(kmem_cache_t *m_cache_ptr, struct slab *slab)**:

**struct slab *cache_grow(kmem_cache_t *cachep)**:先调用alloc_slab_desc()来分配一个slab描述符,如果失败,则直接退出.然后调用kmem_get_page()函数获得一个物理页面,如果失败就调用destroy_slab_desc()摧毁上一步成功分配的slab描述符,退出函数.当以上两项完成之后,初始化page和slab结构相关信息,**无须把新分配的物理页面清0,因为我在下层伙伴系统的函数中已经做过了--我规定每个cache的gfp_flags标记都要包含GFP_ZERO**.然后根据cachep给出的对象大小,选择将对象描述符放在slab描述符内部中,还是其他cache中.如果是放在slab描述符内部,那么初始化objs_desc数组,并且把exte-　rnal_objs_desc置为空,表示对象描述符在内部.否则,就向其他cache申请空间,并将external_objs_desc置为所得空间的起始地址,如果申请失败,则调用**slab_destroy()**摧毁slab.

**void slab_destroy(kmem_cache_t *cachep, struct slab *slabp)**:顾名思义,摧毁一个完整的slab,**不判断该sl-ab的状态,也就是说,尽管属于slabs_partitial或slabs_full,一并删除.**所以上层函数必须负责判断.依次调用mem-set()将所属的页清空,list_del()将slab从所属队列中删除,destroy_slab_desc()将slab描述符和对象描述符删除,再把对应page结构的lru.next和lru.prev设置为空,表示其已不属于slab系统,最后调用kmem_free_page()把页面归还给伙伴系统.

### pmap.c

这个文件夹的函数们对slab系统的函数进行封装,提供了虚拟内存管理所需的全部函数.

首先我们得搞清楚我们有多少内存,这通过调用**mem_dect()**函数来完成,该函数又通过借助"x86.h"中封装的指令函数来读CMOS上的内存区域来取得相关信息.与内存容量相关的偏移量分别是:基础内存--0x15,0x16,小于16MB的扩展内存--0x17,0x18,以及扩展内存--0x34,0x35.在这些不同数据偏移对中,前者代表低八位，后者代表高八位,并且返回数值的单位是KB,更多关于cmos读写的信息可以浏览:<a href="http://bochs.sourceforge.net/techspec/CMOS-reference.txt"> cmos memory map</a>

未完待续...

##进程管理

记录一下关于进程创建和销毁等等的实现.

###proc.h

**struct context**:是用于进程调度的,也只有进程调度时候它才有用,所以我在这里不叙述.

**struct proc_manager**:顾名思义,管理系统内的所有进程,也就是所有cpu下的进程都包括在内,总数不得超**N-PROC = 1024**,也就是WeiOS内总进程数是1024,这是我随便乱选的数字.**proc_table_lock**是个自旋锁,消除竞争.**n_procs_alive**是当前系统内存在的进程总数,当申请一个新进程时,若该值已为NPROC,则拒绝请求.**id_bitmap**用于分配和回收ID,其实际上是32个元素的uint32_t类型的数组,因为32*32=1024,所以每一位代表一个ID,当某位为0时,代表该ID已经分配了,反之则表示ID可用于新进程.**proc_table**是NPROC个struct proc结构指针的数组,当某ID被分配时,把对应的指针设为新的进程描述符的地址,若ID未使用,则为null,即空指针.因为每次分配和回收进程描述符需要调用到slab内存管理系统的函数,开销太过巨大,所以我设置了一个节点为空闲进程描述符的双向链表**procs_desc_cache**,每次回收进程描述符的操作将对象放到该链表中,而不是直接归还给内存管理系统.所以每次申请新的进程描述符时,首先向该链表申请,若该链表为空,再调用kmalloc()向内存管理系统申请.

**struct proc**:进程描述符,没啥好说的.

**struct cpu**:每个CPU都拥有一个此结构,维护每个CPU所需的信息,懒得写了,重点讲下**run_queue**和**exha-　usted_queue**,请看下面:

**struct proc_queue**:O(1)进程调度的核心数据结构.**n_procs**代表该队列中的进程数,如果run_queue中该值为0,那么就将run_queue和exhausted_queue互换,这样一来进程调度函数就会从已经把时间片消耗光的进程中选择一个对象,来进行切换.**priority_bitmap**,是为优先级的位图,之后再解释.**procs_in_queue**是链表头的数组,共40个元素,和40个优先级一一对应.

**enum proc_status**:进程的各种状态,**RUNNABLE**表示进程在某个cpu的run_queue队列中,等待被调度.**RU-NNING**表示该进程正在运行.**READY**描述进程刚被创建却没有准备好运行的状态.**SLEEPING**表示进程正在休眠,等待某事件的发生,并处于该事件的等待队列中.**DYING**表示进程描述符中的killed已被设置为1,处于这种状态的进程将在其下一次从内核态返回到用户态时被杀死--通常不会太久,因为有时钟中断.**ZOMBLE**表示进程已死,占用的内存已被释放,但进程描述符还驻留在内存中,直到父进程调用wait()函数,然后再把描述符释放.**FREE**表示该进程描述符不对应任何存在的进程,是可分配的.

###proc.c

**proc_init()**:初始化进程管理者**proc_manager**(实际上是struct proc_manager的一个对象,我懒得起名了),以及CPU中有关进程管理的部分.

**get_pid()**:从上面所述的进程管理者手中,获取一个新的可用ID.该函数不加锁,也不判断当前进程总数是否达到限制,并把这些工作都交给上层函数.通常是proc_alloc().

**clear_pid()**:与get_pid()工作相反,将pid归还到进程管理系统中.同样把权限检查等工作交给上层函数.

**get_proc_desc()**:申请一个进程描述符结构,首先检查proc_manager中procs_desc_cache链表是否为空,若非空,则从里边获取一个节点,是为新的描述符,并清空里边的信息,返回.若为空,则必须从slab系统中分配一个新的对象,k-malloc()选项中包含__GFP_ZERO,意味着清0,然后返回.

**proc_desc_destroy()**:首先调用clear_pid()回收pid,然后把proc_manager中proc_table对应的元素置为0,把进程状态标记为FREE(可用),然后将进程描述符加入procs_desc_cache链表中.已备下次分配.

**pid2proc()**:首先判断pid参数,若为0则直接返回当前进程的描述符结构,因为我不允许任何进程影响到真正的0号进程.判断所给pid是否合法,若合法则返回对应的进程描述符.如果check参数为1的话,还会执行权限检查,通过条件是:所选进程为当前进程或所选进程的父进程为当前进程.

**proc_setup_vm()**:为新进程创建页目录表,并且把用户空间全设置为0,然后从kern_pgdir-内核页目录表复制内核空间信息到新目录表中.因为进程在用户空间时无权干涉内核内存空间,所以简单地复制页目录表项即可.**注意:我不在这里分配内核栈空间,也不分配用户栈.**

**proc_region_alloc(struct proc *p, void *va, size_t len, int perm)**:为进程分配内存区域,通过一个循环调用page_alloc()分配新的物理页面,再调用page_insert()把新物理页面映射到由va指定的虚拟地址中.**随后我会用这个函数来分配内核栈,所以这里允许所分配的虚拟地址大过UTOP.**

**page_free()**:把进程占据的所有内存空间回收,包括内核栈.只保留进程描述符.

**proc_alloc()**:首先调用spin_lock_irqsave()获取进程表的锁,再判断系统内当前进程数目是否突破限制,若突破则直接解锁退出.然后依次get_proc_desc(),获取一个全新的进程描述符,proc_setup_vm()复制内核空间信息,把用户空间全部设置为0,get_pid()获取一个pid.初始化进程各项基本信息,把status设置为**READY**,**注意:我这里不设置时间片,优先级等信息,因为这些信息依赖于新进程是通过合种方式被创建的,例如fork()或者内核手动初始化0号进程,每种情况下分配都不一样**.**调用proc_region_a-lloc()分配内核栈空间.**解开进程表的锁.接着初始化进程返回信息,这一步很重要,虽然说决定新进程第一次进入用户态时从哪个点开始执行可能依各种不同情况而异,但是,在这里,必须统一设置forkret和trapsret,让新进程好像是因为中断/异常从内核态返回的"老"进程一样.

**load_binary()**:主要用于0号进程的创建.其实际功能是:根据elf头部信息调用proc_region_alloc()把与内核绑定的程序加载到指定进程的用户空间内.并把进程tf中的eip设置为程序入口点,如此一来,当新进程第一次被调度程序选择时,就会从该入口点开始执行.

**proc_create()**:主要用于0号进程的创建.调用proc_alloc()分配进程描述符,进程地址空间,进程内核栈,load_binary()把程序加载到新进程的用户空间.并返回.

**WeiOS_first_process()**:以该函数创建WeiOS的0号进程,首先proc_create()创建将其程序代码,数据段加载到用户空间,并完成基本初始化工作.proc_region_alloc()创建用户栈.接着就是初始化tf中各种寄存器的值,把进程状态设置为**RUNNABLE**,时间片,优先级设置为默认的.最后调用add_proc_to_queue()把0号进程塞到cpu的run_queue中,等到我在init()中执行scheduler()函数,0号进程便可以运行了.

##进程调度

今天是2018年4月6号,沉沦了两天.两个月前我在阅读\<x86从实模式到保护模式\>这本书的时候,我天真的以为当今操作系统进行任务切换是通过调用任务门的形式来进行的,但当我看到\<ULK\>的时候,我意识到了自己的愚蠢.

所以我打算采用的进程调度机制借鉴了别人的.我觉得整套机制非常的巧妙,下面码一下我个人对这套调度机制的理解.注意这里只是机制而不是策略,我的内核中使用的策略是类似与Linux2.6中的那个复杂度为0(1)的算法.

###ＷeiOS的进程调度机制

首先是struct context这个结构,我给它起名叫上下文结构.该结构是整个机制的核心--它通过swtch()函数保存旧进程edi,esi,ebx,ebp和eip的值到旧进程的内核栈中,并从新进程的内核栈中加载这几个寄存器.其中edi,esi,ebx,ebp的存取是通过push/pop指令显式地进行,对于eip的存取比较隐蔽,是通过每次调用swtch()函数的call/ret指令来进行的.这也是上下文结构的布局中,eip处于最高地址的原因.

上下文结构之后,接下来就是每个cpu私有的**scheduler context** -- 我取名为**调度上下文**.

调度上下文实际上就是一个上下文结构,它比较特殊,不属于任何进程,而只属于**scheduler()**这个函数.这么说很奇怪,但事实就是如此,实际上这就是我的调度机制的核心部分.系统规定每次执行任务切换时(除了初始化阶段切换到initprocess-0号进程),都要遵循以下步骤:**1.从旧进程上下文切换到调度上下文. 2.在调度上下文中选择新的进程. 3.从调度上下文切换到新进程的上下文.**

步骤1是在**sched()**函数中执行的,除了内核初始化阶段,之后任何显式或隐式的进程切换/调度,都必须经过这个函数.该函数先检查相关权限,然后执行步骤1.这里有一个小技巧,每次从调度上下文切换到进程上下文时,保存在调度上下文中的eip一定属于**scheduler()**函数中**switchkvm()**语句编译后的若干指令中的第一句.如何做到这一点将在步骤3的解释中讨论.切换到调度上下文之后,第一件事就是加载内核页目录表,主要的原因就是使用CPU自己的内核栈--尽管内核栈逻辑地址相同,通过不同页目录表的映射却可以转换成不同的物理页面.

步骤2很简单,如何选择新的进程就涉及到进程调度的策略了,留到以后再说.那么调度上下文是怎么初始化的呢.其实是通过init.c中的main()函数,没错,就是我们初始化内核的整个函数,最后一条语句就是调用scheduler().该函数直接调用**scheduler()**而不是通过**sched()**来切换到init进程,实际上也是隐式地初始化了调度上下文.

步骤3亦很简单,调用了**switchuvm()**把新进程相关的环境加载到系统中.该函数主要工作是,更新当前CPU的GDT中关于TSS段的描述.整个cpu的所有进程共用一个TSS段,这意味着**1.每次进程切换必须更新该段内容. ２.我们不使用TSS段来存取进程的状态--实际上我们通过struct trapfram结构**. 3.每次进程内核栈都会变为未使用状态,也就是说指针esp0指向栈顶.更新TSS段之后,加载进程页目录表.回到scheduler()中,接着调用swtch()函数切换到新进程上下文,也就是在这,调度上下文的eip值被保存为指向下一条语句--**switchkvm()**.

至此,整个进程调度机制就介绍完毕了,然后就是调度策略了.

###WeiOS的进程调度策略

Linux2.4调度算法的缩小版,基于优先级和时间片的再实现.如果一个进程花费大量时间在休眠上,那么系统认为该进程是一个需要交互的进程.反之,若进程花费大部分时间片用来执行代码,那么就称为消耗型进程(好像是这个名字).对于交互型进程我给予它们越来越高的优先级(数值上越来越低),毕竟谁都不想敲一下键盘,半天才看到字符显示在屏幕上吧.

和LINUX一样,每个进程的时间片是动态调整的,但是我没有管什么static_priority,priority和nice,我只设置了一个priority,范围是0-39,对应到nice的-20-19.数值越低优先级越高,时间片也越长.每次进程进入时间片被消耗光而进入调度上下文scheduler的时候,就会根据sleep_avg的值重新计算优先级和时间片,由此来达到公平的时间片分配.我也不知道原理是什么,这可能需要数学上的证明,比如如何防止进程过度饥饿等等问题,以后我回去研究的.

未完待续...

## 文件系统

和以前一样有个ide_manager结构管理磁盘的驱动部分.该结构内部包含了io等待队列,该队列中块的数目和一个自旋锁.为了使事情简单,我规定单次io操作只进行块为单位进行传输.

**ide_wait()**:等待磁盘操作结束并判断出错,若出错反馈出错信息.

**update_idequeue(struct buf *b)**:把b插入等待io操作的队列中,为了减少寻道时间等的开销,我写了简易的电梯算法,不包含合并机制.

**ide_init()**:初始化系统中的磁盘,并检测是否有第二快磁盘,若存在则把**slave_disk_existed**设置为1,该全局变量在之后非常有用,用于检测申请io的块是否合法.然后初始化ide_manager的各种信息.

**ide_start(struct buf *b)**:根据块中的**flag**标志位判断要进行的操作,若为**B_DIRTY**则进行写操作,否则进行读操作.**因此B_VALID和B_DIRTY两个标志位不能同时存在.上层函数在调用该函数时必须先设置相应的标志位,因为在此函数中我不进行判断!!!**

**ide_intr()**:处理硬盘中断的函数.首先加个锁,保证操作的"原子性",然后判断该中断是否为某种巧合,即判断ide_m-anager.n_requests是否为0,若是则解锁返回.**我规定io操作从ide_queue中的头元素到尾元素顺序执行.**所以若ide_queue中块个数不为0,那么这次中断肯定是由于头节点操作完成而引起的.故我从队列中将头元素移除,更新相关值.如果头元素的操作为写(通过判断标志位是否置B_DIRTY),则从磁盘数据端口中把数据读到块对应的数据区.**我又规定,每次IO操作完成后,都把B_VALID位并且清空B_DIRTY位**,原因显而易见嘛.借着唤醒等待该块完成io操作的进程.最后,若io等待队列中还有块在等待,则立即调用ide_start()开始传输.

**ide_read_write(struct buf *b)**:我把读写放到一块,判断若块已设置了B_VALID则直接返回,毕竟你都有有效了还读个屁,逗老子吗?**所以只有B_DIRTY和空白标志位能进行IO队列,前者表示其需要写,后者表示该块是刚分配的,需要读**．若块标记的设备为附加硬盘而附加硬盘不存在,返回-1报错.然后就是常规操作了,调用update_id-equeue()把块丢到io队列中的合适地方.如果放入之后队列中只有它一个元素,那么调用ide_start()立即开始IO操作.

然后陷入一个循环,睡眠等待标志位在**ide_intr()**中设置为B_VALID.解锁退出.

###块缓冲区

WeiOS中的块缓冲区不是在代码中就分配好的,而是启动时动态分配的.大小为NBUF=60个块.我用struct blk_cache结构的对象--**bcache**来管理整个块缓冲区.

**buffer_init()**:初始化bcache的相关信息.调用kmalloc()在常规内存区分配NBUF个struct buf结构体,在低16MB内存区域中分配等数量的大小为512字节的数据区.(通过向kmalloc()传递__GFP_DMA标记就可以完成低16MB内存的分配).最后把这些块添加到**bcache.free_list_head**代表的队列中,以供分配.

**getblk(uint32_t dev, uint32_t blockno)**:顾名思义,从块缓冲区中获取相应的块.分两类大情况.首先在哈希表中搜寻,若找到,判断该块是否被别的进程占用--B_BUSY是否置位.若是,则休眠等待,**并在休眠结束后跳到循环开始地方重新开始搜寻.这是为了避免竞争,若等待的进程被唤醒和取得控制权的间隙,该块被其他进程所用,那么就会产生冲突.毕竟在休眠过程中放弃了blk_cache_lk这个自旋锁.**若该块在哈希表中,且不忙碌,则置B_BUSY,并设置ow-ner为当前进程.**然后把块从free_list_head代表的队列中删除.**

接下来是第二种大情况,如果我们第一步在哈希表中没有搜寻成功.那就需要在自由队列中进行操作了.首先判断自由队列是否为空,若是则休眠.若否则将选择首元素作分配.将首元素从自由队列中删除.

未完待续...

