```go
type Miner struct {
	mux      *event.TypeMux // 事件锁
	worker   *worker // 真正矿工
	coinbase common.Address // 矿工地址
	eth      Backend // Backend对象，Backend是一个自定义接口封装了所有挖矿所需方法
	engine   consensus.Engine // 共识引擎 以太坊有两种共识引擎ethash和clique
	exitCh   chan struct{}
	startCh  chan common.Address
	stopCh   chan struct{}
}
```

miner只是以太坊对外实现mining功能的开放类，真正干活的是worker。所以，继续深入看看worker的结构

```go
type worker struct {
	config      *Config // 配置
	chainConfig *params.ChainConfig // 链的配置属性
	engine      consensus.Engine // 共识引擎
	eth         Backend // Backend对象，Backend是一个自定义接口封装了所有挖矿所需方法
	chain       *core.BlockChain // 链
	// 通道
	newWorkCh          chan *newWorkReq
	taskCh             chan *task
	resultCh           chan *types.Block
	startCh            chan struct{}
	exitCh             chan struct{}
	resubmitIntervalCh chan time.Duration
	resubmitAdjustCh   chan *intervalAdjust
    
	unconfirmed  *unconfirmedBlocks           // 本地挖出的有待确认的区块
}
```

```go
type ChainConfig struct {Jeffrey Wilcke, 6 years ago: • core: added basic chain configuration
	ChainID *big.Int `json:"chainId"` // 标识当前链，主键唯一id 也用来防止replay attack重放攻击
	HomesteadBlock *big.Int `json:"homesteadBlock,omitempty"` // 以太坊发展蓝图中的一个阶段,当前阶段为Homestead
	DAOForkBlock   *big.Int `json:"daoForkBlock,omitempty"`   // TheDao硬分叉切换，2017年6月18日应对DAO危机做出的调整
	DAOForkSupport bool     `json:"daoForkSupport,omitempty"` // 节点是否支持TheDao硬分叉
	EIP150Block *big.Int    `json:"eip150Block,omitempty"` // eth改善方案硬分叉
	// 发展蓝图的出块
    // 第一阶段为以太坊面世代号frontier，第二阶段为Homestead即当前阶段
	// 第三阶段为Metropolis(大都会)，Metropolis又分为Byzantium(拜占庭硬分叉，引入新型零知识证明算法和pos共识)，
	// 然后是constantinople(君士坦丁堡硬分叉，eth正是应用pow和pos混合链)
	// 第四阶段为Serenity(宁静)，最终稳定版的以太坊
	ByzantiumBlock      *big.Int `json:"byzantiumBlock,omitempty"`      // Byzantium switch block (nil = no fork, 0 = already on byzantium)
	ConstantinopleBlock *big.Int `json:"constantinopleBlock,omitempty"` // Constantinople switch block (nil = no fork, 0 = already activated)
	PetersburgBlock     *big.Int `json:"petersburgBlock,omitempty"`     // Petersburg switch block (nil = same as Constantinople)
	IstanbulBlock       *big.Int `json:"istanbulBlock,omitempty"`       // Istanbul switch block (nil = no fork, 0 = already on istanbul)
	MuirGlacierBlock    *big.Int `json:"muirGlacierBlock,omitempty"`    // Eip-2384 (bomb delay) switch block (nil = no fork, 0 = already activated)
	BerlinBlock         *big.Int `json:"berlinBlock,omitempty"`         // Berlin switch block (nil = no fork, 0 = already on berlin)
	LondonBlock         *big.Int `json:"londonBlock,omitempty"`         // London switch block (nil = no fork, 0 = already on london)

	CatalystBlock *big.Int `json:"catalystBlock,omitempty"` // Catalyst switch block (nil = no fork, 0 = already on catalyst)

	// 共识
	Ethash *EthashConfig `json:"ethash,omitempty"`
	Clique *CliqueConfig `json:"clique,omitempty"`
}
```

为实例化的miner对象创建一个worker对象来真正地干活。

```go
为miner创建worker
func newWorker(config *Config, chainConfig *params.ChainConfig, engine consensus.Engine, eth Backend, mux *event.TypeMux, isLocalBlock func(*types.Block) bool, init bool) *worker {
	// NewTxsEvent面熟吧，前面讲交易时 TxPool会发出该事件，当一笔交易被放入到交易池。这时候如果work空闲会把Tx放到work.txs准备下一次打包进块
	// ChainHeadEvent事件，表示已经有一个块作为链头 work.ipdate监听到该事件会继续挖矿。
    // ChainSideEvent事件，表示一个新块作为链的旁支可能会被放入possibleUncles中。
    // 区块链数据库
    // 可能的叔块
    // 挖出的未被确认的区块
}
```

从上面看出， mainLoop() {}方法来处理上述几个event事件。对交易的提交也是在这处理的。

那么新区块的写入是在func (w *worker) resultLoop() {｝。

