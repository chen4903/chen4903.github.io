---
layout: post
category: blockchain
---

# GAS优化-非内联汇编方式

### 自定义错误

使用自定义错误实例通常比字符串描述便宜得多，因为您可以使用错误的名称来描述它，该名称仅用四个字节编码。

### constant和immutable

与常规状态变量相比，constant和immutable变量的气体成本要低得多。对于constant变量，分配给它的表达式会复制到访问它的所有位置，并且每次都会重新求值。这允许进行局部优化。在构建时对immutable变量求值一次，并将其值复制到代码中访问它们的所有位置。对于这些值，保留32个字节，即使它们可以容纳更少的字节。因此，constant值有时可能比immutable值便宜。

```solidity
//Gas used: 187302
contract Expensive {
   IERC20 public token;
   uint256 public timeStamp = 566777;

   constructor(address token_address) {
       token = IERC20();
   }

}

//Gas used: 142890
contract GasSaver {
   IERC20 public immutable i_token;
   uint256 public constant TIMESTAMP = 566777;

   constructor(address token_address) {
       i_token = IERC20(token_address);
   }
}
```

### anonymous事件

事件签名的哈希结果是topics之一，除非您使用`anonymous`声明事件。这意味着无法按名称筛选特定的匿名事件，只能按合约地址进行筛选。匿名事件的优点是部署和调用它们更便宜。它还允许您声明四个索引参数，而不是三个。

### 内存代替链上数据

获取链上数据的gas(100 gas)比获取内存中的gas(3 gas)多得多。因此，如果方法中需要多次用到同一个链上状态数据，可以先将它获取用一个变量A接收，然后之后的每一处用到的地方用变量A代替即可。例子：UniswapV2Pair.sol。又比如下面这个例子：

```solidity
contract SaveGas {
    uint public var1  = 70;

function testFunction1() external view returns (uint) {
        uint sum = 0;

        for (uint i = 0; i < var1; i++) {
            sum += i;
        }
        return sum;
}

function testFunction2() external view returns (uint) { //这个更省gas
        uint sum = 0;

        uint _var1 = var1;

        for (uint i = 0; i < _var1; i++) {
            sum += i;
        }
        return sum;
    }
}

//=====================================================================================

// SPDX-License-Identifier: MIT
pragma solidity 0.8.7;

contract Gas_Test{
    uint[] public arrayOfNumbers;
    uint public total;

    constructor() {
        arrayOfNumbers = [1,2,3,4,5,6,7,8,9,10,11,12,13];
    }


    function optionA() external {
        for (uint i =0; i < arrayOfNumbers.length; i++){
            total = total + arrayOfNumbers[i];
        }
    }


    function optionC() external { //这个更省gas
        uint _total;
        uint[] memory _arrayOfNumbers = arrayOfNumbers;
        for (uint i =0; i < _arrayOfNumbers.length; i++){
            _total = _total + _arrayOfNumbers[i];
        }
        total = _total;
    }
}

```

### 使用calldata

```solidity
//SPDX-License-Identifier:MIT;
pragma solidity ^0.8.3;
contract GasSaver {
    //Gas used: 22471
    function passParameterAsCallData(string calldata _name) public returns (string memory){}
    //Gas used: 22913
    function passParameterAsMemory(string memory _name) public returns (string memory){}
}
```

### +1操作

示例列出了四个合约，它们显示了我们可以将数字变量增加 1 的方式。合约`Four`在运行时更有效地从批次中取出。要在增加变量时节省更多费用，请用第四种方式

```solidity
//SPDX-License-Identifier:MIT;

pragma solidity ^0.8.3;

contract One{
   uint256 public number;
  //Gas used : 43800
   function incrementByOne() public returns (uint256){
      number += 1;
      return number;
   }
}

contract Two{
   uint256 public number;
   //Gas used : 43787
   function incrementByOne() public returns (uint256){
     number = number + 1;
     return number;
   }
}
contract Three{
   uint256 public number;
   //Gas used : 43634
   function incrementByOne() public returns (uint256){
       return number++;
   }
}

contract Four{
   uint256 public number;
   //Gas used : 43628
   function incrementByOne() public returns (uint256){
       return ++number;
   }
}
```

### 使用view

调用一个view方法不需要消耗gas。但是如果在一个方法A中，调用一个view的方法B，那么方法B将消耗gas

```solidity
contract GasSaver {
    uint256[] private numbers = [2,3,5,67,34];

    function getNumberAt( uint256 _index ) public view returns (uint256){
        return numbers[_index];
    }
    
    function sumAndMultiply() public {
       uint256[] memory _numbers = numbers;
       uint256 arrlength = _numbers.length;
       for(uint256 i=0; i < arrlength; ++i){
          numbers[i] = _numbers[i] * i;
       }
       //如果不调用下面的getNumberAt(2)方法，则sumAndMultiply()消耗44450gas
       //如果调用下面的getNumberAt(2)方法，则sumAndMultiply()消耗44778gas
       //getNumberAt(2);
    }
}
```

### unchecked

使用uncheck处理整数溢出：在 Solidity`0.8.0`版本之前，整数溢出和下溢检查是通过使用该`SafeMath`库执行的。从 Solidity`0.8.0`版本之后，编译器会为我们进行检查。这造成额外费用`gas`。如果我们知道我们将在合约内部执行的数学运算不会上溢或下溢，我们可以告诉编译器不要检查操作中的上溢或下溢。合约`OverFlow`有两个函数用于在合约内部设置两个变量。编译器`setNumberOne`检查并防止数字上溢或下溢。在第二个函数`setNumberTwo`中，我们告诉编译器要注意它的事情，不要检查溢出或下溢。使用`unchecked`意味着不会检查代码块是否有上溢或下溢。如果我们知道我们的数学是安全的，我们可以通过使用`unchecked`

```solidity
contract OverFlow {
    uint256 public numberOne;
    uint256 public numberTwo;

    //Gas used: 43440
    function setNumberOne() public {
        ++numberOne;
    }
    //Gas used: 43339
     function setNumberTwo() public {
       unchecked {
           ++numberTwo;
       }
    }
}
```

### 使用revert代替require

我们`require`用来检查一个语句是否是`true`. 如果语句为假，则交易将被还原，剩余的 gas 将返还给用户。由于 Solidity 中有自定义错误语法，我们可以使用该`revert`语句来发出自定义错误。使用`revert`而不是`require`更省气。您可以运行下面的代码来查看使用时`gas`节省的数量，`revert`而不是`require`.

```solidity
error NotEnough();
contract GasSaver {
   uint256 public number;
   //Gas Used: 21898
    function setNumber(uint256 _number) public {
        require(_number > 10, "number too small please");
        number = _number;
    }
     //Gas Used: 21669
     function setNumberAndRevert(uint256 _number) public {
        if ( _number < 10 ){
            revert NotEnough();
        }
        number = _number;
    }
}
```

### 避免内存输入数据过大

避免在内存变量中填写过大的数据：当一个memory变量中的参数大于32kb的时候，消耗的gas将会平方级增加。我们应避免传入过大的数据给memory变量

```solidity
contract One {
    //Gas used: 29261
    function setArrayInMemory() public pure {
        uint256[1000] memory _array;
    }
}

contract Two {
    //Gas used: 276761
    function setArrayInMemory() public pure {
        uint256[10000] memory _array;
    }
}

//Gas used: 922855
contract Three {
    function setArrayInMemory() public pure {
        uint256[20000] memory _array;
    }
}

contract Four {
    //Gas used: 3386918
    function setArrayInMemory() public pure {
        uint256[40000] memory _array;
    }
}
```

### require

在进行任何状态更改之前，始终将您的`require`语句放在函数的顶部，以便如果`require`语句失败，剩余`gas`的将在还原时退还给用户。使用`&&`或`||`与`require`语句进行比较操作时；您应该首先放置操作中较便宜的部分，这样如果它失败，编译器将不会比较第二部分，从而节省`gas`

### 异或交换

异或交换：因为创建temp变量需要燃料，下面使用一个简单的解决方案：抑或交换法

```solidity
//temp变量交换
if (arr[j] <= pivot){
	i++;
	int temp = arr[i];
	arr[i] = arr[j];
	arr[j] = temp;
}
//异或交换
arr[uint(i)] ^= arr[uint(j)];
arr[uint(j] ^= arr[uint(i)];
arr[uint(i)] ^= arr[uint(j)];
```

### 优化器

Solidity 编译器用于生成高度优化的字节码。

`truffle.config.js`通过在 truffle 配置项目的文件中或 hardhat 配置项目的 hardhat.config.js` 中包含以下配置来启用优化器。

```js
module.exports = {
  compilers: {
    solc: {
      version: <string>, // A version or constraint - Ex. "^0.5.0"
                         // Can be set to "native" to use a native solc or
                         // "pragma" which attempts to autodetect compiler versions
      docker: <boolean>, // Use a version obtained through docker
      parser: "solcjs",  // Leverages solc-js purely for speedy parsing
      settings: {
        optimizer: {
          enabled: <boolean>,
          runs: <number>   // Optimize for how many times you intend to run the code
        },
        evmVersion: <string> // Default: "istanbul"
      },
      modelCheckerSettings: {
        // contains options for SMTChecker
      }
    }
  }
}

```

旁注：确保在开发环境中禁用它，因为它会花费相对更多的时间来执行。

hardhat

```js
module.exports = {
  solidity: {
    version: "0.8.9",
    settings: {
      optimizer: {
        enabled: true,
        runs: 1000,
      },
    },
  },
}
```

### 使用event存储

日志是交易收据的一部分。它们由客户在执行交易时生成，并与区块链一起存储以允许检索它们。日志本身并不是区块链本身的一部分，因为它们不需要达成共识（它们只是历史数据），但是它们由区块链验证，因为交易收据哈希存储在块内。

### 存储槽

solidity的数据存储在一个个slot中，每一个slot为256bit。如果一个存储操剩余空间放不下接下来的一个变量，那么它会被放到下一个slot。

下面的代码会被放到3个slot

```solidity
uint128 x;
uint256 y;
uint128 z;
```

下面的代码会被放到2个slot

```solidity
uint128 x;
uint128 y;
uint256 z;
```

注意，这叫做紧密打包，只有在struct中才有效

### do-while代替for

```solidity
contract SaveGas {

    // 43406 gas
    function loop() public {
        for (uint256 i; i < 200; i++) {}
    }
}

contract SaveGas2 {

    // 40579 gas
    function loop() public {
        uint256 i;
        do {
            i++;
        } while (i < 200);
    }
}
```

该标准`for loop`将在执行语句之前检查条件。A`do while`将至少执行一次语句，然后检查条件，以节省 gas。

> **警告：**仅当您确定代码应至少执行一次时才使用 do-while，并记住处理您的条件以防止无限循环和 gas out。

### 使用uint256

EVM（以太坊虚拟机）由 256 位（32 字节）变量驱动。这意味着在我们下面的循环中，变量i在使用之前会从uint16转换成uint256，缺少的位会补0，这种转换会消耗 gas。

```solidity
contract SaveGas {

    // 28179 gas
    function loop() public {
        uint16 i;
        do {
            unchecked{++i;}
        } while (i < 200);
    }
}

contract SaveGas2 {

    // 26979 gas
    function loop() public {
        uint256 i;
        do {
            unchecked{++i;}
        } while (i < 200);
    }
}
```

### 不用默认值

将存储值从零值更改为非零值将花费`20000 gas`，并且所有非零的第 n 次写入将花费`5000 gas`。因此，我们在定义一个变量的时候，就给它初始化一个值，而不用默认值

```solidity
contract SaveGas {

    uint256 _data;

    // #1: 43494 gas
    // #n: 23594 gas
    function setData(uint256 x) public {
        _data = x;
    }
}

contract SaveGas2 {

    uint256 _data = 1;

    // #1: 23594 gas
    // #n: 23594 gas
    function setData(uint256 x) public {
        _data = x;
    }
}
```

### 位图与位运算符

如 uint8 二进制表示为 00000000，其中每一位有 0、1 两种情况，我们默认为 1 时则为 true，0 为 false，此时可以达到将 bool 值以位的形式来进行管理。下面分别用 bool 数组和 位运算 来管理相同的数据。

```solidity
contract Bitmap {
    bool[10] implementationWithBool;
    uint8 implementationWithBitmap;

    function setDataWithBoolArray(bool[8] memory data) external {
        implementationWithBool = data;
    }

    function setDataWithBitmap(uint8 data) external {
        implementationWithBitmap = data;
    }

    function readWithBoolArray(uint8 index) external returns (bool) {
        return implementationWithBool[index];
    }

    function readWithBitmap(uint indexFromRight) external returns (bool) {
        uint256 bitAtIndex = implementationWithBitmap & (1 << indexFromRight);
        return bitAtIndex > 0;
    }
}
```

| 关键字   | gas消耗 | 节省            | 结果       |
| -------- | ------- | --------------- | ---------- |
| 位运算符 | 22366   | 13363(≈37%)节省 | ✅ 建议结果 |
| 普通变量 | 35729   |                 |            |

### 创建合约

在工厂合约当中，我们经常需要创建若干子合约，常见的方式有三种

- new，通过现有合约进行创建，需随工厂合约将子合约部署在同一合约中；
- create2，通过 creation code 进行创建，需要在子合约部署前，提前在工厂合约中存储一份 creation code；
- clone2，通过对现有合约进行克隆，需要提前部署一份子合约，使用子合约的合约地址进行克隆。

```solidity
function newContract() external returns (address) {
    WattingDeploy metaCoin = new WattingDeploy();
    return address(metaCoin);
}

function createClone(address target) internal returns (address result) {
    bytes20 targetBytes = bytes20(target);
    assembly {
        let clone := mload(0x40)
        mstore(
            clone,
            0x3d602d80600a3d3981f3363d3d373d3d3d363d73000000000000000000000000
        )
        mstore(add(clone, 0x14), targetBytes)
        mstore(
            add(clone, 0x28),
            0x5af43d82803e903d91602b57fd5bf30000000000000000000000000000000000
        )
        result := create(0, clone, 0x37)
    }
}

function create2Contract(uint _salt) external payable returns (address) {
    address addr;
    bytes memory bytecode = creationCode;
    assembly {
        addr := create2(0, add(bytecode, 32), mload(bytecode), _salt)
        if iszero(extcodesize(addr)) {
            revert(0, 0)
        }
    }
    return addr;
}
```

以下是经过三种方式进行合约创建后的 gas 情况，优化建议如下：

使用 new 操作符在提供便利的同时，会将子合约的大小塞入到工厂合约中，极易容易出现合约大小超过 24k 的情况。对比 create2 和 clone2，更推荐使用 clone2 来进行 gas 优化，相比较而言，更节省创建时所耗费的 gas。

| 数据类型 | gas 消耗 | 节省        | 结果   |
| -------- | -------- | ----------- | ------ |
| clone2   | 41493    | 38022(≈48%) | ✅ 建议 |
| create2  | 93031    |             |        |
| new      | 79515    |             |        |

### 函数选择器位置

当合约调用方法的时候，是根据keccak256得到的16进制函数选择器进行匹配的，每匹配一次，消耗22个gas。函数选择器是按照字母顺序进行排序的，然后逐个比较匹配。比如下面的方法：

```solidity
pragma solidity 0.8.7;
// 未排序
contract Colors{
  //21252 
  function blue() external {} // 0xed18f0a7
  //21296 
  function green() external {} // 0xf2f1e132
  //21274 
  function purple() external {} // 0xed44cd44
  //21186 
  function red() external {} // 0x2930cf24
  //21208 
  function white() external {} // 0xa0811074
  //21230 
  function yellow() external {} // 0xbe9faf13
}
```

可以看到`green()`函数将比调用`red()`函数多花费 110 Gas，通过排序我们可以看出是为啥：

```solidity
red() // 0x2930cf24
white()  // 0xa0811074
yellow()  // 0xbe9faf13
blue()  // 0xed18f0a7
purple()  // 0xed44cd44
green() // 0xf2f1e132
```

### 热访问与冷访问

第一次访问storage变量消耗2100gas，如果访问memory变量则消耗100gas，因此在需要多次访问storage变量的时候，在方法里面用局部变量存储storage变量

### 零值&非零值&GAS退款

在以太坊上将值从 0 更改为非零是昂贵的（G sset = 20,000 Gas），而将值从非零更改为 0，可以给您退款 Gas 值。需要注意的是，最多只能获得总交易成本 20% 的退款，这意味着只有交易成本至少为 24,000 Gas 时才能获得退款。Gas 退款的相应公式可以在黄皮书中找到（黄皮书公式72）。

下面有几个案例：

**情景1**

Alice 有 10 个代币，Bob 有 0 个代币。Alice 将向 Bob 发送 5 个代币。这会将 Alice 的余额从非零值 (10) 更改为另一个非零值 (5)，并将 Bob 的余额从 0 更改为非零值（10 个代币）。

- 非零到非零（5,000 个gas）+ 零到非零（20,000 个gas）= 25,000 个gas

**情景2**

Alice 有 10 个代币，Bob 有 0 个代币。Alice 将把她所有的 10 个代币发送给 Bob。这会将 Alice 的余额从非零值更改为零，并将 Bob 的余额从非零 (0) 更改为非零 (10)。

- 非零到零 (5,000 Gas) + 零到非零 (20,000 Gas) - 退款 (4,800 Gas) = 21,200 Gas

显然，情景2 中交易的 Gas 成本更便宜，因为将 Alice 的余额值从非零更改为 0 会奖励退款金额。这应该可以帮助您注意到，对于每个非零到 0 的操作，最好在交易的其他地方花费至少 24,000 Gas

```solidity
pragma solidity 0.8.7;

contract test{
  uint256 public x;

  // 23444 
  function set01() external {
    x = 1;
  }

  // 21422
  function set02() external {
    x = 0;
  }
}
```

### external代替public

如果内部不需要调用本方法，只有外部需要调用，则用external代替Public，因为更加便宜

### 优化器

Solidity Optimizer 可以对智能合约的两个方面产生影响：部署成本或函数调用成本。优化器设置的越小（假设为 200），对部署成本的影响就越大。另一方面，运行次数越多（假设为 10,000 次），对函数调用成本的影响就越大。这是三种不同优化器设置的清晰简单的示例：

为了优化GAS，请始终使用 Solidity 优化器。最好将优化器设置得尽可能高，直到它不再有助于降低函数调用的 Gas 成本。可以推荐这样做，因为函数调用的执行次数比合约的部署执行的次数要多，而合约的部署只发生一次。

但是，如果您正在处理的项目对部署成本非常敏感，那么使用较低的优化器值直到它不再有助于降低部署成本应该会有所帮助。

### payable

标有payable的函数比没标的更加便宜，原因：对于payable函数，合约需要有一些额外的操作码来准备检查另一个合约或外部账户是否正在尝试向其发送 ETH，如果是的话，revert交易。Payable 函数没有那些额外的操作码，允许函数接收 ETH 并使其（与直觉相反）更便宜。

```solidity
pragma solidity 0.8.7;

contract test{

  //21208 
  function set01() public {}
  //21162 
  function set02() public payable {}
}
```

### 内存过大

当合约调用在单笔交易中需要使用超过 32 KB 的内存存储时，内存成本将进入以下表达式的二次部分（黄皮书公式 326）：

```solidity
pragma solidity 0.8.7;

contract test01{
  //29_261 
  function test() external {
    uint256[1000] memory x;
  }
}

  
contract test02{
  //276_761 
  function test() external {
    uint256[10000] memory x;
  }
}

contract test03{
  //922_855 
  function test() external {
    uint256[20000] memory x;
  }
}
```

正如您所看到的，向内存添加 10,000 个uint256的成本接近于向内存添加 1,000 个uint256的成本的十倍（276,761 Gas vs 29,261 Gas），这是有道理的，因为 10,000 是 1,000 的十倍。到目前为止，内存使用量仍然处于方程的线性部分。

但是，向内存添加 20,000 uint256 （922,855 Gas）的成本不仅仅是 10,000 uint256 Gas 值的两倍，而是近 4 倍。在“TwentyK”合约中，内存使用的千字节量达到了等式的第二部分。

为了避免这种内存成本爆炸，请尝试将事务的实现分解为多个片段，并且不要在单个事务中使用大量项目填充数组。

### 不用小于等于或大于等于

在 Solidity 中，≤ 或 ≥ 表达式没有单一的操作码。幕后发生的事情是，Solidity 编译器执行 LT/GT（小于/大于）操作码，然后执行 ISZERO 操作码来检查先前比较的结果 (LT/GT) 是否为零并验证与否。

```solidity
pragma solidity 0.8.7;

contract test01{
  //21391 
  function test() external returns(bool){
    return 3 > 2;
  }
    
}

  
contract test02{
  //21394 
  function test() external returns(bool){
    return 3 >= 3;
  }
}
```

这些合约之间的 Gas 成本相差 3，这是执行 ISZERO 操作码的成本，使得 < 和 > 的使用比 ≤ 和 ≥ 便宜。

### && 或 || 

当require语句需要 2 个或更多表达式时，首先放置消耗较少 Gas 的表达式。因此，在require语句中使用 `&&` 或 `||` 运算符，将最便宜的表达式首先执行，以便可以（有时）绕过第二个也是最昂贵的表达式。

### 条件判断

其实加上`!`更加节省gas，这与我们的想象相反（我们会以为用多一个操作码`!`会消耗更多gas）

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.0;
// https://github.com/ethereum/solidity/issues/13004
// https://twitter.com/paladin_marco/status/1584538632810942464
contract test{
    // function aaa() public pure{
    //     if(!true){
    //         // 140 
    //     }else{

    //     }
    // }
    function aaa() public pure{
        if(true){
            // 153 
        }else{

        } 
    }
}
```

### 尽可能避免0到1的storage写入

初始化存储变量是合约可以执行的最昂贵的操作之一。

当storage变量从0变为非0时，用户必须总共支付 22,100 Gas（0到非0写入为 20,000 Gas，冷存储访问为 2,100 Gas）。

这就是为什么 Openzeppelin 可重入保护寄存器使用 1 和 2 而不是 0 和 1 来确定功能是否有效。将存储变量从非0更改为非0只需花费 5,000 个 Gas。

### 使用的string内容尽可能短

字符串大于等于32字节，会存储到其他地方

### 部署设置payable

让构造函数付费可以在部署时节省 200 个 Gas。这是因为非应付函数中插入了隐式的 `require(msg.value == 0)` 。此外，由于调用数据较小，部署时字节码较少意味着 Gas 成本较低。

有充分的理由使常规函数成为非付费函数，但通常合约是由特权地址部署的，您可以合理地假设该地址不会发送以太币。如果没有经验的用户正在部署合约，这可能不适用。

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.20;

// contract B { // 12440 
//     constructor() payable {}
// }

contract B { // 12464   

}
```

### for循环中+1

```solidity
for (uint256 i; i < limit; ) {
    
    // inside the loop
    
    unchecked {
        ++i;
    }
}
```

### 位移代替乘除2

```
10  *  2 
10  <<  1 # 将10左移1

8  /  4 
8  >>  2 # 将8右移2
```

### 频繁使用的函数应该有最佳的名称

[GitHub repository](https://github.com/jeffreyscholz/solidity-zero-finder-rust).

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.20;

contract FunctionWithLeadingZeros {
    uint256 public totalSupply;
    
    // selector = 0xa0712d68
    function mint(uint256 amount) public {
        totalSupply += amount;
    }
    
    // selector = 0x000071c3 (this cheaper than the above function)
    function mint_184E17(uint256 amount) public {
        totalSupply += amount;
    }
}
```

