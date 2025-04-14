---
layout: post
category: revm
---

## opcode

每个操作码的定义：

```rust
pub struct OpCode(u8);
```

操作码的实际内容：其中`immediate_size`是立即数的字节大小，比如`PUSH1`到 `PUSH32` 操作码的 `immediate_size` 分别是 1-32，表示它们会在操作码后面带有相应字节数的立即数；`not_eof`用于EOF的验证，如果不是EOF操作码则为false，比如`CODESIZE`、`CODECOPY`、`EXTCODESIZE`、`EXTCODECOPY` 等操作码都被标记为 `not_eof`，这些操作码主要与传统合约格式相关，在新的 EOF 格式中不再支持；`terminating`是表示某个操作码是不是终止功能的，例如`RETURN`, `REVERT`等

```rust
pub struct OpCodeInfo {
    name_ptr: NonNull<u8>,
    name_len: u8,
    inputs: u8,
    outputs: u8,
    immediate_size: u8,
    not_eof: bool, // 这是为了支持 EVM 的向前兼容性，同时允许在新格式中禁用一些旧的操作码
    /// If the opcode stops execution. aka STOP, RETURN, ..
    terminating: bool,
}
```

## bytecode

有三种bytecode类型，其中LegacyAnalyzedBytecode是指只有一个`Stop`操作码的，并且它拥有一个`JumpTable`来指示字节码程序流如何跳转。

```rust
pub enum Bytecode {
    /// The bytecode has been analyzed for valid jump destinations.
    LegacyAnalyzed(LegacyAnalyzedBytecode),
    /// Ethereum Object Format
    Eof(Arc<Eof>),
    /// EIP-7702 delegated bytecode
    Eip7702(Eip7702Bytecode),
}
```

LegacyAnalyzedBytecode和Eof的区别：

| 特性         | LegacyAnalyzed                | Eof                                                          |
| :----------- | :---------------------------- | :----------------------------------------------------------- |
| 结构         | 无结构，原始字节              | 结构化，含魔术字节（`0xEF00`开头）和版本号，包含版本号和多个明确节（代码节、数据节等） |
| 跳转目标验证 | 部署时动态分析生成`JumpTable` | 部署时静态验证，无需`JumpTable`                              |
| 数据分离     | 代码末尾可能附加数据          | 代码和数据有明确分离的节                                     |
| 执行效率     | 需要Runtime跳转检查           | 直接执行，无额外检查                                         |
| 格式可塑性   | 可能存在歧义解释              | 严格验证，无歧义                                             |
| 典型用例     | 传统智能合约                  | 需要更高安全性的复杂合约                                     |

并且这三种类型的开头不一样：eof以`ef00`开头，Eip7702以`ef01`开头，普通的字节码则不能以前两者那样开头。

## legacy

`analyze_legacy()`函数，使用字节码来创建JumpTable，它存储了有效的跳转目的地。

```rust
pub fn analyze_legacy(bytetecode: &[u8]) -> JumpTable {
```

对于每一个legacy类型的字节码，其定义如下：
```rust
pub struct LegacyAnalyzedBytecode {
    /// Bytecode with 33 zero bytes padding
    bytecode: Bytes,
    /// Original bytes length
    original_len: usize,
    /// Jump table
    jump_table: JumpTable,
}
```

与其他的EVM实现一般是在运行时中分析字节码然后缓存JumpTable，而revm是提前创建好JumpTable，与bytecode是同级别一起运行的。所有legacy字节码末尾填充33个零字节，确保始终以有效的STOP操作码（0x00）结尾。填充33字节（而非1字节）是为了处理原字节码末尾存在PUSH32操作码但后续数据不足的情况。`original_len`用来支持后续复制。并且字节码长度不得为0。

```rust
impl LegacyRawBytecode {
    /// Converts the raw bytecode into an analyzed bytecode.
    ///
    /// It extends the bytecode with 33 zero bytes and analyzes it to find the jumpdests.
    pub fn into_analyzed(self) -> LegacyAnalyzedBytecode {
        let len = self.0.len();
        let mut padded_bytecode = Vec::with_capacity(len + 33);
        padded_bytecode.extend_from_slice(&self.0);
        padded_bytecode.resize(len + 33, 0);
        let jump_table = analyze_legacy(&padded_bytecode);
        LegacyAnalyzedBytecode::new(padded_bytecode.into(), len, jump_table)
    }
}
```

但是有个疑问，为什么在`LegacyAnalyzedBytecode::new()`创建的时候，不检验这33字节的padding呢？查看此[PR](https://github.com/bluealloy/revm/pull/2409)。

## eof

### Brief

eof是用来优化之前的字节码的，因为之前的字节码有着结构性不强等缺点。eof主要跟下面的这些EIP有关：

```
EIP-3540: EOF - EVM Object Format v1
EIP-3670: EOF - Code Validation
EIP-4200: EOF - Static relative jumps
EIP-4750: EOF - Functions
EIP-5450: EOF - Stack Validation
EIP-6206: EOF - JUMPF and non-returning functions
EIP-7480: EOF - Data section access instructions
EIP-663: SWAPN, DUPN and EXCHANGE instructions
EIP-7069: Revamped CALL instructions
EIP-7620: EOF Contract Creation
EIP-7698: EOF - Creation transaction
```

定义的时候有三部分：

```rust
pub struct Eof {
    pub header: EofHeader,
    pub body: EofBody,
    pub raw: Bytes,
}
```

### EofHeader

```rust
pub struct EofHeader {
    pub types_size: u16, // 类型部分包括输入输出的数量和最大Stack大小
    pub code_sizes: Vec<u16>, // 不可以为0
  	// container_section 是EOF中为合约部署阶段 设计的临时存储区域，主要用于初始化代码和配置数据，与Runtime无关。
    pub container_sizes: Vec<u16>, // 不可以为0
    pub data_size: u16,
    pub sum_code_sizes: usize,
    pub sum_container_sizes: usize,
}
```

其中`code_sizes`和`container_sizes`的存储方式不一样：

```
// code_sizes: 用于存储代码段的大小差值
假设 code_section = [10, 25, 35]
第一次: x = 10, prev_value = 0, ret = 10
第二次: x = 25, prev_value = 10, ret = 15
第三次: x = 35, prev_value = 25, ret = 10
最终 code_sizes = [10, 15, 10]

// container_sizes: 用于存储容器段的实际大小
假设 container_section 包含三个不同长度的容器
container1.len() = 5
container2.len() = 8
container3.len() = 3
最终 container_sizes = [5, 8, 3]
```

实际数据排布如下：可以见一个header的最小长度为15bytes

| Section           | 大小（bytes）                           | Description                                                 |
| ----------------- | --------------------------------------- | ----------------------------------------------------------- |
| Magic             | 2                                       | 用于标识该段字节码是 EOF 格式                               |
| Version           | 1                                       | 版本号，表示当前字节码的版本，方便兼容性处理。目前只可以为0 |
| Types Size        | 3                                       | 包括输入输出的数量和最大堆栈大小。                          |
| code_sizes        | 3                                       | Sizes of EOF code section。不可为0                          |
| num_code_sections | 2 * self.code_sizes.len()               |                                                             |
| container_sizes   | 0 或 3 + 2 * self.container_sizes.len() |                                                             |
| data_size         | 3                                       | EOF data size                                               |
| Terminator        | 1                                       | 终止符，用于标识段信息部分的结束，通常为0字节。             |

### EofBody

```rust
pub struct EofBody {
    pub code_info: Vec<CodeInfo>,
    pub code_section: Vec<usize>, // 每个代码段最后一个字节的索引
    pub code: Bytes,
    pub code_offset: usize,
    pub container_section: Vec<Bytes>,
    pub data_section: Bytes,
    pub is_data_filled: bool,
}
```

### CodeInfo

下面是CodeInfo的定义：

```rust
pub struct CodeInfo {
    // 代码段消耗的栈元素数量。一个字节，范围是`0x00-0x7F`
    pub inputs: u8,
    // 代码段返回的Stack元素数量，如果没有返回值，默认返回0x80。一个字节，范围是`0x00-0x80`
    pub outputs: u8,
    // 代码部分放置到栈上的最大元素数量。2个字节，范围是`0x0000-0x03FF`
    pub max_stack_size: u16,
}
```

### verification

AccessTracker 在EOF验证过程中扮演了状态跟踪与依赖管理的核心角色。通过跟踪代码段访问、管理处理顺序、记录子容器类型，确保EOF容器在结构、跳转和类型上的合法性，防止无效或冲突的操作。

- 跟踪代码段访问状态

  - 字段：`codes: Vec<bool>`

  - 功能：标记每个代码段（`code_section`）是否被访问过。例如，当通过 `CALLF` 或 `JUMPF` 调用其他代码段时，自动标记对应索引为 `true`。

  - 验证：最后检查所有 `codes` 是否全为 `true`，确保无未使用的代码段（避免死代码）。

- 管理代码段处理顺序

  - 字段：`processing_stack: Vec<usize>`
  - 功能：使用栈结构动态管理待处理的代码段索引。例如：初始时压入第一个代码段（索引0）。遇到 `CALLF` 调用新代码段时，压入其索引。

- 记录子容器调用类型

  - 字段：`subcontainers: Vec<Option<CodeType>>`
  - 功能：记录每个子容器（`container_section`）的调用类型（`Initcode` 或 `Runtime`）。例如：通过 `EOFCREATE` 调用子容器时，标记为 `Initcode`。通过 `RETURNCONTRACT` 调用子容器时，标记为 `Runtime`。

- 维护当前容器类型

  - 字段：`this_container_code_type: Option<CodeType>`
  - 功能：标识当前正在处理的容器类型（`Initcode` 或 `Runtime`），用于：
    - 检查 `RETURN`/`STOP` 是否在正确的上下文中使用。
    - 确保子容器调用与当前容器类型兼容。

举个例子：

1. **初始化处理**：`AccessTracker::new` 初始化，标记第一个代码段（索引0）为已访问，压入栈。
2. **处理 `CALLF` 指令**：解析目标代码段索引，调用 `access_code(index)` 标记并压栈。
3. **处理 `EOFCREATE` 指令**：解析子容器索引，调用 `set_subcontainer_type(index, Initcode)` 记录类型。
4. **验证结束**：检查所有代码段是否访问，所有子容器是否调用，确保符合EOF规范。

它的结构为：

```rust
pub struct AccessTracker {
    pub this_container_code_type: Option<CodeType>,
    /// accessed codes
    pub codes: Vec<bool>,
  	// stack中需要被处理的代码部分
    pub processing_stack: Vec<usize>,
    /// Code accessed by subcontainer and expected subcontainer first code type.
    /// EOF code can be invoked in EOFCREATE mode or used in RETURNCONTRACT opcode.
    /// if SubContainer is called from EOFCREATE it needs to be ReturnContract type.
    /// If SubContainer is called from RETURNCONTRACT it needs to be ReturnOrStop type.
    ///
    /// None means it is not accessed.
    pub subcontainers: Vec<Option<CodeType>>,
}
```

EOF container的code部分，叫做CodeType，它有两种类型：Initcode和Runtime

## eip7702

EIP-7702的字节码由这几部分组成：`0xEF00` (MAGIC) + `0x00` (VERSION) + 20 bytes of address。所以它的字节码长度必须为23，并且目前只支持版本0。

```rust
pub struct Eip7702Bytecode {
    pub delegated_address: Address,
    pub version: u8,
    pub raw: Bytes,
}
```





















