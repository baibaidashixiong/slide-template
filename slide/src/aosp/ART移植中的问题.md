### 杂项
- 缺少`com.android.art.testing`：`vndkcorevariant.libraries.txt`问题：在`art/tools/buildbot-build.sh`中去除`vndkcorevariant.libraries.txt`.
- riscv中用call，la中用b还是bl？`ret = jr $ra = jirl $zero, $ra, 0/ call = bl`.
- what is inline cache?
- 高效的指令序列?
- 同步异常和异步异常（AsyncException）？
- 动态基本块和静态基本块：
- dex_instruction和Java Bytecode?
- android runtime中的pass定义在哪，如何工作？定义在optimization.h中的HOptimization.然后在ConstructOptimizations中构建和选择pass。
- art从字节码编译为汇编代码的核心：其中codegen为根据架构不同create的初始化代码生成块。
```bash
OptimizingCompiler::Compile -> OptimizingCompiler::TryCompile/TryCompileIntrinsic -> 
构建HGraph -> 运行优化(baseline时为RunRequiredPasses，其它时候为RunOptimizations) -> 分配寄存器 -> 代码生成code->Compile，最终返回一个CodeGenerator指针
```
- JIT编译需要底层系统支持动态代码生成，对操作系统来说这意味着要支持动态分配带有“可写可执行”权限的内存页。当一个应用程序拥有请求分配可写可执行内存页的权限时，它会比较容易受到攻击从而允许任意代码动态生成并执行，这样就让恶意代码更容易有机可乘。Safari和edge的增强安全性就是通过禁用JIT来进行操作系统保护。（安全方面是否有创新空间？）
- 编译核心：代码生成器code generator, assembler_riscv64.cc/h
- 核心：栈帧结构，代码生成
- JNI_trampoline的作用：作为java到native c的桥梁，进行参数/栈帧转换等操作。JNI函数指的是用C或C++编写的本地函数。JNI compiler的作用？
- assembler_riscv66.cc用于生成机器代码，code_generator_riscv64.cc用于汇编代码，如一条`int c = a + b`的代码，通过code_generator_riscv64生成`assembler_->EmitAdd(dest, src1, src2);`，再通过assembler_riscv64生成`Emit32(0xE0801001 | (dest << 0) | (src1 << 5) | (src2 << 16));`具体的二进制代码。可以说字节码为high level，
- 一个ArtMethod代表一个Java类中的成员方法。
- 需要搞清楚riscv生成汇编时bind，branch，bcond之间的区别。
- jumptable是什么？作用是什么？[跳转表](https://blog.csdn.net/pangshaohua/article/details/6982528).
- virtual关键字是用在基类中声明的，只有用virtual声明过的函数才能在派生类中进行override。
- C++和java的区别？（深层次）
- python VM vs JVM？
- 短跳转是如何变为长跳转的？
- 直接跳转和间接跳转？
- 分支指令中`type`表示分支指令的类型，可以分为有条件分支（bcond）和无条件分支（buncond），`condition_`表示**有条件分支指令**中进行跳转的条件，如（beq）。
- x86的调用惯例分为三种：stdcall，cdecl，fastcall
- 代码解释：
```c++
// 假如有如下汇编
beq  a0,a1, 1f        // beq1
add zero, zero, zero
beq  a2,a3, 2f         // beq2       
add zero, zero, zero

beq  a0,a1, 1f        // beq3
add zero, zero, zero
beq  a2,a3, 2f          // beq4
add zero, zero, zero

1:
add zero, zero, zero

2:
add zero, zero, zero

beq  a0,a1, 1b          // beq5
beq  a0,a1, 2b        // beq6


void Riscv64Assembler::Buncond(Riscv64Label* label, XRegister rd, bool is_bare) {
  uint32_t target = label->IsBound() ? GetLabelLocation(label) : Branch::kUnresolved;
  branches_.emplace_back(buffer_.Size(), target, rd, is_bare);
  FinalizeLabeledBranch(label);
}
```
- 将所有的`break 0`都改为`break 0x5`后，使用gdbinit后仍然会有futex锁死的错误。此时查看gdb，在一直阻塞时ctrl+c以后收到了SIGINT，此时查看bionic中的syscall汇编，发现是`move a7, a0`将保存系统调用号的a0寄存器储存到a7中，此时`info registers`看到a7寄存器中的值为`0062`，对应`libc/kernel/uapi/asm-generic/unistd.h`中`#define __NR3264_lseek 62`的lseek系统调用，可以推测此时一直卡死在lseek中。而`info signals SIGINT`中发现art中没有对SIGINT进行处理，所以会一直引起futex_wait的512问题。
- futex解释：
```c
#include <linux/futex.h>
#include <stdint.h>
#include <sys/time.h>

long futex(uint32_t *uaddr, int futex_op, uint32_t val,  
          const struct timespec *timeout,   /* or: uint32_t val2 */  
          uint32_t *uaddr2, uint32_t val3);
```
- JNI
- suspend points


### pass优化
- gvn：Global Value Numbering，全局值编号
### 代码生成
- ART的寄存器分配策略采用基于SSA的线性扫描寄存器分配算法（LSRA on SSA）
- 需要搞清楚的类：HGraphVisitor？visitor的作用？
- shadow_frame和ManagedStack的区别？
#### 代码生成器中的问题
- 指令选择：指令选择可以说是代码生成阶段最重要的部分。如果生成的IR是高层次的，代码生成器就要使用代码模板把每个IR语句翻译成机器指令序列，但是这样逐个语句生成代码的方式通常会产生质量不佳的代码需要进一步优化。如果IR中反映了相关计算机的某些低层次细节，那么代码生成器就可以使用这些信息来生成更高效的代码序列。

#### 调试
- gdb中的select-frame用于获取bt中某个栈帧的状态。
- gdb中si无法继续往下执行的问题。
- si时无反应，backtrace如下：
```bash
(gdb) bt  
#0  syscall () at bionic/libc/arch-loongarch64/bionic/syscall.S:43  
#1  0x00007ffd590cbd24 in art::futex (uaddr=0x80, op=128, val=0, timeout=0x0, uaddr2=0x0, val3=0) at art/runtime/base/mutex-inl.h:43  
#2  art::ConditionVariable::WaitHoldingLocks (this=0x7ffe43e288f8, self=0x7fff53e22010) at art/runtime/base/mutex.cc:1083  
#3  0x00007ffd590cbc18 in art::ConditionVariable::Wait (this=0x7ffe43e288f8, self=0x7fff53e22010) at art/runtime/base/mutex.cc:1069  
#4  0x00007ffd597c81b0 in art::SignalCatcher::SignalCatcher (this=0x7ffe43e288d0) at art/runtime/signal_catcher.cc:84  
#5  0x00007ffd597a8c48 in art::Runtime::StartSignalCatcher (this=0x7fff13e2be40) at art/runtime/runtime.cc:1179  
#6  art::Runtime::InitNonZygoteOrPostFork (this=0x7fff13e2be40, env=<optimized out>, is_system_server=<optimized out>, is_child_zygote=<optimized out>, action=<optimized out>, isa=<optimized out>,    
   profile_system_server=<optimized out>) at art/runtime/runtime.cc:1132  
#7  0x00007ffd597a6964 in art::Runtime::Start (this=0x7fff13e2be40) at art/runtime/runtime.cc:958  
#8  0x00007ffd59556f10 in JNI_CreateJavaVM (p_vm=0x7fffffffaa60, p_env=0x7fffffffaa58, vm_args=0x7fffffffaa68) at art/runtime/jni/java_vm_ext.cc:1219  
#9  0x00007ffff6a946b0 in JNI_CreateJavaVM (p_vm=0xfffffffffffffe00, p_env=0x80, vm_args=0x0) at libnativehelper/JniInvocation.c:96  
#10 0x0000555555561240 in art::dalvikvm (argc=<optimized out>, argv=<optimized out>) at art/dalvikvm/dalvikvm.cc:178  
#11 0x0000555555561000 in main (argc=-512, argv=0x80) at art/dalvikvm/dalvikvm.cc:220
```
可以看到是卡在futex中，怀疑与`art::Runtime::InitNonZygoteOrPostFork`有关。进入runtime.cc查看`StartSignalCatcher();`源码：
```c++
void Runtime::StartSignalCatcher() {
  if (!is_zygote_) {
    signal_catcher_ = new SignalCatcher();
  }
}
```
查看SignalCatcher实现：
```c++
SignalCatcher::SignalCatcher()
    : lock_("SignalCatcher lock"),
      cond_("SignalCatcher::cond_", lock_),
      thread_(nullptr) {
  SetHaltFlag(false);

  // Create a raw pthread; its start routine will attach to the runtime.
  CHECK_PTHREAD_CALL(pthread_create, (&pthread_, nullptr, &Run, this), "signal catcher thread");

  Thread* self = Thread::Current();
  MutexLock mu(self, lock_);
  while (thread_ == nullptr) {
    cond_.Wait(self);
  }
}
```

- 错误`code_generator.cc:1754] Check failed: stack_offset < codegen->GetFrameSize() - codegen->FrameEntrySpillSize() (stack_offset=24, codegen->GetFrameSize() - codegen->FrameEntrySpillSize()=24)`，打印出`core_spills register`和后`core_spill_mask_`作对比后初步定位为`core_spill_mask_`出问题。而其修改处为`core_spill_mask_ = allocated_registers_.GetCoreRegisters() & core_callee_save_mask_;`，我们需要关注的为`core_registers_`和`core_callee_save_mask_`。但是查看CodeGeneratorLOONGARCH64中CodeGenerator::CodeGenerator，发现每次的`core_callee_save_mask/core_callee_save_mask_`并没有变化，所以变化的应该为`core_registers_`。也有可能是`SetFrameSize`太小。要么就是`POPCOUNT(core_spill_mask_)`(callee saved registers + ra)太多，要么就是`core_spills`(非callee saved registers)太多，要么就是`first_register_slot_in_slow_path_`太大，挤占了core_spills的空间，要么就是framesize太小。`maximum_safepoint_spill_size`代表的值就是`core_spills`应该的值。
- `MaybeGenerateInlineCacheCheck`实现有问题。Difference between method_offset and class_offset?`class_offset`为其示例对象所属类的指针偏移量，`method_offset`表示在vtable中访问特定方法的偏移量。