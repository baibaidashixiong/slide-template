## 术语
- **stw（Stop-the-World）**：停下线程进行垃圾清除。
- **Mark Sweep Compact (MSC)**：分为3步
	1. MARK - traverse live object graph to mark reachable objects.
	2. SWEEP - scans memory to find unmarked memory.
	3. COMPACT - relocating marked objects to defragment free memory.
- card table：
- **MS**：Non concurrent mark-sweep.
- **CMS**：Concurrent Mark Sweep.
- **SS**：Semi-space / mark-sweep hybrid, enables compaction.
- **CC**：A (mostly) concurrent copying collector.
- **TLAB**：Thread Local Allocation Buffer.
- **mutator**：除垃圾收集器之外的、实际执行程序逻辑的用户程序的工作线程。
	- mutator线程的**基本操作**：
```bash
Object obj1 = new Object(); // 分配新对象
obj1.field = null; // 修改引用关系
Object obj2 = obj1; // 创建新的引用
obj1 = null; // 删除引用
```
	- Mutator与GC的关系：
		- 并发模式：
```bash
Mutator线程            GC线程
运行用户代码  <---->  并发执行垃圾回收
修改对象引用  <---->  追踪对象图
分配新对象    <---->  回收死对象
```
		- 停顿模式（Stop-The-World）：
```bash
1. Mutator运行
2. GC启动 -> Mutator暂停
3. GC完成 -> Mutator恢复
```
- allocation：当mutator需要新对象的时候，就会像分配器(allocator)申请一个大小合适的空间。
- 根（root）：根是指向对象的"起点"部分。



- 分代式GC里，年老代常用mark-sweep；或者是mark-sweep/mark-compact的混合方式，一般情况下用mark-sweep，统计估算碎片量达到一定程度时用mark-compact。这是因为传统上大家认为年老代的对象可能会长时间存活且存活率高，或者是比较大，这样拷贝起来不划算，还不如采用就地收集的方式。

|       | mark-sweep | mark-compact | copying             |
| ----- | ---------- | ------------ | ------------------- |
| 速度    | 中等         | 最慢           | 最快                  |
| 空间开销  | 少（但会堆积碎片）  | 少（不堆积碎片）     | 通常需要活对象的2倍大小（不堆积碎片） |
| 移动对象？ | 否          | 是            | 是                   |

## 算法
- GC标记-清除算法（Mark-Sweep）：
	- 使用多个空闲链表可以在allocate时候减少对整个空闲链表的遍历，提高效率。
	- 使用位图标记法可以增加对cow的支持以及使清除操作更加高效。
	- 使用延迟清除法可以将sweep阶段推迟到new obj，也就是allocate时期，节省一次遍历链表的时间。
```bash
mark_sweep() {
  mark_phase()
  sweep_phase()
}
```
- 引用计数法：
- 复制算法（Copying GC）：将存活对象从FROM空间整体压缩移动到TO空间。缺点是需要两倍的空间。
- GC标记-压缩算法（Mark Compact GC）：原地将存活对象压缩到堆的起始处。lisp2算法先将对象指针排序指向堆的的起始处；再更新root的对象指针，再一个一个慢慢移动过去。缺点是需要多次遍历，效率较低。
- **分代垃圾回收**（需要与之前几种算法结合）：将存活的时间长的垃圾放入老年代，存活时间短的为新生代。老年代垃圾只有满了以后才会进行GC，新生代GC就如上述方法。
	- Unger的分代垃圾回收：分为生成空间、2个大小相等的幸存空间以及老年代空间，分别用`$new_start/ $survivor1_start/ $survivor2_start/ $old_start`这4个变量引用它们的开头。
	- 写入屏障：为了将老年代对象记录到**记录集($rs)** 中，在mutator更新对象间的指针操作中，写入屏障不可或缺。**记录集用于高效的寻找从老年代到新生代对象的引用，老年代中每个发出引用的对象会占用记录集一个字。**
```bash
write_barrier(obj, field, new_obj) {
  if(obj >= $obj_start && new_obj < $old_start && obj.remembered == FALSE)
    # 1. 发出引用的对象是不是老年代
    # 2. 引用的目标对象是否是新生代
    # 3. 发出引用的对象是否在记录集中
    $rs[$rs_index] = obj
    $rs_index++
    obj.remembered = TRUE

  *field = new_obj
  # 每个field要么是一个指针要么是一个数据，所以最多只有一个引用
}
# 解释：obj是发出引用的对象，obj内存在要更新的指针，而field指的就是obj内的域，new_obj是在指针更新后成为引用目标的对象。
```
- **卡片标记**：
	- 背景：年轻代GC时，需要找出所有老年代对象对年轻代的跨代引用。，如果每次都扫描整个老年代会很低效。
	- 基本原理：将整个堆空间划分成固定大小的区域，称为"卡页"(Card Page)，每个卡页对应卡表中的一个字节（byte），这个字节标识对应的卡页是否可能存在跨代引用。如果存在，则扫描整个卡页找出其所有的跨代引用。即使每个卡页中都有跨代引用，最坏的情况也只是扫描整个老年代。
```bash
// 当老年代对象引用年轻代对象时
Object oldObj = ...; // 老年代对象
oldObj.field = youngObj; // 引用年轻代对象

// JVM会将对应的卡表项标记为脏
markCard(oldObj);
```


## 问题
- 为什么没有读屏障（CMS）的运行速度要显著低于有读屏障（CC）的运行速度？没有读屏障就没有stw，可以并发执行。
- CMS和MS的异同？为什么MS比CMS的运行速度还要更快？