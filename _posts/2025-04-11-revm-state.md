---
layout: post
category: revm
---

## account_info

比较两个账户的时候，只比较了`balance`、`nonce`和`code_hash`，不包括`code`，并且hash也是只对这三个进行计算。同时我们可以知道，每一次账户进行了转账、交易等操作，那么他们的状态就不一样了

```rust
impl PartialEq for AccountInfo {
    fn eq(&self, other: &Self) -> bool {
        self.balance == other.balance
            && self.nonce == other.nonce
            && self.code_hash == other.code_hash
    }
}

impl Hash for AccountInfo {
    fn hash<H: Hasher>(&self, state: &mut H) {
        self.balance.hash(state);
        self.nonce.hash(state);
        self.code_hash.hash(state);
    }
}
```

并且`AccountInfo`有一些函数：

- `copy_without_code()`: 复制出来一个新的`AccountInfo`，移除了bytecode
- `without_code()`: 在原来的`AccountInfo`基础上，移除bytecode

判断一个账户是否存在，是看账户是否有代码、余额是否为0、nonce是否为0。因此，只要地址上有钱，它就是存在的，比如0地址，虽然没人控制

```rust
#[inline]
pub fn is_empty(&self) -> bool {
    let code_empty = self.is_empty_code_hash() || self.code_hash.is_zero();
    code_empty && self.balance.is_zero() && self.nonce == 0
}

/// Returns `true` if the account is not empty.
#[inline]
pub fn exists(&self) -> bool {
    !self.is_empty()
}
```

下面的函数，估计是合约创建的时候使用的，直接把nonce设置为1了。对应了[stackexchange](https://ethereum.stackexchange.com/questions/31809/why-do-contracts-start-with-nonce-1)上的疑问：

```rust
/// Initializes an [`AccountInfo`] with the given bytecode, setting its balance to zero, its
/// nonce to `1`, and calculating the code hash from the given bytecode.
#[inline]
pub fn from_bytecode(bytecode: Bytecode) -> Self {
    let hash = bytecode.hash_slow();

    AccountInfo {
        balance: U256::ZERO,
        nonce: 1,
        code: Some(bytecode),
        code_hash: hash,
    }
}
```

## types

- EvmState：每一个地址，对应一个账户状态，用mapping来映射
- EvmStorage：每个账户也是通过mapping来映射状态，从0开始，一一对应每个slot
- TransientStorage：这个就很有意思了，对应了Solidity中的`tstore`操作码。可以发现它的键是元组`(Address, U256)`，值是U256，说明瞬时存储只会简单地记录某个值，独立于storage和state之外

```rust
/// EVM State is a mapping from addresses to accounts.
pub type EvmState = HashMap<Address, Account>;

/// Structure used for EIP-1153 transient storage
pub type TransientStorage = HashMap<(Address, U256), U256>;

/// An account's Storage is a mapping from 256-bit integer keys to [EvmStorageSlot]s.
pub type EvmStorage = HashMap<U256, EvmStorageSlot>;
```

## lib

每一个EVM账户的基本信息：

- `info`：上面说过了

- `storage`: `HashMap<U256, EvmStorageSlot>`

- `status`: 有这么几种类型

  - Loaded: 默认状态，账户已经被加载，但是没有任何交互操作

  - Created: 账户刚刚被创建，用来指示我们不需要去数据库获取它的storage信息

  - SelfDestructed: 是否自毁了

  - Touched: 只有被标记为这个状态的时候，我们才会把这个账户存到数据库

  - LoadedAsNotExisting: 用于兼容硬分叉。

    - 在以太坊的 Spurious Dragon 硬分叉（EIP-161）之前：`空账户`和`不存在的账户`是两个不同的状态，这种区分导致了一些问题，比如可能会在状态树中存储大量的空账户。用于处理 Spurious Dragon 硬分叉之前的区块

    - 当一个账户被标记为 LoadedAsNotExisting 时，表示这个账户从数据库中加载时就不存在；这与账户存在但为空（balance = 0，nonce = 0，code = empty）是不同的状态

    - 创建一个新的账户的时候，`new_not_existing()`，用的是LoadedAsNotExisting

    - 比如下面的代码：在 Spurious Dragon 之后：直接检查账户是否为空；在 Spurious Dragon 之前：需要检查账户是否被标记为 LoadedAsNotExisting 且未被触及

      ```rust
      pub fn state_clear_aware_is_empty(&self, spec: SpecId) -> bool {
          if SpecId::is_enabled_in(spec, SpecId::SPURIOUS_DRAGON) {
              self.is_empty()
          } else {
              let loaded_not_existing = self.is_loaded_as_not_existing();
              let is_not_touched = !self.is_touched();
              loaded_not_existing && is_not_touched
          }
      }
      ```

  - Cold: 是冷状态还是热状态，用于gas计算

```rust
pub struct Account {
    /// Balance, nonce, and code
    pub info: AccountInfo,
    /// Storage cache
    pub storage: EvmStorage,
    /// Account status flags
    pub status: AccountStatus,
}
```

oh？合约被销毁之后，仍然可以重新活过来。这种`|=`和`-=`作为相反操作第一次见，用bit运算来修改状态，效率很高

```rust
/// Marks the account as self destructed.
pub fn mark_selfdestruct(&mut self) {
    self.status |= AccountStatus::SelfDestructed;
}

/// Unmarks the account as self destructed.
pub fn unmark_selfdestruct(&mut self) {
    self.status -= AccountStatus::SelfDestructed;
}
```

返回所有被修改的slot，这个在fuzz等场景我觉得非常有用

```rust
/// Returns an iterator over the storage slots that have been changed.
///
/// See also [EvmStorageSlot::is_changed].
pub fn changed_storage_slots(&self) -> impl Iterator<Item = (&U256, &EvmStorageSlot)> {
    self.storage.iter().filter(|(_, slot)| slot.is_changed())
}
```

每一个slot，有一个旧值和一个新值，并且通过`is_cold`来指示这个slot是否已经热了

```rust
pub struct EvmStorageSlot {
    /// Original value of the storage slot
    pub original_value: U256,
    /// Present value of the storage slot
    pub present_value: U256,
    /// Represents if the storage slot is cold
    pub is_cold: bool,
}
```

































































