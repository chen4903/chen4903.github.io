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





































































