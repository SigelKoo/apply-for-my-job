# EVM

EVM用来执行以太坊上的交易

![业务流程](https://img.learnblockchain.cn/2019/15548145070948.jpg)

输入一笔交易，内部会转换成一个Message对象，传入EVM执行。

- 一笔普通转账交易，直接修改`StateDB`中对应的账户余额
- 智能合约的创建或者调用通过EVM中的解释器加载和执行字节码，执行过程中可能会查询或者修改`StateDB`

#### Intrinsic Gas 固定Gas

每笔交易过来，不管三七二十一先需要收取一笔固定油费，计算方法如下：

![交易油费计算](https://img.learnblockchain.cn/2019/15548145839008.jpg)

如果你的交易不带额外数据（Payload），比如普通转账，那么需要收取21000的Gas。

如果你的交易携带额外数据，那么这部分数据也是需要收费的，具体来说是按字节收费：字节为0的收4Gas，字节不为0收68Gas，所以你会看到很多做合约优化的，目的就是减少数据中不为0的字节数量，从而降低油费Gas消耗。

#### 生成Contract对象

交易会被转换成一个Message对象传入EVM，而EVM则会根据Message生成一个Contract对象以便后续执行：

![交易生成对象](https://img.learnblockchain.cn/2019/15548149590538.jpg)

可以看到，**Contract中会根据合约地址，从`StateDB`中加载对应的代码，后面就可以送入解释器执行了**。

另外，执行合约能够消耗的油费有一个上限，就是节点配置的每个区块能够容纳的`GasLimit`。

#### 送入解释器执行

代码跟输入都有了，就可以送入解释器执行了。EVM是基于栈的虚拟机，解释器中需要操作四大组件：

- PC：类似于CPU中的程序计数器，指向当前执行的指令
- Stack：执行堆栈，位宽为256 bits，最大深度为1024
- Memory：内存空间
- Gas：油费池，耗光邮费则交易执行失败

![解释器四大组件](https://img.learnblockchain.cn/2019/15548151682923.jpg)

![解释器执行流程](https://img.learnblockchain.cn/2019/15548151793832.jpg)

EVM的每条指令称为一个OpCode（https://ethervm.io/），占用一个字节，所以指令集最多不超过256。比如下图就是一个示例（PUSH1=0x60, MSTORE=0x52）：

![OpCode指令](https://img.learnblockchain.cn/2019/15548153378078.jpg)

首先程序计数器会从合约代码中读取一个OpCode，然后从一个JumpTable中检索出对应的operation，也就是与其相关联的函数集合。接下来会计算该操作需要消耗的油费，如果油费耗光则执行失败，返回ErrOutOfGas错误。如果油费充足，则调用execute()执行该指令，根据指令类型的不同，会分别对Stack、Memory或者StateDB进行读写操作。

JumpTable，是一个 [256] operation 的数据结构。每个下标对应了一种指令，使用operation来存储了指令对应的处理逻辑，gas消耗，堆栈验证方法，memory使用的大小等功能。

#### 调用合约函数

EVM怎么知道交易想调用的是合约里的哪个函数呢？跟合约代码一起送到解释器里的还有一个Input，而这个Input数据是由交易提供的。

![Input数据](https://img.learnblockchain.cn/2019/15548155517748.jpg)

Input数据通常分为两个部分：

- 前面4个字节被称为“4-byte signature”，是某个函数签名的Keccak哈希值的前4个字节，作为该函数的唯一标识。
- 后面跟的就是调用该函数需要提供的参数了，长度不定。

举个例子：我在部署完A合约后，调用add(1)对应的Input数据是

```
0x87db03b70000000000000000000000000000000000000000000000000000000000000001
```

而在我们编译智能合约的时候，编译器会自动在生成的字节码的最前面增加一段函数选择逻辑：

首先通过`CALLDATALOAD`指令将“4-byte signature”压入堆栈中，然后依次跟该合约中包含的函数进行比对，如果匹配则调用JUMPI指令跳入该段代码继续执行。

这么讲可能有点抽象，我们可以看一看上图中的合约对应的反汇编代码就一目了然了：

![函数signature](https://img.learnblockchain.cn/2019/15548157930961.jpg)

![反汇编代码](https://img.learnblockchain.cn/2019/15548158001820.jpg)

这里提到了`CALLDATALOAD`，就顺便讲一下数据加载相关的指令，一共有4种：

- CALLDATALOAD：把输入数据加载到Stack中
- CALLDATACOPY：把输入数据加载到Memory中
- CODECOPY：把当前合约代码拷贝到Memory中
- EXTCODECOPY：把外部合约代码拷贝到Memory中

最后一个EXTCODECOPY不太常用，一般是为了审计第三方合约的字节码是否符合规范，消耗的Gas一般也比较多。这些指令对应的操作如下图所示：

![指令对应的操作](https://img.learnblockchain.cn/2019/15548158875900.jpg)

#### 合约调用合约

合约内部调用另外一个合约，有4种调用方式：

- CALL
- CALLCODE
- DELEGATECALL
- STATICALL

![CALL调用流程](https://img.learnblockchain.cn/2019/15548159348670.jpg)

可以看到，调用者把调用参数存储在内存中，然后执行CALL指令。

CALL指令执行时会创建新的Contract对象，并以内存中的调用参数作为其Input。

解释器会为新合约的执行创建新的`Stack`和`Memory`，从而不会破环原合约的执行环境。

新合约执行完成后，通过RETURN指令把执行结果写入之前指定的内存地址，然后原合约继续向后执行。

#### 合约的四种调用方式

在中大型的项目中，我们不可能在一个智能合约中实现所有的功能，而且这样也不利于分工合作。一般情况下，我们会把代码按功能划分到不同的库或者合约中，然后提供接口互相调用。

在`Solidity`中，如果只是为了代码复用，我们会把公共代码抽出来，部署到一个library中，后面就可以像调用C库、Java库一样使用了。但是library中不允许定义任何storage类型的变量，这就意味着library不能修改合约的状态。如果需要修改合约状态，我们需要部署一个新的合约，这就涉及到合约调用合约的情况。

合约调用合约有下面4种方式：

调用者——>合约A——>合约B

- CALL：发送者是上一跳接收者，接收者是被调用者，发送者为合约A，接收者为合约B
- CALLCODE：发送者是上一跳接收者，接收者是上一跳接收者，发送者为合约A，接收者为合约A
- DELEGATECALL：发送者是上一跳发送者，接收者是上一跳接收者，发送者为调用者，接收者为合约A
- STATICCALL：发送者是上一跳接收者，接收者是被调用者，在调用合约的过程中发生了转账

###### CALL vs. CALLCODE

CALL和CALLCODE的区别在于：代码执行的上下文环境不同。

具体来说，CALL修改的是**被调用者**的storage，而CALLCODE修改的是**调用者**的storage。

![storage](https://img.learnblockchain.cn/2019/15548164197499.jpg)

###### CALLCODE vs. DELEGATECALL

实际上，可以认为DELEGATECALL是CALLCODE的一个bugfix版本，官方已经不建议使用CALLCODE了。

CALLCODE和DELEGATECALL的区别在于：`msg.sender`不同。

具体来说，DELEGATECALL会一直使用原始调用者的地址，而CALLCODE不会。

![CALLCODE和DELEGATECALL区别](https://img.learnblockchain.cn/2019/15548165586862.jpg)

###### STATICCALL

计划将来在编译器层面把调用view和pure类型的函数编译成STATICCALL指令。



#### 创建合约

前面都是讨论的合约调用，那么创建合约的流程时怎么样的呢？

如果某一笔交易的to地址为nil，则表明该交易是用于创建智能合约的。

首先需要创建合约地址，采用下面的计算公式：`Keccak(RLP(call_addr, nonce))[:12]`。也就是说，对交易发起人的地址和nonce进行RLP编码，再算出Keccak哈希值，取后20个字节作为该合约的地址。

下一步就是根据合约地址创建对应的`stateObject`，然后存储交易中包含的合约代码。该合约的所有状态变化会存储在一个`storage trie`中，最终以`Key-Value`的形式存储到StateDB中。代码一经存储则无法改变，而`storage trie`中的内容则是可以通过调用合约进行修改的，比如通过SSTORE指令。

![生成合约地址](https://img.learnblockchain.cn/2019/15548162475872.jpg)