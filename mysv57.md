# 简单实现riscv架构下linux内核对Sv57的扩展以及使用qemu+gdb调试

## 环境配置

1.plct实验室汪老师的文章

> 在 QEMU 上运行 RISC-V 64 位版本的 Linux - 汪辰的文章 - 知乎 https://zhuanlan.zhihu.com/p/258394849

搭建好基本的riscv内核的运行环境以及编译工具链

2.要修改的代码  https://github.com/RV-VM/linux-napot-support.git

3.需要生成compile_commands.json（vscode在linux上表现的很拉，我使用的是nvim+coc），具体方法是安装bear，将1中两个编译linux内核的命令改为

```text
$ bear make ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- defconfig
$ bear make ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- -j $(nproc)
```

在这里强烈安利汪老师的知乎，，，如果有时间建议去看一下，，，会避免很多坑

## 代码分析

​		这是Sv57的实现草案 https://github.com/riscv/virtual-memory/tree/main/specs，简单来说就是使rv架构下面的linux内核支持56位物理地址。clone下来的这个版本为sv39，只有三层页表：pte、pmd、pgd，而扩展后的sv57则是五层页表：pte、pmd、pud、p4d以及pgd。具体页表的偏移量麻烦自行在草案里面确认。

​		接下来就是看代码的环节，整个实现事实上比较简单，主要是对一些宏的添加和新增一些页表初始化的函数。先看看启动过程，`arch/riscv/kernel/head.S`这个汇编代码展示了整个系统启动的过程，比较关键的信息在_start_kernel段中，如下：

```call setup_vm
       call setup_vm
#ifdef CONFIG_MMU
	la a0, early_pg_dir
	XIP_FIXUP_OFFSET a0
	call relocate
#endif /* CONFIG_MMU */

	call setup_trap_vector
	/* Restore C environment */
	la tp, init_task
	sw zero, TASK_TI_CPU(tp)
	la sp, init_thread_union + THREAD_SIZE

#ifdef CONFIG_KASAN
	call kasan_early_init
#endif
	/* Start the kernel */
	call soc_early_init
	tail start_kernel```
```

​		这个是和虚拟内存初始化有关的汇编部分，需要注意的就是这里riscv和网上内核相关教程比较常见arm体系有些许不同：arm在进入start_kernel之前仅仅只是初始化了物理内存块，并且在start_kernel完成初步虚拟内存的初始化；而riscv是在汇编时期调用setup_vm函数初始化虚拟内存（代码第一行），这一点需要注意一下。

​		基本上我们核心的任务就是处理setup_vm部分以及之后到MMU初始化完毕的这部分代码。

​		先来找setup_vm，在`arch/riscv/mm/init.c`文件当中，在这个函数当中我们主要关注建立fixmap和trampolinemap的部分，如下

```pt_ops.alloc_pte = alloc_pte_early;
	pt_ops.alloc_pte = alloc_pte_early;
	pt_ops.get_pte_virt = get_pte_virt_early;
#ifndef __PAGETABLE_PMD_FOLDED
	pt_ops.alloc_pmd = alloc_pmd_early;
	pt_ops.get_pmd_virt = get_pmd_virt_early;
#endif
	/* Setup early PGD for fixmap */
	create_pgd_mapping(early_pg_dir, FIXADDR_START,
			   (uintptr_t)fixmap_pgd_next, PGDIR_SIZE, PAGE_TABLE);

#ifndef __PAGETABLE_PMD_FOLDED
	/* Setup fixmap PMD */
	create_pmd_mapping(fixmap_pmd, FIXADDR_START,
			   (uintptr_t)fixmap_pte, PMD_SIZE, PAGE_TABLE);
	/* Setup trampoline PGD and PMD */
	create_pgd_mapping(trampoline_pg_dir, kernel_map.virt_addr,
			   (uintptr_t)trampoline_pmd, PGDIR_SIZE, PAGE_TABLE);
#ifdef CONFIG_XIP_KERNEL
	create_pmd_mapping(trampoline_pmd, kernel_map.virt_addr,
			   kernel_map.xiprom, PMD_SIZE, PAGE_KERNEL_EXEC);
#else
	create_pmd_mapping(trampoline_pmd, kernel_map.virt_addr,
			   kernel_map.phys_addr, PMD_SIZE, PAGE_KERNEL_EXEC);
#endif
#else
	/* Setup trampoline PGD */
	create_pgd_mapping(trampoline_pg_dir, kernel_map.virt_addr,
			   kernel_map.phys_addr, PGDIR_SIZE, PAGE_KERNEL_EXEC);
#endif

	/*
	 * Setup early PGD covering entire kernel which will allow
	 * us to reach paging_init(). We map all memory banks later
	 * in setup_vm_final() below.
	 */
	create_kernel_page_table(early_pg_dir, true);

	/* Setup early mapping for FDT early scan */
	create_fdt_early_page_table(early_pg_dir, dtb_pa);

	/*
	 * Bootime fixmap only can handle PMD_SIZE mapping. Thus, boot-ioremap
	 * range can not span multiple pmds.
	 */```
```

关于fixmap为内核虚拟地址和物理地址的一些固定映射（一些在之后完成mmu时会被清除），trampoline为内核在虚拟地址的映射位置，用于储存内核代码。

之后我们可以看到，需要使用到create_xxx_map()这个时候我们的事情就来 了，需要在原来的结果中加入对接下来页表的支持。

先来看create_pte_map和create_pmd_map两个函数

```static void __init create_pte_mapping
static void __init create_pte_mapping(pte_t *ptep,				      
				      uintptr_t va, phys_addr_t pa,
				      phys_addr_t sz, pgprot_t prot)
{
	uintptr_t pte_idx = pte_index(va);

	BUG_ON(sz != NAPOT_CONT64KB_SIZE && sz != PAGE_SIZE);
	if (sz == NAPOT_CONT64KB_SIZE) {
		do {
			ptep[pte_idx] = pte_mknapot(pfn_pte(PFN_DOWN(pa), prot), NAPOT_CONT64KB_ORDER);
			pte_idx++;
			sz -= PAGE_SIZE;
		} while(sz > 0);
		return;
	}

	if (pte_none(ptep[pte_idx]))
		ptep[pte_idx] = pfn_pte(PFN_DOWN(pa), prot);
}```
```

这里输入的参数中

- pte_t为一个pte的指针，用来表示页表
-  va为对应的虚拟地址
- pa为对应的物理地址
- sz为需要进行map的内存大小
- prot为内存保护

在8-15行为处理snapot的情况，我们主要关注17行之后，pte_id为对应表项在页表中的位置pfn_pte为将物理页表块号（pfn）映射为虚拟地址（pte）并且存在页表（ptep）的映射中

```create_pmd_map
static void __init create_pmd_mapping(pmd_t *pmdp,
				      uintptr_t va, phys_addr_t pa,
				      phys_addr_t sz, pgprot_t prot)
{
	pte_t *ptep;
	phys_addr_t pte_phys;
	uintptr_t pmd_idx = pmd_index(va);

	if (sz == PMD_SIZE) {
		if (pmd_none(pmdp[pmd_idx]))
			pmdp[pmd_idx] = pfn_pmd(PFN_DOWN(pa), prot);
		return;
	}

	if (pmd_none(pmdp[pmd_idx])) {
		pte_phys = pt_ops.alloc_pte(va);
		pmdp[pmd_idx] = pfn_pmd(PFN_DOWN(pte_phys), PAGE_TABLE);
		ptep = pt_ops.get_pte_virt(pte_phys);
		memset(ptep, 0, PAGE_SIZE);
	} else {
		pte_phys = PFN_PHYS(_pmd_pfn(pmdp[pmd_idx]));
		ptep = pt_ops.get_pte_virt(pte_phys);
	}

	create_pte_mapping(ptep, va, pa, sz, prot);
}```
```

对于create_pmd_mapping，基本参数的含义和上面创建pte的参数含义相同，第5行建立了一个pte的表用来当作对应pmd表项映射的表，这里需要注意一下sz的问题：如果sz是pmd的大小，证明这个映射设置的是一个pmd大小的映射，也就是在这里只需要创建一个pmd级的映射就行。如果sz为pte大小的话，则证明这一步本身的目标是创建一个pte的映射，则从15行开始走向创建pte的逻辑，其中15-20行为pmd没有被创建是新建一个pmd，而后面是在已有的pmd中寻找一块内存去准备创建pte，在25行为创建对应的pte。

这里注意一点：非pte的页表创建都会根据size选择两种模式，一种是创建自己本身大小的映射，这也是最直观的情况，另外一种是要考虑创建更小的表的情况（建立的时候是从大层次表到小表的如先建立pmd再建立pte），这个时候不但创建一个本身的表，并且需要创建下一层表。

这里也就是说需要创建对应的create_pud_mapping和create_p4d_mapping逻辑基本和上面差不多，，，

之后是关于一些其他关于页表需要的函数，以pmd为例：

```#endif /* CONFIG_XIP_KERNEL */

static pmd_t *__init get_pmd_virt_early(phys_addr_t pa)
{
	/* Before MMU is enabled */
	return (pmd_t *)((uintptr_t)pa);
}

static pmd_t *__init get_pmd_virt_fixmap(phys_addr_t pa)
{
	clear_fixmap(FIX_PMD);
	return (pmd_t *)set_fixmap_offset(FIX_PMD, pa);
}

static pmd_t *__init get_pmd_virt_late(phys_addr_t pa)
{
	return (pmd_t *) __va(pa);
}

static phys_addr_t __init alloc_pmd_early(uintptr_t va)
{
	BUG_ON((va - kernel_map.virt_addr) >> PGDIR_SHIFT);

	return (uintptr_t)early_pmd;
}

static phys_addr_t __init alloc_pmd_fixmap(uintptr_t va)
{
	return memblock_phys_alloc(PAGE_SIZE, PAGE_SIZE);
}

static phys_addr_t __init alloc_pmd_late(uintptr_t va)
{
	unsigned long vaddr;

	vaddr = __get_free_page(GFP_KERNEL);
	BUG_ON(!vaddr);
	return __pa(vaddr);
}```
```

分别针对pmd的虚拟地址以及fixmap的虚拟地址的生存和查询，其中early和late分别对应为mmu生成前后之间的情况。

故而我们需要在之后的pud和p4d中加入这些函数。

关于宏，主要为判断页表是否重叠，这一点请看linux公共内核那边的宏。

好，到这里我们回到setup_vm函数，这里就很明白了pt_ops作为全局变量根据不同的情况（mmu建立的前后）为产生的变量决定其分配的函数，之后则是加入对fixmap的映射和trampoline（用于内核代码）的映射，注意建立fixmap和trampoline中对应的虚拟地址，其定义：

``````
static pmd_t trampoline_pmd[PTRS_PER_PMD] __page_aligned_bss;
static pmd_t fixmap_pmd[PTRS_PER_PMD] __page_aligned_bss;
``````

这两个定义为对应生成pmd页面的地址（所以在生成对应映射的调用是pgd调用的是fixmap_pmd），同理使用pud和p4d时要加入对应的过程和数据结构。

setup_vm之后的就是创建dtb映射

```
create_fdt_early_page_table(early_pg_dir, dtb_pa);
```

和上面类似，不过需要在上面两个映射完成后执行，，，，逻辑差不多这里 不再阐述

回到head.S,在setup_vm后就进入了start_kernel正式进入了c语言时期，这里需要关注的是start_kernel中会进行一系列调用最终调用掉setup_vm_final函数，如下：

```
static void __init setup_vm_final(void)
{
	uintptr_t va, map_size;
	phys_addr_t pa, start, end;
	u64 i;

	/**
	 * MMU is enabled at this point. But page table setup is not complete yet.
	 * fixmap page table alloc functions should be used at this point
	 */
	pt_ops.alloc_pte = alloc_pte_fixmap;
	pt_ops.get_pte_virt = get_pte_virt_fixmap;
#ifndef __PAGETABLE_PMD_FOLDED
	pt_ops.alloc_pmd = alloc_pmd_fixmap;
	pt_ops.get_pmd_virt = get_pmd_virt_fixmap;
#endif
	/* Setup swapper PGD for fixmap */
	create_pgd_mapping(swapper_pg_dir, FIXADDR_START,
			   __pa_symbol(fixmap_pgd_next),
			   PGDIR_SIZE, PAGE_TABLE);

	/* Map all memory banks in the linear mapping */
	for_each_mem_range(i, &start, &end) {
		if (start >= end)
			break;
		if (start <= __pa(PAGE_OFFSET) &&
		    __pa(PAGE_OFFSET) < end)
			start = __pa(PAGE_OFFSET);
		if (end >= __pa(PAGE_OFFSET) + memory_limit)
			end = __pa(PAGE_OFFSET) + memory_limit;

		map_size = best_map_size(start, end - start);
		for (pa = start; pa < end; pa += map_size) {
			va = (uintptr_t)__va(pa);

			create_pgd_mapping(swapper_pg_dir, va, pa, map_size,
					   pgprot_from_va(va));
		}
	}

#ifdef CONFIG_64BIT
	/* Map the kernel */
	create_kernel_page_table(swapper_pg_dir, false);
#endif

	/* Clear fixmap PTE and PMD mappings */
	clear_fixmap(FIX_PTE);
	clear_fixmap(FIX_PMD);

	/* Move to swapper page table */
	csr_write(CSR_SATP, PFN_DOWN(__pa_symbol(swapper_pg_dir)) | SATP_MODE);
	local_flush_tlb_all();

	/* generic page allocation functions must be used to setup page table */
	pt_ops.alloc_pte = alloc_pte_late;
	pt_ops.get_pte_virt = get_pte_virt_late;
#ifndef __PAGETABLE_PMD_FOLDED
	pt_ops.alloc_pmd = alloc_pmd_late;
	pt_ops.get_pmd_virt = get_pmd_virt_late;
#endif
}
```

这里首先需要将swapper部分的内存进行映射，之后清处掉不用的fixmap，刷新cache，此时mmu已经配置完成，将后面的寻址和分配函数的后缀改为late。

至于怎么改应该已经很明白了所以不再写了，，，，反正也是自己看，，，也不会有谁能看到这个吧，，，，

## 怎么调试捏

调试方法：gdb+qemu

首先这里需要使用之前编译链中的gdb调试工具

在.config中加入以下选项

重新编译

用一下命令让qemu监听对应的端口

gdb链接对应的端口

这里会有一个问题就是，在正常打开vmlinux的时候由于内核初始时虚拟内存并没有初始化，会造成在mmu初始化之前无法找到段和程序地址的情况，具体解决方法如下：

使用编译工具链中的objdump反编译vmlinux，这是我产生的对应的段地址

之后opensbi的习惯是物理初始地址为

在物理地址之上加上虚拟地址，作为载入vmlinux的指定，就可以正常测试了

## 关于怎么测试

先看编译完成的内核对应的虚拟地址

使用mmap映射并输出对应地址

在host中交叉编译，注意使用静态编译，并将二进制文件加入虚拟机中

具体加入方法；

执行

完成
