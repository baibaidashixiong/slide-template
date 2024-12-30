- runtime.cc中创建JIT需要有jit_code_cache和NotSafeMode的支持：
```c++
void Runtime::CreateJit() {
  if (jit_code_cache_.get() == nullptr) {
    if (!IsSafeMode()) {
      LOG(WARNING) << "Missing code cache, cannot create JIT.";}
    return;}
  if (IsSafeMode()) {
    LOG(INFO) << "Not creating JIT because of SafeMode.";
    jit_code_cache_.reset();
    return;
  }

  jit::Jit* jit = jit::Jit::Create(jit_code_cache_.get(), jit_options_.get());
  DoAndMaybeSwitchInterpreter([=](){ jit_.reset(jit); });
  if (jit == nullptr) {
    LOG(WARNING) << "Failed to allocate JIT";
    // Release JIT code cache resources (several MB of memory).
    jit_code_cache_.reset();
  } else {
    jit->CreateThreadPool();
  }
}
```
## ART方法调用解析
### 一、提前优化
使用 Android-Studio 编译应用时，实际上是通过 Java 编译器先将 `.java` 代码编译为对应的 Java 字节码，即 `.class` 类文件；然后用 `dx`(在新版本中是`d8`) 将 Java 字节码转换为 Dalvik 字节码，并将所有生成的类打包到统一的 DEX 文件中，最终和资源文件一起 zip 压缩为 `.apk` 文件。
在安装用户的 APK 时，Android 系统主要通过 PacketManager 对应用进行解包和安装。其中在处理 DEX 文件时候，会通过 **installd** 进程调用对应的二进制程序对字节码进行优化，这对于 Dalvik 虚拟机而言使用的是 **dexopt** 程序，而 ART 中使用的是 **dex2oat** 程序。
dexopt 将 dex 文件优化为 odex 文件，即 optimized-dex 的缩写，其中包含的是优化后的 Dalvik 字节码，称为 quickend dex；dex2oat 基于 LLVM，优化后生成的是对应平台的二进制代码，以 oat 格式保存，oat 的全称为 Ahead-Of-Time。oat 文件实际上是以 ELF 格式进行存储的，并在其中 oatdata 段(section) 包含了原始的 DEX 内容。
在 Android 8 之后，将 OAT 文件一分为二，原 oat 仍然是 ELF 格式，但原始 DEX 文件内容被保存到了 VDEX 中，VDEX 有其独立的文件格式。整体流程如下图所示:
![[Pasted image 20240130213724.png]]
值得一提的是，在 Andorid 系统中 dex2oat 会将优化后的代码保存在 `/data/app` 对应的应用路径下，系统应用会保存在 `/data/dalvik-cache/` 下，对于后者，产生的实际有三个文件，比如:
```bash
$ ls -l | grep Settings.apk
-rw-r----- 1 system system           77824 2021-12-10 10:33 system_ext@priv-app@Settings@Settings.apk@classes.art
-rw-r----- 1 system system          192280 2021-11-19 12:50 system_ext@priv-app@Settings@Settings.apk@classes.dex
-rw-r----- 1 system system           59646 2021-12-10 10:33 system_ext@priv-app@Settings@Settings.apk@classes.vdex
```
`system_ext@priv-app@Settings@Settings.apk@classes.dex` 实际上是 ELF 格式的 OAT 文件，所以我们不能以貌(后缀)取人；`.art` 也是一个特殊的文件格式，如前文所言，Android 实现了自己的 Java 虚拟机，这个虚拟机本身是用 C/C++ 实现的，其中的一些 Java 原语有对应的 C++ 类，比如:

- java.lang.Class 对应 art::mirror::Class
- java.lang.String 对应 art::mirror::String
- java.lang.reflect.Method 对应 art::mirror::Method
- ……

当创建一个 Java 对象时，内存中会创建对应的 C++ 对象并调用其构造函数，JVM 管理者这些 C++ 对象的引用。为了加速启动过程，避免对这些常见类的初始化，Android 使用了 `.art` 格式来保存这些 C++ 对象的实例，简单来说，art 文件可以看做是一系列常用 C++ 对象的内存 dump。

不论是 oat、vdex 还是 art，都是 Android 定义的内部文件格式，官方并不保证其兼容性，事实上在 Android 各个版本中这些文件格式都有不同程度的变化，这些变化是不反映在文档中的，只能通过代码去一窥究竟。因此对于这些文件格式我们现在只需要知道其大致作用，无需关心其实现细节。
### 二、方法调用
本来按照时间线来看的话，这里应该先介绍 ART 运行时类和方法的加载过程，但从实践出发，先看 Java 方法的调用过程，并针对其中涉及到的概念在下一节继续介绍。

在 Web 安全中，Java 服务端通常带有一个称为 RASP (Runtime Application Self-Protection) 的动态防护方案，比如监控某些执行命令的敏感函数调用并进行告警，其实际 hook 点是在 JVM 中，不论是方法直接调用还是反射调用都可以检测到。因此我们有理由猜测在 Android 中也有类似的调用链路，为了方便观察，这里先看反射调用的场景，一般反射调用的示例如下:
```java
import java.lang.reflect.*;
public class Test {
    public static void main(String args[]) throws Exception {
        Class c = Class.forName("com.evilpan.DemoClass");
        Method m = c.getMethod("foo", null);
        m.invoke();
    }
}
```
因此一个方法的调用会进入到 `Method.invoke` 方法，这是一个 **native 方法**，实际实现在 _art/runtime/native/java_lang_reflect_Method.cc_:
```c++
static jobject Method_invoke(JNIEnv* env, jobject javaMethod, jobject javaReceiver,
                             jobjectArray javaArgs) {
  ScopedFastNativeObjectAccess soa(env);
  return InvokeMethod<kRuntimePointerSize>(soa, javaMethod, javaReceiver, javaArgs);
}
```
InvokeMethod 定义在 _art/runtime/reflection.cc_，其实现的核心代码如下:
```c++
template <PointerSize kPointerSize>
jobject InvokeMethod(const ScopedObjectAccessAlreadyRunnable& soa, jobject javaMethod,
                     jobject javaReceiver, jobject javaArgs, size_t num_frames) {
    ObjPtr<mirror::Executable> executable = soa.Decode<mirror::Executable>(javaMethod);
    const bool accessible = executable->IsAccessible();
    ArtMethod* m = executable->GetArtMethod();

    if (UNLIKELY(!declaring_class->IsVisiblyInitialized())) {
        Thread* self = soa.Self();
        Runtime::Current()->GetClassLinker()->EnsureInitialized(
            self, h_class,
            /*can_init_fields=*/ true,
            /*can_init_parents=*/ true)
    }

    if (!m->IsStatic()) {
        if (declaring_class->IsStringClass() && m->IsConstructor()) {
            m = WellKnownClasses::StringInitToStringFactory(m);
        } else {
            m = receiver->GetClass()->FindVirtualMethodForVirtualOrInterface(m, kPointerSize);
        }
    }

    if (!accessible && !VerifyAccess(/*...*/)) {
        ThrowIllegalAccessException(
        StringPrintf("Class %s cannot access %s method %s of class %s", ...));
    }

    InvokeMethodImpl(soa, m, np_method, receiver, objects, &shorty, &result);
}
```
上面省略了许多细节，主要是做了一些调用前的检查和预处理工作，流程可以概况为:

1. 判断方法所属的类是否已经初始化过，如果没有则进行初始化；
2. 将 `String.<init>` 构造函数调用替换为对应的工厂 `StringFactory` 方法调用；
3. 如果是虚函数调用，替换为运行时实际的函数；
4. 判断方法是否可以访问，如果不能访问则抛出异常；
5. 调用函数；

值得注意的是，jobject 类型的 javaMethod 可以转换为 `ArtMethod` 指针，该结构体是 ART 虚拟机中对于具体方法的描述。之后经过一系列调用:

- InvokeMethodImpl
- InvokeWithArgArray
- `method->Invoke()`

最终进入 `ArtMethod::Invoke` 函数，还是只看核心代码:
```c++
void ArtMethod::Invoke(Thread* self, uint32_t* args, uint32_t args_size, JValue* result,
                       const char* shorty) {
    Runtime* runtime = Runtime::Current();
    if (UNLIKELY(!runtime->IsStarted() ||
               (self->IsForceInterpreter() && !IsNative() && !IsProxyMethod() && IsInvokable()))) {
        art::interpreter::EnterInterpreterFromInvoke(...);
    } else {
        bool have_quick_code = GetEntryPointFromQuickCompiledCode() != nullptr;
        if (LIKELY(have_quick_code)) {
            if (!IsStatic()) {
                (*art_quick_invoke_stub)(this, args, args_size, self, result, shorty);
            } else {
                (*art_quick_invoke_static_stub)(this, args, args_size, self, result, shorty);
            }
        } else {
            LOG(INFO) << "Not invoking '" << PrettyMethod() << "' code=null";
        }
    }
    self->PopManagedStackFragment(fragment);
}
```
ART 对于 Java 方法实现了两种执行模式，一种是像 Dalvik 虚拟机一样解释执行字节码，姑且称为解释模式；另一种是快速模式，即直接调用通过 OAT 编译后的本地代码。
阅读上述代码可以得知，当 ART 运行时尚未启动或者指定强制使用解释执行时，虚拟机执行函数使用的是解释模式，ART 可以在启动时指定 `-Xint` 参数强制使用解释执行，但即便指定了使用解释执行模式，还是有一些情况无法使用解释执行，比如:

1. 当所执行的方法是 Native 方法时，这时只有二进制代码，不存在字节码，自然无法解释执行；
2. 当所执行的方法无法调用，比如 access_flag 判定无法访问或者当前方法是抽象方法时；
3. 当所执行的方式是代理方法时，ART 对于代理方法有单独的本地调用方式；

### 三、解释执行

解释执行的入口是 `art::interpreter::EnterInterpreterFromInvoke`，该函数定义在 _art/runtime/interpreter/interpreter.cc_，关键代码如下：
```c++
void EnterInterpreterFromInvoke(Thread* self,
                                ArtMethod* method,
                                ObjPtr<mirror::Object> receiver,
                                uint32_t* args,
                                JValue* result,
                                bool stay_in_interpreter) {
    CodeItemDataAccessor accessor(method->DexInstructionData());
    if (accessor.HasCodeItem()) {
        num_regs =  accessor.RegistersSize();
        num_ins = accessor.InsSize();
    }
    // 初始化栈帧 ......
    if (LIKELY(!method->IsNative())) {
        JValue r = Execute(self, accessor, *shadow_frame, JValue(), stay_in_interpreter);
        if (result != nullptr) {
        *result = r;
        }
  }
}
```
其中的 `CodeItem` 就是 DEX 文件中对应方法的字节码，还是老样子，直接看简化的调用链路：

| method | file |
| ---- | ---- |
| Execute | art/runtime/interpreter/interpreter.cc |
| ExecuteSwitchImpl | art/runtime/interpreter/interpreter_switch_impl.h |
| ExecuteSwitchImplAsm | … |
| ExecuteSwitchImplAsm | art/runtime/arch/arm64/quick_entrypoints_arm64.S |
| ExecuteSwitchImplCpp | art/runtime/interpreter/interpreter_switch_impl-inl.h |
`ExecuteSwitchImplAsm` 为了速度直接使用汇编实现，在 ARM64 平台中的定义如下：
```c
//  Wrap ExecuteSwitchImpl in assembly method which specifies DEX PC for unwinding.
//  Argument 0: x0: The context pointer for ExecuteSwitchImpl.
//  Argument 1: x1: Pointer to the templated ExecuteSwitchImpl to call.
//  Argument 2: x2: The value of DEX PC (memory address of the methods bytecode).
ENTRY ExecuteSwitchImplAsm
    SAVE_TWO_REGS_INCREASE_FRAME x19, xLR, 16
    mov x19, x2                                   // x19 = DEX PC
    CFI_DEFINE_DEX_PC_WITH_OFFSET(0 /* x0 */, 19 /* x19 */, 0)
    blr x1                                        // Call the wrapped method.
    RESTORE_TWO_REGS_DECREASE_FRAME x19, xLR, 16
    ret
END ExecuteSwitchImplAsm
```
本质上是调用保存在 x1 寄存器的第二个参数，调用处的代码片段如下：
```c++
template<bool do_access_check, bool transaction_active>
ALWAYS_INLINE JValue ExecuteSwitchImpl() {
    //...
    void* impl = reinterpret_cast<void*>(&ExecuteSwitchImplCpp<do_access_check, transaction_active>);
    const uint16_t* dex_pc = ctx.accessor.Insns();
    ExecuteSwitchImplAsm(&ctx, impl, dex_pc);
}
```
即调用了 `ExecuteSwitchImplCpp`，在该函数中，可以看见典型的解释执行代码：
```c++
template<bool do_access_check, bool transaction_active>
void ExecuteSwitchImplCpp(SwitchImplContext* ctx) {
    Thread* self = ctx->self;
    const CodeItemDataAccessor& accessor = ctx->accessor;
    ShadowFrame& shadow_frame = ctx->shadow_frame;
    self->VerifyStack();

    uint32_t dex_pc = shadow_frame.GetDexPC();
    const auto* const instrumentation = Runtime::Current()->GetInstrumentation();
    const uint16_t* const insns = accessor.Insns();
    const Instruction* next = Instruction::At(insns + dex_pc);

    while (true) {
        const Instruction* const inst = next;
        dex_pc = inst->GetDexPc(insns);
        shadow_frame.SetDexPC(dex_pc);
        TraceExecution(shadow_frame, inst, dex_pc);
        uint16_t inst_data = inst->Fetch16(0); // 一条指令 4 字节

        if (InstructionHandler(...).Preamble()) {
            switch (inst->Opcode(inst_data)) {
                case xxx: ...;
                case yyy: ...;
                ...
            }
        }
    }
}
```
在当前版本中 (Android 12)，实际上是通过宏展开去定义了所有 op_code 的处理分支，不同版本实现都略有不同，但解释执行的核心思路从 Android 2.x 版本到现在都是一致的，因为字节码的定义并没有太多改变。

### 四、快速执行
再回到 ArtMethod 真正调用之前，如果不使用解释模式执行，则通过 **`art_quick_invoke_stub`** 去调用。stub 是一小段中间代码，用于跳转到实际的 native 执行，该符号使用汇编实现，在 ARM64 中的定义在 _art/runtime/arch/arm64/quick_entrypoints_arm64.S_，核心代码如下：
```c++
.macro INVOKE_STUB_CALL_AND_RETURN
    REFRESH_MARKING_REGISTER
    REFRESH_SUSPEND_CHECK_REGISTER

    // load method-> METHOD_QUICK_CODE_OFFSET
    ldr x9, [x0, #ART_METHOD_QUICK_CODE_OFFSET_64]
    // Branch to method.
    blr x9
.endm

/*
 *  extern"C" void art_quick_invoke_stub(ArtMethod *method,   x0
 *                                       uint32_t  *args,     x1
 *                                       uint32_t argsize,    w2
 *                                       Thread *self,        x3
 *                                       JValue *result,      x4
 *                                       char   *shorty);     x5
 */
ENTRY art_quick_invoke_stub
    // ...
    INVOKE_STUB_CALL_AND_RETURN
END art_quick_invoke_static_stub
```
中间省略了一些保存上下文以及调用后恢复寄存器的代码，其核心是调用了 `ArtMethod` 结构体偏移 `ART_METHOD_QUICK_CODE_OFFSET_64` 处的指针，该值对应的代码为：
```c++
ASM_DEFINE(ART_METHOD_QUICK_CODE_OFFSET_64,
           art::ArtMethod::EntryPointFromQuickCompiledCodeOffset(art::PointerSize::k64).Int32Value())
```
即 `entry_point_from_quick_compiled_code_` 属性所指向的地址。
```c++
// art/runtime/art_method.h
static constexpr MemberOffset EntryPointFromQuickCompiledCodeOffset(PointerSize pointer_size) {
return MemberOffset(PtrSizedFieldsOffset(pointer_size) + OFFSETOF_MEMBER(
    PtrSizedFields, entry_point_from_quick_compiled_code_) / sizeof(void*)
        * static_cast<size_t>(pointer_size));
}
```
可以认为这就是所有快速模式执行代码的入口，至于该指针指向什么地方，又是什么时候初始化的，可以参考下一节代码加载部分。实际在方法调用时，快速模式执行的方法可能在其中执行到了需要以解释模式执行的方法，同样以解释模式执行的方法也可能在其中调用到 JNI 方法或者其他以快速模式执行的方法，所以在单个函数执行的过程中运行状态并不是一成不变的，但由于每次切换调用前后都保存和恢复了当前上下文，使得不同调用之间可以保持透明，这也是模块化设计的一大优势所在。
### 五、代码加载
在上节我们知道在 ART 虚拟机中，Java 方法的调用主要通过 `ArtMethod::Invoke` 去实现，那么 ArtMethod 结构是什么时候创建的呢？为什么 jmethod/jobject 可以转换为 `ArtMethod` 指针呢？
在 Java 这门语言中，方法是需要依赖类而存在的，因此要分析方法的初始化需要先分析类的初始化。虽然我们前面知道如何从 OAT/VDEX/DEX 文件中构造对应的 ClassLoader 来进行类查找，但那个时候类并没有初始化，可以编写一个简单的类进行验证：
```java
public class Demo {
    static {
        Log.i("Demo", "static block called");
    }
    {
        Log.i("Demo", "IIB called");
    }
}
```
如果 Demo 类在代码中没有使用，那么上述两个打印都不会触发；如果使用 `Class.forName("Demo")` 进行反射引用，则 static block 中的代码会被调用。跟踪 Class.forName 调用:
```java
@CallerSensitive
public static Class<?> forName(String className)
            throws ClassNotFoundException {
    Class<?> caller = Reflection.getCallerClass();
    // 笔者注: initialize = true
    return forName(className, true, ClassLoader.getClassLoader(caller));
}
```
最终调用到名为 `classForName` 的 native 方法，其定义在 _art/runtime/native/java_lang_Class.cc_:
```c++
// "name" is in "binary name" format, e.g. "dalvik.system.Debug$1".
static jclass Class_classForName(JNIEnv* env, jclass, jstring javaName, jboolean initialize,
                                 jobject javaLoader) {
    ScopedFastNativeObjectAccess soa(env);
    ScopedUtfChars name(env, javaName);

    std::string descriptor(DotToDescriptor(name.c_str()));
    Handle<mirror::ClassLoader> class_loader(
      hs.NewHandle(soa.Decode<mirror::ClassLoader>(javaLoader)));
    ClassLinker* class_linker = Runtime::Current()->GetClassLinker();
    Handle<mirror::Class> c(
      hs.NewHandle(class_linker->FindClass(soa.Self(), descriptor.c_str(), class_loader)));

    if (initialize) {
        class_linker->EnsureInitialized(soa.Self(), c, true, true);
    }
    return soa.AddLocalReference<jclass>(c.Get());
}
```
首先将 Java 格式的类表示转换为 smali 格式，然后通过指定的 class_loader 去查找类，查找过程主要通过 `class_linker` 实现。由于 forName 函数中指定了 `initialize` 为 `true`，因此在找到对应类后还会额外执行一步 `EnsureInitialized`，在后文会进行详细介绍。
### 六、FindClass
FindClass 实现了根据类名查找类的过程，定义在 _art/runtime/class_linker.cc_ 中，关键流程如下:
```c++
ObjPtr<mirror::Class> ClassLinker::FindClass(Thread* self,
                                             const char* descriptor,
                                             Handle<mirror::ClassLoader> class_loader) 
    if (descriptor[1] == '\0') 
        return FindPrimitiveClass(descriptor[0]);

    const size_t hash = ComputeModifiedUtf8Hash(descriptor);
    // 在已经加载的类中查找
    ObjPtr<mirror::Class> klass = LookupClass(self, descriptor, hash, class_loader.Get());
    if (klass != nullptr) {
        return EnsureResolved(self, descriptor, klass);
    }
    // 尚未加载
    if (descriptor[0] != '[' && class_loader == nullptr) {
        // 类加载器为空，且不是数组类型，在启动类中进行查找
        ClassPathEntry pair = FindInClassPath(descriptor, hash, boot_class_path_);
        return DefineClass(self, descriptor, hash,
                           ScopedNullHandle<mirror::ClassLoader>(),
                           *pair.first, *pair.second);
    }

    ObjPtr<mirror::Class> result_ptr;
    bool descriptor_equals;
    ScopedObjectAccessUnchecked soa(self);
    // 先通过 classLoader 的父类查找
    bool known_hierarchy =
        FindClassInBaseDexClassLoader(soa, self, descriptor, hash, class_loader, &result_ptr);
    if (result_ptr != nullptr) {
        descriptor_equals = true;
    } else if (!self->IsExceptionPending()) {
        // 如果没找到，再通过 classLoader 查找
        std::string class_name_string(descriptor + 1, descriptor_length - 2);
        std::replace(class_name_string.begin(), class_name_string.end(), '/', '.');
        ScopedLocalRef<jobject> class_loader_object(
            soa.Env(), soa.AddLocalReference<jobject>(class_loader.Get()));
        ScopedLocalRef<jobject> result(soa.Env(), nullptr);
        result.reset(soa.Env()->CallObjectMethod(class_loader_object.get(),
                                                 WellKnownClasses::java_lang_ClassLoader_loadClass,
                                                 class_name_object.get()));
    }

    // 将找到的类插入到缓存表中
    ClassTable* const class_table = InsertClassTableForClassLoader(class_loader.Get());
    class_table->InsertWithHash(result_ptr, hash);

    return result_ptr;
}
```
首先会通过 `LookupClass` 在已经加载的类中查找，已经加载的类会保存在 ClassTable 中，以 hash 表的方式存储，该表的键就是类对应的 hash，通过 descriptor 计算得出。如果之前已经加载过，那么这时候就可以直接返回，如果没有就需要执行真正的加载了。从这里我们也可以看出，类的加载过程属于懒加载 (lazy loading)，如果一个类不曾被使用，那么是不会有任何加载开销的。
然后会判断指定的类加载器是否为空，为空表示要查找的类实际上是一个系统类。系统类不存在于 APP 的 DEX 文件中，而是 Android 系统的一部分。由于每个 Android (Java) 应用都会用到系统类，为了提高启动速度，实际通过 zygote 去加载，并由所有子进程一起共享。上述 `boot_class_path_` 数组在 `Runtime::Init` 中通过 ART 启动的参数进行初始化，感兴趣的可以自行研究细节。
### 七、LinkCode
LinkCode 顾名思义是对代码进行链接，关键代码如下：
```c++
static void LinkCode(ClassLinker* class_linker,
                     ArtMethod* method,
                     const OatFile::OatClass* oat_class,
                     uint32_t class_def_method_index) {
    Runtime* const runtime = Runtime::Current();
    const void* quick_code = nullptr;
    if (oat_class != nullptr) {
         // Every kind of method should at least get an invoke stub from the oat_method.
         // non-abstract methods also get their code pointers.
         const OatFile::OatMethod oat_method = oat_class->GetOatMethod(class_def_method_index);
         quick_code = oat_method.GetQuickCode();
    }
    runtime->GetInstrumentation()->InitializeMethodsCode(method, quick_code);

    if (method->IsNative()) {
    // Set up the dlsym lookup stub. Do not go through `UnregisterNative()`
    // as the extra processing for @CriticalNative is not needed yet.
        method->SetEntryPointFromJni(
            method->IsCriticalNative() ? GetJniDlsymLookupCriticalStub() : GetJniDlsymLookupStub());
  }
}
```
其中 quick_code 指针指向的是 OatMethod 中的 **`code_offset_`** 偏移处的值，该值指向的是 OAT 优化后的本地代码位置。`InitializeMethodsCode` 是 **`Instrumentation`** 类的方法，实现在 _art/runtime/instrumentation.cc_，如果看过之前分析应用启动流程的文章应该对这个类不会陌生，尽管不是同一个类，但它们的功能却是类似的，即作为某些关键调用的收口，并在其中实现可插拔的追踪行为。其内部实现如下：
```c++
void Instrumentation::InitializeMethodsCode(ArtMethod* method, const void* aot_code) {
    // Use instrumentation entrypoints if instrumentation is installed.
    if (UNLIKELY(EntryExitStubsInstalled())) {
        if (!method->IsNative() && InterpretOnly()) {
            UpdateEntryPoints(method, GetQuickToInterpreterBridge());
        } else {
            UpdateEntryPoints(method, GetQuickInstrumentationEntryPoint());
        }
        return;
    }
    if (UNLIKELY(IsForcedInterpretOnly())) {
        UpdateEntryPoints(
            method, method->IsNative() ? GetQuickGenericJniStub() : GetQuickToInterpreterBridge());
        return;
    }
    // Use the provided AOT code if possible.
    if (CanUseAotCode(method, aot_code)) {
        UpdateEntryPoints(method, aot_code);
        return;
    }
    // Use default entrypoints.
    UpdateEntryPoints(
      method, method->IsNative() ? GetQuickGenericJniStub() : GetQuickToInterpreterBridge());
}
```
第一部分正是用于追踪的判断，如果当前已经安装了追踪监控，那么会根据当前方法的类别分别设置对应的入口点；否则就以常规方式设置方法的调用入口:

- 对于强制解释执行的运行时环境:
    - 如果是 Native 方法则将入口点设置为 `art_quick_generic_jni_trampoline`，用于跳转执行 JNI 本地代码；
    - 对于 Java 方法则将入口点设置为 `art_quick_to_interpreter_bridge`，使方法调用过程会跳转到解释器继续；
- 如果 AOT 编译的本地代码可用，则直接将方法入口点设置为 AOT 代码；
- 如果 AOT 代码不可用，那么就回到解释执行场景进行处理；

设置 ArtMethod 入口地址的方法是 UpdateEntryPoints，其内部实现非常简单:
```c++
static void UpdateEntryPoints(ArtMethod* method, const void* quick_code)
    REQUIRES_SHARED(Locks::mutator_lock_) {
    if (kIsDebugBuild) {
        ...
    }
    // If the method is from a boot image, don't dirty it if the entrypoint
    // doesn't change.
    if (method->GetEntryPointFromQuickCompiledCode() != quick_code) {
        method->SetEntryPointFromQuickCompiledCode(quick_code);
    }
}
```
内部实质上是调用了 `ArtMethod::SetEntryPointFromQuickCompiledCode`:
```c++
void SetEntryPointFromQuickCompiledCode(const void* entry_point_from_quick_compiled_code)
      REQUIRES_SHARED(Locks::mutator_lock_) {
    SetEntryPointFromQuickCompiledCodePtrSize(entry_point_from_quick_compiled_code,
                                              kRuntimePointerSize);
  }
```
回顾我们前面分析方法调用的章节，对于快速执行的场景，`ArtMethod::Invoke` 最终是跳转到 **`entry_point_from_quick_compiled_code`** 进行执行，而这个字段就是在这里进行设置的。

## ART-JIT记录
- 与dex2oat相同，JIT编译也使用OptimizingCompiler。
- ART虚拟机中与JIT模块相关的类：
	- **Jit**类是ART JIT模块的门户，JIT提供的主要功能将借助该类的相关成员函数对外输出。
	- **JitCodeCache**管理用来存储JIT编译结果的一段内存空间。当内存不够用时，这部分内存空间可以被回收。
	- **JitCompiler**用于处理JIT相关的编译。其内部用到的编译模块和dex2oat里AOT编译用到的模块同为OptimizingCompiler。从编译角度看ART中的JIT和AOT用到的技术没有区别，只不过编译的时机不同而已。
	- **ProfilingInfo**类用于管理一个方法的性能统计信息。
- JIT->OptimizingCompiler调用链条：`Jit::Create -> JitCompilerInterface* jit_create -> JitCompiler::Create() -> JitCompiler::JitCompiler() -> Compiler::Create -> Compiler* CreateOptimizingCompiler -> OptimizingCompiler::Compile`进入OptimizingCompiler进行编译。
```bash
+ JitCompiler: +Create()
|              +CompileMethod()
|
+-+-> Jit: +CompileMethod()  # 对方法method进行JIT编译
  |        +Create()         # 用于创建Jit对象
  |        +MethodEntered()
  |        +AddSamples()     # 对某个方法进行性能统计
  |        +LoadCompiler()
  |
  +-+->JitCodeCache: +Create()
    |                +GarbageCollectCache()
    |
    +-->ProfilingInfo: +Create()
                       +AddInvokeInfo()
```

## Compile方法记录
- CompileMethod记录编译方法，同时也为jit和aot编译的并经之路。其有一个成员变量`const LengthPrefixedArray<uint8_t>* const quick_code_`，指向一个定长数组(LengthPrefixedArray)，表示一个常量对象的常量指针，元素的类型为uint8_t。该数组存储的就是编译得到的机器码。
- QuickMethodFrameInfo中有几个重要的成员变量：
	- `frame_size_in_bytes_`：栈帧大小。
	- `core_spill_mask_`：32位长（uint32_t），每一位对应一个核心寄存器，值为1表示寄存器的内容可能会存储在栈山，分配栈时需要预留相应的空间。
	- `fp_spill_mask_`：作用和`core_spill_mask_`一样，每一位代表一个浮点寄存器。