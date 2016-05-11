# OS lab3 实验报告

# 思考题

## Thinking 3.1:
	为什么我们在构造空闲进程链表时使用了逆序插入的方式？

env_init()的overview中有这样的提示：

Insert in reverse order,so that the first call to env_alloc() return envs[0].

逆序插入，以便首次调用env_alloc()可以返回envs[0],也就是env_alloc()可以顺序分配进程；

至于为什么需要返回0，顺序分配进程，我认为这样更符合人们的习惯，方便其他地方的编程；比如说在sched_yield()中，我对envs的遍历就是顺序遍历的，如果之前的env_alloc[]是逆序的，这里就会造成不公平，即分配时间片的顺序与进程分配的顺序相反，对先分配的进程不利。

## Thinking 3.2:
	思考env_ setup_ vm 函数：

	1. 第三点注释中的问题: 为什么我们要执行pgdir[i] = boot_pgdir[i]这个赋值操作?
	   换种说法，我们为什么要使用boot_pgdir作为一部分模板？(提示:mips 虚拟空间布局)
	2. UTOP 和ULIM 的含义分别是什么，在UTOP 到ULIM 的区域与其他用户区相比有什么最大的区别？
	3. (选做) 我们为什么要让pgdir[PDX(UVPT)]=env_cr3?(提示: 结合系统自映射机制)

1. 这个思考题我认为指导书已经说的很明确了，在这里我只是总结一下，用自己的语言进行复述：

	我们MIPS的虚拟地址空间采用的是2G/2G的模式，即高2G是内核区，这个区域的虚拟地址到物理地址的映射关系都是一样的，低2G是用户区，对于所有进程都有这样的虚拟空间；
	这也就意味着所有进程的页目录的一半都是一样的，所以我们可以用boot_pgdir作为一部分的模板。

2. 根据include/mmu.h中的图示和宏定义，我们可以知道：
	
	ULIM:User space limit,值为`0x8000 0000`;即用户空间的限制，也就是高2G(内核空间)和低2G(用户空间)的分水岭。

	UTOP:User space top,置为`0x7f40 0000`;与ULIM相差3个4MB(一个页目录项所映射的空间)，是用户进程可以控制的上限。

3. 这个操作是把，UVPT(进程的页表在虚拟空间的起始位置)所代表的4MB虚拟地址所在的页目录项，置为env_cr3所对应的物理地址；
	
	我们可以发现env_cr3与pgdir的关系是env_cr3 = PADDR(env_pgdir)，而env_pgdir代表的是进程页目录的内核虚拟地址，也就是说，env_cr3表示进程页目录的物理地址。

	所以这一操作的结果就是，指向UVPT（进程的页表）的页目录项等于进程页目录的物理地址，也就是实现了自映射。

## Thinking 3.3
	思考user_data 这个参数的作用。没有这个参数可不可以？为什么？
	（如果你能说明哪些应用场景中可能会应用这种设计就更好了。可以举一个实际的库中的例子）

可以看到，在函数`load_icode_mapper`中`void *user_data`被强制转换为`struct Env *env`,这也就意味着，这个用户数据指针，在函数中一直是当作一个进程指针在使用。观察这个`env`的行为(我们自己填的)，也正是我们所预料的，他在作为一个进程指针行使功能。

所以没有这个参数是不可以的，很难想象，没有进程指针，我们如何进行映射和内存装载。

我不太明白括号中这种设计具体指的是什么，我的理解是“使用`void *`类型作为形参，在函数体内在进行类型转换，来完成预定的功能”。这种设计在C语言标准库中很常见，比如我们常用的`void qsort(void*, size_t, size_t,int (*)cmp(const void*, const void*));`,在函数指针cmp的参数列表中，以`void *`为形参，实现qsort的通用性，与比较函数无关。类比发现，我们小操作系统中`load_icode_mapper()`的地位与`cmp()`很接近，都是作为函数指针被另一个函数(在我们的小操作系统中是`int load_elf(u_char *binary, int size, u_long *entry_point, void *user_data,int (*map)(u_long va, u_int32_t sgsize,u_char *bin, u_int32_t bin_size, void *user_data))`)以参数形式调用。

## Thinking 3.4
	思考上面这一段话，并根据自己在lab2 中的理解，回答：
	• 我们这里出现的” 指令位置” 的概念，你认为该概念是针对虚拟空间，还是物
	理内存所定义的呢？
	• 你觉得entry_point其值对于每个进程是否一样？该如何理解这种统一或不
	同？
	• 从布局图中找到你认为最有可能是entry_point的值。

1. 虚拟空间。在lab2内存管理中，我们学习到，“一切皆虚拟地址”。也就是说无论进程还是内核直接操纵的地址都是虚拟地址，虚拟地址转物理地址在硬件mmu中完成。

2. 我认为是不一样的。

	我们可以发现，entry_point的值是在`load_elf()`中修改的`*entry_point = ehdr->e_entry`;
	那这个`ehdr->e_entry`又是什么东西呢？追本溯源，在load_elf()的开头有`Elf32_Ehdr *ehdr = (Elf32_Ehdr *)binary`，而根据注释，binary是“an elf format binary file”，即“elf格式的二进制文件”，这行代码实际上是获得的elf的文件头；在`include/kerelf.h`中我们可以找到`Elf32_Ehdr结构体`的定义，找到e_entry,有这样的注释“Entry point virtual address”，这也成为了第一问的佐证。
	关于elf的文件头和相关域的作用，wiki百科上有更详细的解释:<https://en.wikipedia.org/wiki/Executable_and_Linkable_Format>

	结论就是，entry_point是在elf格式的二进制文件中写好的，每个进程的entry_point值是可以不一样的，只要它们可以在自己的elf文件头中设置自己的ehdr->e_entry。

	实验验证：在我们实验中所用到的elf是“code_a.c"和"code_b.c"这两个二进制文件，里面写的什么我也看不懂，但我通过printf输出两个进程的`ehdr->e_entry`发现，都是`0x4000b0`(这也为下一问提供佐证)，可见这两个进程的entry_point是相同的，但这也不代表所有进程的entry_point相同。

	这种entry_point不同可以为各个进程决定自己的程序入口提供方便，增加编程的灵活性。

3. entry_point的值最可能为`UTEXT(0x40 0000)+offset`,这里的偏移量也是有进程的elf文件头决定的，在我们的实验中 两个进程的偏移量均为`0xb0`.

## Thinking 3.5
	思考一下，要保存的进程上下文中的env_tf.pc的值应该设置为多少？
	为什么要这样设置？

先贴出我填的代码，

	struct Trapframe *old;
	old = (struct Trapframe*)TIMESTACK - sizeof(struct Trapframe);
	bcopy(old, &curenv->env_tf, sizeof(struct Trapframe));
	curenv->env_tf.pc = old->cp0_epc;

这4行代码是将当前进程的状态保存下来，以便下次切换到该进程时可以接着运行。关键点在这里的`old->cp0_epc`,让我们来看看《See MIPS run Linux》上是怎样解释`cp0_epc`这个寄存器的吧：

	异常返回地址(EPC)寄存器

	这是一个保存异常返回点的寄存器。导致（或遭受）异常的指令地址存入EPC，
	除非Cause寄存器的BD位置位了，这种情况下EPC指向前一条（分支）指令。如果CPU是64位，那么EPC也是64位。

这样我们就不难理解，要保存的进程上下文中的env_tf.pc的值应该设置为old->cp0_epc。

## Thinking 3.6
	思考TIMESTACK 的含义，并找出相关语句与证明来回答以下关于
	TIMESTACK 的问题：
	• 请给出一个你认为合适的TIMESTACK 的定义
	• 请为你的定义在实验中找出合适的代码段作为证据(请对代码段进行分析)
	• 思考TIMESTACK 和第18 行的KERNEL_SP 的含义有何不同

## Thinking 3.7
	思考一下你的调度程序，这种调度方式由于某种不可避免的缺陷而造
	成对进程的不公平。
	• 这种不公平是如何产生的？
	• 如果实验确定只运行两个进程，你如何改进可以降低这种不公平？


# 实验难点
1. 在指导书中，rfe这条命令反复出现，但当我在`MIPS_Vol2（指令集）`中试图寻找此指令的更加详细的信息，却根本发现没有这条指令。通过进一步的寻找，我在`《see MIPS run Linux》`中发现这样的描述：rfe已过时，被替换为eret。并被描述为“在那些失传已久的原始CPU(`MIPS-I`)上，需要转移指令后接一条rfe作为延迟槽”。关于rfe的解释已经很少了。

	当然，紧跟着的Note也解释了MIPS3000中SR寄存器的功能描述。而`MIPS R3000`实现的也是`MIPS-I`指令集，所以我认为如果指导书在这里简要说明一下这些历史渊源比较好，否则在我们找资料的时候刚开始会比较晕。

2. load_icode_mapper()，这个函数我认为是本次实验的难点，我也多次修改这个函数才得到最后的正确结果。

	首先，这个函数的功能是很好理解的，被核心就是要把bin段的内容加载到内存。

# 残留疑点

1. struct Trapframe 中定义的寄存器类型是unsigned long,而不是unsigned int；咱们们的小操作系统是MIPS32，所有的寄存器也都是32位的，为什么要用64位的存储寄存器值呢？

# 感想与体会
这次的实验真的让我体会到什么叫做“蜜汁错误”。bug定位非常难，到现在调试的效率还是比较低。

由于lab3用到了lab2的代码，所以上次实验的错误可能传导到本次实验；而且如果你的结果出错，不一定是env.c的问题，还有可能是Makefile,ids，include.mk的错误，我在做lab2的时候就被lds坑了一次，替换了lds,但本次得到的lds还是错的，所以我推测，lab3从lab2继承的只有pmap.c；还有要注意的一点是，你和同学同样错误的实验现象,错误的原因和源头的位置可能大相径庭。

OS远端测试也是谜一样的结果。听同学反映，本地测试有问题，push上去也能得100，但我并没有测试只有20分，后来有听说和你提交的时间有关系，曾经有一段时间交上去都是100分。想想也是醉了。

本次实验的难度比上次实验有了进一步提高，耳闻lab4更加变态，猪脚们已经给我们打了很多预防针，lab4的代码至今不对，指导书也不完善，还有恐怖的"蜜汁bug",感觉这两星期有的一拼了。