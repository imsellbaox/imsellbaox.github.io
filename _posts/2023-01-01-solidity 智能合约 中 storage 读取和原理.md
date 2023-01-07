---
layout: post
title:  Solidity 智能合约 中 storage 读取和原理
categories: [Solidity,区块链安全,智能合约]
excerpt: 如何读取solidity中不同数据类型的storage，利用web3的方式对slot....
---
# meet it
最近在打CTF时，用到了web3的getStorageAt()。其实之前开发时候接入某Dex合约数据时也用到过这个。
# Storage
Storage的实现，是slot以键值对的形式存储，每个slot 32 字节。默认值是0。

* slot只存储变量，不存储常量
* 每个slot最多可存储 256 位（32 bytes）
* 变量依次按顺序在slot中( 入栈规则 )
* 一个slot可以存储多个变量，前提是大小保证在32 bytes 以下。
* 如果变量的大小超过slot的剩余大小，这个变量将储存在新的slot中。
* Struct创建一个新的slot，struct里的变量按顺序放入slot。
* array和struct一样，创建一个新的slot，依次放入。我这里做个扩展（例如：bytes8[4] 那么里面的4个元素都存入同一个slot中）。
* string这样的动态数组创建一个新的slot，这个slot存储数组长度和入口指针，而数组中的值会存储在其他位置，如果string是大于32字节的值，则会被分成·多段存入多个slot中，当然我们只需要关注存入长度和指针的slot就行了。
* Mapping总是创建一个新的slot，这个slot存放keccak256散列后的key，值将存储在其他位置。

# 布局
## 简单常用和一些重要的知识



```solidity
contract MyContract {
uint8 public a = 1; // slot 0
uint16 public b = 8888; // slot 0
uint32 public c; // slot 0
uint64 public d; // slot 0
uint128 public e; // slot 0
uint256 public f; // slot 1
int8 public g; // slot 2
int16 public h; // slot 2
int32 public i; // slot 2
int64 public j; // slot 2
int128 public k; // slot 2
int256 public l; // slot 3
bytes8 public m = "a"; // slot 4
bytes16 public n = "b"; // slot 4
bytes32 public o; // slot 5
bool public p; // slot 6
address public q; //slot 7
uint8 public r; //slot 7
}
```

首先uint和int后面的8，16....代表位，而bytes后面的数字则是代表字节。
32bytes = 256位Binary   （这里不做扩展，不理解去Google）

一个slot的大小是32bytes，所以在存储uint和int时，只要不超过32bytes（256位）就可以同时存储在同一slot中。

bytes同理，所以你可以看到bytes8和bytes16是存储在一个slot中的，但是如果bytes32就超了slot的size，所以单独开辟
bool类型，默认是1 byte，可是它的数据独特性，所以默认用一个slot单独存储。
address类型，size是20 bytes，所以依然可再存入12bytes的数据在同一slot中。

<font color=yellow>再次提醒！字节不是位，搞清uint8和bytes8！</font>

uint，int，bytes，bool由于数据结构简单，所以理解起来也比较容易。



**<font color=red>slot合并数据的布局</font>**

例如上述的a，b两个变量存入同一个slot是什么样的？

```java
slot得到如下数据：0x000000000000000000000000000000000000000000000000000000000022b801
```

a在前先入栈，uint8则为8位2进制， 1 byte， hex是2位，16进制值为1则为 01

b后入栈，uint16为16位8位2进制，2 bytes ， hex是4位，16进制值为 022b
放入同一slot顺序如上述结果。

## struct AND  array

```solidity

contract MyContract {
struct mystruct{
    uint8  a; 
    uint16 b; 
    bool c; 
    uint64 d; 
    uint128 e; 
}
uint8 public a; // slot 0
mystruct public b; //slot 1,2,3
/*
   b{
    uint8  a; // slot 1
    uint16 b; // slot 1
    bool c; // slot 2
    uint64 d; // slot 3
    uint128 e; // slot 3
}
**/
}
```



struct 会开辟一个新slot，来存入内部的变量，如果不够，则会继续存储下一个slot。

```solidity
contract MyContract {

uint8 public a; // slot 0
bytes8[3] public b; //slot 1
bytes32[3] public c; //slot 2,3,4

constructor() public{
    a = 255; // slot 0
    b[0] = "a"; //slot 1
    b[1] = "hello"; //slot 1
    b[2] = "fucc"; //slot 1

    c[0] = "aaaaaaaaaaaaa"; //slot 2
    c[1] = "bbbbbbbbbbbbb"; //slot 3
	c[2] = "ccccccccccccc"; //slot 4
    }
}
```





array的布局一样遵循原则 开辟新slot，存放value，不够则存储到下一个slot

```solidity
我们看一下slot1的数据
'0x00000000000000006675636b0000000068656c6c6f0000006100000000000000'
bytes8 ，所以32字节拆成4份8字节。
0x0000000000000000  // index 3！
0x6675636b00000000  // index 2
0x68656c6c6f000000  // index 1
0x6100000000000000  // index 0
```

遵循入栈，出栈规则，index0 为 61 对应ascill码 ‘a’ 即为b[0]。b[1],b[2] 同理。
这里注意一下，bytes8[3] 的size是3，所以并没有index 3 ，但EVM会用0补全这个32字节的slot。

<font color=bulue>其实struct的slot数据布局也是一样的，发散思维，可以用remix和web3自己在测试网上验证struct的结构猜想。【前置技能：solidity，web3.js，Metamask 】</font>

## String

```solidity
contract MyContract {
string public a = "hello"
string public b = "this is xiaofeng ,you should be like me,do you wanna?"
}
```



string 稍微有点特殊，它其实在solidity中是一个动态的字节数组，它在slot的结构又两种方式：

```solidity
function 1:
sting的size如果小于等于31字节
结构布局为：slot = data(31 bytes) + len(1 byte)

读取结果:
0x68656c6c6f000000000000000000000000000000000000000000000000000a
// 68656c6c6f是hello的hex，而0a 是 string 的 length

function 2:
sting的size如果大于32字节
结构布局为：slot = len(32 byte)
读取结果:
0x0000000000000000000000000000000000000000000000000000000000006a
// 而6a 是 string 的 length
```


所以读取这两种字节数量的string的方法是不一样的。
读取function 1 类型取前面的hex转ascill即可。

读取function 2 的代码我放到下面了。

```solidity
const contract = "0x67A7FC61Ef83dff0B9C15e6d189f724359761c8e";
const length = web3.utils.toNumber(await web3.eth.getStorageAt(contract, 4));
const dataSlot = web3.utils.toBN(
  web3.utils.soliditySha3({ t: "uint256", v: 4 }) // v 是 string的slot索引
);
const data = [];
for (let i = 0; i * 64 < length; i = i + 1) {
  data.push(
    await web3.eth.getStorageAt(contract, dataSlot.add(web3.utils.toBN(i)))
  );
}
```




这种类型，我们要先要用string的slot索引获得keccak256哈希，转成BN（bignumber），这个值 dataSlot 是存储数据的入口索引。接下来就可以用getStorageAt读取data了。

<font color=red>但是注意一个ascill码是2字符（1字节），string的存储空间是连续的，你需要自己定义索引，把整个length的hex全部读出来，再转换成ascill码</font>



## Mapping
### uint,int等常用类型



```solidity
pragma solidity ^0.8.0;

contract tes {

mapping(uint160=>uint256) public myMapping; //slot0
mystruct public a ; //slot1

struct mystruct{
    uint256 a;
}
    constructor() public {
        setValue(1000);
    }
    function setValue(uint256 value) public {
        myMapping[uint160(msg.sender)] =  myMapping[uint160(msg.sender)] + value;
    }

    function getValue(address key) public view returns (uint256) {
        return myMapping[uint160(msg.sender)];

    }
}
```





Mapping的storage布局是比较特殊的，当你试图用getStorageAt()去获取上面的Mapping，你将会得到0x0，和array不一样。其实很好理解，Mapping这种 <font color=red>动态的数据结构</font> 的数据是不断变化和叠加的，如果用array那种形式，显然写数据时，非常不合理。
Mapping在storage中的布局是这样的，当前slot的索引，和要查询的key 进行keccak256哈希散列，目的是保证数据不会被更改，否则无法查询到正确的value。如上面代码中：slot 0 和  uint160(msg.sender) 进行了sha3（keccak256）算法的散列。 

```javascript
let vae = web3.utils.toBN("822828716627677038787365832882249040538522448789");
let vae = web3.utils.toBN("0x9020ea6e39b844d126fe75c5cb342d8a64d57395");
const dataSlot =await web3.utils.soliditySha3({type:"uint256", value:vae}, {type:"uint256", value:0});
await web3.eth.getStorageAt("0x2a336886d67333a329d49AA282eC45E965D53dF2", dataSlot)
```

这个是读取代码，两个vae其实都可以，一个是unit160转过来的，一个是用bytes20转过来的。
在{type:"uint256", value:vae}，也许你会想填uint160，但事实上，EVM对变量的存储默认都是32字节为一组，不足的它会帮你补0，所以真正的类型应该是uint256，或者 bytes32。

### address

```solidity
pragma solidity ^0.8.0;

contract tes {

mapping(address=>uint256) public myMapping; //slot0
mystruct public a ; //slot1

struct mystruct{
    uint256 a;
}

    constructor() public {
        setValue(1000);
    }
    
    function setValue(uint256 value) public {
        myMapping[msg.sender] =  myMapping[msg.sender] + value;
    }

    function getValue(address key) public view returns (uint256) {
        return myMapping[msg.sender];

    }
}
```




address 布局是这样的：slot 0 和  address（bytes32），我之前提到，EVM对变量的存储默认都是32字节为一组。所以针对address 其实EVM存储时会把它补到32字节。即为：

```
0x0000000000000000000000009020ea6e39b844d126fe75c5cb342d8a64d57395
```




那么读取时应该这样：

```javascript
let key = "0x0000000000000000000000009020ea6e39b844d126fe75c5cb342d8a64d57395";
let slot = 0;
const dataSlot =await web3.utils.soliditySha3({type:"bytes32", value:key}, {type:"uint256", value:slot});
await web3.eth.getStorageAt("0x000fBcb4709458a03f45218f6c382654B260b0ed", dataSlot)

```



我提到的 **{type:"bytes32", value:key} **  在这里应该是32位，所以你的address 也应该扩展为32位，就是key。

### String，Mapping嵌套类型

我们来点疯狂的。

```solidity
pragma solidity ^0.8.0;

contract tes {

mapping(address=>string) public myMapping;
mystruct public a ;

struct mystruct{
    uint256 a;
}

    constructor() public {
        setValue("hello");
    }
    
    function setValue(string memory value) public {
        myMapping[msg.sender] =  value;
    }

    function getValue() public view returns (string memory) {
        return myMapping[msg.sender];
    }
}
```



我们知道string的特殊性，31字节和 32字节以上是两种不同的储存方式。我们结合mapping，怎么查出这个string呢？
当然对于string 小于32 bytes的，那么很简单。

```javascript
let key = "0x0000000000000000000000009020ea6e39b844d126fe75c5cb342d8a64d57395";
let dataSlot = web3.utils.toBN(await web3.utils.soliditySha3({type:"bytes32", value:key}, {type:"uint256", value:0}));
await web3.eth.getStorageAt("0x07904EF23C01Ab1C24a4d0C349Aae862354E0440", dataSlot)
```



因为数据出来以后是data+len的形式，所以data很容易获取。

那大于32字节的string，我们把hello换成

```
thismytime!I cant sleep! how can I do,Im so fine,don't worry me I love you forever
```



我们又如何获取呢？

```javascript
let key = "0x0000000000000000000000009020ea6e39b844d126fe75c5cb342d8a64d57395";
let dataSlot = web3.utils.toBN(await web3.utils.soliditySha3({type:"bytes32", value:key}, {type:"uint256", value:0}));
let length = web3.utils.toNumber(await web3.eth.getStorageAt("0x07904EF23C01Ab1C24a4d0C349Aae862354E0440", dataSlot));
let strSlot = web3.utils.toBN(web3.utils.soliditySha3({ t: "uint256", v: dataSlot }));

let data = [];
for (let i = 0; i * 64 < length; i = i + 1) {
  data.push(
    await web3.eth.getStorageAt("0x07904EF23C01Ab1C24a4d0C349Aae862354E0440", strSlot .add(web3.utils.toBN(i)))
  );
```



这里我只给出读取的代码，你应该打开 浏览器console，remix，web3 去自己实操这段代码，感受其中的魅力。然后尝试自己理解它，这比我讲出来更有用。

关于Mapping嵌套，你也一定能自己独立写出来，因为掌握原理，你可以从storage读取如何你想要的数据。

**作者：萧峰  xhf1098516987@gmail.com**