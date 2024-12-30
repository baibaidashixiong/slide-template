jvmti是类似llvm插桩的一个工具。
-  `move-wide/from16 vAA, vBBBB`：
	- `move`”基础运算码，表示基础运算（移动寄存器的值）。
    - `wide`”为名称后缀，表示指令对宽（64 位）数据进行运算。
    - `from16`为运算码后缀，表示具有 16 位寄存器引用源的变体。
    - `vAA`为目标寄存器（隐含在运算中；并且，规定目标参数始终在前），取值范围为 `v0` - `v255`。（8位）
    - `vBBBB`是源寄存器，取值范围为 `v0` - `v65535`。(16位)
- dexdump自动转换字节序为顺序。
- **smali和dex文件的区别**：Smali是一种human-friendly的中间代码表示,方便编码和分析;而Dex是一种机器友好的紧凑分发格式,被Android运行时直接加载执行。dex转换为smail方法：生成在out目录下
```bash
./java -jar baksmali-2.5.2.jar d TestClass.dex
```
- dex中加入了**LEB128**（“**L**ittle-**E**ndian **B**ase **128**”）用于给32位数字进行编码，由N(N=1~5)个字节组成。每个字节的第7位数据用于表示这个LEB128数据是否结束，第7位取0表示此字节为最后一个字节，也叫结尾字节；第7位取值为1表示此数据后还有数据。这样对于大多数较小的数，原本需要4个字节现在只需要1个字节。
- java中name的`<init>`为java默认自动生成的构造函数/方法。
- dex文件中**field的作用**：字段（field）指的是类中的**变量或数据成员**，其可以被sget-object来获取到寄存器中。**偏移量（Offset）**：字段在DEX文件中的位置偏移量，用于在运行时准确地定位字段的值。`field@0001` 表示这个操作所引用的字段在 Dex 文件中的偏移量为 `0001`。
- **dex指令的指令格式**：以`new-instance vAA, type@BBBB`为例，其指令码为22,**指令格式**为k21c，其中前两个是十进制数，最后一个是字母，第一个十进制数表示格式中16位代码单元的数量，第二个数字表示格式所含寄存器的数量**上限**（因为某些格式使用的寄存器数量是可变的）。最后一个字母以半助记符的形式表示该格式编码的任何其他数据类型。例如，“`21t`”格式的长度为 2，包含一个寄存器引用，另外还包含一个分支目标。如dex指令`sget-object v1, Ljava/lang/System;.out:Ljava/io/PrintStream; // field@0001`，其十六进制为`0162 0001`，而其指令格式`k21c`如下，
```bash
sstaticop vAA, field@BBBB
# sget-object指令格式
AA|op BBBB
op vAA, field@BBBB
01|62 0001  # AA为1表示v1寄存器，62表示操作码，0001表示字段池索引为1
```
- java中的**abstract关键字**用于定义抽象类和方法，**抽象类是不能被实例化的类，它通常只有方法签名，用作其他类的基类**，包含一些通用的属性和方法的定义，**子类必须实现这些抽象方法**来提供具体的实现。
- java中**除static, private, final，构造函数之外的所有函数默认都是virtual function**。而剩余的就是direct method，direct method在编译时就确定了调用的具体方法和地址。
- dex程序示例：
```c
# 源程序 TestClass.java
public class TestClass {                                                                         
       private int m = 0;                                                                       
                                                                                                
       public int inc() {                                                                       
               return m + 1;                                                                    
       }                                                                                        
       public static void main(String[] args) {                                                 
               TestClass a = new TestClass();                                                   
               System.out.println(a.inc()+"Hello, world!");                                     
       }                                                                                                                                                                                    
}

# 编译方法
javac TestClass.java
dx --dex --output=TestClass.dex TestClass.class
dexdump -d TestClass.dex

# 输出
Processing 'TestClass.dex'...
Opened 'TestClass.dex', DEX version '035'
Class #0            -  
 Class descriptor  : 'LTestClass;'  // L为类描述符
 Access flags      : 0x0001 (PUBLIC) // 访问标志
 Superclass        : 'Ljava/lang/Object;'//超类
 Interfaces        -  
 Static fields     -  
 Instance fields   -  // 示例字段
   #0              : (in LTestClass;)
     name          : 'm'
     type          : 'I'
     access        : 0x0002 (PRIVATE)
 Direct methods    -  // 构造方法
   #0              : (in LTestClass;)  
     name          : '<init>'  
     type          : '()V'  
     access        : 0x10001 (PUBLIC CONSTRUCTOR)  
     code          -  
     registers     : 2  
     ins           : 1  
     outs          : 1  
     insns size    : 7 16-bit code units  
0001bc:                                        |[0001bc] TestClass.<init>:()V  
0001cc: 7010 0400 0100                         |0000: invoke-direct {v1}, Ljava/lang/Object;.<init>:()V // method@0004  
0001d2: 1200                                   |0003: const/4 v0, #int 0 // #0  
0001d4: 5910 0000                              |0004: iput v0, v1, LTestClass;.m:I // field@0000  
0001d8: 0e00                                   |0006: return-void  
     catches       : (none)  
     positions     :  
       0x0000 line=1  
       0x0003 line=2  
     locals        :  
       0x0000 - 0x0007 reg=1 this LTestClass;  
  
   #1              : (in LTestClass;)  
     name          : 'main'  
     type          : '([Ljava/lang/String;)V'  
     access        : 0x0009 (PUBLIC STATIC)  
     code          -  
     registers     : 4  
     ins           : 1  
     outs          : 2  
     insns size    : 34 16-bit code units  
0001dc:                                        |[0001dc] TestClass.main:([Ljava/lang/String;)V  
0001ec: 2200 0100                              |0000: new-instance v0, LTestClass; // type@0001  
0001f0: 7010 0000 0000                         |0002: invoke-direct {v0}, LTestClass;.<init>:()V // method@0000  
0001f6: 6201 0100                              |0005: sget-object v1, Ljava/lang/System;.out:Ljava/io/PrintStream; // field@0001
// Ljava/lang/System;表示引用java/lang/System类，.out:Ljava/io/PrintStream;表示引用java/lang/System类中名为out的类型为java/io/PrintStream的静态字段
0001fa: 2202 0500                              |0007: new-instance v2, Ljava/lang/StringBuilder; // type@0005  
0001fe: 7010 0500 0200                         |0009: invoke-direct {v2}, Ljava/lang/StringBuilder;.<init>:()V // method@0005  
000204: 6e10 0100 0000                         |000c: invoke-virtual {v0}, LTestClass;.inc:()I // method@0001  
00020a: 0a00                                   |000f: move-result v0  
00020c: 6e20 0600 0200                         |0010: invoke-virtual {v2, v0}, Ljava/lang/StringBuilder;.append:(I)Ljava/lang/StringBuil  
der; // method@0006  
000212: 0c00                                   |0013: move-result-object v0  
000214: 1a02 0100                              |0014: const-string v2, "Hello, world!" // string@0001  
000218: 6e20 0700 2000                         |0016: invoke-virtual {v0, v2}, Ljava/lang/StringBuilder;.append:(Ljava/lang/String;)Ljav  
a/lang/StringBuilder; // method@0007  
00021e: 0c00                                   |0019: move-result-object v0  
000220: 6e10 0800 0000                         |001a: invoke-virtual {v0}, Ljava/lang/StringBuilder;.toString:()Ljava/lang/String; // me  
thod@0008  
000226: 0c00                                   |001d: move-result-object v0  
000228: 6e20 0300 0100                         |001e: invoke-virtual {v1, v0}, Ljava/io/PrintStream;.println:(Ljava/lang/String;)V // me  
thod@0003  
00022e: 0e00                                   |0021: return-void  
     catches       : (none)  
     positions     :  
       0x0000 line=8  
       0x0005 line=9  
       0x0021 line=10  
     locals        :  
       0x0000 - 0x0022 reg=3 (null) [Ljava/lang/String;  
  
 Virtual methods   -  
   #0              : (in LTestClass;)  
     name          : 'inc'  
     type          : '()I'  
     access        : 0x0001 (PUBLIC)  
     code          -  
     registers     : 2  
     ins           : 1  
     outs          : 0  
     insns size    : 5 16-bit code units  
000230:                                        |[000230] TestClass.inc:()I  
000240: 5210 0000                              |0000: iget v0, v1, LTestClass;.m:I // field@0000  
000244: d800 0001                              |0002: add-int/lit8 v0, v0, #int 1 // #01  
000248: 0f00                                   |0004: return v0  
     catches       : (none)  
     positions     :  
       0x0000 line=5  
     locals        :  
       0x0000 - 0x0005 reg=1 this LTestClass;  
  
 source_file_idx   : 12 (TestClass.java)
```