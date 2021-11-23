1. KERNELBASE：内核虚拟地址空间的起始地址，一般和PAGE_OFFSET相同，但是也可能不同
2. PAGE_OFFSET：内核虚拟地址空间中低端内存的起始地址
3. PHYSICAL_START：内核物理地址的起始地址
4. MEMORY_START：内核低端内存的物理起始地址

arch/risc/include/asm:

​	page.h 这个莫名的好像不用改？

​	pagetable.h 加入对pud、p4d虚拟地址转换的部分

​    pagetable-64bit.h  加入pud、p4d。。。

​	pgalloc.h  加入对pud、p4d分配和释放的支持

arch/risc/mm：

​	init.c  setpup_vm，还有setup_vmfinal中添加pud和p4d映等

​	hugetlbpage.c  这个没怎么看明白但是似乎要改的样子，，

