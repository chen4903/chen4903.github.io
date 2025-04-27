---
layout: post
category: revm
---

## interface

定义了几个类型：Database、DatabaseRef、WrapDatabaseRef、DatabaseCommit。DatabaseCommit是写入数据库的接口，而前三者都实现了这几个与数据库交互的类似接口：

```rust
/// Gets basic account information.
fn basic_ref(&self, address: Address) -> Result<Option<AccountInfo>, Self::Error>;

/// Gets account code by its hash.
fn code_by_hash_ref(&self, code_hash: B256) -> Result<Bytecode, Self::Error>;

/// Gets storage value of address at index.
fn storage_ref(&self, address: Address, index: U256) -> Result<U256, Self::Error>;

/// Gets block hash by block number.
fn block_hash_ref(&self, number: u64) -> Result<B256, Self::Error>;
```

有另外三个不同的文件，包装了上面的接口：

- async_db：为Database进行了异步的实现，方便在异步和同步的上下文中调用上面四个接口

- empty_db：定义了一个名为 `EmptyDBTyped<E>`的结构体，它是一个“空数据库”的实现。这个数据库在查询时总是返回默认值，通常用于测试或作为占位符。
- try_commit：字如其名，尝试写入数据库

## in_memory_db

其设计了一个cache：它把`accounts`和`Bytecode`分开了，但是`accounts`中有`code_hash`可以在`contracts`中找到对应的内容。

```rust
pub struct Cache {
    /// Account info where None means it is not existing. Not existing state is needed for Pre TANGERINE forks.
    /// `code` is always `None`, and bytecode can be found in `contracts`.
    pub accounts: HashMap<Address, DbAccount>,
    /// Tracks all contracts by their code hash.
    pub contracts: HashMap<B256, Bytecode>,
    /// All logs that were committed via [DatabaseCommit::commit].
    pub logs: Vec<Log>,
    /// All cached block hashes from the [DatabaseRef].
    pub block_hashes: HashMap<U256, B256>,
}
```

并且，这个缓存很有意思，分内外层缓存。可以通过`flatten()`函数将外层的缓存写进内层：如果存在则覆盖，不存在则添加，除了Logs一定是追加。并且本身可以不断通过`nest()`函数来不断嵌套。

```rust
pub struct CacheDB<ExtDB> {
    /// The cache that stores all state changes.
    pub cache: Cache,
    /// The underlying database ([DatabaseRef]) that is used to load data.
    ///
    /// Note: This is read-only, data is never written to this database.
    pub db: ExtDB,
}
```

CacheDB有以下几个实用的函数：

- `load_account()`和`basic()`：从cache（内存）中，根据一个Address获取其对应的DbAccount，如果没有则从数据库中获取，如果数据库中也没有，则返回不存在。
- `insert_contract()`：插入一个`AccountInfo`，并且会在函数中计算其hash作为mapping的键。
- `insert_account_storage()`：对应于Solidity中的存储模型，给一个地址、slot位置、值，然后插入内容
- `replace_account_storage()`：对应于Solidity中的存储模型，给一个地址、slot位置、值，然后替换内容
- `commit()`：将改动的内容`changes`通过循环写进`CacheDB`，涉及到合约是否被销毁、是不是新创建等，然后才追加进缓存
- `code_by_hash()`：根据`code_hash`从缓存中获取Bytecode
- `storage()`：从缓存或底层数据库中获取指定账户的slot的值。它会优先从缓存中查找数据，如果缓存中没有找到，则从底层数据库中加载数据，并将其存储到缓存中

来看看`DbAccount`：

```rust
pub struct DbAccount {
    pub info: AccountInfo,
    /// If account is selfdestructed or newly created, storage will be cleared.
    pub account_state: AccountState,
    /// Storage slots
    pub storage: HashMap<U256, U256>,
}

pub enum AccountState {
		// 在Spurious Dragon硬分叉之前，空的和不存在的之间存在差异。我们在这里标记它。
    NotExisting,
    // EVM触及了这个账户。对于较新的硬分叉来说，这意味着它可以从状态中被清除/移除。
    Touched,
		// EVM清除了此账户的存储，主要是通过自毁操作，我们不向数据库请求存储槽，并假设它们是U256::ZERO
    StorageCleared,
    // EVM没有与这个账户交互
    #[default]
    None,
}
```

还有个BenchmarkDB，用来作为测试的，不用管。

## alloydb

用Alloy这个库，包装了一个database，实现了相同的接口，估计是方便用户用alloy直接和数据库进行交互。

## states

### account_status

将一个账户从数据库磁盘加载到内存之后，有这几种状态：

- `LoadedNotExisting`: 账户已加载，但不存在。
- `Loaded`: 账户已加载且存在。
-  `LoadedEmptyEIP161`: 账户已加载且为空，符合EIP-161标准。
- `InMemoryChange`: 账户中存在仅在内存中的更改。
-  `Changed`: 账户已被修改。
-  `Destroyed`: 账户已被销毁。
- `DestroyedChanged`: 账户已被销毁，然后被修改。
-  `DestroyedAgain`: 账户再次被销毁。

这三个Destroy状态也侧面说明了，同一个account可以不断地被创建和销毁，从而做到在同一个address更新字节码的功能，做到合约字节码可更新，现实中有黑客已经用过这种攻击手法。

账户状态可以进行转换：

- `on_created()`：当一个账户被创建的时候，会根据之前的状态转换为新状态
- `on_changed()`：当一个账户被改动的时候，根据之前的状态转换为新状态
- `on_touched_empty_post_eip161()`：处理账户在“被触碰但为空”的情况下的状态转换，确保符合 EIP-161 的要求。它根据当前账户的状态，返回一个新的状态，以反映账户是否被销毁或是否需要被清除
- `on_touched_created_pre_eip161()`：处理的是 EIP-161 之前 的账户状态转换

| 方法 | on_touched_empty_post_eip161 | on_touched_created_pre_eip161 |
|------------------------------|------------------------------------------------------------|------------------------------------------------------------|
| 适用场景 | 处理 EIP-161 之后“被触碰但为空”的账户状态转换。 | 处理 EIP-161 之前“被触碰或创建”的账户状态转换。 |
| 返回值 | 返回一个新的 AccountStatus。 | 返回 `Option<AccountStatus>`，表示状态可能不变（None）。 |
| 处理逻辑 | 根据 EIP-161 规则，将空账户标记为 Destroyed。 | 根据 EIP-161 之前的规则，处理账户的创建或触碰。 |
| 触发 unreachable! 的情况 | 如果账户状态是 Loaded 或 Changed，则触发 unreachable!。 | 如果账户状态是 Loaded 或 Changed，则触发 unreachable!。 |

- `on_selfdestructed()`：当一个账户被销毁的时候，会根据之前的状态转换为新状态
- `transition()`：用于合并账户状态，确保状态变化的一致性和正确性。比如：
  - 交易执行中的账户状态变更：转账，合约交互等
  - 区块执行中的账户状态合并：多个交易修改同一个账户，状态回滚等
  - 账户销毁与创建：比如账户 A 在交易 1 中被销毁，账户 A 在交易 2 中被重新创建并修改，在这种情况下，transition 函数可以用于合并账户 A 的状态变更，确保最终状态为 DestroyedChanged，表示账户被销毁后又发生了变更。

### bundle_account

这个模块的设计是为了解决 EVM中账户状态管理，它通过 BundleAccount 结构体，来跟踪、回滚和生成账户状态的变化。

```rust
pub struct BundleAccount {
    pub info: Option<AccountInfo>,
    pub original_info: Option<AccountInfo>,
    /// 包含原始状态和当前状态。
    /// 提取变更集时，我们比较原始值是否与当前值不同。
    /// 如果不同，我们将其添加到变更集中。
    /// 如果账户被销毁，我们忽略原始值，并将当前状态与U256::ZERO进行比较。
    pub storage: StorageWithOriginalValues, // 这个存储表示在区块改变之前的值。
    /// Account status.
    pub status: AccountStatus,
}

pub type StorageWithOriginalValues = HashMap<U256, StorageSlot>;

pub struct StorageSlot {
    /// 存储槽被更改之前的值。
    /// 当插槽首次加载时，这是原始值。
    /// 如果插槽没有被更改，这等于当前值。
    pub previous_or_original_value: U256,
    /// 当加载sload时，当前值被设置为原始值
    pub present_value: U256,
}
```

这个函数有点奇怪，为什么要加1：在 BundleAccount 结构体中，size_hint 方法返回 1 + self.storage.len()，这里的 "+1" 是因为账户的变更包含两个部分：

1. 账户基本信息的变更（对应 "+1"）：

   - 账户的基本信息（如余额、nonce、代码等）的变更算作一个变更单位。

   - 这些信息存储在 AccountInfo 中。

2. 存储槽的变更（对应 `self.storage.len()`）：

   - 每个存储槽（storage slot）的变更都算作一个变更单位。

   - 存储在 storage 字段中的每个键值对都代表一个存储槽的变更

```rust
pub fn size_hint(&self) -> usize {
    1 + self.storage.len()
}
```

- `storage_slot()`：获取某个slot的值
- `rever()`：
  - 根据传入的 AccountRevert 参数，将账户的状态（包括基本信息和存储）回滚到之前的状态。它会更新账户的 status、info 和 storage 字段，并根据情况决定是否可以移除账户。
  - 主要在`status`, `storage`和`accountInfo`三个方面回滚
- `update_and_create_revert()`：
  - 更新账户状态并生成一个 AccountRevert 对象，该对象可以用于将账户状态回滚到更新前的状态。如果没有需要回滚的变更，则返回 None。
  - 这个函数很复杂，主要分两部分：第一部分准备了一些helper函数，第二部分是根据transition.status来更新状态，并且返回更改之前的状态

### bundle_state

实现了一个区块链状态管理模块，核心是BundleState和BundleBuilder两个结构体，提供状态构建、转换、扩展和回滚的核心功能。很好的回滚机制，包括账户回滚和slot storage回滚

```rust
pub struct BundleBuilder { // builder模式，用于逐步构建BundleState
    states: HashSet<Address>,
    state_original: HashMap<Address, AccountInfo>,
    state_present: HashMap<Address, AccountInfo>,
    state_storage: HashMap<Address, HashMap<U256, (U256, U256)>>,

    reverts: BTreeSet<(u64, Address)>,
    revert_range: RangeInclusive<u64>,
    revert_account: HashMap<(u64, Address), Option<Option<AccountInfo>>>,
    revert_storage: HashMap<(u64, Address), Vec<(U256, U256)>>,

    contracts: HashMap<B256, Bytecode>,
}

pub struct BundleState { // 核心数据结构
    pub state: HashMap<Address, BundleAccount>,
    pub contracts: HashMap<B256, Bytecode>, // 这个区块中创建的所有合约，hash=>字节码
    pub reverts: Reverts, // 存储回滚信息，按块高度组织，允许回滚到之前的状态
    pub state_size: usize, // bundle state中plain state的大小
    pub reverts_size: usize, // bundle state中reverts的大小
}

pub enum OriginalValuesKnown { // 用于控制在转换为普通状态（plain state）时是否考虑原始值
    Yes, // 表示原始值已知，适用于不期望父块被提交或回滚的场景。
    No,  // 表示原始值不可靠，适用于状态可能被拆分或扩展的场景。
}

pub enum BundleRetention { // 决定是否保留回滚信息
    PlainState, // 只更新普通状态。
    Reverts, // plain state和reverts状态都会保留
}
```

其中的函数实现较为复杂，我们宏观上分析它们的功能。

- 状态构建（`BundleBuilder::build()`）

  - **流程**：

    - 遍历states，为每个地址创建BundleAccount，包含原始信息、当前信息和存储槽。
    - 将存储槽从(U256, U256)转换为StorageSlot（包含原始值和当前值）。
    - 按revert_range初始化回滚映射，遍历reverts生成AccountRevert（包含账户和存储槽的回滚信息）。
    - 整合状态、合约和回滚，生成BundleState。

  - **特点**：

    - 确保回滚信息仅在revert_range范围内有效，忽略无效输入。

    - 计算state_size和reverts_size，为性能优化提供依据。

- 状态转换（`to_plain_state()`）

  - 功能：
    - 将BundleState转换为StateChangeset，包含账户信息变更、存储变更和合约。
    - 根据OriginalValuesKnown决定是否检查原始值。
  - 逻辑：
    - 遍历state，检查账户信息是否变化（is_info_changed）或是否被销毁（was_destroyed）。
    - 对于存储槽，检查是否被销毁或值是否变化，生成PlainStorageChangeset。
    - 过滤空字节码的合约，确保仅记录有效合约。
  - **用途**：用于将内存中的状态变化持久化到数据库。

- 状态扩展（`extend()`）

  - 功能：合并两个BundleState，处理状态、合约和回滚的合并。
  - 逻辑
    - 如果另一个状态（other）的账户被销毁，将当前状态的存储槽转移到other的回滚中。
    - 合并状态时，优先保留other的最新信息，更新存储槽的当前值。
    - 扩展合约和回滚，保持一致性。
  - 用途：支持多块状态的合并，适用于区块链分叉或批量处理。

- 回滚（`revert()` 和 `revert_latest()`）

  - 功能：通过revert_latest回滚最近一次状态变化，revert支持回滚指定次数。
  - 逻辑：
    - 从reverts中弹出最近的回滚记录，遍历每个账户的回滚信息。
    - 根据AccountInfoRevert（恢复、删除或无操作）和存储槽回滚，更新或删除state中的账户。
    - 维护state_size和reverts_size的准确性
  - 用途：
    - 用于处理交易失败或分叉回滚，恢复到之前的状态。



































































































