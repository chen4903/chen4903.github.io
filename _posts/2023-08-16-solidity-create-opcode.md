---
layout: post
category: blockchain
---

# create合约地址计算

地址address是20字节，160位。

## 原理

create操作码智能合约地址的计算公式：

- senderAddress：创建这个新合约的msg.sender的地址，可以是EOA地址，也可以是合约地址。
- nonce：随机数
  - 如果是合约来创建，那么新合约的随机数从1开始，合约每创建一个新合约，nonce+1（ EIP 161版本之前从0开始）。
  - 如果是EOA用户来创建，那么随机数就是该EOA用户迄今为止调用了多少笔交易
- rlp：将`senderAddress`和`nonce`进行RLP编码
- keccak256：进行Keccak256计算，取[12:31]，即结果的最右边 160 位

```solidity
// last 20 bytes of hash of rlp encoding of tx.origin and tx.nonce
keccak256(rlp(senderAddress, nonce))[12:31]
```

注意：EOA 地址创建与此不同，EOA 地址创建只是生成一个 256 位私钥。

## 计算代码

**solidity**

```solidity
pragma solidity 0.8.0;

contract a {
    function test() public returns(address){
        address lostContract = address(
        uint160(uint256(keccak256(abi.encodePacked(bytes1(0xd6), bytes1(0x94), address(????????), bytes1(0x01))))));
        return lostContract;
    }
}
//bytes1(0x01)是nonce
```

# create2合约地址计算

`CREATE2` 操作码使我们在智能合约部署在以太坊网络之前就能预测合约的地址。`Uniswap`创建`Pair`合约用的就是`CREATE2`而不是`CREATE`。这一讲，我将介绍`CREATE2`的用法

## CREATE2如何计算地址

`CREATE2`的目的是为了让合约地址独立于未来的事件。不管未来区块链上发生了什么，你都可以把合约部署在事先计算好的地址上。用`CREATE2`创建的合约地址由4个部分决定：

- `0xFF`：一个常数，避免和`CREATE`冲突
- 创建者地址
- `salt`（盐）：一个创建者给定的数值
- 待部署合约的字节码（`bytecode`）

```text
新地址 = hash("0xFF",创建者地址, salt, bytecode)
```

`CREATE2` 确保，如果创建者使用 `CREATE2` 和提供的 `salt` 部署给定的合约`bytecode`，它将存储在 `新地址` 中。

## 计算包含特定字符的地址

以下以计算一个包含'5a54'的地址为例

```python
//使用python，引入web3库
from web3 import Web3

s1 = '0xff406187E1b3366B5da3539D99C4E88E42FC60De50'

s3= 'da647010355608442b3eab68e7dcc6d5b836f2628d2366ff8ae853413a643965'
i = 0
while(1):
    salt = hex(i)[2:].rjust(64,'0')
    s = s1+salt+s3
    hashed = Web3.sha3(hexstr=s)
    hashed_str = ''.join(['%02x' % b for b in hashed])
    if '5a54' in hashed_str[26:]:
        print(salt,hashed_str)
        break
    i += 1
    print(salt)
//输出结果：
//salt:0000000000000000000000000000000000000000000000000000000000000314 //hashed_str:ed2bb3003f33323b6ab51649857345a545213f2cf30379035ce2d922d1bf1f9d
```

为什么`hashed_str[26:]`这样子设计呢？原因：这个脚本返回的第二个字符串是一个六十四位的数据，我们只需要后面的四十位作为地址，又因为题目合约的限定需要从第三位开始满足拥有“5a54”，因此我们需要从第二十六位之后的三十八位中含有"5a54"

注意：我们create2生成的地址是hashed_str后面的40位。即`857345a545213f2cf30379035ce2d922d1bf1f9d`。原因：判断5a54的时候是从第二十六位开始的，地址只需要四十位。

## 如何使用`CREATE2`

（1）一般使用

```solidity
Contract x = new Contract{salt:_salt,value:_value}(params)
```

- Contract：创建的合约名
- x：合约对象（地址）
- _salt：指定的盐
- value：如果构造函数是 payable，可以创建时转入value 数量的 ETH（wei）
- params：新合约构造函数的参数

（2）编码模式

```solidity
predictedAddress = address(uint160(uint(keccak256(abi.encodePacked(
                bytes1(0xff),
                address(this),
                salt,
                keccak256(type(Pair).creationCode)
            )))));
```

（3）内联汇编

```solidity
assembly {
	addr := create2(0, add(bytecode, 0x20), mload(bytecode), salt)
}
```

1. 发送给新合约的wei数(msg.value).
2. 跳过bytecode前面32个字节长度的数据
3. 获取bytecode的长度
4. salt ，我们将它作为可控参数，可以在计算后再提供，是一个随机数nonce

解释：bytecode的前32个字节存储的是这个bytecode的总长度，32个字节后面的数据才是bytecode真正的数据

注意：使用过一次create2部署之后，不可以再次使用相同的（再次调用方法），否则它会返回0地址，因为不可能部署两次相同的地址

例子：

```solidity
pragma solidity 0.8.20;

contract deployed{
    
  bytes bytecode = hex"xxxx";
  
  function deploy(bytes32 salt) public returns(address) {
    bytes memory _bytecode = bytecode;
    address addr;
    
    assembly {
      addr := create2(0, add(_bytecode, 0x20), mload(_bytecode), salt)
    }
    return addr;
  }

  function getHash()public view returns(bytes32){
    return keccak256(bytecode);
  }
}
```

