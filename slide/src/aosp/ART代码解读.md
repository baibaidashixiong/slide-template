### 解释执行过程
#### 纯C++的解释执行
- 解释执行的大switch生成在`interpreter_switch_impl-inl.h`中，通过`gcc -E`编译生成。最重要的处理字节码的就是`OP_MOVE_FROM16<transaction_active>`，其中创建InstructionHandler类的对象handler，并使用`handler.OPCODE_NAME()`来调用InstructionHandler类中的函数来处理字节码。从而生成字节码处理逻辑`OP_MOVE_FROM16<transaction_active>`，该处理过程是架构无关的。
```c++
// 预处理生成
#define DEX_INSTRUCTION_LIST(V) \
  V(0x00, NOP, "nop", k10x, kIndexNone, kContinue, 0, kVerifyNothing) \
  V(0x01, MOVE, "move", k12x, kIndexNone, kContinue, 0, kVerifyRegA | kVerifyRegB) \
  V(0x02, MOVE_FROM16, "move/from16", k22x, kIndexNone, kContinue, 0, kVerifyRegA | kVerifyRegB) \
  V(0x03, MOVE_16, "move/16", k32x, kIndexNone, kContinue, 0, kVerifyRegA | kVerifyRegB) \
  V(0x04, MOVE_WIDE, "move-wide", k12x, kIndexNone, kContinue, 0, kVerifyRegAWide | kVerifyRegBWide) \

#define OPCODE_CASE(OPCODE, OPCODE_NAME, NAME, FORMAT, i, a, e, v)                                \
        case OPCODE: {                                                                            \
          next = inst->RelativeAt(Instruction::SizeInCodeUnits(Instruction::FORMAT));             \
          success = OP_##OPCODE_NAME<transaction_active>(                                         \
              ctx, instrumentation, self, shadow_frame, dex_pc, inst, inst_data, next, exit);     \
          if (success && LIKELY(!interpret_one_instruction)) {                                    \
            continue;                                                                             \
          }                                                                                       \
          break;                                                                                  \
        }
  DEX_INSTRUCTION_LIST(OPCODE_CASE)
#undef OPCODE_CASE

// 预处理生成结果
case 0x00: { next = inst->RelativeAt(Instruction::SizeInCodeUnits(Instruction::k10x)); success = OP_NOP<transaction_active>( ctx, instrumentation, self, shadow_frame, dex_pc, inst, inst_data, next, exit); if (success && LIKELY(!interpret_one_instruction)) { continue; } break; } 
case 0x01: { next = inst->RelativeAt(Instruction::SizeInCodeUnits(Instruction::k12x)); success = OP_MOVE<transaction_active>( ctx, instrumentation, self, shadow_frame, dex_pc, inst, inst_data, next, exit); if (success && LIKELY(!interpret_one_instruction)) { continue; } break; } 
case 0x02: { next = inst->RelativeAt(Instruction::SizeInCodeUnits(Instruction::k22x)); success = OP_MOVE_FROM16<transaction_active>( ctx, instrumentation, self, shadow_frame, dex_pc, inst, inst_data, next, exit); if (success && LIKELY(!interpret_one_instruction)) { continue; } break; } 
case 0x03: { next = inst->RelativeAt(Instruction::SizeInCodeUnits(Instruction::k32x)); success = OP_MOVE_16<transaction_active>( ctx, instrumentation, self, shadow_frame, dex_pc, inst, inst_data, next, exit); if (success && LIKELY(!interpret_one_instruction)) { continue; } break; } case 0x04: { next = inst->RelativeAt(Instruction::SizeInCodeUnits(Instruction::k12x)); success = OP_MOVE_WIDE<transaction_active>( ctx, instrumentation, self, shadow_frame, dex_pc, inst, inst_data, next, exit); if (success && LIKELY(!interpret_one_instruction)) { continue; } break; }

// 字节码处理代码生成
#define OPCODE_CASE(OPCODE, OPCODE_NAME, NAME, FORMAT, i, a, e, v)                                \
template<bool transaction_active>                                                                 \
ASAN_NO_INLINE NO_STACK_PROTECTOR static bool OP_##OPCODE_NAME(                                   \
    SwitchImplContext* ctx,                                                                       \
    const instrumentation::Instrumentation* instrumentation,                                      \
    Thread* self,                                                                                 \
    ShadowFrame& shadow_frame,                                                                    \
    uint16_t dex_pc,                                                                              \
    const Instruction* inst,                                                                      \
    uint16_t inst_data,                                                                           \
    const Instruction*& next,                                                                     \
    bool& exit) REQUIRES_SHARED(Locks::mutator_lock_) {                                           \
  InstructionHandler<transaction_active, Instruction::FORMAT> handler(                            \
      ctx, instrumentation, self, shadow_frame, dex_pc, inst, inst_data, next, exit);             \
  return LIKELY(handler.OPCODE_NAME());                                                           \
}
DEX_INSTRUCTION_LIST(OPCODE_CASE)
#undef OPCODE_CASE

// 字节码处理代码生成结果
template<bool transaction_active> ASAN_NO_INLINE NO_STACK_PROTECTOR static bool OP_NOP( SwitchImplContext* ctx, const instrumentation::Instrumentation* instrumentation, Thread* self, ShadowFrame& shadow_frame, uint16_t dex_pc, const Instruction* inst, uint16_t inst_data, const Instruction*& next, bool& exit) REQUIRES_SHARED(Locks::mutator_lock_) { InstructionHandler<transaction_active, Instruction::k10x> handler( ctx, instrumentation, self, shadow_frame, dex_pc, inst, inst_data, next, exit); return LIKELY(handler.NOP()); } 
template<bool transaction_active> ASAN_NO_INLINE NO_STACK_PROTECTOR static bool OP_MOVE( SwitchImplContext* ctx, const instrumentation::Instrumentation* instrumentation, Thread* self, ShadowFrame& shadow_frame, uint16_t dex_pc, const Instruction* inst, uint16_t inst_data, const Instruction*& next, bool& exit) REQUIRES_SHARED(Locks::mutator_lock_) { InstructionHandler<transaction_active, Instruction::k12x> handler( ctx, instrumentation, self, shadow_frame, dex_pc, inst, inst_data, next, exit); return LIKELY(handler.MOVE()); } 
template<bool transaction_active> ASAN_NO_INLINE NO_STACK_PROTECTOR static bool OP_MOVE_FROM16( SwitchImplContext* ctx, const instrumentation::Instrumentation* instrumentation, Thread* self, ShadowFrame& shadow_frame, uint16_t dex_pc, const Instruction* inst, uint16_t inst_data, const Instruction*& next, bool& exit) REQUIRES_SHARED(Locks::mutator_lock_) { InstructionHandler<transaction_active, Instruction::k22x> handler( ctx, instrumentation, self, shadow_frame, dex_pc, inst, inst_data, next, exit); return LIKELY(handler.MOVE_FROM16()); } 
template<bool transaction_active> ASAN_NO_INLINE NO_STACK_PROTECTOR static bool OP_MOVE_16( SwitchImplContext* ctx, const instrumentation::Instrumentation* instrumentation, Thread* self, ShadowFrame& shadow_frame, uint16_t dex_pc, const Instruction* inst, uint16_t inst_data, const Instruction*& next, bool& exit) REQUIRES_SHARED(Locks::mutator_lock_) { InstructionHandler<transaction_active, Instruction::k32x> handler( ctx, instrumentation, self, shadow_frame, dex_pc, inst, inst_data, next, exit); return LIKELY(handler.MOVE_16()); } 
template<bool transaction_active> ASAN_NO_INLINE NO_STACK_PROTECTOR static bool OP_MOVE_WIDE( SwitchImplContext* ctx, const instrumentation::Instrumentation* instrumentation, Thread* self, ShadowFrame& shadow_frame, uint16_t dex_pc, const Instruction* inst, uint16_t inst_data, const Instruction*& next, bool& exit) REQUIRES_SHARED(Locks::mutator_lock_) { InstructionHandler<transaction_active, Instruction::k12x> handler( ctx, instrumentation, self, shadow_frame, dex_pc, inst, inst_data, next, exit); return LIKELY(handler.MOVE_WIDE()); }
```
#### 汇编解释执行
##### IsNterpSupported
- 在`RuntimeImageHelper::CopyMethodArrays`，`ArtMethod::CopyFrom`中使用该函数来判断是否支持nterp来为最终生成优化的app image（相当于可执行文件的加速缓存）作准备。
- 在`CanRuntimeUseNterp`中通过该函数判断runtime是否只支持interpreter，后续在解释执行进行optimizition的时候都会通过此函数进行判断是否能进行优化。（注：区分`CanMethodUseNterp`）


### ART Assembler将汇编转换为机器码过程——直接跳转
基于art commmit f15e067185a832a3d0b3f574c28faf808f6e2dff。
#### 指令发射
- Emit有overwriting和append两种指令发射方式，使用一个`overwriting_`标志来判断来区分这两种方式。
	- 对于分支指令，`overwriting_`标志位为真，使用Store将指令存储到**buffer相对`overwrite_location_`偏移量**的位置，**即label表**的位置（后续会详细解释）。
	- 对于普通指令，使用Emit将指令存储到buffer顺序指令流末尾的位置。
	- 注：
		- `AssemblerBuffer::buffer_`：指令缓冲区。
		- `AssemblerBuffer::contents_`：指令缓冲区内存起始位置。
		- `AssemblerBuffer::cursor_`：指令缓冲区顺序指令流末尾位置。
		- `Riscv64Assembler::overwriting_`：true代表需要对指令缓冲区`buffer_`中的分支指令的占位信息进行重写。
		- `Riscv64Assembler::overwrite_location_`：`buffer_`中需要重写的分支指令地址，根据Branch中的`location_`得出。
		- `Riscv64Assembler::Branch::location_`：每条分支指令在指令缓冲区中的起始位置。
		- `Riscv64Assembler::Branch::target_`：分支跳转目标label的地址。
		- `ArenaVector<Branch> branches_;`：分支指令容器队列。
		- `Label::position_`：
			- **标签未绑定时(position_ > 0)**：分支指令label所处在分支指令容器队列`branches_`中的位置+8（用于存储一些额外信息）。与`Emit32(label->position_)`结合形成某个Label的跳转链表。
			- **标签绑定时(position_ < 0)**：为该label与其在指令序列中上一条跳转指令的距离的负数-8。（ `-(addr(label) - addr(prev_branch) + 8)`）
		- `IsBound`和`IsLinked`的区别：`IsLinked`表示指向**同一label的跳转指令处于同一条链表**，**此时target地址未确定**；`IsBound`表示这些同一链表上的跳转指令的**目标地址已被绑定**，此时label的`position_`为`-(addr(label) - addr(prev_branch) + 8)`。
```c++
void Emit32(uint32_t value) { Emit(value); }

// Emit data (e.g. encoded instruction or immediate) to the instruction stream.
template <typename T>
void Emit(T value) {
    static_assert(std::is_same_v<T, uint32_t> || std::is_same_v<T, uint16_t>,
                "Only Integer types are allowed");// 模板内变量类型只许使用16或32位整数
	if (overwriting_) {
      // Branches to labels are emitted into their placeholders here.
      buffer_.Store<T>(overwrite_location_, value);
      overwrite_location_ += sizeof(T);
    } else {
      // Other instructions are simply appended at the end here.
      AssemblerBuffer::EnsureCapacity ensured(&buffer_);
      buffer_.Emit<T>(value);
	}
}

template<typename T> void Store(size_t position, T value) {
    *reinterpret_cast<T*>(contents_ + position) = value;
}

template<typename T> void Emit(T value) {
    *reinterpret_cast<T*>(cursor_) = value;
    cursor_ += sizeof(T);
}
```

#### 非跳转指令
- 以Add指令为例生成机器码：
```c++
// assembler_riscv64.cc
void Riscv64Assembler::Add(XRegister rd, XRegister rs1, XRegister rs2) {
  EmitR(0x0, rs2, rs1, 0x0, rd, 0x33);
}

// assembler_riscv64.h
  // R-type instruction:
  //
  //    31         25 24     20 19     15 14 12 11      7 6           0
  //   -----------------------------------------------------------------
  //   [ . . . . . . | . . . . | . . . . | . . | . . . . | . . . . . . ]
  //   [   funct7        rs2       rs1   funct3     rd        opcode   ]
  //   -----------------------------------------------------------------
  template <typename Reg1, typename Reg2, typename Reg3>
  void EmitR(uint32_t funct7, Reg1 rs2, Reg2 rs1, uint32_t funct3, Reg3 rd, uint32_t opcode) {
	DCHECK(...);
    uint32_t encoding = funct7 << 25 | static_cast<uint32_t>(rs2) << 20 |
                        static_cast<uint32_t>(rs1) << 15 | funct3 << 12 |
                        static_cast<uint32_t>(rd) << 7 | opcode;
    Emit32(encoding);
  }
```

#### 跳转指令
- 示例测试汇编代码
```assembler
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
```
##### 一、对跳转指令进行占位
不同于Add这种简单的指令，跳转指令`beq a0 a1 1f`，间接跳转`call reg`等由于不知道`1f`和`reg`的值没法直接生成机器码。这种情况下就需要**先做标记然后占位**。具体做法是先把变量目的地址target设置为极大值`kUnresolved = 0xffffffff`，标记指**记录下此时beq相对buffer_的地址**，并且**占据beq所需的长度**。主要为以下三个步骤：
1. **设置跳转目标地址target**。`uint32_t target = label->IsBound() ? GetLabelLocation(label) : Branch::kUnresolved;`如果label已被绑定则获取该label的地址绑定为跳转目标地址target（**TODO**：如何获取？），否则将target设置为一个极大值。
2. **将beq1生成的对象放入分支指令容器队列结尾**。`branches_.emplace_back(buffer_.Size(), target, condition, lhs, rhs, is_bare);`。其中`buffer_.Size()`表示当前buffer中指令流的大小；`target`表示Label地址。`is_bare`表示是否为**不可变长分支**（与**RISCV的压缩指令有关**）。（emplace_back可以直接传入参数自动调用类的构造函数，push_back只能传入实例）
3. **对跳转指令进行占位，弄清该逻辑的*关键是只有一个buffer_，一个branches_中有多个Branch，每个Branch对应一个Label，一个Label对应多个Branch***。`FinalizeLabeledBranch(label);`。其中`length`为最后一条插入的（分支）指令的长度，`1 length = 4 bytes`，对于不同的分支指令有不同的`length`，以`kCondBranch`为例，其为短跳转有4个字节；`kLongCondBranch`为长跳转占12个字节。该函数主要作用如下：
	1. 判断label是否已被绑定。
		- **未被绑定**：说明是一个前向（forward）分支，此时目标地址未知，此时执行如下操作：
			1. `Emit32(label->position_)`在指令流中为跳转指令进行占位，其中**对应每个新label的分支指令首次都用0进行占位**，表示该label所对应的跳转指令链表的终止符。
			2. 获取分支指令容器队列大小-1作为**branch_id**，其表示**当前分支指令在branches_向量中的索引**。通过`LinkTo(int branch_id):: position_ = branch_id + sizeof(void*)`来将此条分支的索引存储到`label::position_`中，**下一次处理目标label相同的跳转指令时就用该`position_`进行占位。这样每条指向相同label的跳转指令就可以由占位符形成一个链表**。当标签的实际地址确定后，编译器就可以遍历这个链表，并且修改所有指向该标签的前向分支指令，让它们的目标地址指向正确的位置，从而完成代码重定位（PIC）。而`position_`中要加`sizeof(void*)`一个指针大小的偏移是为了用于在起始位置存储一些额外的信息（比如链接信息）。
	2. 如为长跳转或者已绑定的分支跳转指令则使用Nop(`Addi(Zero, Zero, 0)`)使用全0为其占位（为分支指令占位的目的？**为什么label绑定了地址还需要占位**？所有跳转指令都放入同一张表中）。
![[aosp/pic/跳转指令标签链表.png]]
```c++
// 条件跳转
void Riscv64Assembler::Bcond(
    Riscv64Label* label, bool is_bare, BranchCondition condition, XRegister lhs, XRegister rhs) {
  // If lhs = rhs, this can be a NOP.
  if (Branch::IsNop(condition, lhs, rhs)) {
    return;
  }
  if (Branch::IsUncond(condition, lhs, rhs)) {
    Buncond(label, Zero, is_bare);
    return;
  }

  uint32_t target = label->IsBound() ? GetLabelLocation(label) : Branch::kUnresolved;
  branches_.emplace_back(buffer_.Size(), target, condition, lhs, rhs, is_bare);// 在分支指令队列中初始化添加一条分支指令
  FinalizeLabeledBranch(label);
}

//跳转指令占位
void Riscv64Assembler::FinalizeLabeledBranch(Riscv64Label* label) {
  DCHECK_ALIGNED(branches_.back().GetLength(), sizeof(uint32_t));
  uint32_t length = branches_.back().GetLength() / sizeof(uint32_t);// 上一条插入的分支指令长度
  if (!label->IsBound()) {
    Emit32(label->position_);
    length--;
    uint32_t branch_id = branches_.size() - 1; // 分支指令数量
    label->LinkTo(branch_id);
  }
  for (; length != 0u; --length) {// 使用Nop()为分支跳转指令占位
    Nop();
  }
}

void LinkTo(int position) { position_ = position + sizeof(void*); }
```

##### 二、计算target
在code_generator_riscv64.cc中**每当指令序列遇到label时候便会调用Bind函数对之前的分支指令进行跳转目标地址绑定**。以Bind(label1)为例。**注：所有label都是相对地址**（PIC）
- 首先获取当前指令序列相对地址，然后`while (label->IsLinked())`：当`pos > 0`时一直循环，向前(forward)遍历获取所有跳转目标为该label，并位于该label之前的分支指令。在这过程中绑定分支指令跳转地址的关键函数就是`branch->Resolve(bound_pc)`，`void Riscv64Assembler::Branch::Resolve(uint32_t target) { target_ = target; }`将分支跳转的目标地址绑定为label的地址。
- 为该链表上所有该label的分支指令绑定目标地址`target_`后即可**将该label的分支设置为`Bounded`**，因为其地址已被知晓，当后续跳转指令目标为该label时可直接根据其地址绑定`target_`。此时pos被设置为负数，根据pos正负即可知道该分支指令应该向前（forward）还是向后（backward）跳转，pos>0为向前跳转，label地址未知；pos<0为向后跳转，label地址已知。
- **问**：能否在`Riscv64Label`中设置为一个类似`branches_`的队列来保存label的位置？可以但是可能消耗空间太多，而branch的location还可用于指令重写，此处使用队列保存label地址代价高收益小。
```c++
void Riscv64Assembler::Bind(Riscv64Label* label) {
  uint32_t bound_pc = buffer_.Size();// 获取指令序列边界相对pc（即指令序列大小），标签相对于buffer_起始位置的偏移量

  while (label->IsLinked()) { // 遍历回到指向该label的第一条分支指令，对应图中的beq1
    uint32_t branch_id = label->Position(); // 获取此时label位置所对应的分支指令位置
    Branch* branch = GetBranch(branch_id);
    
    branch->Resolve(bound_pc); // 关键步骤，绑定label地址为此条Branch的跳转地址target_ = bound_pc

    uint32_t branch_location = branch->GetLocation(); // 获取此条分支指令在指令序列buffer_中的相对位置
    uint32_t prev = buffer_.Load<uint32_t>(branch_location); // 读取当前指令地址中的内容，即第一步中占位的内容
    label->position_ = prev; // 链表回溯
  }
  // 循环结束后所有目标为该label的分支指令目标地址都被确定并写入，此时label.position_的作用已经发挥

  // Now make the label object contain its own location (relative to the end of the preceding
  // branch, if any; it will be used by the branches referring to and following this label).
  uint32_t prev_branch_id = Riscv64Label::kNoPrevBranchId;
  if (!branches_.empty()) {
    prev_branch_id = branches_.size() - 1u;
    const Branch* prev_branch = GetBranch(prev_branch_id);
    bound_pc -= prev_branch->GetEndLocation(); // 此label与处于其前一个的分支指令间的距离
  }
  label->prev_branch_id_ = prev_branch_id;
  label->BindTo(bound_pc); // 
}

uint32_t Riscv64Assembler::Branch::GetLocation() const { return location_; } // 返回Branch在指令序列中的位置
void Riscv64Assembler::Branch::Resolve(uint32_t target) { target_ = target; } // 绑定跳转指令的target_
void BindTo(int position) { position_ = -position - sizeof(void*); } // position为负数表示此时label地址已被绑定
uint32_t Riscv64Assembler::Branch::GetEndLocation() const { return GetLocation() + GetLength(); } // 获取当前指令末尾地址

uint32_t Riscv64Assembler::GetLabelLocation(const Riscv64Label* label) const {
  uint32_t target = label->Position();
  if (label->prev_branch_id_ != Riscv64Label::kNoPrevBranchId) {
    // Get label location based on the branch preceding it.
    const Branch* prev_branch = GetBranch(label->prev_branch_id_);
    target += prev_branch->GetEndLocation();
  }
  return target;
}
```

##### 三、计算向前跳转的target
- 当跳转时如`position_ < 0`则说明label地址已被绑定，此时直接获取label地址（即与上一个Branch之间的距离），再加上上一个Branch的地址即为此label的地址，便可不用`kUnresolved`占位而赋值给`target_`。而此时`FinalizeLabeledBranch(label)`变可以直接对该跳转指令用`Nop()`占位。
```c++
uint32_t target = label->IsBound() ? GetLabelLocation(label) : Branch::kUnresolved;
int Position() const { return IsBound() ? -position_ - sizeof(void*) : position_ - sizeof(void*); } // 加上偏移量方便添加诸如链接信息的内容
```

##### 四、生成机器码
以上操作都只是对分支指令进行占位，但是还为真正生成机器码，真正生成分支指令的机器码通过`FinalizeCode() -> EmitBranches()`来完成。问：为什么需要如此多的跳转类型？
```c++
void Riscv64Assembler::EmitBranches() {
  overwriting_ = true; // overwriting_ 为true表明需要对buffer_中的内容进行重写
  for (auto& branch : branches_) { // 遍历branch_对分支指令的占位进行重写
    EmitBranch(&branch);
  }
  overwriting_ = false;
}
```
- `[&](XRegister reg, auto next)`为lambda
```c++
void Riscv64Assembler::EmitBranch(Riscv64Assembler::Branch* branch) {
  overwrite_location_ = branch->GetLocation(); // 获取要重写的location_
  const int32_t offset = branch->GetOffset();  // 计算分支指令（的目标地址？）与label的偏移量
  BranchCondition condition = branch->GetCondition();
  XRegister lhs = branch->GetLeftRegister();
  XRegister rhs = branch->GetRightRegister();

  auto emit_auipc_and_next = [&](XRegister reg, auto next) { // 用于长跳转的回调函数
    auto [imm20, short_offset] = SplitOffset(offset); // 拆分高20位和低12位
    Auipc(reg, imm20);
    next(short_offset);
  };

  switch (branch->GetType()) {
    // Short branches.
    case Branch::kUncondBranch:
    case Branch::kBareUncondBranch:
      J(offset);
      break;
    case Branch::kCondBranch:
    case Branch::kBareCondBranch:
      CHECK_EQ(overwrite_location_, branch->GetOffsetLocation());
      EmitBcond(condition, lhs, rhs, offset);
      break;
    ...
    case Branch::kLongUncondBranch:
      emit_auipc_and_next(TMP, [&](int32_t short_offset) { Jalr(Zero, TMP, short_offset); });
      break;
}}

// 计算偏移量
int32_t Riscv64Assembler::Branch::GetOffset() const {
  // Calculate the byte distance between instructions and also account for
  // different PC-relative origins.
  uint32_t offset_location = GetOffsetLocation();  // 分支指令中的地址的偏移位置（因为此处需要重写offset）
  int32_t offset = static_cast<int32_t>(target_ - offset_location);// 分支指令（地址？）与label的偏移量
  return offset;
}

// 用于长跳转中进行地址掩码计算的回调函数
uint32_t Riscv64Assembler::Branch::GetOffsetLocation() const {
  return location_ + branch_info_[type_].pc_offset;
  // 只有中跳转的kCondBranch21和长跳转的kLongCondBranch中的pc_offset为4，其于都为0
}
```
- 对于重写跳转指令跳转，先判断类型（`type_`，如`kCondBranch`）再判断跳转条件（`cond`，如`kCondEQ`）其中**类型分为有条件和分支和无条件分支**，有条件分支才要进一步判断具体跳转条件。
```c++
case Branch::kCondBranch21:
      EmitBcond(Branch::OppositeCondition(condition), lhs, rhs, branch->GetLength());
      CHECK_EQ(overwrite_location_, branch->GetOffsetLocation());
      J(offset);
      break;
```
- 讲解用于拆分长跳转地址偏移的SplitOffset函数：
```c++
// Split 32-bit offset into an `imm20` for LUI/AUIPC and
// a signed 12-bit short offset for ADDI/JALR/etc.
ALWAYS_INLINE static inline std::pair<uint32_t, int32_t> SplitOffset(int32_t offset) {
  DCHECK_LT(offset, 0x7ffff800); // 确保偏移量小于0x7ffff800
  // Round `offset` to nearest 4KiB offset because short offset has range [-0x800, 0x800).
  int32_t near_offset = (offset + 0x800) & ~0xfff; // 对offset向最近的4K四舍五入，0x800为2K
  // Calculate the short offset.
  int32_t short_offset = offset - near_offset;
  DCHECK(IsInt<12>(short_offset));
  // Extract the `imm20`.
  uint32_t imm20 = static_cast<uint32_t>(near_offset) >> 12;
  // Return the result as a pair.
  return std::make_pair(imm20, short_offset);
}
```

- **插播解释分支指令类型和条件之间的关系**：当Bcond使用`branches_.emplace_back`初始化Branch时，其运行`InitializeType(is_bare ? kBareCondBranch : kCondBranch);`，
```c++
void Riscv64Assembler::Branch::InitializeType(Type initial_type) {
  OffsetBits offset_size_needed = GetOffsetSizeNeeded(location_, target_); // 对于CondBranch来说，返回值一定是kOffset13

  switch (initial_type) {
    case kCondBranch:
      if (condition_ != kUncond) {
        InitShortOrLong(offset_size_needed, kCondBranch, kCondBranch21, kLongCondBranch);
        break;
      }
      FALLTHROUGH_INTENDED;
  ...
}}

Riscv64Assembler::Branch::OffsetBits Riscv64Assembler::Branch::GetOffsetSizeNeeded(
    uint32_t location, uint32_t target) {
  if (target == kUnresolved) { return kOffset13; }
  int64_t distance = static_cast<int64_t>(target) - location;
  if (IsInt<kOffset13>(distance)) {
    return kOffset13; // 对应条件分支，12 + 1 = 13, 其中多出的1位（可能）表示正负
  } else if (IsInt<kOffset21>(distance)) {
    return kOffset21; // 对应jal
  } else {
    return kOffset32; // （maybe）对应jalr长跳转
  }
}

void Riscv64Assembler::Branch::InitShortOrLong(Riscv64Assembler::Branch::OffsetBits offset_size,
                                               Riscv64Assembler::Branch::Type short_type,
                                               Riscv64Assembler::Branch::Type long_type,
                                               Riscv64Assembler::Branch::Type longest_type) {
  Riscv64Assembler::Branch::Type type = short_type;
  if (offset_size > branch_info_[type].offset_size) { // 对于条件分支来说，该if一定不会执行，因为条件分支都是短跳转
    type = long_type;
    if (offset_size > branch_info_[type].offset_size) {
      type = longest_type;
    }
  }
  type_ = type;
}
```

### assembler_test
- 以以下函数为例：
```c++
TEST_F(AssemblerRISCV64Test, BcondForward5KiB) {
  TestBcondForward("BcondForward5KiB", 5 * KB, "1", GetPrintBcondOppositeAndJ("2"));
}
```
- TestBcondForward函数用于测试前向（forward）分支跳转
```c++
template <typename PrintBcond>
  void TestBcondForward(const std::string& test_name,
                        size_t gap_size,
                        const std::string& target_label,
                        PrintBcond&& print_bcond,
                        bool is_bare = false) {
    std::string expected;
    Riscv64Label label;
    expected += EmitBcondForAllConditions(&label, target_label + "f", print_bcond, is_bare);
    expected += EmitNops(gap_size); // 根据输入的gap_size(5 * KB)来填充nop指令
    __ Bind(&label); // apply relocation
    expected += target_label + ":\n";
    DriverStr(expected, test_name);
  }
```
- `PromoteIfNeeded`函数用于判断是否需要条件分支扩展(Conditional Branch Expansion)，如kCondBranch21可表示branch后跟一个跳转指令。执行路径：
```bash
DriverWrapper -> FinalizeCode -> PromoteBranches -> PromoteIfNeeded
```
- GetPrintBcondOppositeAndJ用于条件分支扩展。beq跳转立即数范围为`12 + 1 shift = 13`位，即
```c++
auto GetPrintBcondOppositeAndJ(const std::string& skip_label) {
  return [=]([[maybe_unused]] const std::string& cond,
             const std::string& opposite_cond,
             const std::string& args,
             const std::string& target) {
    return "b" + opposite_cond + args + ", " + skip_label + "f\n" +
           "j " + target + "\n" +
          skip_label + ":\n";
  };
}

// 小于等于
if (a <= b) { // then code }
// 在 RISC-V 指令集中,并没有直接对应"小于等于"的分支指令。但我们可以通过以下方式实现:
bgt a, b, else_label 
// 如果a > b, 跳转到else_label
# then code
j end_label
else_label:
// else code
end_label:
```
- EmitBranch中对分支指令占位进行填充
```c++
void Riscv64Assembler::EmitBranch(Riscv64Assembler::Branch* branch) {
  ...
  switch (branch->GetType()) {
    // Short branches.
    case Branch::kUncondBranch:
    case Branch::kBareUncondBranch:
      J(offset);
      break;
    case Branch::kCondBranch:
    case Branch::kBareCondBranch:
      EmitBcond(condition, lhs, rhs, offset);
      break;
    case Branch::kCall:
    case Branch::kBareCall:
      Jal(lhs, offset);
      break;

    // Medium branch.
    case Branch::kCondBranch21:
      EmitBcond(Branch::OppositeCondition(condition), lhs, rhs, branch->GetLength());
      J(offset);
      break;

    // Long branches.
    case Branch::kLongCondBranch:
      EmitBcond(Branch::OppositeCondition(condition), lhs, rhs, branch->GetLength());
      FALLTHROUGH_INTENDED;
    case Branch::kLongUncondBranch:
      emit_auipc_and_next(TMP, [&](int32_t short_offset) { Jalr(Zero, TMP, short_offset); });
      break;
    case Branch::kLongCall:
      emit_auipc_and_next(lhs, [&](int32_t short_offset) { Jalr(lhs, lhs, short_offset); });
      break;

    // label.
    case Branch::kLabel:
      emit_auipc_and_next(lhs, [&](int32_t short_offset) { Addi(lhs, lhs, short_offset); });
      break;
    // literals.
    case Branch::kLiteral:
      emit_auipc_and_next(lhs, [&](int32_t short_offset) { Lw(lhs, lhs, short_offset); });
      break;
    case Branch::kLiteralUnsigned:
      emit_auipc_and_next(lhs, [&](int32_t short_offset) { Lwu(lhs, lhs, short_offset); });
      break;
    case Branch::kLiteralLong:
      emit_auipc_and_next(lhs, [&](int32_t short_offset) { Ld(lhs, lhs, short_offset); });
      break;
    case Branch::kLiteralFloat:
      emit_auipc_and_next(
          TMP, [&](int32_t short_offset) { FLw(branch->GetFRegister(), TMP, short_offset); });
      break;
    case Branch::kLiteralDouble:
      emit_auipc_and_next(
          TMP, [&](int32_t short_offset) { FLd(branch->GetFRegister(), TMP, short_offset); });
      break;
  }
}
```

### codegen过程
- `HGraphVisitor`：
- 堆污染（Heap poison）：用于在程序中检测和防止对已释放的内存或无效内存的访问。它的基本思想是，在释放内存时，将内存内容标记为无效或无法访问状态，从而使得任何试图访问已释放内存的操作都会触发错误或异常。堆污染通常被用于调试和安全性方面，它可以帮助发现一些内存相关的错误，比如使用已释放的内存或访问越界的内存。然而，在生产环境中，通常会关闭堆污染，以提高性能，因为它会引入额外的运行时开销。`PoisonHeapReference`使用`__ Sub_d(reg, Zero, reg);`来将寄存器值取负，使原来有效的引用变成无效的引用。在后续程序中尝试使用这个引用，就会引发错误或异常，从而提前暴露潜在的内存错误。
- 虚调用（virtual call）？
- GenerateIntLongCompareAndBranch函数的原理？

### JIT过程
- 首先`Jit::LoadCompilerLibrary`加载`libartd-compiler.so`。然后
- jit optimization中分为CriticalNative和FastNative，它们用于加速jni转换，它们主要靠略过加载类或线程的检查来直接调用native代码。它们的主要区别是CriticalNative不应该调用**托管对象**managed objects（即在java代码中创建并由虚拟机管理的对象，且传入和返回的都为非基本类型，如jobject），而FastNative则可使用托管对象。
- JIT编译入口函数：`OptimizingCompiler::JitCompile`，根据传入CompilationKind的不同(Osr, Baseline, Optimized)进行不同程度的JIT编译。
#### Native方法的jni stub编译过程
- `JNI_CreateJavaVM(java_vm_ext.cc)` -> `Runtime::Start(runtime.cc)` ->  `ClassLinker::EnsureInitialized(class_linker.cc)` -> `ClassLinker::InitializeClass(class_linker.cc)` -> `ArtMethod::Invoke(art_method.cc)` -> `art_quick_to_interpreter_bridge(quick_entrypoints_loongarch64.S)` -> `artQuickToInterpreterBridge(quick_trampoline_entrypoints.cc)` -> `EnterInterpreterFromEntryPoint(interpreter.cc)` -> `interpreter::Execute(interpreter.cc)` -> `Jit::MethodEntered(jit.cc)` -> `JitCompileTask::Run(jit.cc)` -> `Jit::CompileMethod(jit.cc)` -> `JitCompiler::CompileMethod(jit_compiler.cc)` -> `OptimizingCompiler::JitCompile(optimizing_compiler.cc)` -> `ArtQuickJniCompileMethod(jni_compiler.cc)`.然后最终调用`ArtJniCompileMethodInternal`来生成具体的jni bridge机器码。
- `compiler/jni/quick/jni_compiler.cc`：
```c++
// Generate the JNI bridge for the given method, general contract:
// - Arguments are in the managed runtime format, either on stack or in
//   registers, a reference to the method object is supplied as part of this
//   convention.
//
template <PointerSize kPointerSize>
static JniCompiledMethod ArtJniCompileMethodInternal(const CompilerOptions& compiler_options,
                                                     uint32_t access_flags,
                                                     uint32_t method_idx,
                                                     const DexFile& dex_file) {
......
// 为callee register， 方法指针和return address构造栈帧. 对于CriticalNative方法直接构造包含所要传递参数的栈帧，跳过后续的检查。
// 1. Build the frame saving all callee saves, Method*, and PC return address.
//    For @CriticalNative, this includes space for out args, otherwise just the managed frame.
......
// 更新quick frame指针，将构造出来的栈顶指针分配给ART Thread Register。CriticalNative不需要将栈顶指针保存到thread register的原因是由于@CriticalNative的特性，其不会调用java的托管对象，所以其在执行过程中GC被禁用，所以不需要保存。(quick frame是机器执行所用的栈，解释执行所用的栈称为shadow frame)
// 2. (if not critical_native) Write out the end of the quick frames.
__ StoreStackPointerToThread(Thread::TopOfManagedStackOffset<kPointerSize>());
......
// 3. Move frame down to allow space for out going args.
......
// 一些检查，对于@CriticalNative来说因为没有jclass的传入，所以可以不用做这些检查
// 4. Check if we need to go to the slow path to emit the read barrier for the declaring class in the method for a static call.
//    Skip this for @CriticalNative because we're not passing a `jclass` to the native method.
......
// 调用JniMethodStart函数传递Thread*，为从java native进入jni方法代码做准备。jni_start表示JniMethodStart，其会通过self->TransitionFromRunnableToSuspended(kNative)将线程状态从Runnable切换到Native，同时env->SetLocalRefCookie(env->GetLocalsSegmentState())创建JNI函数的local reference table. local reference table是记录在JNI函数中使用到的Java对象，这些对象是GC Roots的一部分，防止其还在使用的时候被GC回收。
// 5. Call into appropriate JniMethodStart passing Thread* so that transition out of Runnable
//    can occur. The result is the saved JNI local state that is restored by the exit call. We
//    abuse the JNI calling convention here, that is guaranteed to support passing 2 pointer arguments.
......
// 对于@CriticalNative，就是简单的将mr_conv中的从A寄存器开始放置的Arguments放入到main_jni_conv中的T寄存器。而对于非@CriticalNative方法，需要用到CopyParameter来拷贝变量，因为此时的方法中可能有托管对象jobject.
// 6. Fill arguments.
......
// 注：第7,8步为第六步中非@CriticalNative的方法的延续
	// 
	// 7. For static method, create jclass argument as a pointer to the method's declaring class.
```

#### dex/jit/java相关的一些概念
- GetMethodShorty（获取art method的[ShortyDescriptor描述符](https://source.android.google.cn/docs/core/dalvik/dex-format?hl=zh-cn#shortydescriptor)）
```bash

ShortyDescriptor的描述规则：（()表示一个Group，*  表示Group可有 0 到多个）
        ShortyDescriptor -> ShortyReturnType (ShortyFieldType)*
    ShortyReturnType的描述规则：
        ShortyReturnType -> 'V' | ShortyFieldType
    ShortyFieldType的描述规则：（引用类型统一用 L 表示）
        ShortyFieldType -> Z | B | S | C | I | J | F | D | L

# 举例
int calculateSum(float a, double b, String c)
# 对应的shorty描述符为
"IFDLjava/lang/String;" (return int; float/double)

void handleData(byte[] data, boolean isCompressed)
# 对应的shorty描述符为
"V[BZ" (void return; byte数组/boolean)
```
- 局部引用表(local reference table)：
- JniCallingConvention的作用？为什么main_jni_conv的传入为shorty，end_jni_conv传入的为jni_end_shorty。其造成的不同有什么？

#### HInstruction生成
- 比如`kGtBias`的参数是如何传入的HInstruction中的：其首先通过ProcessDexInstruction中的
```c++
case Instruction::CMPL_FLOAT: {
  Binop_23x_cmp(instruction, DataType::Type::kFloat32, ComparisonBias::kLtBias, dex_pc);
  break;
}

void HInstructionBuilder::Binop_23x_cmp(const Instruction& instruction,
                                        DataType::Type type,
                                        ComparisonBias bias,
                                        uint32_t dex_pc) {
  HInstruction* first = LoadLocal(instruction.VRegB(), type);
  HInstruction* second = LoadLocal(instruction.VRegC(), type);
  AppendInstruction(new (allocator_) HCompare(type, first, second, bias, dex_pc));
  UpdateLocal(instruction.VRegA(), current_block_->GetLastInstruction());
}
```
来生成H

####  Java方法编译
- 对于Main.java中的方法TestAdd：
```java
public class Main {
	public static int TestAdd(int a1, int a2) {
	    return a1 + a2;
	}

	public static void main(String args[]) {
     int ret = TestAdd(2, 2);
   }
}
```
对其进行编译：
```bash
javac Main.java
dx --dex --output=test.dex Main.class
dexdump -d test.dex
```
其中TestAdd的dex字节码为：
```bash
   #1              : (in LMain;)  
     name          : 'TestAdd'  
     type          : '(II)I'  
     access        : 0x0009 (PUBLIC STATIC)  
     code          -  
     registers     : 3  
     ins           : 2  
     outs          : 0  
     insns size    : 3 16-bit code units  
00012c:                                        |[00012c] Main.TestAdd:(II)I  
00013c: 9000 0102                              |0000: add-int v0, v1, v2  
000140: 0f00                                   |0002: return v0  
     catches       : (none)  
     positions     :    
       0x0000 line=19  
     locals        :    
       0x0000 - 0x0003 reg=1 (null) I    
       0x0000 - 0x0003 reg=2 (null) I
```
关键的dex字节码就是：`add-int v0, v1, v2; return v0`。其被分解为的HInstruction为`HParameterValue; HParameterValue; HGoto; HAdd ;HReturn; HExit`。
对于每一个Java方法，首先其dex字节码经过`AllocateRegisters`（这一步之前与架构关系都不大）后再将运行`CodeGenerator::Compile`，遍历HBasicBlock并运行其中的HInstruction，然后在为其中的一些HInstruction`GenerateSlowPaths()`。


#### some trick
- 为什么`XRegister temp = temp_location.AsRegister<XRegister>();`其中`temp_location = {<art::ValueObject> = {<No data fields>}, static kBitsForKind = 4, static kBitsForPayload = 60, static kLocationConstantMask = 3, static kStackIndexBias = 576460752303423488, value_ = 68}`时temp会被赋值为A0。
	- art中code-cache生成步骤：art里重定位这要是assembler_loongarch64.c这里branch.Resolve进行填充的，这里存放jit code的地址都是在高的虚地址处（首先存放在anon:dalvik-CompilerMetadata地址段中）。然后art里先用JitCodeCache::Reserve开辟一段比较低的地址（此处是PA，也就是jit code最终执行的地址），然后JitMemoryRegion::CommitCode中std::copy(code.begin(), code.end(), w_memory + header_size);将CompilerMetadata中的内容拷贝到高地址（VA）可写的jit-cache中地址中，而这个VA又是上面开辟的低地址的去掉可写权限的映射。（w_memory是VA，x_memory是PA）
- 判断字符串是否相等：`IntrinsicCodeGeneratorLOONGARCH64::VisitStringEquals`.