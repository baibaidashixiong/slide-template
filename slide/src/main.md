---
title: 2024 年度总结
separator: <!--s-->
verticalSeparator: <!--v-->
theme: simple
highlightTheme: github
css: custom.css
revealOptions:
    transition: 'slide'
    transitionSpeed: fast
    center: false
    slideNumber: "c/t"
    width: 1000
    data-maxlines: 10
---

<div class="middle center">
<div style="width: 100%">

# 2024 年度总结
<hr/>

By [朱奇正](https://github.com/baibaidashixiong)

<div style="text-align: right; margin-top: 1em;">
<p>2024.1.9&emsp;&emsp;&emsp;</p>
</div>
</div>
</div>

<!--s-->

<div class="middle center">
<div style="width: 100%">

# 目录 
<hr/>
1. 完成JNI/汇编器(1~4月)<br>
2. 天基云项目(7~10月)<br>
3. 安卓移植(11月~现在)<br>

</div>
</div>


<!--s-->

<div class="middle center">
<div style="width: 100%">

# Part.1 
<hr/>
完成JNI/汇编器

</div></div>
<!--v-->
## &nbsp;&nbsp;&nbsp;&nbsp;完成JNI

- 设置通用入口点
  - art_quick_generic_jni_trampoline
    - jni trampoline

```mermaid
                                   +--> lookup stub(for lazy load)
                                   |
art_quick_generic_jni_trampoline --+--> native code
                                   |
                                   +--> jni_trampoline --> native code
```


<!--v-->

## &nbsp;&nbsp;&nbsp;完成JNI

<hr/>

- JNI Compiler

```java [1-3|4-5|6-7|9-17|19-20] {data-maxlines=5}
// 1. Build the frame saving all callee saves, Method*, and PC return address.
//    For @CriticalNative, this includes space for out args, otherwise just the managed frame.
  __ BuildFrame(current_frame_size, method_register, callee_save_regs);
// 2. Write out the end of the quick frames.
  __ StoreStackPointerToThread(Thread::TopOfManagedStackOffset<kPointerSize>());
// 3. Move frame down to allow space for out going args.
  __ IncreaseFrameSize(main_out_arg_size);
...
// 8. Create 1st argument, the JNI environment ptr.
// Register that will hold local indirect reference table
  if (main_jni_conv->IsCurrentParamInRegister()) {
    ManagedRegister jni_env = main_jni_conv->CurrentParamRegister();
    __ LoadRawPtrFromThread(jni_env, Thread::JniEnvOffset<kPointerSize>());
  } else {
    FrameOffset jni_env = main_jni_conv->CurrentParamStackOffset();
    __ CopyRawPtrFromThread(jni_env, Thread::JniEnvOffset<kPointerSize>());
  }
...
// 18. Finalize code generation
  __ FinalizeCode();
```

<!--v-->
## &nbsp;&nbsp;&nbsp;&nbsp;完成汇编器

- Compiler Test(Based on GTEST)

<!--v-->
## &nbsp;&nbsp;&nbsp;&nbsp;完成汇编器

- Linker
  - Emit - Reverse(nop) - Relocate
<div style="text-align: center;">
<br/>
<img src="linker.png" width="100%" style="margin: 0 auto;">
</div>

<!--v-->

## 标题

<div class="three-line">

|表头 a|表头 b|表头 c|
|:--:|:--:|:--:|
|这是一个|一些内容|...|
|三线表|...|...|

</div>

|表头 a|表头 b|表头 c|
|:--:|:--:|:--:|
|这是一个|一些内容|...|
|普通表格|...|...|

<!--s-->

<div class="middle center">
<div style="width: 100%">

# Part.2 布局

</div>
</div>

<!--v-->

## 多列布局

<div class="mul-cols">
<div class="col">

第一列

- list
- list

</div>
<div class="col">

第二列

```python
class MyClass:
    def __init__(self, ...):
        ...
    def method(self, ...):
        ...
```

</div>
</div>

<div class="mul-cols">
<div class="col">

第一列

- list
- list

</div>
<div class="col">

第二列

1. list 
2. list 
    - list

</div>
<div class="col">

第三列

```python
class MyClass:
    def __init__(self, ...):
        ...
    def method(self, ...):
        ...
```

</div>
</div>
