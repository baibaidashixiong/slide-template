所需要的代码：CodeGenerator（基类），需要子类CodeGeneratorLOONG64，其又衍生出InstructionCodeGeneratorLOONG64，InstructionCodeGenerator，LocationsBuilderLOONG64，Loong64Assembler。

CodeGenerator的AllocateLocations会为IR对象创建一个LocationSummary对象，LocationSummary的目的是让CodeGenerator和Register Allocation隔开，从而让寄存器分配时只需设置好LocationSummary，CodeGenerator将根据LocationSummary里所包含的信息来生成机器码。RegisterSet是一个用于记录LocationSummary使用了哪些核心寄存器和浮点寄存器的辅助类。
- 首先需要移植测试程序，`buildbot-vm.sh`来创建虚拟机环境。
- 然后需要移植JNI接口，JNI接口即JAVA应用通过调用native函数来运行。通过test中的001-HelloWorld来运行。
- 需要实现`runtime/interpreter/mterp/[arch]`中的内容。
	- main.S：
		- 需要实现**suspend point**，经过suspend point的处理（保存/恢复栈状态）可以从jit/nterp模式deoptimization切换到解释执行模式，来用于GC，性能分析，调试等，
- 解释执行的switch case dex指令定义在libdexfile/dex/dex_instruction_list.h中
- **解释执行实现**：由gen_mterp.py生成的指令表为mterp_riscv64.S，其中指令元素按照NTERP_HANDLER_SIZE（老版本为MTERP_HANDLER_SIZE）字节（128字节）对齐，即每个指令元素占用128字节。其中指令定义在`runtime/interpreter/mterp/riscv64`下，这些定义的指令主要就是用host汇编实现dex字节码。其中有以下规则：
	1. 以%开头的行是Python代码。它们将按原样复制到脚本中（不包括%）
	2. 其他行是文本，调用write_line输出。
	3. 模板$可以用于从代码中插入变量。是脚本运行是动态填充的。
- 启动流程（注意：Runtime代表art虚拟机，为全局唯一的）：`AndroidRuntime::start -> AndroidRuntime::startVm -> jint JNI_CreateJavaVM -> Runtime::Create -> Runtime::Init -> `.
```bash
frameworks/base/core/jni/AndroidRuntime.cpp
AndroidRuntime::start(启动android runtime) -> AndroidRuntime::startVm ->
art/runtime/jni/java_vm_ext.cc
extern "C" jint JNI_CreateJavaVM ->  
art/runtime/runtime.cc
Runtime::Create -> Runtime::Init -> Runtime::Start(where call it?) -> Runtime::InitNativeMethods ->
art/runtime/well_known_classes.cc
WellKnownClasses::LateInit # 调用LateInit缓存使用频率比较高的Runtime.nativeLoad,Reflect构造器，以及Invoke方法
```
- `frameworks/base/core/jni/AndroidRuntime.cpp`中`AndroidRuntime::start`调用startReg(主要用于注册android function)注册android特有的jni方法，这是和dalvikvm的不同，app_process会注册android jni方法，而dalvikvm不可以。所以是不能用dalvikvm启动android app的。`env->PushLocalFrame(200);`创建容量为200的局部变量作用域，将gRegJNI存入其中。`androidSetCreateThreadFunc((android_create_thread_fn) javaCreateThreadEtc);`设置创建线程的函数指针为javaCreateThreadEtc。然后通过register_com_android_internal_os_ZygoteInit_nativeZygoteInit调用`com/android/internal/os/ZygoteInit`的main函数进入java世界。
- 虚拟机的栈：`runtime/stack.h`.
- art从jit进入HInstruction流程：`art::jit::JitCompiler::CompileMethod`->`art::OptimizingCompiler::JitCompile`->`art::OptimizingCompiler::TryCompile`->`art::CodeGenerator::Compile`->`art::HInstanceOf::Accept`->`art::loongarch64::InstructionCodeGeneratorLOONGARCH64::VisitInstanceOf`.
- HBasicBlock，HGraph之间的关系？
- `optimizing_compiler.cc`中JitCompile、Compile、TryCompile的区别？JitCompile->TryCompile->Compile.
- HInstruction到底做了什么？HInstruction中的location该如何确定？
- codegen中的slowpath就是解释执行吗？还是什么其它的路径？
- 默认用的寄存器分配算法是线性扫描法。

## 零、soong构建系统
- `out/soong/.intermediates`：存放所有Android.bp所指定的编译出来的预处理后的源文件和目标文件。
	- `out/soong/.intermediates/art/runtime/libart_mterp.riscv64/gen/mterp_riscv64.S`是由脚本`art/runtime/interpreter/mterp/gen_mterp.py`生成的。
	- 其通过`gen_setup.py`中generate的`header, entry, instruction_start, opcodes, balign, instruction_end`来生成汇编代码
- android源码编译后得到system.img,ramdisk.img,userdata.img映像文件。其中， ramdisk.img是emulator的文件系统，system.img包括了主要的包、库等文件，userdata.img包括了一些用户数据，emulator负责加载这3个映像文件后，会把system.img和userdata.img分别加载到 ramdisk文件系统中的system和 userdata目录下。（但是貌似只需要system.img和vendor.img？）
- 得到`compile_commands.json`文件提供调试：
```bash
lunch aosp_loongarch64-eng
export SOONG_GEN_COMPDB=1
export SOONG_GEN_COMPDB_DEBUG=1
make nothing
```
- 
## 一、CodeGen
首先在`compiler/optimizing`目录下添加code_generator，为目标架构添加指令集说明。
在`compiler/utils/loongarch64`中添加架构相关信息，汇编代码生成。
- Intrinsics为特定平台优化的代码/指令和代码内联。
- fastpath与slowpath的区别：
- 解释执行中nterp执行流程：
- art中执行过程有两个栈，机器执行产生的栈称为quick_frame，解释执行产生的栈称为shadow_frame。还有一个managed stack.

## 工具
- c++ demangle：c++filt
- 代码行数统计：`cloc ./`.
- 调试android：
```bash
# 创建文件系统
dd if=/dev/zero of=androidroot.img bs=1M count=16384
sudo losetup /dev/loop21 ./androidroot.img
sudo mkfs.ext4 /dev/loop21 # 为androidroot.img设置文件系统
sudo mount /dev/loop21 /mnt/
sudo cp -a /home/zqz/android/aosp.la/out/target/product/generic_loongarch64/system/* /mnt/
sudo cp -a /home/zqz/android/aosp.la/out/target/product/gennric_loongarch64/data/* /mnt/data/
sudo umount /mnt
sudo losetup -d /dev/loop21
```
- 安卓逆向工具：frida，Xposed。
- 技术：inline-hook。
- 生成测试代码工具：Java-fuzzer.
- TEST_F：gtest框架中用于测试的宏。
- dex文件用例：.java->.class->.dex
```bash
# hello.java
public class hello {
   public static void main(String[] args) {
       System.out.println("Hello World!");
   }
}
# 编译java文件为class文件
javac hello.java
# 执行文件
java hello
# 反汇编hello.class文件
javap -c hello
# 使用build-tools目录下的dx工具生成dex文件
./common/bin/dx obj
# 反编译dex文件
dexdump -d Hello.dex
```
- 查看伪指令：`binutils/opcodes/loongarch-opc.c`文件。
- 编译qemu：`../configure --target-list=loongarch64-softmmu,loongarch64-linux-user --disable-werror --enable-capstone --enable-slirp`
- 跟踪文件是否被读取：`inotifywait -rm /usr/lib/x86_64-linux-gnu/debug/`。递归监控`/usr/lib/x86_64-linux-gnu/debug/`中的文件是否被访问到。
- dexlist将dex文件中的方法列出。
- 编译dex文件：
```bash
dex2oat64 --dex-file=004-StackWalk.jar --oat-file=004.oat
oatdump --oat-file=004.oat > 004.txt
```
- jit-cache的可执行代码的内存容量为32M。(r--s表示有读无写/执行权限且共享映射)
```bash
4f3b3000-513b3000 r--s 00000000 00:01 6931                               /memfd:jit-cache (deleted)  
513b3000-533b3000 r-xs 02000000 00:01 6931                               /memfd:jit-cache (deleted)
```
- 安装ART包：`adb install */system/apex/com.android.art.testing.apex`.
### JVMTI





## 测试 
- 堆污染：heap-poisoning，由`ART_HEAP_POISONING`控制。
- 控制JIT启用：`Xjitthreshold`。
- ART单元测试：`art/test.py --target --64 -v  -t 001-Main`。
```bash
art/test.py --target -r --no-prebuild --ndebug --no-image -v --64 --interpreter -t 001-HelloWorld --gdb
```
- `_test.cc`文件采用gtest框架来编写，使用`m test-art-host-gtest`来对其进行测试。
- gtest测试：
```bash
./build/soong/soong_ui.bash --make-mode -j art_compiler_tests
out/host/linux-x86/nativetest/art_compiler_tests/art_compiler_tests
```
测试问：host和target分别测什么？
## 问题
- `FRAME_SIZE_SAVE_REFS_AND_ARGS`中的寄存器的作用？为什么不是32个寄存器？
- codegen的作用和工作原理？bear
- jni macro compiler的作用：为每个managed code->native code的过程中生成封装。
- 分支扩展？用于进行跳转距离延长
- la.pcrel/la.abs
- 补值问题（大坑）
- Devirtualization（虚函数去虚化）
- 条件编译。字节序，对齐，地址空间大小，页大小
- graal aot和art aot的区别？
- GC通常分为引用计数法和可达性分析法。
- SuspendCheck/checkpoint/safepoint的作用？
- 为什么规定jit只编译`Main.test()`的情况下，会生成两次FrameEntry?
- `MoveLocation`，
- `ArtMethod**` 和`ArtMethod`的关系：
```bash
`ArtMethod**` 是一个指向 `ArtMethod*` 的指针，而 `ArtMethod*` 则是指向 `ArtMethod` 对象的指针。
为了理解它们的关系，先分解一下：
### 1. **`ArtMethod`**
   - `ArtMethod` 是 Android Runtime (ART) 中一个非常重要的结构体，它代表了一个 Java 方法的元数据。`ArtMethod` 存储了与方法相关的信息，如方法的名称、访问权限、编译状态、代码指针等。
   - 在 ART 中，每个方法在运行时都对应一个 `ArtMethod` 实例，用来描述这个方法的各种属性和状态。

### 2. **`ArtMethod*`**
   - `ArtMethod*` 是一个指向 `ArtMethod` 结构体的指针，也就是单个 `ArtMethod` 对象的地址。它允许在代码中通过指针访问特定方法的元数据信息。例如，`ArtMethod*` 指向的 `ArtMethod` 对象可以包含关于某个 Java 方法的详细信息。
   - 简单来说，`ArtMethod*` 代表一个具体的 Java 方法实例的元数据信息。

### 3. **`ArtMethod**`**
   - `ArtMethod**` 是指向 `ArtMethod*` 的指针，也就是指向一个 `ArtMethod*` 的地址。
   - 在堆栈帧中，Java 方法的调用会涉及到保存 `ArtMethod*` 指针，`ArtMethod**` 通常用于栈帧中保存这些 `ArtMethod*`。每当 Java 方法调用发生时，ART 会在栈帧中存储当前方法的 `ArtMethod*`，指向当前执行的方法。通过 `ArtMethod**`，可以方便地在栈帧中管理和操作这些指针。
   - 可以把 `ArtMethod**` 看作是用于管理或遍历方法指针的工具，比如从栈帧中取出当前正在执行的方法指针。

### **`ArtMethod**` 与 `ArtMethod` 的关系**
   - `ArtMethod**` 是指向方法指针的指针，而 `ArtMethod*` 是指向具体方法的指针。
   - 在栈帧中，`ArtMethod**` 用于表示当前执行方法的指针位置，指向实际的 `ArtMethod*`。而 `ArtMethod*` 进一步指向具体的 `ArtMethod` 实例，保存着方法的具体信息。
   - 在代码中，操作 `ArtMethod**` 可以间接操作 `ArtMethod`，例如通过解引用 `ArtMethod**` 得到 `ArtMethod*`，再进一步访问 `ArtMethod` 的内容。

### 实际应用场景
在执行过程中，每个栈帧会包含对当前执行方法的 `ArtMethod*`，通过这种方法指针，ART 可以通过栈帧快速找到当前执行的 Java 方法的相关信息。而 `ArtMethod**` 则是为了灵活操作这些方法指针，尤其在调用栈遍历或栈帧管理时，会经常用到它。
```
- 为什么X86上可以在jit生成的代码遇到访存出错后si便可进入到异常处理的函数，而LA上si便直接跑到下一处错误地址处。
- Managed Stack：Managed Stack主要是为托管代码（即Java代码）提供执行上下文的地方，它保存了在 Java 方法执行过程中需要管理的信息，比如局部变量、方法调用链、返回地址等。ART 中的 Managed Stack 与传统的栈相似，但专门用于管理 ART 运行时中执行的托管线程。
- **从jit生成代码的访存出错，到跳转到`art::SignalChain::Handler()`进行异常处理，这其中发生了什么**？跳入内核进行异常处理再返回。
- 编译器何时会乱序？当两个变量并不存在依赖的时候，在编译器看来它们只是两个独立的地址，只是在我们的视角里给它们赋予了不同的含义，所以编译器认为并没有必要保证它们不能乱序，这意味着编译器生成的代码从我们眼里可能会变成这个意思。
- 哨兵(sentinal)和poison的区别关系？
- CMC是如何利用userfaultfd机制去除读屏障的？缺点：需要对活的引用对象拷贝两次。
## 优化
- max, min伪指令的实现
- `volatile`关键字可以禁止指令进行重排序优化。其实现方式是在`HandleFieldGet`中插入以下代码来判断实现。
```c++
if (is_volatile) {
    codegen_->GenerateMemoryBarrier(MemBarrierKind::kAnyAny);
}
 ...
if (is_volatile) {
    codegen_->GenerateMemoryBarrier(MemBarrierKind::kLoadAny);
}
```

## 启动错误
1. 没有eth0网卡。通过`cat /sys/class/net/enp0s3f0/device/uevent`来查看网卡所需的驱动。然后在x86上为la内核增加stm eth驱动以后重新编译内核：
```bash
CC_PREFIX=/home/zqz/android/tools/loongson-gnu-toolchain-8.3-x86_64-loongarch64-linux-gnu-rc1.3-1
export PATH=$CC_PREFIX/bin:$PATH
export LD_LIBRARY_PATH=$CC_PREFIX/lib:$LD_LIBRARY_PATH
export LD_LIBRARY_PATH=$CC_PREFIX/loongarch64-linux-gnu/lib64:$LD_LIBRARY_PATH
make ARCH=loongarch CROSS_COMPILE=loongarch64-linux-gnu- -j
```

## 调试结果记录：
- 与长跳转有关的测试用例
```bash
021-string2
580-checker-round

```
- osr相关问题：为什么对于一个函数来说（如Main.main），先执行kOsr再执行kBaseline？
- 负载存储消除技术（load-store elimination, lse）
- kBaseline和kOptimized优化对比：
```java
public static double nextDown(double d) {
      long a1 = ((d > 0.0d)? -1L:1L);
      int z1 = 0x222;
      long a =  Double.doubleToRawLongBits(d);
      if(a1 == -1) {
        z = 0x333;
        e = ((d > 0.0d)?-0x111L:+0x444L);
      }
      return Double.longBitsToDouble(a + a1);
  }
```
以`e = ((d > 0.0d)?-0x111L:+0x444L);`为例，对于Baseline，只固化了一些偏移地址以及将变量分配在寄存器上，基本上没做什么优化：
```bash
      0 0 v23 If [z22] dex_pc:26 loop:none
0x000000e4: 64001005	bge zero, a1, 16 (addr 0xf4)
      0 0 v78 ParallelMove dex_pc:n/a liveness:64 moves:[invalid->invalid,invalid->invalid] loop:none
0x000000e8: 0015009a	move s3, a0
0x000000ec: 02bbbc19	addi.w s2, zero, -273 (0x111)
      0 0 v27 Goto dex_pc:30 loop:none
0x000000f0: 50000c00	b 12 (addr 0xfc)
      0 0 v79 ParallelMove dex_pc:n/a liveness:68 moves:[invalid->invalid,invalid->invalid] loop:none
0x000000f4: 0015009a	move s3, a0
0x000000f8: 03911019	li.w s2, 0x444
```
而对于Optimized来说，其消除了跳转指令：
```bash
      0 0 v106 ParallelMove dex_pc:n/a liveness:40 moves:[invalid->invalid,invalid->invalid,invalid->invalid] loop:none
0x00000084: 00150097	move s0, a0
0x00000088: 03911005	li.w a1, 0x444
0x0000008c: 02bbbc04	addi.w a0, zero, -273 (0x111)
 <|@
      0 1 j87 Select [j26,j24,z9] dex_pc:26 loop:none
0x00000090: 0012e414	sltu t8, zero, s2
0x00000094: 0011d014	sub.d t8, zero, t8
0x00000098: 00159493	xor t7, a0, a1
0x0000009c: 0014ce94	and t8, t8, t7
0x000000a0: 0015929b	xor s4, t8, a0
```
- `716-jli-jit-samples`错误问题：`ArtMethod::SetIntrinsic`中`kIsDebugBuild`对`is_fast_native`进行了赋值。应该是Android runtime本身的bug。
- `989`错误：实际读取的地址比实际存放的地址低了208,也就是`FRAME_SIZE_SAVE_REFS_AND_ARGS`的距离，估计是少restore了一次栈。`fpr_args_`有问题。
- RISCV架构中拿S1作xSelf是因为S1是它的保留寄存器。