# EVM

EVM用来执行以太坊上的交易

![在这里插入图片描述](https://img-blog.csdnimg.cn/20181106141126929.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1R1cmtleUNvY2s=,size_16,color_FFFFFF,t_70)

输入一笔交易，内部会转换成一个Message对象，传入EVM执行。

- 一笔普通转账交易，直接修改StateDB中对应的账户余额
- 智能合约的创建或者调用通过EVM中的解释器加载和执行字节码，执行过程中可能会查询或者修改StateDB

### Intrinsic Gas 固定Gas

每笔交易过来，不管三七二十一先需要收取一笔固定油费，计算方法如下：

![交易油费计算](https://img.learnblockchain.cn/2019/15548145839008.jpg)

如果你的交易不带额外数据（Payload），比如普通转账，那么需要收取21000的Gas。

如果你的交易携带额外数据，那么这部分数据也是需要收费的，具体来说是按字节收费：字节为0的收4Gas，字节不为0收68Gas，所以你会看到很多做合约优化的，目的就是减少数据中不为0的字节数量，从而降低油费Gas消耗。

### 生成Contract对象

Transaction对象会被转换成一个Message对象传入EVM，而EVM则会根据Message生成一个Contract对象以便后续执行：

![交易生成对象](https://img.learnblockchain.cn/2019/15548149590538.jpg)

可以看到，**Contract中会根据合约地址，从StateDB中加载对应的代码，后面就可以送入解释器执行了**。

另外，执行合约能够消耗的油费有一个上限，就是节点配置的每个区块能够容纳的GasLimit。

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

首先PC会从合约代码中读取一个OpCode，然后从一个JumpTable中检索出对应的operation，也就是与其相关联的函数集合。接下来会计算该操作需要消耗的油费，如果油费耗光则执行失败，返回ErrOutOfGas错误。如果油费充足，则调用execute()执行该指令，根据指令类型的不同，会分别对Stack、Memory或者StateDB进行读写操作。

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

### 操作

算术操作

```
ADD                     //对栈顶的两个条目进行加法
MUL                     //对栈顶的两个条目进行乘法
SUB                     //对栈顶的两个条目进行减法
DIV                     //整数除法
SDIV                    //带符号的整数除法
MOD                     //模运算
SMOD                    //带符号的模运算
ADDMOD                  //先做加法然后进行模运算
MULMOD                  //先做乘法然后进行模运算
EXP                     //乘方运算
SIGNEXTEND              //符号扩展操作
SHA3                    //对内存中的一段数据进行Keccak-256哈希运算
```

注意，所有算术运算都对2256取了模（除非明确标注为不做此处理），并且0的0次方，即00，会被计算为1。

栈操作

```
POP                  //移除栈顶的一个条目
MLOAD                //从内存中加载一个“字”
MSTORE               //向内存中保存一个“字”
MSTORE8              //向内存中保存一个字节
SLOAD                //从存储中加载一个“字”
SSTORE               //向存储中保存一个“字”
MSIZE                //获得当前已分配内存的字节数大小
PUSHx                //将x字节的一个条目放到栈顶，其中x的数值可以是1到32（一个整“字”）的整数
DUPx                 //复制栈顶的第x个条目到栈顶，其中x的数值可以是1到16的整数
SWAPx                //交换栈顶条目和第x+1个栈内条目，其中x的数值可以是1到16的整数
```

处理流程操作

```
STOP                 //停止执行
JUMP                 //将程序计数器设置为任意数值
JUMPI                //基于条件修改程序计数器的值
PC                   //取得程序计数器的数值（增加这个指令本身的计数之前的数值）
JUMPDEST             //标记一个有效的跳转地址
```

系统操作

```
LOGx                  //增加一条带有x个主题的日志数据，其中x的数值可以是0到4的整数
CREATE                //用关联代码创建一个新账户
CALL                  //向另一个账户发起消息调用，也就是运行另一个账户的代码
CALLCODE              //用另一个账户的代码向当前账户发起消息调用
RETURN                //停止执行并返回输出数据
DELEGATECALL          //用其他账户的代码向当前账户发起消息调用，但sender和value的数值保持不变
STATICCALL            //向一个账户发起静态消息调用
REVERT                //停止执行并撤销状态修改，但保持返回数据和剩余gas
INVALID               //预设的无效指令
SELFDESTRUCT          //停止执行，并将当前账户标记为自毁账户
```

逻辑操作

```
LT                         //小于比较操作
GT                         //大于比较操作
SLT                        //有符号小于比较操作
SGT                        //有符号大于比较操作
EQ                         //等于比较操作
ISZERO                     //简单的非操作
AND                        //按位与操作
OR                         //按位或操作
XOR                        //按位异或操作
NOT                        //按位非操作
BYTE                       //从一个“字”中取得一个字节数据
```

环境操作

```
GAS                    //取得可用gas的数量（减去这个指令的消耗）
ADDRESS                //取得当前账户的地址
BALANCE                //取得指定账户的余额
ORIGIN                 //取得触发这次EVM执行的EOA地址
CALLER                 //取得当前执行的调用者地址
CALLVALUE              //取得当前执行的调用者所发送的以太币数量
CALLDATALOAD           //取得当前执行的输入数据
CALLDATASIZE           //取得当前输入数据的字节大小
CALLDATACOPY           //将当前输入数据复制到内存中
CODESIZE               //当前环境运行的代码的字节大小
CODECOPY               //将当前环境运行的代码复制到内存中
GASPRICE               //取得由初始交易所制定的gas价格
EXTCODESIZE            //取得任意账户代码的字节大小
EXTCODECOPY            //将任意账户的代码复制到内存中
RETURNDATASIZE         //取得在当前环境中的前一次调用的输出数据字节大小
RETURNDATACOPY         //将前一次调用的输出数据复制到内存中
```

区块操作

```
BLOCKHASH               //取得最新的256个完整区块中某个区块的哈希
COINBASE                //取得当前区块的区块奖励受益人地址
TIMESTAMP               //取得当前区块的时间戳
NUMBER                  //取得当前区块的区块号
DIFFICULTY              //取得当前区块的难度
GASLIMIT                //取得当前区块的gas上限
```

# 源码

```
type BlockContext struct {
	账户是否包含足够的用来转账的以太
	将以太从一个帐户转移到另一个帐户

	区块相关信息
}

EVM是以太坊虚拟机基础对象，并提供必要的工具，以使用提供的上下文运行给定状态的合约。
应该指出的是，任何调用产生的任何错误都应该被认为是一种回滚修改状态和消耗所有GAS操作，不应该执行对具体错误的检查。 解释器确保生成的任何错误都被认为是错误的代码。

type EVM struct {
	Context BlockContext
	为EVM提供StateDB相关操作
	当前调用的栈

	链配置信息
	链规则
	虚拟机配置
	解释器
	用于中止EVM调用操作
	当前call可用的gas
}

func NewEVM(ctx Context, statedb StateDB, chainConfig *params.ChainConfig, vmConfig Config) *EVM {}
```

```
数据库中的以太坊智能合约，包括合约代码和调用参数
type Contract struct {
	合约调用者
	JUMPDEST分析的结果

	合约代码
	合约地址
}
func NewContract(caller ContractRef, object ContractRef, value *big.Int, gas uint64) *Contract {}
```

**创建EVM对象的代码：**

```
1. 组装BlockContext
2. 创建EVM对象
func NewEVM(blockCtx BlockContext, txCtx TxContext, statedb StateDB, chainConfig *params.ChainConfig, config Config) *EVM {}
3. 创建解释器
func NewEVMInterpreter(evm *EVM, cfg Config) *EVMInterpreter {}
```

**创建合约**

```go
func (evm *EVM) create(caller ContractRef, codeAndHash *codeAndHash, gas uint64, value *big.Int, address common.Address) ([]byte, common.Address, uint64, error) {
	执行深度检查，如果超出设定的深度限制  创建失败
	账户余额不足，创建失败
	确保指定地址没有已存在的相同合约
	创建合约地址
	创建数据库快照，为了迅速回滚
	在当前状态新建合约账户
	转账操作
	创建合约
	设置合约代码
	执行合约的初始化
	检查初始化生成的代码长度是否超过限制
    合约创建成功
    计算存储代码所需要的Gas，当前拥有的Gas足不足以存储代码
    合约创建失败，借助上面创建的快照快速回滚
}
```

**调用合约**

```go
func (evm *EVM) Call(caller ContractRef, addr common.Address, input []byte, gas uint64, value *big.Int) (ret []byte, leftOverGas uint64, err error) {
}
```

Call方法和create方法的逻辑大体相同，这里分析下他们的不同之处:

- 1.call调用的是一个已经存在合约账户的合约，create是新建一个合约账户。
- 2.call里evm.Transfer发生在合约的发送方和接收方，create里则是创建合约用户的账户和该合约用户之间。

CallCode，它与Call不同的地方在于它使用调用者的EVMContext来执行给定地址的合约代码。

DelegateCall，它与CallCode不同的地方在于它调用者被设置为调用者的调用者。

StaticCall，它不允许执行任何状态的修改。



**解释器**

```go
解释器配置
type Config struct {
	启用调试
	操作码记录器
	禁用解释器调用，代码库调用，委托调用
	操作码opcode对应的操作表
}
用来运行智能合约的字节码
type EVMInterpreter struct {
	evm *EVM
	解释器配置
	最后一个call调用的返回值
}
```

**实现智能合约的执行**

```go
执行合约代码
func (in *EVMInterpreter) Run(contract *Contract, input []byte, readOnly bool) (ret []byte, err error) {
	调用深度递增，evm执行栈的深度不能超过1024
	重置上一个call的返回数据
	合约代码为空
	执行停止时将栈回收为int值缓存池
    解释器主循环，循环运行直到执行显式STOP，RETURN或SELFDESTRUCT，发生错误 {
    	捕获预执行的值进行跟踪
    	从合约的二进制数据i获取第pc个opcode操作符 opcode是以太坊虚拟机指令，一共不超过256个，正好一个byte大小能装下
    	从JumpTable表中查询op对应的操作
    	计算新的内存大小以适应操作，必要时进行扩容
    	计算执行操作所需要的gas
    	执行操作
    	验证int值缓存池
    	将最后一次返回设为操作结果
    }
}
```

**JumpTable(opCode-operation)**

在执行合约的时候涉及到contract.GetOp(pc)方法从合约二进制代码中取出第pc个操作符opcode，然后再按对应关系找到opcode对应的操作operation。这里的对应关系就保存在jump_table中。

这里先要理解操作符opcode的概念，它是EVM的操作符。通俗地讲，一个opcode就是一个byte，solidity合约编译形成的bytecode中，一个byte就代表一个opcode。opcodes.go中定义了所有的操作符，并将所有的操作符按功能分类。

每一个opcode都会对应一个具体的操作operation，一个操作包含其操作函数以及一些必要的参数。

```go
type operation struct {
	execute     executionFunc // 操作函数
	constantGas uint64
	dynamicGas  gasFunc // 操作需要多少gas
	minStack int // 需要最小栈
	maxStack int // 不可超过的栈数量
	memorySize memorySizeFunc // 操作需要的内存大小

	halts   bool // 操作终止
	jumps   bool // 操作跳转
	writes  bool // 是否写入
	reverts bool // 出错回滚
	returns bool // 操作返回
}
```

不同操作对应了不同的operation结构体

**Stack栈**

EVM是基于栈的虚拟机，这里栈的作用是用来保存操作数的。

```go
type Stack struct {
	data []uint256.Int
}

func newstack() *Stack {
	return stackPool.Get().(*Stack)
}

func (st *Stack) push(d *uint256.Int) {
	// NOTE push limit (1024) is checked in baseCheck
	st.data = append(st.data, *d)
}

func (st *Stack) pop() (ret uint256.Int) {
	ret = st.data[len(st.data)-1]
	st.data = st.data[:len(st.data)-1]
	return
}
```

**Memory & stateDB**

Memory类为EVM实现了一个简单的内存模型。它主要在执行合约时针对operation进行一些内存里的参数拷贝。

```
// Memory implements a simple memory model for the ethereum virtual machine.
type Memory struct {
	// 内存
	store       []byte
	// 最后一次的gas花费
	lastGasCost uint64
}

// NewMemory returns a new memory memory model.
func NewMemory() *Memory {
	return &Memory{}
}
```

## 总结

当有这样一段智能合约代码：

```solidity
pragma solidity ^0.4.0;
contract SimpleStorage {
    uint storedData;

    function set(uint x) public {
        storedData = x;
    }

    function get() public returns (uint) {
        return storedData;
    }
}
```

在Remix编译器进行编译后得到字节码：

```
{
	"object": "606060405260a18060106000396000f360606040526000357c01000000000000000000000000000000000000000000000000000000009004806360fe47b11460435780636d4ce63c14605d57603f565b6002565b34600257605b60048080359060200190919050506082565b005b34600257606c60048050506090565b6040518082815260200191505060405180910390f35b806000600050819055505b50565b60006000600050549050609e565b9056",
	"opcodes": "PUSH1 0x60 PUSH1 0x40 MSTORE PUSH1 0xA1 DUP1 PUSH1 0x10 PUSH1 0x0 CODECOPY PUSH1 0x0 RETURN PUSH1 0x60 PUSH1 0x40 MSTORE PUSH1 0x0 CALLDATALOAD PUSH29 0x100000000000000000000000000000000000000000000000000000000 SWAP1 DIV DUP1 PUSH4 0x60FE47B1 EQ PUSH1 0x43 JUMPI DUP1 PUSH4 0x6D4CE63C EQ PUSH1 0x5D JUMPI PUSH1 0x3F JUMP JUMPDEST PUSH1 0x2 JUMP JUMPDEST CALLVALUE PUSH1 0x2 JUMPI PUSH1 0x5B PUSH1 0x4 DUP1 DUP1 CALLDATALOAD SWAP1 PUSH1 0x20 ADD SWAP1 SWAP2 SWAP1 POP POP PUSH1 0x82 JUMP JUMPDEST STOP JUMPDEST CALLVALUE PUSH1 0x2 JUMPI PUSH1 0x6C PUSH1 0x4 DUP1 POP POP PUSH1 0x90 JUMP JUMPDEST PUSH1 0x40 MLOAD DUP1 DUP3 DUP2 MSTORE PUSH1 0x20 ADD SWAP2 POP POP PUSH1 0x40 MLOAD DUP1 SWAP2 SUB SWAP1 RETURN JUMPDEST DUP1 PUSH1 0x0 PUSH1 0x0 POP DUP2 SWAP1 SSTORE POP JUMPDEST POP JUMP JUMPDEST PUSH1 0x0 PUSH1 0x0 PUSH1 0x0 POP SLOAD SWAP1 POP PUSH1 0x9E JUMP JUMPDEST SWAP1 JUMP ",
	"sourceMap": "24:189:0:-;;;;;;;;;",
	"linkReferences": {}
}
```

其中，opcodes字段便是合约代码编译后的操作码集合。

以PUSH1 0x60为例，可以在jump_table.go中找到对应的operation:

```
PUSH1: {
			execute:       makePush(1, 1),
			gasCost:       gasPush,
			validateStack: makeStackFunc(0, 1),
			valid:         true,
		}
```

此时EVM就会去执行makePush函数，同时通过gasPush计算该操作需要的gas费用。EVM内部通过pop不断进行出栈操作来处理整个操作码集，当栈为空的时候表示整个合约代码执行完毕得到最后的执行结果。
