PoA为权威证明

PoA共识中出块权掌握在部分签名者手里，而普通人是无法参与的（无论你有多少算力、多少权益）。可见PoA共识牺牲了一部去中心化的特性，换来了一种可控性。

**问题1：如何实现签名者的引进和踢出**

PoA的第一个问题是需要解决签名者更换的问题。在PoA中，签名者必须保证多数情况下在线出块。然而随着项目的不断运行，不可能所有签名者都一直有资源、有意愿继续工作；另外偶尔签名者也会作恶，必须及时将作恶的人踢出。

**问题2：如何控制出块时机**

首先要明确的是，出块时机由两方面决定：一是出块时间；二是由谁出块。在PoA中，签名者之间是合作关系，大家“和和气气”，什么时间出块、由谁出都要按规则来，不能争不能抢。因此需要有良好的规则控制出块时机。

# 设计概要

- **checkpoint**：一个特殊的block，它的高度是EPOCH_LENGTH的整数倍，block中不包含投票信息但包含当时所有的签名者列表

- **SIGNER_COUNT**：某一时刻签名者的数量
- **SIGNER_LIMIT**：连续的块的数量，在这些连续的块中，某一签名者最多只能签一个块；同时也是投票生效的票数的最小值
- **BLOCK_PERIOD**：两个相邻的块的Time字段的最小差值，也是出块周期
- **EPOCH_LENGTH**：两个checkpoint之间的block的数量。达到这个数量后会生成checkpoint以及清除当前所有未生效的投票信息
- **DIFF_INTURN**：出块状态（difficulty）之一，此状态代表“按道理已经轮到我出块”
- **DIFF_NOTURN**：出块状态（difficulty）之一，此状态代表“按道理还没轮到我出块”

问题1解决

- clique中签名者的引进和踢出是通过已有签名者进行投票实现的，并且加入了更加详细的控制。下面我们看一下clique中的投票规则：
  - 投票信息保存在block中。一个block只有一个投票信息，且只能在自己生成的block上保存。
  - 针对某个被投人的票数超过SIGNER_LIMIT时，投票结果立即生效。
  - 投票生效后，立即清除所有被投人是当前生效人的投票信息。如果投的是踢出票，则被投人之前投出的、但还未生效的投票全部失效。
  - 踢出一个签名者以后，可能会导致原来不通过的投票理论上可以通过。clique不特意处理这种情况，等待下次统计时再判断。
  - 发起一个投票后，客户端不会被立即清除投票信息，而是在之后每次出块时都会选一个继续投票。因为区块链中的有效区块有重新调整的可能性，所以不能认为投票生效了之后就会一直生效。
  - 无效的投票：被投票人不在当前签名者列表中但投的是踢出票，或被投票人在当前签名列表中但投的是引进票
  - 为了使编码简单，无效的投票不会受到惩罚（其实我认为有些功能实现也依赖于无效的投票）
  - 在每个EPOCH_LENGTH内，一个签名者给同一个账号地址重复投票时，会先将上次的投票信息清除，然后再统计本次的投票信息（如果本次为无效的投票不会恢复已经清除的上次投票信息）
  - 每个checkpoint不进行投票，而只是包含当前签名者列表信息。对于其它区块，可以用来携带投票信息。

关于上面重复投票的处理需要多说一下，这种处理方式会产生两个结果（假设投票人是A，被投票人是B）：

1. 在当前EPOCH_LENGTH内，A给B只能投一票
2. 在当前EPOCH_LENGTH内，如果给B的投票未生效（总票数未超过SIGNER_LIMIT）时A想把投给B的票撤消，那么A可以投一次跟之前相反的票。因为新的投票会导致旧的投票信息清除，而如果旧的投票是有效的则新的投票必定是无效的，因而也不会进入投票统计。

问题2解决

- 前面我们说过，出块时机由两方面决定：一是出块时间；二是由谁出块。下面我们看看clique是如何解决这些问题的。

  - 出块时间
    在clique中，出块时间是固定的，由BLOCK_PERIOD决定。

  - 由谁出块

    clique中出块权的确定稍微复杂，具体规则为：

    - 签名者在签名者列表中且在SIGNER_LIMIT内没出过块
    - 如果签名者是DIFF_INTURN状态，则拥有较高出块权（等待出块时间到来，签名区块并立即广播出去）
    - 如果签名者是DIFF_NOTURN状态，则拥有较低出块权（等待出块时间到来，再延迟一下（延迟时间为rand(SIGNER_COUNT * 500ms)）

可见出块权由两方面确定：一是最近是否出过块，如果出过则没有出块权；二是DIFF_INTURN / DIFF_NOTURN状态，DIFF_INTURN拥有较高出块权。



Header.Coinbase：在clique中用来保存投票时被投票人的地址。而出块者的地址通过签名数据计算得出（ecrecover）。

Header.Nonce：在clique中不需要根据Nonce调整header的哈希，因此直接用来保存投票目的。如果值为nonceAuthVote(0xffffffffffffffff)则代表这是一次授权投票；如果值为nonceDropVote(0x0000000000000000)则代表这是一次踢出投票。

Header.Extra：在clique中Extra除了依然保存vanity数据和Seal数据，还在checkpoint中增加了所有签名者地址数据。其结构为：
vanityData(固定32字节)+signer1Address+signer2Address+…+SealData(固定65字节)

在clique中，有一个值叫做”epoch”。当一个block的高度恰好是”epoch”值的整数倍时，这个block便不会包含任何投票信息，而是包含了当前所有的签名者列表。这个block被叫做checkpoint。可以看出，checkpoint类似于一个“里程碑”，可以用来表示“到目前为止，有效的签名者都记录在我这里了”；而epoch就是设立里程碑的距离。

简单来说，”epoch”的存在，是为了避免没有尽头的投票窗口，也是为了周期性的清除除旧的投票提案。更进一步地，在checkpoint中存在的签名者列表，可以让节点间基于中间某个checkpoint就可以同步到签名者列表，而不需要整个链上的数据。

```go
func (c *Clique) Prepare(chain consensus.ChainReader, header *types.Header) error {
  //others code
  ......

  // number为将要生成的块的高度

  //如果number不是epoch的整数倍（不是checkpoint），则进行投票信息的填充
  if number%c.config.Epoch != 0 {
    ......

    //填写投票信息（投票信息存储在Coinbase和Nonce字段中）
    if len(addresses) > 0 {
      header.Coinbase = addresses[rand.Intn(len(addresses))]
      if c.proposals[header.Coinbase] {
        copy(header.Nonce[:], nonceAuthVote)
      } else {
        copy(header.Nonce[:], nonceDropVote)
      }
    }
  }

  ......

  //如果number是epoch的整数倍（将要生成一个checkpoint），则填充签名者列表
  if number%c.config.Epoch == 0 {
    for _, signer := range snap.signers() {
      header.Extra = append(header.Extra, signer[:]...)
    }
  }
}
```

在`Snapshot.apply`方法中的代码，则体现了“避免没有尽头的投票窗口，周期性的清除除旧的投票提案”的功能：

```go
func (s *Snapshot) apply(headers []*types.Header) (*Snapshot, error) {
  ......

  for _, header := range headers {
    // Remove any votes on checkpoint blocks
    number := header.Number.Uint64()
    //如果当前header的高度是epoch的整数倍（当前block是一个checkpoint），则清除所有投票信息和统计
    //Votes和Tally字段的详细信息参看对Snapshot的介绍。
    if number%s.config.Epoch == 0 {
      snap.Votes = nil
      snap.Tally = make(map[common.Address]Tally)
    }

    ......
  }
}
```

这里稍微说明一下为什么这一点代码实现了epoch的设计目的。一个Snapshot对象中保存了在某个高度上的投票信息，而创建一个Snapshot对象其实是分两步的，apply方法属于第二步。假如给apply传入10个header，其中第8个的高度恰好是epoch的整数倍，那么前面那7个的投票信息就被抹去了。

除了epoch，代码中还有一个值为checkpointInterval。当在某个block上创建一个Snapshot对象，且这个block的高度是checkpointInterval的整数倍时，则将这个Snapshot对象写入到数据库中永久存储；以后想到再次在这个block上生成Snapshot对象时，直接从数据库中读取就可以了。摘录代码如下：

```go
func (c *Clique) snapshot(chain consensus.ChainReader, number uint64, hash common.Hash, parents []*types.Header) (*Snapshot, error) {
  ......

  for snap == nil {
    ......

    //如果高度为checkpointInterval的整数倍，则直接尝试从数据库中读取Snapshot对象
    if number%checkpointInterval == 0 {
      if s, err := loadSnapshot(c.config, c.signatures, c.db, hash); err == nil {
        snap = s
        break
      }
    }

    ......
  }

  ......

  //如果高度为checkpointInterval的整数倍，则将Snapshot对象存储到数据库中
  if snap.Number%checkpointInterval == 0 && len(headers) > 0 {
    if err = snap.store(c.db); err != nil {
      return nil, err
    }
  }
  return snap, err
}
```

##### Snapshot

Snapshot对象是clique中比较重要的一个对象，它的作用是统计并保存链的某段高度区间的投票信息和签名者列表。这个统计区间是从某个checkpoint开始（包括genesis block），到某个更高高度的block。在Snapshot对象中用到了两个重要的结构体：Vote和Tally，我们先对它们进行一下说明，再来详细说一下Snapshot结构体。

Vote struct

Vote代表的是一次投票的详细信息，包括谁给谁投的票、投的加入票还是踢出票等等。它的结构体定义如下：

```go
type Vote struct {
  Signer    common.Address // 此次投票是由谁投的
  Block     uint64         // 此次投票是在哪个高度的block上投的
  Address   common.Address // 此次投票是投给谁的
  Authorize bool           // 这是一个加入票（申请被投人成为签名者）还是踢出票（申请将被投人踢出签名者列表）
}
```

Tally struct

Tally结构体是对所有被投人的投票结果统计。注意它与Vote结构体的区别：Vote是投票过程的记录（如A给B投了一个授权票），而Tally是对结果的统计（类似于选班长唱票时计票员在黑板上画的“正”字）。Tally的定义如下：

```go
type Tally struct {
  Authorize bool // 这是加入票的统计还是踢出票的统计
  Votes     int  // 目前为止累计的票数
}
```

如果只看这里你可能会意外这里并没有“针对谁进行的统计”的信息，这是因为Tally在Snapshot结构体是是作为map的一部分的，参看下面对Snapshot结构体字段的说明。

Snapshot struct

```go
type Snapshot struct {
  config   *params.CliqueConfig
  sigcache *lru.ARCCache        

  Number  uint64                      // Block number where the snapshot was created
  Hash    common.Hash                 // Block hash where the snapshot was created
  Signers map[common.Address]struct{} // 当前的所有有效的签名者
  Recents map[uint64]common.Address   // 最近生成过block的签名者。其中map的key是生成的block的高度
  Votes   []*Vote                     // 按时间先后顺序保存的投票信息
  Tally   map[common.Address]Tally    // 目前为止累计各被投人的票数
}
```

除了Votes和Tally，比较重要的字段就是Signers和Recents了。Signers字段比较好理解，就是当前可以出块的所有的签名者。各个节点根据这个字段的信息来判断某个块的签名者是否真的有权出块。注意这个字段是可以不断动态变化的，当某个账号地址的累计投票数超过一半时，就会被加入到Signers中（或从Signers中移除），实现代码摘录如下：

```go
func (s *Snapshot) apply(headers []*types.Header) (*Snapshot, error) {
  for _, header := range headers {
    ......

    //如果累计投票数超过一半
    if tally := snap.Tally[header.Coinbase]; tally.Votes > len(snap.Signers)/2 {
      if tally.Authorize {
        //如果是加入票，则将被投人加入到Signers列表中
        snap.Signers[header.Coinbase] = struct{}{}
      } else {
        //如果是踢出票，则把被投人从Signers中删除
        delete(snap.Signers, header.Coinbase)

        //将被投人从Recents信息中删除
        if limit := uint64(len(snap.Signers)/2 + 1); number >= limit {
          delete(snap.Recents, number-limit)
        }

        //丢弃被投人之前投出的所有投票
        for i := 0; i < len(snap.Votes); i++ {
          if snap.Votes[i].Signer == header.Coinbase {
            snap.uncast(snap.Votes[i].Address, snap.Votes[i].Authorize)
            snap.Votes = append(snap.Votes[:i], snap.Votes[i+1:]...)
            i--
          }
        }
      }

      //清除所有被投人是当前生效人的投票信息
      for i := 0; i < len(snap.Votes); i++ {
        if snap.Votes[i].Address == header.Coinbase {
          snap.Votes = append(snap.Votes[:i], snap.Votes[i+1:]...)
          i--
        }
      }
      delete(snap.Tally, header.Coinbase)  //清空当前生效人的票数统计信息
    }
  }

  ......
}
```

`Snapshot.Recents`字段保存了最近出块的签名者和所出的块的高度。这个“最近”的定义是最新的`len(Snapshot.Signers)/2 + 1`个块。我们先看一下代码是如何操作这个字段的：

```go
func (s *Snapshot) apply(headers []*types.Header) (*Snapshot, error) {
  for _, header := range headers {
    ......

    //将高度与当前块相差limit的块从Recents中删除掉
    if limit := uint64(len(snap.Signers)/2 + 1); number >= limit {
      delete(snap.Recents, number-limit)
    }
    ......

    //将当前块的高度和签名者加入Recents中
    snap.Recents[number] = signer
  }
}
```

比如目前有6个签名者，当前块的高度是10，那么高度为”10 - (6/2 + 1) = 6”的块将从Recents中删除，然后将高度为10的块和其签名者加入Recents中；处理一下个块即高度为11时，高度为7的块又会从Recents中删除，然后高度为11的块会被加入。总之，这个”最近”的基点是当前块的高度，囊括的范围为`len(Snapshot.Signers)/2 + 1`。下面我们看看在出块时Recents字段是如何起作用的：

```go
func (c *Clique) Seal(chain consensus.ChainReader, block *types.Block, results chan<- *types.Block, stop <-chan struct{}) error {
  ......

  for seen, recent := range snap.Recents {
    if recent == signer {
      // Signer is among recents, only wait if the current block doesn't shift it out
      if limit := uint64(len(snap.Signers)/2 + 1); number < limit || seen > number-limit {
        log.Info("Signed recently, must wait for others")
        return nil
      }
    }
  }

  ......
}
```

最后，我们要说一下Snapshot的创建过程。前面其实提到过，Snapshot的创建分两步：一是在某个checkpoint(包括genesis block)上调用`newSnapshot`生成一个Snapshot对象，然后调用这个对象的apply方法。这两步被封装在了`Clique.snapshot`方法中，即`Clique.snapshot`才是正确生成Snapshot对象的方法。因此我们来详细看一下Clique的snapshot方法。下面是摘录的代码，为了更清楚的表达逻辑，对代码进行了简化，并加了一些伪代码：

```go
func (c *Clique) snapshot(chain consensus.ChainReader, number uint64, hash common.Hash, parents []*types.Header) (*Snapshot, error) {
  headers []*types.Header

  for snap == nil {
    //首先从缓存或数据库中查找
    if snapshot in cache or database {
      get and break
    }

    //既不在缓存中也不在数据库中，那么看是否是创世块或没有父块的checkpoint。
    //关于为什么要判断没有父块的checkpoint，后面有详细说明
    if number == 0 || (number%c.config.Epoch == 0 && chain.GetHeaderByNumber(number-1) == nil) {
      checkpoint := chain.GetHeaderByNumber(number)

      //从checkpoint中取出signers列表
      signers := make([]common.Address, (len(checkpoint.Extra)-extraVanity-extraSeal)/common.AddressLength)
      for i := 0; i < len(signers); i++ {
        copy(signers[i][:], checkpoint.Extra[extraVanity+i*common.AddressLength:])
      }

      //调用newSnapshot在checkpoint上创建Snapshot对象，并将其存入数据库中
      snap = newSnapshot(c.config, c.signatures, number, checkpoint.Hash(), signers)
      snap.store(c.db)
      break
    }

    //如果以上情况都不是，则往前回溯区块的链，并保存回溯过程中遇到的header
    header := get parent by number and hash
    headers = append(headers, header)
    number, hash = number-1, header.ParentHash
  }

  //把headers中保存的头从按高度从小到大排序。
  reverse(headers)

  //将回溯中遇到的headers传给apply方法，得到一个新的snap对象
  snap, err := snap.apply(headers)

  //保存snap对象
  store snap to cache
  if should store to database {
    snap.store(c.db)
  }
  return snap, err
}
```

从上面简化的代码中可以看到，想要创建一个Snapshot对象，需要从给定的block开始向前回溯，直到从缓存或数据库中找到对应的Snapshot，或者满足`number == 0 || (number%c.config.Epoch == 0 && chain.GetHeaderByNumber(number-1) == nil)`这个奇怪的条件。然后将回溯过程中遇到的header传给apply。在apply内部会根据传入的headers逐个统计投票等信息。

关于上面提到的这个奇怪的判断条件的分析，参看[奇怪的判断条件](http://yangzhe.me/2019/02/01/ethereum-clique/#odd_if)这一小节里的说明。

##### inturn and noturn

前面说过，clique作为PoA的实现，挖矿的人之间是合作关系，因此需要有规则规定某一时刻应该由谁出块。在clique中，`inturn`状态代表的是“按道理轮到我出块了”，而`noturn`正好相反。

在代码中，inturn的值为`diffInTurn`，noturn的值为`diffNoTurn`。Header.Difficulty字段用来保存相应的值，它的计算方式非常简单，具体可以查看Snapshot.inturn方法，这里不再多说。

在`Clique.Seal`方法中，签名时会进行一定时间的等待。如果Header.Difficulty的值为`diffNoTurn`，则会比`diffInTurn`的块随机多等待一些时间，通过这种方式可以保证轮到出块的人可以优先出块。代码如下：

```
func (c *Clique) Seal(chain consensus.ChainReader, block *types.Block, results chan<- *types.Block, stop <-chan struct{}) error {
  ......

  //计算正常的等待出块时间
  delay := time.Unix(header.Time.Int64(), 0).Sub(time.Now()) // nolint: gosimple
  if header.Difficulty.Cmp(diffNoTurn) == 0 {
    //没有轮到我们出块，多等一会
    // It's not our turn explicitly to sign, delay it a bit
    wiggle := time.Duration(len(snap.Signers)/2+1) * wiggleTime
    delay += time.Duration(rand.Int63n(int64(wiggle)))

    log.Trace("Out-of-turn signing requested", "wiggle", common.PrettyDuration(wiggle))
  }

  ......
}
```







### 出块流程

在理解了前面提到的一些概念以后，我觉得再加上一个整体的视图，clique模块就非常容易理解了，因此这里我们从一个更高的视角来看一下clique的工作流程。

Clique对象包含两大块功能：Header的有效性验证和生成新的Header。有效性验证的方法全都是以”Verify”开头，比较容易理解，就不多说了。这里主要介绍一下生成新的block的流程。我将关键信息整理成了一张流程图。由于Clique对象只是实现了以太坊中共识接口的方法，主要流程还是由miner模块控制，因此这里也加入了简单的miner模块的流程。

![img](http://yangzhe.me/pic/ethereum-clique/flowchart.png)

# 关键问题

前面的分析已经涉及了几乎clique模块的所有重点，但为了清晰，这里把一些需要重点关注的关键问题单独拿出来，再针对性的进行一次说明。

### 如何控制出块时机

所有的共识机制都需要有一套控制出块时机的方法，因为不可能在同一时间让所有人都出块。clique是通过两个方面对出块时机进行控制的：

1. recent列表
2. inturn或noturn

在clique中，不允许最近出过块的人再出块，必须间隔一定的高度，这是通过`Snapshot.Recents`字段控制的。在`Clique.Seal`方法中有如下一段代码：

```go
  for seen, recent := range snap.Recents {
    if recent == signer {
      // Signer is among recents, only wait if the current block doesn't shift it out
      if limit := uint64(len(snap.Signers)/2 + 1); number < limit || seen > number-limit {
        log.Info("Signed recently, must wait for others")
        return nil
      }
    }
  }
```

代码中`snap.Recents`保存的就是最近出的块的高度和块的签名者（signer），`seen`变量是块的高度，`recent`是这个块的签名者。这段代码表达的意思是，如果当前的签名者刚出过块（在Recents中），并且这个历史块的高度离新出块的高度相差在所有签名者数量的一半（严格来说是一半加1）以内，则不允许再出块。

比如目前共有7个签名者，将要新出的块高度为100，而我刚出过一个块的高度是97，那么这个高度为100的块我就不能再出了。（97 > 100 - (7/2 + 1)）

那么snap.Recents又是怎么来的呢？它是在`Snapshot.apply`中生成的：

```go
  // Delete the oldest signer from the recent list to allow it signing again
  if limit := uint64(len(snap.Signers)/2 + 1); number >= limit {
    delete(snap.Recents, number-limit)
  }
  // Resolve the authorization key and check against signers
  signer, err := ecrecover(header, s.sigcache)

  snap.Recents[number] = signer
```

可以看到在填充Recents时，会将高度比当前块小太多的块从Recents中踢出去（number - limit）。

从上面的分析中我们可以看到，在调用Seal时，其实是有大约一半数量的签名者都是可以出块的。那这一半的签名者都要出块吗？当然不是的。理想情况下，每一个时刻最好只有一个人出块。在clique中是使用inturn/noturn的机制实现的。

在前面的概念介绍时我们已经介绍过inturn概念，简单来说就是“当前是不是轮到我出块了”，判断方法是看当前块的高度是否和自己在签名者列表中的顺序一致。`Snapshot.inturn`方法实现了这一功能：

```go
func (s *Snapshot) inturn(number uint64, signer common.Address) bool {
  signers, offset := s.signers(), 0
  for offset < len(signers) && signers[offset] != signer {
    offset++
  }
  return (number % uint64(len(signers))) == uint64(offset)
}
```

在`Clique.Prepare`中会将“是否轮到自己”的值写到Header.Difficulty字段中：

```go
func (c *Clique) Prepare(chain consensus.ChainReader, header *types.Header) error {
  ......
  header.Difficulty = CalcDifficulty(snap, c.signer)
  ......
}

func CalcDifficulty(snap *Snapshot, signer common.Address) *big.Int {
  if snap.inturn(snap.Number+1, signer) {
    return new(big.Int).Set(diffInTurn)
  }
  return new(big.Int).Set(diffNoTurn)
}
```

然后在`Clique.Seal`中，会根据Header.Difficulty中的值判断“是否轮到自己”出块。如果不是，则要随机多等待点时间，这就给了该出块的人优先的出块权：

```go
func (c *Clique) Seal(chain consensus.ChainReader, block *types.Block, results chan<- *types.Block, stop <-chan struct{}) error {
  ......
  delay := time.Unix(header.Time.Int64(), 0).Sub(time.Now())
  if header.Difficulty.Cmp(diffNoTurn) == 0 {
    // It's not our turn explicitly to sign, delay it a bit
    wiggle := time.Duration(len(snap.Signers)/2+1) * wiggleTime
    delay += time.Duration(rand.Int63n(int64(wiggle)))
  }
}
```

所以，clique通过recent和inturn两种方式控制出块时机，最终给了该出块的人优先的出块权。

### 如何动态调整签名者列表

前面我们也说过，在项目运行过程中，签名者可能会发生变化，需要一种机制能动态的调整签名者名单。在clique中，这种调整是通过投票实现的。即对是否要加入或踢除某人，现有签名者可以发起投票，当投票超过半数时投票通过，被投人被自动加入或踢出。下面我们来看看这种投票机制是怎么实现的。

我们先来看看如何发起投票。你可以在console中调用`clique.propose`来进行一次投票，比如：

> clique.propose(“0x8D5cC3e43CE479d81c7e1e4a6DebC8D6c126a9eF”, true)

调用此方法以后，投票信息就会写入`Clique.proposals`字段中。在随后出块调用`Clique.Prepare`方法时，会从`Clique.proposals`中随机选择一条投票信息，放到Header.Coinbase和Header.Nonce中：

```go
func (c *Clique) Prepare(chain consensus.ChainReader, header *types.Header) error {
  ......

  //非checkpoint才可以携带投票信息
  if number%c.config.Epoch != 0 {
    //从c.proposals中收集所有有意义的被投地址
    addresses := make([]common.Address, 0, len(c.proposals))
    for address, authorize := range c.proposals {
      if snap.validVote(address, authorize) {
        addresses = append(addresses, address)
      }
    }

    //如果确实有有意义的被投地址，随机选一个填入Header中
    if len(addresses) > 0 {
      header.Coinbase = addresses[rand.Intn(len(addresses))]
      if c.proposals[header.Coinbase] {
        copy(header.Nonce[:], nonceAuthVote)
      } else {
        copy(header.Nonce[:], nonceDropVote)
      }
    }
  }

  ......
}
```

现在票已经投出去了，我们再来看看如何统计投票信息，以及投票通过后如何生效。这些功能都是在`Snapshot.apply`方法中实现的：

```go
func (s *Snapshot) apply(headers []*types.Header) (*Snapshot, error) {
  for _, header := range headers {
    number := header.Number.Uint64()

    //如果达到一个epoch，则清空当前所有的投票信息
    if number%s.config.Epoch == 0 {
      snap.Votes = nil
      snap.Tally = make(map[common.Address]Tally)
    }

    //如果当前的投票人已经给某人投过票，则清除之前的投票信息
    //这保证了一个epoch内一个signer只能给某人投一次票，或用来撤消投票
    for i, vote := range snap.Votes {
      if vote.Signer == signer && vote.Address == header.Coinbase {
        snap.uncast(vote.Address, vote.Authorize)
        snap.Votes = append(snap.Votes[:i], snap.Votes[i+1:]...)
        break
      }
    }

    //将本次的投票信息纳入统计
    var authorize bool
    switch {
      case bytes.Equal(header.Nonce[:], nonceAuthVote):
        authorize = true
      case bytes.Equal(header.Nonce[:], nonceDropVote):
        authorize = false
      default:
        return nil, errInvalidVote
    }
    if snap.cast(header.Coinbase, authorize) {
      snap.Votes = append(snap.Votes, &Vote{
        Signer:    signer,
        Block:     number,
        Address:   header.Coinbase,
        Authorize: authorize,
        })
    }

    //如果投票数量超过一半，则投票通过。
    if tally := snap.Tally[header.Coinbase]; tally.Votes > len(snap.Signers)/2 {
      //如果是加入票，则将被投人加入到Signers列表中
      //之后此人就可以出块了
      if tally.Authorize {
        snap.Signers[header.Coinbase] = struct{}{}
      } else {
        //如果是踢出票，则将被投人从Signers列表中踢出
        //之后这人就无法出块了
        delete(snap.Signers, header.Coinbase)

        //将被投人从Recents信息中删除
        if limit := uint64(len(snap.Signers)/2 + 1); number >= limit {
          delete(snap.Recents, number-limit)
        }

        //丢弃被投人之前投出的所有投票
        for i := 0; i < len(snap.Votes); i++ {
          if snap.Votes[i].Signer == header.Coinbase {
            snap.uncast(snap.Votes[i].Address, snap.Votes[i].Authorize)
            snap.Votes = append(snap.Votes[:i], snap.Votes[i+1:]...)
            i--
          }
        }
      }

      //清除所有被投人是当前生效人的投票信息
      for i := 0; i < len(snap.Votes); i++ {
        if snap.Votes[i].Address == header.Coinbase {
          snap.Votes = append(snap.Votes[:i], snap.Votes[i+1:]...)
          i--
        }
      }
      delete(snap.Tally, header.Coinbase) //清空当前生效人的票数统计信息
    }
  }
}
```

这段代码比较复杂，但通过代码中的中文注释，相信可以很容易弄明白其基本逻辑：当我们逐个遍历链上的block时，我们记录每个block上的投票信息，当投票总数达到半数以上时，对被投人作相应的操作——加入或踢除（踢除包含了更多操作）；如果遇到epoch，则之前统计的所有投票信息作废，全部清空（这在epoch概念中有详细讨论）。

### 作恶的代价和处理

PoA机制无法保证所有签名者都不作恶。那么万一有人作恶，会造成什么影响呢？

相信经过前面不断的介绍，读者已经能理解clique的基本原理。签名者在clique中无法连续出块（Recent机制限制），因此如果有人作恶，它最多也只能每隔“出块人数量一半”的高度出一个块，这可以保证多数块是正确的，这一点与PoW类似（只要作恶者的能力（对于PoA来说是作恶的人的数量，对于PoW来说是算力）不超过一半，就可以保证多数块是正常的）。

clique有投票机制，如果及时发现某人作恶，其他签名者可以快速反应，将其强制踢出签名者列表。所以总得来说，clique的实现可以保证少量人作恶的情况下只能造成很小的影响，并且可以及时强制制止。



## 基于可验证随机函数的权威证明共识算法改进

PoA算法通过可信身份授权来提供快速的交易，因为其高性能和对故障的高容错能力而被人关注

De Angelis 等分析了 PBFT 和 PoA 在联盟链上 CAP 理论的分析[10]，发现在联盟链中的使用 PoA 的确比 PBFT 有更好的性能， 同时分析了两种主要的 PoA 算法，即 Aura [11]和 Clique [12]，最终发现 PoA 无法为数据完整性提供足够 的一致性保证。

Ekparinya 等[13]人发现在 PoA 共识中存在克隆攻击(The Cloning Attack)，即 可以通过克隆一个权威节点来实现对 PoA 算法的攻击，致使这种攻击的原因在于，在 PoA 中的领导节点 取余法易暴露领导节点的选取顺序，导致下一轮权威节点被选取的过程在网络中是完全公开的，权威节 点的公开可能导致安全性和第三方操纵，从而容易遭受被拒绝服务攻击(Distributed Denial-of-service attacks, DDos)攻击和审查攻击(Censorship attacks)，即攻击者可以通过恶意攻击导致权威节点的不诚实行为。

共识算法的基本流程[14]如下： 1) 选举出块者：出块者是指区块链中负责产生区块的节点，又称领导者。 2) 生成区块：出块者主要完成生成区块的工作，即将一段时间内网络中产生的交易数据打包放到当 前区块中。一个领导者在其“任职”期间，能够生成多个区块，一般将一个领导者的任职时间称为一个 时期(epoch)，每个时期由多个轮(round)组成，每一轮生成一个区块。 3) 节点验证更新区块链：出块者生成区块后，将区块在网络中广播，收到区块的节点验证区块正确 性并更新本地区块链。节点可能还需要验证区块中交易合法性和出块者身份合法性等。

可验证随机函数是一种基于公私钥的密码学哈希函数，只有 VRF 私钥的持有者才能计算哈希，但是具有相应公钥的任何人都可以验证哈希的正确性。在此应用中，证明人 Prover 持有 VRF 私钥并使用 VRF 哈希在输入数据上构建基于哈希的数据结构，由于 VRF 的性质，只有 Prover 才能给出有关哈希正确性的证明，知道其 VRF 公钥的 任何人都可以验证 Prover 的证明是否正确，却不能对存储在数据结构中的数据做出推断。可验证随机函 数可以基于私钥对一个输入，产生一个唯一的固定长度的输出，以及一个对应的证明。其他人通过公钥、 输出、证明之后就一定能验证这三者的正确性，并且也只有在知道这三者之后才能验证其正确性。

VRF 附带一个密钥生成算法，可以生成公钥 PK 和私钥 SK。Prover 通过其私钥对某个输入 α 进行哈 希处理可得到随机输出 β

Prover 还使用私钥来构造零知识证明 π，用来证明 β 是正确输出，

任何人都可直接通过 π 获得 β

零知识证明 π 允许 Verifier 持有公钥 PK 以验证 β 是否基于 PK 下的 α 的正确输出 β。因此，VRF 还 附带了算法，首先计算公式(3)，若计算出的 β 等于 PK 下的正确 β，输出 VALID，否则输出 INVALID：

权威证明共识算法依赖于 N 个经过身份认证过的可信节点，也称权威节点(Authorities)，然后在每一 轮中选举出一个或多个权威节点担任该次共识过程中的领导者(Leader)，负责提出新的区块，当此次提议 被至少 N 2 1+ 个验证节点(Validators)确认后，该打包区块被最终确定下来，从而实现整个系统的分布式 共识，此算法几乎不需要任何算力去竞争，因此可以通过减少每个区块的间隔时间和处理更多的交易。 PoA 算法中的共识采用轮换模式，通过取余法在权威节点之间公平地选取领导节点。