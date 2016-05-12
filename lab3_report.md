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
	• 我们这里出现的” 指令位置” 的概念，你认为该概念是针对虚拟空间，还是物理内存所定义的呢？
	• 你觉得entry_point其值对于每个进程是否一样？该如何理解这种统一或不同？
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
	除非Cause寄存器的BD位置位(此时异常受害指令处于延迟操中)了，这种情况下EPC指向前一条（分支）指令。如果CPU是64位，那么EPC也是64位。

这样我们就不难理解，要保存的进程上下文中的env_tf.pc的值应该设置为old->cp0_epc。

## Thinking 3.6
	思考TIMESTACK 的含义，并找出相关语句与证明来回答以下关于
	TIMESTACK 的问题：
	• 请给出一个你认为合适的TIMESTACK 的定义
	• 请为你的定义在实验中找出合适的代码段作为证据(请对代码段进行分析)
	• 思考TIMESTACK 和第18 行的KERNEL_SP 的含义有何不同

1. TIMESTACK是时钟中断后存储进程状态的栈区。

2. 	从mmu.h中我们可以了解到TIMESTACK的值为`0x8200 0000`，而且从内存布局图中，我们可以看出，它的位置位于Kernel stack和Kernel text之间，是属于内核区的。

	在我们的实验中有两处显式用到TIMESTACK：

		env_destory():

		bcopy((void *)KERNEL_SP - sizeof(struct Trapframe),
				  (void *)TIMESTACK - sizeof(struct Trapframe),
				  sizeof(struct Trapframe));

		env_run():

		struct Trapframe *old;
		old = (struct Trapframe*)TIMESTACK - sizeof(struct Trapframe);
		bcopy(old, &curenv->env_tf, sizeof(struct Trapframe));
		curenv->env_tf.pc = old->cp0_epc;

	根据使用TIMESTACK的行为，无论是在env_run(),还是在env_destory()中，我们都是把TIMESTACK之下的一段大小为sizeof(struct Trapframe)的内存作为存储进程状态的地方.在env_destory()中我们把存于KERNEL_SP的进程状态复制到TIMESTACK中；在env_run()中,我们从TIMESTACK中取出进程状态复制给当前进程，至于机器什么把进程的状态存入TIMESTACK，我在下面会解释。

	在实验中还有一处隐式用到TIMESTACK：在include/stackframe.h中，我们可以发现，这样的代码：

		.macro get_sp
			mfc0	k1, CP0_CAUSE
			andi	k1, 0x107C
			xori	k1, 0x1000
			bnez	k1, 1f
			nop
			li	sp, 0x82000000
			j	2f
			nop
		1:
			bltz	sp, 2f
			nop
			lw	sp, KERNEL_SP
			nop

		2:	nop

		.endm

	这段代码是获取sp栈指针值，这里将其置为了0x8200000。关键来了，在1:中又把sp置为了KERNEL，也就是说，在有时调用get_sp时，sp置为TIMESTACK(运行2时),有时置为KERNEL_SP处存的值(运行1时)。至于什么情况下运行1，什么情况下运行2，原谅我才疏学浅，这里只给出我的一种猜测：可能与某个状态寄存器的相关位有关。

	可以看出,TIMESTACK 和 KERNEL_SP 有密切的关系。

	在这个头文件中还有`.macro SAVE_ALL`这个宏函数，阅读其中的汇编代码发现，其调用了`get_sp`并把各个通用寄存器和CP0中的寄存器按照Trapdrame的格式存入sp的栈区。

		.macro SAVE_ALL    
	                                  
			mfc0	k0,CP0_STATUS                   
			sll		k0,3      /* extract cu0 bit */  
			bltz	k0,1f                            
			nop       
			/*                                       
			 * Called from user mode, new stack      
			 */                                      
			//lui	k1,%hi(kernelsp)                 
			//lw	k1,%lo(kernelsp)(k1)  //not clear right now 
			           
		1:				
			move	k0,sp 
			get_sp      
			move	k1,sp                     
			subu	sp,k1,TF_SIZE                    
			sw	k0,TF_REG29(sp)                  
			sw	$2,TF_REG2(sp)                   
			mfc0	v0,CP0_STATUS                    
			sw	v0,TF_STATUS(sp)                 
			mfc0	v0,CP0_CAUSE                     
			sw	v0,TF_CAUSE(sp)                  
			mfc0	v0,CP0_EPC                       
			sw	v0,TF_EPC(sp)
			mfc0	v0, CP0_BADVADDR        
			sw	v0, TF_BADVADDR(sp)            
			mfhi	v0                               
			sw	v0,TF_HI(sp)                     
			mflo	v0                               
			sw	v0,TF_LO(sp)                     
			sw	$0,TF_REG0(sp)
			sw	$1,TF_REG1(sp)                    
			//sw	$2,TF_REG2(sp)                   
			sw	$3,TF_REG3(sp)                   
			sw	$4,TF_REG4(sp)                   
			sw	$5,TF_REG5(sp)                   
			sw	$6,TF_REG6(sp)                   
			sw	$7,TF_REG7(sp)                   
			sw	$8,TF_REG8(sp)                   
			sw	$9,TF_REG9(sp)                   
			sw	$10,TF_REG10(sp)                 
			sw	$11,TF_REG11(sp)                 
			sw	$12,TF_REG12(sp)                 
			sw	$13,TF_REG13(sp)                 
			sw	$14,TF_REG14(sp)                 
			sw	$15,TF_REG15(sp)                 
			sw	$16,TF_REG16(sp)                 
			sw	$17,TF_REG17(sp)                 
			sw	$18,TF_REG18(sp)                 
			sw	$19,TF_REG19(sp)                 
			sw	$20,TF_REG20(sp)                 
			sw	$21,TF_REG21(sp)                 
			sw	$22,TF_REG22(sp)                 
			sw	$23,TF_REG23(sp)                 
			sw	$24,TF_REG24(sp)                 
			sw	$25,TF_REG25(sp)                 
			sw	$26,TF_REG26(sp) 				 
			sw	$27,TF_REG27(sp) 				 
			sw	$28,TF_REG28(sp)                 
			sw	$30,TF_REG30(sp)                 
			sw	$31,TF_REG31(sp)
		.endm

	那么我们什么时候调用的SAVE_ALL呢？在lib/genex.S中有这样的函数，这个函数也就是在trap_init(lib/traps.c)异常向量组中设置的0号异常的处理函数 handler_int.（在其他的异常处理函数中也有SAVE_ALL的调用）

		NESTED(handle_int, TF_SIZE, sp)
		.set	noat

		//1: j 1b
		nop

		SAVE_ALL
		CLI
		.set	at
		mfc0	t0, CP0_CAUSE
		mfc0	t2, CP0_STATUS
		and	t0, t2

		andi	t1, t0, STATUSF_IP4
		bnez	t1, timer_irq
		nop
		END(handle_int)

	有没有豁然开朗的感觉，一切都在这里串起来了。我再按照时间顺序，捋一遍。首先产生timer产生时间中断,根据我们的中断向量组的设置，0号异常（时间中断）调用`handler_int`,在`handler_int`中我们调用SAVE_ALL,将当前CPU状态上下文以Trapframe的形式存入TIMESTACK，之后进入调度函数sched_yield(),其调用env_run(),调用下一个就绪进程;在env_run()中,我们在把状态存入curenv->env_tf中，完成上下文的保存。

3. TIMESTACK是时钟中断后进程状态的存储区，KERNEL_SP是系统调用后的进程状态的存储区。


## Thinking 3.7
	思考一下你的调度程序，这种调度方式由于某种不可避免的缺陷而造
	成对进程的不公平。
	• 这种不公平是如何产生的？
	• 如果实验确定只运行两个进程，你如何改进可以降低这种不公平？

首先贴出我的调度程序：

	void sched_yield(void)
    {
            static unsigned int count = -1;
            while(1)
            {
                    struct Env *e;
                    count = (count + 1)%NENV;
                    e = &envs[count];
                    if(e->env_status==ENV_RUNNABLE)
                    {
                            env_run(e);
                            break;
                    }
            }
    }

1. 由于我的调度`sched_yield`程序对进程的调度遍历是从`0~NENV-1`这样顺序遍历的。遍历的同时计时器也在运行，所以可能会造成这样的不公平：后面遍历所花的时间会计算到下一个就绪进程的时间片里,对下一个就绪进程不利。

	考虑这样的例子：有两个就绪进程envs[0]和env[1],0运行时时间中断到达，运行调度程序，会立即遍历到1，1开始运行，下一个时间中断到达后，运行调度程序，会进行`NENV-2`次循环才会遍历到0，而调度程序花的时间实际上是算到0里面的，这样就对进程0很不公平。

2. 如果实验确定只运行两个进程，只需遍历一次envs数组，找到这两个正在运行的进程，用静态变量记录这两个进程号，之后的进程调度只需在这两个进程之间切换就可以了。

# 实验难点
1. 在指导书中，rfe这条命令反复出现，但当我在`MIPS_Vol2（指令集）`中试图寻找此指令的更加详细的信息，却根本发现没有这条指令。通过进一步的寻找，我在`《see MIPS run Linux》`中发现这样的描述：rfe已过时，被替换为eret。并被描述为“在那些失传已久的原始CPU(`MIPS-I`)上，需要转移指令jr后接一条rfe作为延迟槽，切换回异常前的模式”。关于rfe的解释已经很少了。

	当然，紧跟着的Note也解释了MIPS3000中SR寄存器的功能描述。而`MIPS R3000`实现的也是`MIPS-I`指令集，所以我认为如果指导书在这里简要说明一下这些历史渊源比较好，否则在我们找资料的时候刚开始会比较晕。

2. load_icode_mapper()，这个函数我认为是本次实验的难点，我也多次修改这个函数才得到最后的正确结果。

	首先贴出我的代码：
	
		static int load_icode_mapper(u_long va, u_int32_t sgsize,
		                                                         u_char *bin, u_int32_t bin_size, void *user_data)
		{
		        struct Env *env = (struct Env *)user_data;
		        struct Page *p = NULL;
		        u_long i;
		        int r;
		        u_long offset = va - ROUNDDOWN(va, BY2PG);
		        if(offset>0)
		        {
		                if(page_alloc(&p)==-E_NO_MEM)
		                        return -1;
		                if(page_insert(env->env_pgdir, p, va, PTE_R)!=0)
		                        return -1;
		                if(bin_size<BY2PG-offset)
		                {
		                        bcopy(bin, page2kva(p)+offset, bin_size);
		                        i = bin_size;
		                }
		                else
		                {
		                        bcopy(bin, page2kva(p)+offset, BY2PG-offset);
		                        i = BY2PG-offset;
		                }
		        }
		        /*Step 1: load all content of bin into memory. */
		        for (; i < bin_size; i += BY2PG) {
		                /* Hint: You should alloc a page and increase the reference count of it. */
		                if(page_alloc(&p)==-E_NO_MEM)
		                        return -1;
		                // p->pp_ref++;
		                if(page_insert(env->env_pgdir, p, va+i, PTE_R)!=0)
		                        return -1;
		                bcopy(bin+i, page2kva(p), BY2PG);
		        }
		        /*Step 2: alloc pages to reach `sgsize` when `bin_size` < `sgsize`.
		    * i has the value of `bin_size` now. */
		        while (i < sgsize) {
		                if(page_alloc(&p)==-E_NO_MEM)
		                        return -1;
		                if(page_insert(env->env_pgdir, p, va+i, PTE_R)!=0)
		                        return -1;;
		                i += BY2PG;
		        }
		        return 0;
		}

	这个函数的功能是很好理解的，其核心就是要把bin段的内容加载到内存。

	但是，在做的时候还是遇到了很多问题。首先，是被注释坑了，注释规定了“前置条件：va是4K对齐的”；然而后来发现并不是这样的，不是4K对齐的情况下需要自己特殊处理。其次，对于分配页装不满的时候的处理，需不需要将空余部分置为0呢？后来发现，置0的操作其实是冗余的，在page_alloc()中就已经把新分配的页清了0。最后，就是要注意分配页和复制bin内容的范围和边界，这些值是很值得考虑和深思的，一不留神就会出错，而且出了错后，你很可能不知道是这个函数错了，难以定位bug。

# 残留疑点

1. struct Trapframe 中定义的寄存器类型是unsigned long,而不是unsigned int；咱们们的小操作系统是MIPS32，所有的寄存器也都是32位的，为什么要用64位的存储寄存器值呢？

# 感想与体会
这次的实验真的让我体会到什么叫做“蜜汁错误”。bug定位非常难，到现在调试的效率还是比较低。

由于lab3用到了lab2的代码，所以上次实验的错误可能传导到本次实验；而且如果你的结果出错，不一定是env.c的问题，还有可能是Makefile,ids,include.mk的错误，我在做lab2的时候就被lds坑了一次，替换了lds,但本次得到的lds还是错的，所以我推测，lab3从lab2继承的只有pmap.c；还有要注意的一点是，你和同学同样错误的实验现象,错误的原因和源头的位置可能大相径庭。

OS远端测试也是谜一样的结果。听同学反映，本地测试有问题，push上去也能得100，但我的测试只有20分，后来又听说分数和你提交的时间有关系，曾经有一段时间交上去都是100分。想想也是醉了。

本次实验的难度比上次实验有了进一步提高，耳闻lab4更加变态，猪脚们也已经给我们打了很多预防针，lab4的代码至今不对，指导书也不完善，还有恐怖的"蜜汁bug",感觉这两星期有的一拼了。