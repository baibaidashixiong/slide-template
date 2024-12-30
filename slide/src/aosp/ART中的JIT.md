	虚拟机会对方法的执行次数进行统计，当某个方法的执行次数到达一定阈值后（ART虚拟机称之为hot method）虚拟机会将这些hot method编译成本地机器码。后续这些方法将以机器码的方式来执行。
	还有一种类型的方法，执行次数可能不多，但方法内部却包含了一个循环次数特别多的循环，当循环到一定次数后JIT会被触发，该方法也会被编译成机器码。但该机器码并不会等到该方法的下一次调用才执行，而是作为后续指令执行。假设某个方法包含一个10w次循环，循环次数为1w之前该方法为解释执行，1w次之后切换为机器码执行。该处理方式需要将机器码执行所需的栈信息替换之前以解释执行的栈，所以也叫On Stack Replacement(OSR)技术。
和dex2oat一样，JIT编译也是使用OptimizingCompiler。
ART虚拟机中JIT模块包含的几个重要类，其中：
- **Jit**类是ART JIT模块的门户。JIT模块提供的主要功能将借助该类的相关成员函数对外输出。
- **JitCodeCache**管理用来存储JIT编译结果的一段内存空间。当内存不够用时这部分内存空间可以被回收。
- **JitCompiler**用于处理JIT相关的编译。其内部用到的编译模块和dex2oat里AOT编译用到的编译模块同为**OptimizingCompiler**。从编译角度来看，ART中的JIT和AOT用到的技术没有区别，只不过编译的时机不同而已。
- **ProfilingInfo**类用于管理一个方法的性能统计信息。
```c++
// jit.h 
class Jit {
 public:
   // 1. Create用于创建Jit对象
   static Jit* Create(JitCodeCache* code_cache, JitOptions* options);
   // 2. AddSamples用于对某个方法进行性能统计
    void AddSamples(Thread* self, ArtMethod* method, uint16_t samples, bool with_backedges);
   // 3. 对方法method进行JIT编译
   bool CompileMethod(ArtMethod* method, Thread* self, bool baseline, bool osr, bool prejit);
   // 4. 判断是否可以进行OSR，后续将转入机器码执行模式
   static bool MaybeDoOnStackReplacement(Thread* thread, ArtMethod* method, uint32_t dex_pc, int32_t dex_pc_offset, JValue* result);
   ...
 private:
   // code_cache_指向一个JitCodeCache对象，它用于管理JIT所需的内存空间
   jit::JitCodeCache* const code_cache_;
   // 线程池对象，内部通过工作线程来处理和JIT相关的工作，如编译等。
   std::unique_ptr<ThreadPool> thread_pool_;
   ...
};
```
# 一、Jit、JitCodeCache等
## 1.1 LoadCompilerLibrary
- `LoadCompilerLibrary`通过dlopen的方式加载`libart(d)-compiler.so`，并从中取出`jit_load`等函数并赋值给Jit对应的成员变量。
- `jit_load_`为Jit的成员函数，是一个函数指针，执行jit_load函数.
```c++
// jit.cc
bool Jit::LoadCompilerLibrary(std::string* error_msg) {
  jit_library_handle_ = dlopen(
      kIsDebugBuild ? "libartd-compiler.so" : "libart-compiler.so", RTLD_NOW);
  if (jit_library_handle_ == nullptr) {
...
    return false;
  }
  if (!LoadSymbol(&jit_load_, "jit_load", error_msg)) {
    dlclose(jit_library_handle_);
    return false;
  }
  return true;
}
```
### 1.1.1 jit_load
```c++
//jit_compiler.cc jit_load
extern "C" JitCompilerInterface* jit_load() {
  auto* const jit_compiler = JitCompiler::Create(); // create jit compiler
  ...
  return jit_compiler;
}

// JitCompiler()
JitCompiler::JitCompiler() {
  compiler_options_.reset(new CompilerOptions());
  ParseCompilerOptions();
  /* Compiler::Create内部用于创建OptimizingCompiler对象 */
  compiler_.reset(Compiler::Create(*compiler_options_, /*storage=*/ nullptr, Compiler::kOptimizing));
}
```
### 1.1.2 JitCompiler::CompileMethod
```c++
// jit_compiler.cc -> JitCompiler::CompileMethod
/* method为代编译的方法，osr表示是否针对OSR的编译 */
bool JitCompiler::CompileMethod(
    Thread* self, JitMemoryRegion* region, ArtMethod* method, bool baseline, bool osr) {
... // 初始化类
  Runtime* runtime = Runtime::Current();

  // Do the compilation.
  bool success = false;
  {
    JitCodeCache* const code_cache = runtime->GetJit()->GetCodeCache();
...
    success = compiler_->JitCompile(
        self, code_cache, region, method, baseline, osr, jit_logger_.get());
...
  }
...
  return success;
}
```
## 1.2 JitCodeCache
JitCodeCache提供了一个存储空间，用来存放JIT编译的结果。当内存不足时，该部分空间可以被释放。
- 首先了解JitCodeCache的创建，其中`GetCodeCacheInitialCapacity`获取的`code_cache_initial_capacity_`在jit.cc中由`JitOptions* JitOptions::CreateFromRuntimeArguments`方法通过`jit_options->code_cache_initial_capacity_ = options.GetOrDefault(RuntimeArgumentMap::JITCodeCacheInitialCapacity);`在`runtime_options.def`中获取。非DebugBuild下默认为64K。
```c++
// jit_code_cache.cc
// JitCodeCache::Create
JitCodeCache* JitCodeCache::Create(bool used_only_for_profile_data,
                                   bool rwx_memory_allowed,
                                   bool is_zygote,
                                   std::string* error_msg) {
...
  /*
    initial_capacity和max_capacity默认为64K和64M
  */
  size_t initial_capacity = Runtime::Current()->GetJITOptions()->GetCodeCacheInitialCapacity();
  // Check whether the provided max capacity in options is below 1GB.
  size_t max_capacity = Runtime::Current()->GetJITOptions()->GetCodeCacheMaxCapacity();
  // We need to have 32 bit offsets from method headers in code cache which point to things
  // in the data cache. If the maps are more than 4G apart, having multiple maps wouldn't work.
  // Ensure we're below 1 GB to be safe.
  if (max_capacity > 1 * GB) {
    std::ostringstream oss;
    oss << "Maxium code cache capacity is limited to 1 GB, "
        << PrettySize(max_capacity) << " is too big";
    *error_msg = oss.str();
    return nullptr;
  }
...
  std::unique_ptr<JitCodeCache> jit_code_cache(new JitCodeCache());
  if (is_zygote) {
    // Zygote should never collect code to share the memory with the children.
    jit_code_cache->garbage_collect_code_ = false;
    jit_code_cache->shared_region_ = std::move(region);
  } else {
    jit_code_cache->private_region_ = std::move(region);
  }
...
  return jit_code_cache.release();
}


// jit_code_cache.cc
// JitCodeCache::JitCodeCache
/* 冒号用于初始化类的成员变量 */
JitCodeCache::JitCodeCache()
    : is_weak_access_enabled_(true),
      inline_cache_cond_("Jit inline cache condition variable", *Locks::jit_lock_),
      zygote_map_(&shared_region_),
      lock_cond_("Jit code cache condition variable", *Locks::jit_lock_),
      collection_in_progress_(false),
      last_collection_increased_code_cache_(false),
      garbage_collect_code_(true),
      number_of_compilations_(0),
      number_of_osr_compilations_(0),
      number_of_collections_(0),
      histogram_stack_map_memory_use_("Memory used for stack maps", 16),
      histogram_code_memory_use_("Memory used for compiled code", 16),
      histogram_profiling_info_memory_use_("Memory used for profiling info", 16) {
}
```

### 1.2.1 CommitCode
当JIT模块编译完一个方法后，该方法的编译结果将通过调用JitCodeCache::Commit函数，再调用其中的JitMemoryRegion::CommitCode函数以存放到对应的存储空间里。
```cpp
// jit_memory_region.cc
// JitMemoryRegion::CommitCode
const uint8_t* JitMemoryRegion::CommitCode(ArrayRef<const uint8_t> reserved_code,
                                           ArrayRef<const uint8_t> code,
                                           const uint8_t* stack_map,
                                           bool has_should_deoptimize_flag) {
  DCHECK(IsInExecSpace(reserved_code.data()));
  ScopedCodeCacheWrite scc(*this);

  size_t alignment = GetInstructionSetAlignment(kRuntimeISA);
  size_t header_size = OatQuickMethodHeader::InstructionAlignedSize();
  size_t total_size = header_size + code.size();

  // Each allocation should be on its own set of cache lines.
  // `total_size` covers the OatQuickMethodHeader, the JIT generated machine code,
  // and any alignment padding.
  DCHECK_GT(total_size, header_size);
  DCHECK_LE(total_size, reserved_code.size());
  uint8_t* x_memory = const_cast<uint8_t*>(reserved_code.data());
  uint8_t* w_memory = const_cast<uint8_t*>(GetNonExecutableAddress(x_memory));
  // Ensure the header ends up at expected instruction alignment.
  DCHECK_ALIGNED_PARAM(reinterpret_cast<uintptr_t>(w_memory + header_size), alignment);
  const uint8_t* result = x_memory + header_size;

  // Write the code.
  std::copy(code.begin(), code.end(), w_memory + header_size);

  // Write the header.
  OatQuickMethodHeader* method_header =
      OatQuickMethodHeader::FromCodePointer(w_memory + header_size);
  new (method_header) OatQuickMethodHeader(
      (stack_map != nullptr) ? result - stack_map : 0u,
      code.size());
  if (has_should_deoptimize_flag) {
    method_header->SetHasShouldDeoptimizeFlag();
  }

  // Both instruction and data caches need flushing to the point of unification where both share
  // a common view of memory. Flushing the data cache ensures the dirty cachelines from the
  // newly added code are written out to the point of unification. Flushing the instruction
  // cache ensures the newly written code will be fetched from the point of unification before
  // use. Memory in the code cache is re-cycled as code is added and removed. The flushes
  // prevent stale code from residing in the instruction cache.
  //
  // Caches are flushed before write permission is removed because some ARMv8 Qualcomm kernels
  // may trigger a segfault if a page fault occurs when requesting a cache maintenance
  // operation. This is a kernel bug that we need to work around until affected devices
  // (e.g. Nexus 5X and 6P) stop being supported or their kernels are fixed.
  //
  // For reference, this behavior is caused by this commit:
  // https://android.googlesource.com/kernel/msm/+/3fbe6bc28a6b9939d0650f2f17eb5216c719950c
  //
  bool cache_flush_success = true;
  if (HasDualCodeMapping()) {
    // Flush d-cache for the non-executable mapping.
    cache_flush_success = FlushCpuCaches(w_memory, w_memory + total_size);
  }

  // Invalidate i-cache for the executable mapping.
  if (cache_flush_success) {
    cache_flush_success = FlushCpuCaches(x_memory, x_memory + total_size);
  }

  // If flushing the cache has failed, reject the allocation because we can't guarantee
  // correctness of the instructions present in the processor caches.
  if (!cache_flush_success) {
    PLOG(ERROR) << "Cache flush failed triggering code allocation failure";
    return nullptr;
  }

  // Ensure CPU instruction pipelines are flushed for all cores. This is necessary for
  // correctness as code may still be in instruction pipelines despite the i-cache flush. It is
  // not safe to assume that changing permissions with mprotect (RX->RWX->RX) will cause a TLB
  // shootdown (incidentally invalidating the CPU pipelines by sending an IPI to all cores to
  // notify them of the TLB invalidation). Some architectures, notably ARM and ARM64, have
  // hardware support that broadcasts TLB invalidations and so their kernels have no software
  // based TLB shootdown. The sync-core flavor of membarrier was introduced in Linux 4.16 to
  // address this (see mbarrier(2)). The membarrier here will fail on prior kernels and on
  // platforms lacking the appropriate support.
  art::membarrier(art::MembarrierCommand::kPrivateExpeditedSyncCore);

  return result;
}
```

# 二、JIT阈值控制与处理
JIT阈值控制与处理包含两个部分
- **性能统计埋点**：解释执行某个Java方法时，在一些JIT关注的地方埋点以更新统计信息，所谓的“埋点”，就是指在一些关键地方调用JIT模块的性能统计相关的成员函数，在ART JIT模块中，最终的性能统计和处理函数是Jit类的AddSamples。
- Jit AddSamples检查当前所执行方法的性能统计信息，根据不同阈值的设置进行不同的处理。
## 2.1 性能统计埋点
JIT的性能统计埋点主要用AddSamples来计数，一共有如下几处地方。
### 2.1.1 EnterInterpreterFromEntryPoint
```cpp
JValue EnterInterpreterFromEntryPoint(Thread* self, const CodeItemDataAccessor& accessor,
                                      ShadowFrame* shadow_frame) {
...
// 通知JIT模块，表示本次调用是从一个机器码到解释执行的转换。其内部将调用AddSamples最终的计算的执行次数为
  if (jit != nullptr) {
    jit->NotifyCompiledCodeToInterpreterTransition(self, shadow_frame->GetMethod());
  }
  return Execute(self, accessor, *shadow_frame, JValue());
}
```
### 2.1.2 Execute
```cpp
static inline JValue Execute(Thread* self, const CodeItemDataAccessor& accessor, ShadowFrame& shadow_frame, JValue result_register, bool stay_in_interpreter = false, bool from_deoptimize = false) REQUIRES_SHARED(Locks::mutator_lock_) {
...
    if (!stay_in_interpreter && !self->IsForceInterpreter()) {
      jit::Jit* jit = Runtime::Current()->GetJit();
      if (jit != nullptr) {
      /* 内部调用Addamples，性能统计次数增加1 */
        jit->MethodEntered(self, shadow_frame.GetMethod());
        ...
    }
  }
}
```

# 三、OSR的处理
On Stack Replacement是从解释执行模式切换到机器码执行模式的关键技术。
```cpp

```