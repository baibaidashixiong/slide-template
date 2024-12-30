### riscv压缩指令集扩展（C Extension）
- 
- code block的分界点：间接跳转(indirect branch)，如`jalr a0, a1, 4`，jit模式下每执行完一个code block更新一次pc，而interprete每执行一条指令更新一次pc
- 为什么要进行地址转换？内存布局，使用pmap查看内存布局。GUEST_MEMORY_OFFSET偏移量为0为什么会在运行到一定程度的时候触发段错误？
- 关闭aslr：`setarch -R ./program`.
- gdb打硬监视点：`watch -location m->state.reenter_pc`.
- gdb监视内存地址`watch *(int*0)0xaaaa`.
- x86和arm的存储一致性（load store顺序）导致的性能问题。
- p_memsz的值大于p_filesz的含义：表示该segment在内存中所分配的空间大小超过文件中实际的大小，这部分多余的部分则全部填充为0,这样的好处是在构造elf可执行文件时不需要再额外设立bss的segment，可以把数据段segment的p_memsz扩大，那些额外的部分就是bss。
- rvemu在la上无法运行的原因：la上系统的页大小为16K，x86/riscv上的页大小为4K，mmu.c中所用的load_elf_segment无法使用mmap正确分配程序到内存中。如何使其能正确运行需要修改mmu。