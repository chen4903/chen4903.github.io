---
layout: post
category: blockchain
---

# ABIEncoderV2 Array

`solc` versions `0.4.7`-`0.5.9` contain a [compiler bug](https://blog.ethereum.org/2019/06/25/solidity-storage-array-bugs) leading to incorrect ABI encoder usage.

```solidity
pragma solidity 0.5.1;
pragma experimental ABIEncoderV2;
contract A {
    uint[2][3] bad_arr = [[1, 2], [3, 4], [5, 6]];
    
    /* Array of arrays passed to abi.encode is vulnerable */
    function bad() public view returns(bytes memory){                                                                                          
        bytes memory b = abi.encode(bad_arr);
        return b;
    }
}


// result:
// 0000000000000000000000000000000000000000000000000000000000000001
// 0000000000000000000000000000000000000000000000000000000000000002
// 0000000000000000000000000000000000000000000000000000000000000002
// 0000000000000000000000000000000000000000000000000000000000000003
// 0000000000000000000000000000000000000000000000000000000000000003
// 0000000000000000000000000000000000000000000000000000000000000004
```

`abi.encode(bad_arr)` in a call to `bad()` will incorrectly encode the array as `[[1, 2], [2, 3], [3, 4]]` and lead to unintended behavior.

Remediation: Use a compiler >= `0.5.10`.

# consturctor

In Solidity [0.4.22](https://github.com/ethereum/solidity/releases/tag/v0.4.23), a contract with both constructor schemes will compile. The first constructor will take precedence over the second, which may be unintended.

```solidity
pragma solidity 0.4.22;
contract A {
    uint public x; // x = 1
    constructor() public {
        x = 0;
    }
    function A() public {
        x = 1;
    }
    
    function test() public returns(uint) {
        return x;
    }
}
```

`abi.encode(bad_arr)` in a call to `bad()` will incorrectly encode the array as `[[1, 2], [2, 3], [3, 4]]` and lead to unintended behavior.

Remediation: Only declare one constructor, preferably using the new scheme `constructor(...)` instead of `function <contractName>(...)`.

# nested structs

Prior to Solidity 0.5, a public mapping with nested structures returned [incorrect values](https://github.com/ethereum/solidity/issues/5520).

```solidity
pragma solidity 0.4.26;

contract test {

    struct A {
        uint x;
    }

    struct B {
        A y;
    }

    // The map that maps a game ID to a specific game.
    // hello[any] is always equal to 128
    mapping(uint256 => B) public hello;

    constructor() public {
        A memory a;
        a.x = 10;
        B memory b;
        b.y = a;
        hello[0] = b;
        hello[1] = b;
        hello[2] = b;
    }

}
```

Remediation: use high compiler

```solidity
pragma solidity 0.8.17;

contract test {

    struct A {
        uint x;
        uint256 get;
    }

    struct B {
        uint256 num;
        A y;
    }

    // The map that maps a game ID to a specific game.
    mapping(uint256 => B) public hello;

    constructor() public {
        A memory a;
        a.x = 10;
        a.get = 2;
        B memory b;
        b.y = a;
        b.num = 11;
        hello[1] = b;
    }
// this works: hello[1]:
// 0:uint256: num 11
// 1:tuple(uint256,uint256): y 10,2
}
```

# immutable&view bug

**The following execution result is:** the value of `a` is `0x5B38Da6a701c568545dCfcB03FcB875f56beddC4`.

```solidity
pragma solidity 0.8.21;
contract C {
    address public immutable a;
    constructor() public {
        a = 0x4B20993Bc481177ec7E8f571ceCaE8A9e22C02db;
        F(msg.sender);
    }
    function F(address witnessAddress) view public returns(address){
        assembly{
            mstore(0x80,witnessAddress)
        }
        return a;
    }
}
```

**Reason:** Immutable variables are stored starting at memory position `0x80`, and the compiler uses `sload` to retrieve the value from this position to the stack. This means that if, during this period, the content of certain memory locations after `0x80` is modified, it will affect the immutable variable, even if the function is marked as `view`. This is because `view` operations are performed in memory; although they do not write to the blockchain, if some operations need to use data in memory, those operations can be affected. This situation can be caused by using inline assembly-related operations, such as `mstore`, `codecopy`, etc.

**Reference:** [issues#14049](https://github.com/ethereum/solidity/issues/14049)

