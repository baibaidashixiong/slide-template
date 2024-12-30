只分析重要类和重要成员变量
#### CFG构造相关代码
- **HBasicBlock**：代表基本块。成员变量：
```c++
HGraph* graph_; // 指向的CFG对象
ArenaVector<HBasicBlock*> predecessors_; // 前驱基本块
ArenaVector<HBasicBlock*> successors_;   // 后继基本块
HInstructionList instructions_; // 保存基本块中的所有HInstruction
HInstructionList phis_; // 存储HPhi对象
HLoopInformation* loop_information_;
HBasicBlock* dominator_;
ArenaVector<HBasicBlock*> dominated_blocks_;
uint32_t block_id_; // 基本块编号
// The dex program counter of the first instruction of this block.
const uint32_t dex_pc_; // 基本块起始的dex字节码（why？）（重要）
size_t lifetime_start_;
size_t lifetime_end_;
```
- **HGraph**：代表CFG。`ArenaVector<HBasicBlock*> blocks_`表示该有向无环图所属的所有基本块。`HBasicBlock* entry_block_/exit_block_;`为入口/出口基本块，为特意构造，与代码无关，其`dex_pc_`取值为-1。
- **HBasicBlockBuilder**：构造CFG的辅助类。`const DexFile* const dex_file_`指向输入的dex文件。`CodeItemDataAccessor code_item_accessor_`表示用于访问代码项的不同字段和信息。而代码项用`struct CodeItem : public dex::CodeItem`类表示。`ScopedArenaVector<HBasicBlock*> branch_targets_/throwing_blocks_`为ART自定义的数组容器`ArenaVector`，其功能类似STL的vector，但是为虚拟机做了特殊优化。（TODO）

#### HInstruction
IR之间分为线性关系和非线性关系。
- **HInstructionList**：保存基本块中的所有HInstruction。
```c++
public:
  void AddInstruction(HInstruction* instruction);
  void RemoveInstruction(HInstruction* instruction);
...
  void AddAfter(HInstruction* cursor, const HInstructionList& instruction_list);
  void AddBefore(HInstruction* cursor, const HInstructionList& instruction_list);
  void Add(const HInstructionList& instruction_list);
private:
  HInstruction* first_instruction_; // IR列表头元素
  HInstruction* last_instruction_;  // IR列表尾元素
```
- **HInstruction**：
```c++
private:
  HInstruction* previous_; // 当前IR的前驱指令
  HInstruction* next_;     // 当前IR的后继指令
  HBasicBlock* block_;
  const uint32_t dex_pc_;
  // An instruction gets an id when it is added to the graph.
  // It reflects creation order. A negative id means the instruction
  // has not been added to the graph.
  int id_; // 针对整个CFG而言，默认从0开始
```
除了线性关系外，一些IR类之间还存在使用和被使用的关系。
1. 使用关系：一些IR需要使用别的IR，如HPhi需要保存代表变量不同版本的HInstruction对象，所以HPhi使用了这些对象，一个IR对象可以用一个或多个其它对象。
2. 被使用关系：被使用的IR需要自己被谁使用了，一个IR对象可以被一个或多个其它IR对象使用。
