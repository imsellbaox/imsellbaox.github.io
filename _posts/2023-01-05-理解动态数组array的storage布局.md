---
layout: post
title:  理解动态数组array的storage布局
categories: [Solidity,区块链安全,智能合约]
excerpt: 一个理论上依然可以发起的攻击，了解array的布局，你将...
---
## meet it

对于 ```bytes32[] public array;```这种动态数组（包括string）最近又发现了新特性。

* **EVM 存储大小是2²⁵⁶ 个32 字节的slot。**
* **存在int下溢问题**





## Array

```sol
address Owner;
bytes32[4] public array;
```

当我们创建array后，EVM从底部获取足够的空间，创建array数组的保留空间，这里slot空间其实应该到2²⁵⁶-1，但为了直观，我们把最大值设为256，而int溢出原本应该为： ```int a = 0-1 = 2²⁵⁶-1```，改为```int a = 0-1 =256-1```

下面是该array的布局情况。

| slot | variables     | array |
| ---- | ------------- | ----- |
| 0    | owner         |       |
| 1    | array[size=4] |       |
| 2    |               |       |
| ...  |               |       |
| 251  |               |       |
| 252  | array[0]      | 0     |
| 253  | array[1]      | 1     |
| 254  | array[2]      | 2     |
| 255  | array[3]      | 3     |





如果这是一个动态数组，size被我们设置为了256 是什么样的呢？

| slot |    variables    |   array    |
| :--: | :-------------: | :--------: |
|  0   |      owner      | array[254] |
|  1   | array[size=256] | array[255] |
|  2   |    array[0]     |            |
|  3   |     array1]     |            |
|  4   |    array[2]     |            |
| .... |   array[...]    |            |
| 252  |   array[250]    |            |
| 253  |   array[251]    |            |
| 254  |   array[252]    |            |
| 255  |   array[253]    |            |
 当array溢出，它将回到EVM storage的头部，于是，array[254]，array[255] 可以看到是slot 0 和 1

这样我们可以利用这个特性，修改array[254]的值，获得onwer的权限。

## 修复

上述的漏洞存在于solidity 0.5版本以下。对于这个bug 在我用0.8版本时，发现array.length 我们已经不再拥有写权限，即 要么为固定数组，若为动态数组，length的改变只能通过push，pop修改，并且已经做了溢出处理：

````sol
bytes[16] array;
bytes[] array;
array.push(a);
array.pop();
````

从理论上来讲，这个攻击依然可以发动，前提是你拥有一个宇宙级别的资金。

你可以通过push不断叠加，达到2²⁵⁶-1-[Ownerslot+1] 的位置，修改为你的address，不过你也需要耗费数年完成这件事。所以实际上是不可能的。