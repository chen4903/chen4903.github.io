---
layout: post
category: revm
---

## constants

列出一些有意义的常量

| 常量               | 值                                                           | 说明                                         |
| ------------------ | ------------------------------------------------------------ | -------------------------------------------- |
| BLOCK_HASH_HISTORY | 256                                                          | 最多访问最多前256区块的hash                  |
| PRECOMPILE3        | [0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 3] | 预编译地址0x3                                |
| STACK_LIMIT        | 1024                                                         | stack最大深度                                |
| MAX_INITCODE_SIZE  | 2 * eip170::MAX_CODE_SIZE                                    | 最大的init code                              |
| CALL_STACK_LIMIT   | 1024                                                         | call的调用栈深度（说明重入攻击最多深度1024） |
| MAX_CODE_SIZE      | 0x6000                                                       | 合约大小限制，这里应该是指runtime code       |

## eip170

| 常量          | 值     | 说明                                   |
| ------------- | ------ | -------------------------------------- |
| MAX_CODE_SIZE | 0x6000 | 合约大小限制，这里应该是指runtime code |

## eip4844

eip4844: TODO，一大堆常量，先挖个坑，暂时不看

## eip7702

eip7702: TODO，暂时不看这个常量，先挖个坑，暂时不看

## hardfork

记录了硬分叉的名字和对应的区块号

```rust
FRONTIER = 0,     // Frontier               0
FRONTIER_THAWING, // Frontier Thawing       200000
HOMESTEAD,        // Homestead              1150000
DAO_FORK,         // DAO Fork               1920000
TANGERINE,        // Tangerine Whistle      2463000
SPURIOUS_DRAGON,  // Spurious Dragon        2675000
BYZANTIUM,        // Byzantium              4370000
CONSTANTINOPLE,   // Constantinople         7280000 is overwritten with PETERSBURG
PETERSBURG,       // Petersburg             7280000
ISTANBUL,         // Istanbul	              9069000
MUIR_GLACIER,     // Muir Glacier           9200000
BERLIN,           // Berlin	                12244000
LONDON,           // London	                12965000
ARROW_GLACIER,    // Arrow Glacier          13773000
GRAY_GLACIER,     // Gray Glacier           15050000
MERGE,            // Paris/Merge            15537394 (TTD: 58750000000000000000000)
SHANGHAI,         // Shanghai               17034870 (Timestamp: 1681338455)
CANCUN,           // Cancun                 19426587 (Timestamp: 1710338135)
#[default]
PRAGUE, // PRAGUE                 TBD
OSAKA,            // Osaka                  TBD
```











