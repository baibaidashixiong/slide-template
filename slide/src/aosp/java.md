- `art_quick_alloc_string_from_string_region_tlab_instrumented`.
- TLAB
- 新生代Eden
- 老年代
- field
- classloader
- 在Java中，描述符(Descriptor)是一种紧凑的形式,用于表示Java类型系统中的基本类型和引用类型。其是一个字符串，代表字段field或方法的类型。其常用于描述字段、方法参数和返回类型等。
```bash
# 字段描述符 Field Descriptors
FieldDescriptor: [FieldType]
FieldType:[BaseType] | [ObjectType] | [ArrayType]
BaseType:(one of) `B(yte)` `C(har)` `D(ouble)` `F(loat)` `I(nt)` `J(long)` `S(hort)` `Z(boolean)`
ObjectType: `L` ClassName `;`  (e.g. Ljava/lang/Object; Ljava/lang/String)
ArrayType: `[` [ComponentType] (e.g. `double[][][]` is `[[[D` )
ComponentType: [FieldType]

# 方法描述符 Method Descriptors
MethodDescriptor:`(` {[ParameterDescriptor]} `)` [ReturnDescriptor]
ParameterDescriptor: [FieldType]
ReturnDescriptor: [FieldType] | [VoidDescriptor]
VoidDescriptor: `V`
e.g. `Object m(int i, double d, Thread t) {...}` is `(IDLjava/lang/Thread;)Ljava/lang/Object;`
      `int[] foo(int i, int i2, Object o) {...}` is `(IILjava/lang/Object;)[I`
```
- 何为native method？一个java method是如何被判定为native method的？
- art的`runtime/native/*.cc`中`static JNINativeMethod gMethods`定义了native method。
-  `java.lang/java.net`等其中的各个方法为java api。
- jdk中生成jni assembler的方法为`SharedRuntime::generate_native_wrapper`。
- 启动类加载器？java中的`java.lang.ClassLoader`类.
- 一大难点：java的类加载机制
- java的合成方法(Synthetic Methods)和内部类。
### java native方法
- `libcore/ojluni/src/main/native/jvm.h`中定义了各个native jvm api的方法，其native实现位与art中，如**java.lang.System.currentTimeMillis**()方法，其定义在`libcore/ojluni/src/main/java/java/lang/System.java`中（类似的，在jdk中其位与目录`jdk21u/src/java.base/share/classes/java/lang`下）：
```bash
@CriticalNative
public static native long currentTimeMillis();
```
- 其在`libcore/ojluni/src/main/native/System.c`被转换：
```bash
static jlong System_currentTimeMillis() {
  return JVM_CurrentTimeMillis(NULL, NULL);
}
```
- 其在`libcore/ojluni/src/main/native/jvm.h`中被定义：
```bash
/*
 * java.lang.System
 */
JNIEXPORT jlong JNICALL
JVM_CurrentTimeMillis(JNIEnv *env, jclass ignored);
```
- 其在`art/openjdkjvm/OpenjdkJvm.cc`中被实现：
```bash
JNIEXPORT jlong JVM_CurrentTimeMillis(JNIEnv* env ATTRIBUTE_UNUSED,
                                      jclass clazz ATTRIBUTE_UNUSED) {
    struct timeval tv;
    gettimeofday(&tv, (struct timezone *) nullptr);
    jlong when = tv.tv_sec * 1000LL + tv.tv_usec / 1000;
    return when;
}
```

- 对于**java.lang.Thread.currentThread**()方法，其被声明在`libcore/ojluni/src/main/java/java/lang/Thread.java`中：
```bash
    /**
     * Returns a reference to the currently executing thread object.
     *
     * @return  the currently executing thread.
     */
    @FastNative
    public static native Thread currentThread();
```
- 其被定义在`art/runtime/native/java_lang_Thread.cc`中，也是一个native方法：
```bash
static jobject Thread_currentThread(JNIEnv* env, jclass) {
  ScopedFastNativeObjectAccess soa(env);
  return soa.AddLocalReference<jobject>(soa.Self()->GetPeer());
}
```
- 通过比较以上两个函数的声明，可以看出@CriticalNative和@FastNative方法在返回值上的区别，更详细的说明在[CriticalNative与FastNative](https://source.android.google.cn/docs/core/runtime/improvements?hl=zh-cn)。