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

对于create_pmd_mapping，基本参数的含义和上面创建pte的参数含义相同
