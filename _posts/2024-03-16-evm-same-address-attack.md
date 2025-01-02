---
layout: post
category: blockchain
---

## Theory

To begin with, there is an attack event about this article: [Tornado.Cash DAO attack](https://www.tuoniaox.com/news/p-562673.html). Its attack logic is that:

Using CREATE2 to deploy an contract and returning a address, the rumtimecode will be deployed to EVM. If this contract has a `selfdestruct()`, it can remove the runtimecode, balance, **nonce** and so on in this address. As we know, the theory of using CREATE to create a contract is:

```solidity
keccak256(rlp(senderAddress, nonce))[12:31]
```

Because the senderAddress isn't change and nonce is reset to zero again, we can deploy different bytecode in the same address.

## Example

This is all the contract we need.

```solidity
pragma solidity 0.8.16;

contract A{
	// _bytecode: contract B
    bytes _bytecode = hex"6080604052600a60005534801561001557600080fd5b50610425806100256000396000f3fe608060405234801561001057600080fd5b506004361061004c5760003560e01c80630c55699c1461005157806335f469941461006f57806345c8159014610079578063517cf1ca14610097575b600080fd5b6100596100b5565b6040516100669190610172565b60405180910390f35b6100776100bb565b005b6100816100d5565b60405161008e91906101ce565b60405180910390f35b61009f61010a565b6040516100ac91906101ce565b60405180910390f35b60005481565b600073ffffffffffffffffffffffffffffffffffffffff16ff5b6000806040516100e49061013f565b604051809103906000f080158015610100573d6000803e3d6000fd5b5090508091505090565b6000806040516101199061014c565b604051809103906000f080158015610135573d6000803e3d6000fd5b5090508091505090565b610103806101ea83390190565b610103806102ed83390190565b6000819050919050565b61016c81610159565b82525050565b60006020820190506101876000830184610163565b92915050565b600073ffffffffffffffffffffffffffffffffffffffff82169050919050565b60006101b88261018d565b9050919050565b6101c8816101ad565b82525050565b60006020820190506101e360008301846101bf565b9291505056fe6080604052600b60005534801561001557600080fd5b5060df806100246000396000f3fe6080604052348015600f57600080fd5b506004361060325760003560e01c806335f46994146037578063a56dfe4a14603f575b600080fd5b603d6059565b005b60456073565b604051605091906090565b60405180910390f35b600073ffffffffffffffffffffffffffffffffffffffff16ff5b60005481565b6000819050919050565b608a816079565b82525050565b600060208201905060a360008301846083565b9291505056fea264697066735822122031f4cde3577d87814c482dd53e9673d6b0c8232c6ad9ea609b9c1ed776d8ae3964736f6c634300081000336080604052600c60005534801561001557600080fd5b5060df806100246000396000f3fe6080604052348015600f57600080fd5b506004361060325760003560e01c806335f46994146037578063c5d7802e14603f575b600080fd5b603d6059565b005b60456073565b604051605091906090565b60405180910390f35b600073ffffffffffffffffffffffffffffffffffffffff16ff5b60005481565b6000819050919050565b608a816079565b82525050565b600060208201905060a360008301846083565b9291505056fea2646970667358221220b2a0ebd03886067eb14fc3cac337f335ec2f7423f81bb506a317c0ddb7cb419a64736f6c63430008100033a2646970667358221220cb864211dc25005a67499d35383b700ead9a1148c4d299691a4ba5d7ec9c0a0964736f6c63430008100033";
    
    function deploy(bytes32 _salt) public returns(address){
        bytes memory bytecode = _bytecode;
        address addr;
        assembly {
            addr := create2(0, add(bytecode, 0x20), mload(bytecode), _salt)
        }
        return addr;
    }

    function getHash()public view returns(bytes32){
        return keccak256(_bytecode);
    }
}

contract B{
    uint256 public x = 10;

    function die() public{
        selfdestruct(payable(0));
    }

    function createC_1() public returns(address){
        C_1 c_1 = new C_1();
        return address(c_1);
    }

    function createC_2() public returns(address){
        C_2 c_2 = new C_2();
        return address(c_2);
    }
}

contract C_1{ 
    uint256 public y = 11;

    function die() public{
        selfdestruct(payable(0));
    }
}

contract C_2{
    uint256 public z = 12;

    function die() public{
        selfdestruct(payable(0));
    }
}
```

1. Deploy contract A
2. calculate bytecodehash and deploy the contract B
3. contract B: call `createC_1()` to deploy C_1, now we can get `y` in contract C_1. Attention, contract use CREATE to create a contract:  `keccak256(rlp(contract B, 0))[12:31]`, nonce=0. After that, the nonce turns to 1.
4. contract C_1: call `die()`, and then we can't get `y`. Now there is no code in the contract C_1's address.
5. contract B: call `die()`, now the nonce of contract B is reset to 0 and no code in the contract B's address.
6. use the contract A and the same salt to call `deploy()`, because the salt and senderAddress are the same, `deploy()` returns the same address of B in step 5.
7. contract B: call `createC_2()` to deploy C_2. Because `keccak256(rlp(contract B, 0))[12:31]` is the same as step 3, so CREATE returns the same address of step 3. Now we make it: deploy the same address with different bytecodes by CREATE, CREATE2 and `selfdestruct().`

## Summary

By these seven steps, we understand the advanced technique of CREATE, CREATE2 and `selfdestruct()`. All in all, the main idea is that:

- the same address can be deployed with different bytecode, the code in other words.
- an address can arbitrarily add and remove bytecode.

