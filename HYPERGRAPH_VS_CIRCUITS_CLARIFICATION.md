# 超图 vs 布尔电路：关系澄清
# Hypergraph vs Boolean Circuits: Relationship Clarification

## 核心问题

**问题**: 工作节点最终不可能既计算超图逻辑又计算布尔电路逻辑吧？超图和布尔电路（基于OT的）的关系是什么？

**简短回答**: 
- ✅ 工作节点**确实有两种职责**，但它们**完全分离**
- ✅ 超图和布尔电路**不是编译关系**
- ✅ 它们是**两个独立的系统层**，服务于不同目的

---

## 1. 两个独立的系统层

```
┌─────────────────────────────────────────────────────────────┐
│                    Quilibrium系统分层                         │
└─────────────────────────────────────────────────────────────┘

Layer 4: 应用计算层 (Application Computation)
┌──────────────────────────────────────────────────────────┐
│ 布尔电路 (Boolean Circuits)                               │
│ ─────────────────────────────────────────────────────── │
│ • 用户定义的计算逻辑                                       │
│ • 排序、比较、加密算法等                                   │
│ • 通过乱码电路+OT执行                                      │
│ • P2P直接通信 (Alice ↔ Bob)                              │
│                                                          │
│ 执行位置: 计算参与者的内存中 (OFF-CHAIN)                  │
│ 数据流: Alice ↔ P2P Channel ↔ Bob                        │
└──────────────────────────────────────────────────────────┘
                          ↑
                          │ 读取代码
                          │ (GetVertexData)
                          ↓
Layer 3: 状态管理层 (State Management)
┌──────────────────────────────────────────────────────────┐
│ 超图 (Hypergraph CRDT)                                   │
│ ─────────────────────────────────────────────────────── │
│ • 存储和检索数据                                          │
│ • AddVertex, RemoveVertex, GetVertex                    │
│ • CRDT合并操作                                           │
│ • 存储：代码、元数据、Token状态                           │
│                                                          │
│ 执行位置: 工作节点的PebbleDB (ON-CHAIN STATE)             │
│ 数据流: Worker ↔ PebbleDB ↔ Hypergraph CRDT             │
└──────────────────────────────────────────────────────────┘
                          ↑
                          │ 读写状态
                          ↓
Layer 2: 共识层 (Consensus)
┌──────────────────────────────────────────────────────────┐
│ Blockchain Consensus                                     │
│ ─────────────────────────────────────────────────────── │
│ • 交易排序                                                │
│ • 状态转换验证                                            │
│ • VDF时间证明                                             │
└──────────────────────────────────────────────────────────┘
                          ↑
                          │ 消息路由
                          ↓
Layer 1: 网络层 (Network)
┌──────────────────────────────────────────────────────────┐
│ P2P Network (BlossomSub)                                 │
│ ─────────────────────────────────────────────────────── │
│ • 消息传播                                                │
│ • Peer发现                                                │
│ • 加密通道                                                │
└──────────────────────────────────────────────────────────┘
```

## 2. 工作节点的双重职责（完全分离）

### 2.1 职责分析

```
Worker Node (例如: Core 1, Shard 0x80-0xBF)
┌──────────────────────────────────────────────────────────┐
│                                                          │
│  职责A: 超图状态管理 (ON-CHAIN)                           │
│  ══════════════════════════════════════════════════════ │
│                                                          │
│  1. 接收共识层的状态更新                                  │
│     • CodeDeployment交易 → 存储电路                      │
│     • Token交易 → 更新余额                               │
│     • 执行记录 → 更新元数据                               │
│                                                          │
│  2. 执行超图CRDT操作                                      │
│     ┌────────────────────────────────────┐              │
│     │ AddVertex(domain, address, data)   │              │
│     │   ↓                                │              │
│     │ 1. 计算ShardKey (Bloom filter)     │              │
│     │ 2. 检查是否属于本shard              │              │
│     │ 3. 插入VectorCommitmentTree        │              │
│     │ 4. 更新CRDT sets                   │              │
│     │ 5. 持久化到PebbleDB                 │              │
│     │ 6. 返回Merkle proof                │              │
│     └────────────────────────────────────┘              │
│                                                          │
│  3. 服务超图查询                                          │
│     ┌────────────────────────────────────┐              │
│     │ GetVertexData(address)             │              │
│     │   ↓                                │              │
│     │ 1. 从PebbleDB读取                   │              │
│     │ 2. 反序列化VectorCommitmentTree    │              │
│     │ 3. 返回数据                         │              │
│     └────────────────────────────────────┘              │
│                                                          │
│  这些是 **数据库操作**，不涉及用户逻辑计算                 │
│                                                          │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  职责B: P2P消息路由 (INFRASTRUCTURE)                      │
│  ══════════════════════════════════════════════════════ │
│                                                          │
│  1. 运行BlossomSub节点                                    │
│     • 订阅相关topics                                     │
│     • 转发加密消息                                       │
│     • 不解密内容!                                        │
│                                                          │
│  2. 为MPC参与者提供网络基础设施                           │
│     ┌────────────────────────────────────┐              │
│     │ Alice发送: 加密的garbled circuit   │              │
│     │   ↓                                │              │
│     │ Worker: 仅仅转发，不解密           │              │
│     │   ↓                                │              │
│     │ Bob接收: 加密的garbled circuit     │              │
│     └────────────────────────────────────┘              │
│                                                          │
│  这是 **网络路由**，不涉及内容处理                        │
│                                                          │
└──────────────────────────────────────────────────────────┘

关键点:
  • 职责A和职责B在 **不同的时间、不同的数据流** 中发生
  • 它们 **不冲突、不混淆**
  • Worker节点 **不执行布尔电路**，只存储和路由
```

### 2.2 时间线分离

```
T0: 代码部署阶段
────────────────
Worker职责A (超图操作):
  1. 接收CodeDeployment交易
  2. 验证签名和支付
  3. 计算code_address = Hash(circuit)
  4. 存储到Hypergraph:
     AddVertex(COMPUTE_DOMAIN, code_address, circuit_binary)
  5. 持久化到PebbleDB
  
  ⏱️ 耗时: ~50ms
  💾 数据: 156 KB (quicksort电路)
  
Worker职责B (P2P路由): 
  (无操作，此阶段无MPC流量)

═══════════════════════════════════════════════════════════

T1: 执行请求阶段
────────────────
Worker职责A (超图操作):
  1. 接收CodeExecute交易
  2. 验证支付证明
  3. 存储执行元数据:
     AddVertex(COMPUTE_DOMAIN, exec_id, {
       code_ref: code_address,
       rendezvous: 0x9b8a...
     })
  
  ⏱️ 耗时: ~20ms
  💾 数据: ~500 bytes (元数据)
  
Worker职责B (P2P路由):
  (无操作，此阶段无MPC流量)

═══════════════════════════════════════════════════════════

T2: MPC执行阶段
───────────────
Worker职责A (超图操作):
  1. 被动服务查询:
     Alice请求: GetVertexData(code_address)
     → 返回: circuit_binary (156 KB)
  
  ⏱️ 耗时: ~5ms (数据库读取)
  💾 数据: 156 KB (从PebbleDB读)
  
Worker职责B (P2P路由):
  1. 转发Alice → Bob的消息:
     • Garbled circuit (752 KB)
     • Input labels (41 KB)
     • Output labels (41 KB)
  
  2. 关键: Worker **只转发，不解密**
     Message format:
       [Header] [Encrypted Payload]
       Worker看到: Header (routing info)
       Worker看不到: Payload (密文)
  
  ⏱️ 耗时: ~900ms (网络传输)
  💾 数据: 834 KB (转发，不存储)

═══════════════════════════════════════════════════════════

T3: 结果记录阶段 (可选)
───────────────────────
Worker职责A (超图操作):
  1. 接收ComputeFinalize交易
  2. 存储结果承诺:
     AddVertex(COMPUTE_DOMAIN, result_id, {
       commitment: Hash(result)
     })
  
  ⏱️ 耗时: ~20ms
  💾 数据: ~100 bytes (仅承诺，无明文)
  
Worker职责B (P2P路由):
  (无操作，MPC已完成)

═══════════════════════════════════════════════════════════

总结:
  • Worker的两种职责在 **时间上交错**
  • 职责A (超图): 存储电路、元数据 (数据库操作)
  • 职责B (P2P): 转发加密消息 (网络路由)
  • Worker **从不执行布尔电路逻辑**
```

## 3. 超图与布尔电路的关系

### 3.1 不是编译关系

```
错误理解 ✗:
  超图逻辑 → 编译 → 布尔电路
  或: 布尔电路 → 存储 → 超图查询语言
  
正确理解 ✓:
  超图 = 存储介质
  布尔电路 = 存储的内容之一
  
  类比:
    超图 : 布尔电路 = 硬盘 : 程序二进制
    你不会说"硬盘编译为exe文件"
    而是说"硬盘存储exe文件"
```

### 3.2 实际关系

```
┌─────────────────────────────────────────────────────────┐
│                     关系图谱                             │
└─────────────────────────────────────────────────────────┘

用户逻辑 (QCL代码)
    ↓
  编译 (Bedlam Compiler)
    ↓
布尔电路二进制
    ↓
  存储 (通过CodeDeployment交易)
    ↓
超图Vertex
    ↓
  持久化 (PebbleDB)
    ↓
工作节点磁盘

MPC执行时:
  工作节点提供 → 布尔电路二进制 (读取)
  Alice & Bob → 执行乱码电路协议 (计算)
  
关系:
  • 超图: 电路的 **存储位置**
  • 布尔电路: 超图中 **存储的数据**
  • OT/乱码电路: 电路的 **执行方式**
```

### 3.3 具体示例

```go
// Worker节点的代码 - 它不执行布尔电路!

// ════════════════════════════════════════════════════════
// 职责A: 超图操作 (存储和检索)
// ════════════════════════════════════════════════════════

// 1. 存储电路
func (w *Worker) HandleCodeDeployment(tx *CodeDeploymentTx) error {
    // 这只是数据库写入，不执行电路!
    circuitBinary := tx.Circuit  // 156 KB的二进制数据
    codeAddress := Hash(circuitBinary)
    
    // 创建Vertex (仅存储操作)
    vertex := &Vertex{
        Domain: COMPUTE_DOMAIN,
        Address: codeAddress,
        Data: circuitBinary,  // 存为不透明的字节数组
    }
    
    // 调用超图CRDT操作 (数据结构操作，非逻辑计算)
    err := w.hypergraph.AddVertex(vertex)
    
    // 持久化 (磁盘I/O，非逻辑计算)
    return w.pebbleDB.Put(key, value)
}

// 2. 检索电路
func (w *Worker) RetrieveCircuit(codeAddress [32]byte) ([]byte, error) {
    // 这只是数据库读取，不执行电路!
    key := MakeKey(COMPUTE_DOMAIN, codeAddress)
    
    // 从PebbleDB读取 (磁盘I/O)
    value, err := w.pebbleDB.Get(key)
    if err != nil {
        return nil, err
    }
    
    // 反序列化 (数据解析，非逻辑计算)
    vertex := DeserializeVertex(value)
    
    // 返回原始二进制 (不解释、不执行)
    return vertex.Data, nil  // 这是opaque binary blob
}

// ════════════════════════════════════════════════════════
// 职责B: P2P路由 (消息转发)
// ════════════════════════════════════════════════════════

// 3. 转发MPC消息
func (w *Worker) HandleP2PMessage(msg *P2PMessage) error {
    // Worker只看到加密的payload
    // 不知道、也不关心内容是什么
    
    if msg.Topic == "/quilibrium/compute/9b8a7c..." {
        // 检查路由规则
        if shouldForward(msg) {
            // 直接转发，不解密!
            return w.blossomSub.Publish(msg.Topic, msg.EncryptedPayload)
        }
    }
    
    return nil
}

// ════════════════════════════════════════════════════════
// Worker节点 **从不** 做这些事情:
// ════════════════════════════════════════════════════════

// ✗ 错误示例1: Worker不执行电路
func (w *Worker) ExecuteCircuit(circuit []byte, inputs []byte) []byte {
    // ❌ 这个函数不存在!
    // Worker不知道如何执行布尔电路
    // 它只存储电路的二进制数据
}

// ✗ 错误示例2: Worker不garble电路
func (w *Worker) GarbleCircuit(circuit []byte) *GarbledCircuit {
    // ❌ 这个函数不存在!
    // Garbling只发生在Alice端 (MPC参与者)
    // Worker不参与MPC协议
}

// ✗ 错误示例3: Worker不解密MPC消息
func (w *Worker) DecryptMPCMessage(encrypted []byte) []byte {
    // ❌ 这个函数不存在!
    // Worker没有解密密钥
    // 只有Alice和Bob能解密
}
```

## 4. 实际执行流程对比

```
════════════════════════════════════════════════════════════
场景A: Token转账 (超图内置逻辑)
════════════════════════════════════════════════════════════

用户: 发送Token交易
  ↓
区块链共识: 验证并排序交易
  ↓
Worker节点: 执行Token intrinsic逻辑
  ┌──────────────────────────────────┐
  │ 1. 读取发送者余额 (超图查询)     │
  │    GetVertex(sender_address)     │
  │    → balance = 1000 QUIL         │
  │                                  │
  │ 2. 检查余额充足 (内置逻辑)       │
  │    if balance < amount:          │
  │      return ERROR                │
  │                                  │
  │ 3. 更新余额 (超图操作)           │
  │    UpdateVertex(sender, -100)    │
  │    UpdateVertex(receiver, +100)  │
  │                                  │
  │ 4. 持久化 (数据库写入)           │
  │    PebbleDB.Put(...)             │
  └──────────────────────────────────┘
  ↓
结果: 余额更新完成

关键点:
  • Worker执行的是 **Token intrinsic的预定义逻辑**
  • 这是 **内置的、固定的** 业务逻辑
  • 操作超图 (读/写状态)
  • 不涉及用户自定义计算

════════════════════════════════════════════════════════════
场景B: QuickSort (用户自定义布尔电路)
════════════════════════════════════════════════════════════

用户: 发送CodeExecute交易
  ↓
区块链共识: 验证并记录请求
  ↓
Worker节点: 仅存储元数据
  ┌──────────────────────────────────┐
  │ 1. 记录执行请求 (超图操作)       │
  │    AddVertex(exec_id, metadata)  │
  │                                  │
  │ 2. 持久化 (数据库写入)           │
  │    PebbleDB.Put(...)             │
  │                                  │
  │ 3. 不执行任何计算逻辑!           │
  │    Worker的工作到此为止          │
  └──────────────────────────────────┘
  ↓
  ╔══════════════════════════════════╗
  ║ MPC执行 (OFF-CHAIN, P2P)         ║
  ╚══════════════════════════════════╝
  ↓
Alice: 从Worker查询电路
  ┌──────────────────────────────────┐
  │ GetVertexData(code_address)      │
  │ → 获取: quicksort.circ (156 KB)  │
  └──────────────────────────────────┘
  ↓
Alice: 本地执行Garbling
  ┌──────────────────────────────────┐
  │ 1. 解析电路 (45,231 gates)       │
  │ 2. 生成wire labels (1.5 MB)     │
  │ 3. 加密真值表 (752 KB)           │
  │ 4. 发送给Bob (P2P直连)           │
  └──────────────────────────────────┘
  ↓
Bob: 本地执行Evaluation
  ┌──────────────────────────────────┐
  │ 1. 接收加密电路                  │
  │ 2. 通过OT获取输入labels          │
  │ 3. 逐门解密和计算                │
  │ 4. 发送输出labels给Alice         │
  └──────────────────────────────────┘
  ↓
Alice: 解码结果
  ┌──────────────────────────────────┐
  │ sorted_array = Decode(labels)    │
  │ → ["apple", "banana", ...]       │
  └──────────────────────────────────┘
  ↓
(可选) Worker: 记录结果承诺
  ┌──────────────────────────────────┐
  │ AddVertex(result_id,             │
  │   Hash(sorted_array))            │
  └──────────────────────────────────┘

关键点:
  • Worker **不执行QuickSort逻辑**
  • Worker只提供: 电路存储 + 消息路由
  • 真正的计算在 **Alice和Bob之间** (P2P)
  • Worker永远看不到明文的输入或输出
```

## 5. 为什么这样设计？

### 5.1 架构优势

```
┌─────────────────────────────────────────────────────────┐
│ 如果Worker执行布尔电路会怎样? (假设的错误设计)           │
├─────────────────────────────────────────────────────────┤
│                                                         │
│ ✗ 隐私泄露:                                             │
│   Worker能看到所有输入和输出                            │
│   → 破坏了MPC的隐私保证                                 │
│                                                         │
│ ✗ 可扩展性差:                                           │
│   每个Worker需要执行所有用户的计算                      │
│   → 节点负载过重，无法水平扩展                          │
│                                                         │
│ ✗ 信任问题:                                             │
│   用户必须信任Worker节点                                │
│   → 违背去中心化原则                                    │
│                                                         │
│ ✗ 激励错位:                                             │
│   Worker做计算但得不到直接报酬                         │
│   → 缺乏动力运行节点                                    │
│                                                         │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│ 当前设计的优势: Worker不执行布尔电路                     │
├─────────────────────────────────────────────────────────┤
│                                                         │
│ ✓ 隐私保护:                                             │
│   Worker只看到加密数据                                  │
│   → 无法推断用户输入或结果                              │
│                                                         │
│ ✓ 可扩展性好:                                           │
│   计算在用户之间P2P进行                                 │
│   → Worker只需处理状态存储，负载可控                    │
│                                                         │
│ ✓ 去信任化:                                             │
│   用户不需要信任任何节点                                │
│   → 真正的去中心化                                      │
│                                                         │
│ ✓ 激励清晰:                                             │
│   Worker: 提供存储和网络，收取租金                      │
│   Alice/Bob: 执行计算，支付gas                          │
│   → 角色分明，激励对齐                                  │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 5.2 分工明确

```
┌──────────────┬──────────────────────────────────────────┐
│ 角色         │ 职责                                      │
├──────────────┼──────────────────────────────────────────┤
│ Worker节点   │ • 存储状态 (超图)                         │
│              │ • 路由消息 (P2P)                          │
│              │ • 达成共识 (区块链)                       │
│              │ • 不执行用户计算                          │
├──────────────┼──────────────────────────────────────────┤
│ MPC参与者    │ • 执行布尔电路 (乱码电路)                 │
│ (Alice/Bob)  │ • 交换输入 (OT)                           │
│              │ • 计算结果                                │
│              │ • 不信任任何节点                          │
├──────────────┼──────────────────────────────────────────┤
│ 编译器       │ • QCL → 布尔电路                          │
│ (Bedlam)     │ • 电路优化                                │
│              │ • 仅在开发时运行                          │
└──────────────┴──────────────────────────────────────────┘
```

## 6. 代码示例：Worker视角

让我们看看Worker节点的实际代码，验证它不执行布尔电路：

```go
// node/execution/intrinsics/compute/compute_intrinsic.go

// ComputeIntrinsic - Worker节点的Compute intrinsic实现
type ComputeIntrinsic struct {
    hypergraph        hypergraph.Hypergraph  // 超图接口
    inclusionProver   crypto.InclusionProver
    bulletproofProver crypto.BulletproofProver
    // ... 其他字段
    
    // 注意: 没有 CircuitExecutor 或 GarbledCircuitEvaluator!
}

// Deploy - 部署代码 (存储到超图)
func (c *ComputeIntrinsic) Deploy(
    domain [32]byte,
    provers [][]byte,
    creator []byte,
    fee *big.Int,
    contextData []byte,  // 包含电路二进制
    frameNumber uint64,
    hgstate state.State,
) (state.State, error) {
    // 1. 解析部署参数
    codeDeployment := &CodeDeployment{}
    err := codeDeployment.FromBytes(contextData, c.compiler)
    
    // 2. 验证 (不执行!)
    valid, err := codeDeployment.Verify(frameNumber)
    
    // 3. 存储到超图 (关键: 作为opaque data存储)
    codeAddressBI, err := poseidon.HashBytes(
        concat(domain[:], codeDeployment.Circuit),
    )
    codeAddress := codeAddressBI.FillBytes(make([]byte, 32))
    
    // 创建VectorCommitmentTree存储电路
    codeTree := &VectorCommitmentTree{}
    err := codeTree.Insert(
        []byte{0 << 2},           // Index 0
        codeDeployment.Circuit,   // 电路作为字节数组存储!
        nil,
        big.NewInt(int64(len(codeDeployment.Circuit))),
    )
    
    // 4. Materialize (持久化)
    value := NewVertexAddMaterializedState(
        domain,
        codeAddress,
        frameNumber,
        nil,
        codeTree,  // 存储的是原始二进制，不是解析后的结构
    )
    
    err = hypergraph.Set(
        domain[:],
        codeAddress,
        VertexAddsDiscriminator,
        frameNumber,
        value,
    )
    
    return hypergraph, nil
    
    // ════════════════════════════════════════════════════
    // 注意这个函数做了什么:
    // ✓ 接收电路二进制
    // ✓ 计算地址 (hash)
    // ✓ 存储到超图
    // 
    // 这个函数没有做什么:
    // ✗ 解析电路的gates和wires
    // ✗ 执行任何计算
    // ✗ Garble或evaluate
    // 
    // 电路被当作 **不透明的字节数组** 存储!
    // ════════════════════════════════════════════════════
}

// InvokeStep - 处理执行请求
func (c *ComputeIntrinsic) InvokeStep(
    state state.State,
    input []byte,
    frameNumber uint64,
    feePaid *big.Int,
    feeMultiplier *big.Int,
) (state.State, error) {
    typePrefix := binary.BigEndian.Uint32(input[:4])
    
    switch typePrefix {
    case protobufs.CodeExecuteType:
        // 解析执行请求
        var codeExecute CodeExecute
        err := codeExecute.FromBytes(input, ...)
        
        // 验证请求
        valid, err := codeExecute.Verify(frameNumber)
        
        // 检查费用
        cost, err := codeExecute.GetCost()
        if feePaid.Cmp(cost.Mul(feeMultiplier)) < 0 {
            return nil, errors.New("insufficient fee")
        }
        
        // ════════════════════════════════════════════════
        // 关键: 这里 **不执行** 布尔电路!
        // Worker只是:
        // 1. 验证请求格式
        // 2. 检查支付
        // 3. 记录元数据
        // 
        // 实际的MPC执行在链外进行!
        // ════════════════════════════════════════════════
        
        return state, nil  // 仅返回状态，无计算结果
    }
}

// ════════════════════════════════════════════════════════
// Worker节点没有这些函数:
// ════════════════════════════════════════════════════════

// ✗ 这个函数不存在于Worker中!
func (c *ComputeIntrinsic) ExecuteCircuit(
    circuit []byte,
    inputs []byte,
) []byte {
    // ❌ Worker不实现此功能
    // Circuit execution在Alice/Bob的本地机器
}

// ✗ 这个函数不存在于Worker中!
func (c *ComputeIntrinsic) GarbleCircuit(
    circuit *Circuit,
) *GarbledCircuit {
    // ❌ Worker不实现此功能
    // Garbling在Alice端进行
}
```

## 7. 总结答案

### 问题：工作节点最终不可能既计算超图逻辑又计算布尔电路逻辑吧？

**回答**: 
```
✓ 工作节点确实有两种职责，但它们完全不冲突:

1. 超图操作 (数据库逻辑):
   • AddVertex, GetVertex, UpdateVertex
   • 这些是 **数据结构操作**
   • 类似: SQL INSERT, SELECT, UPDATE
   • 不涉及用户定义的计算逻辑

2. P2P路由 (网络逻辑):
   • 转发加密消息
   • 提供网络基础设施
   • 不解密、不执行内容

工作节点 **从不执行布尔电路逻辑**!
布尔电路的执行在 MPC参与者(Alice/Bob) 之间进行。
```

### 问题：超图和布尔电路（基于OT的）的关系是什么？

**回答**:
```
关系: 存储与被存储

超图 (Storage Layer):
  • 角色: 持久化存储介质
  • 功能: 存储和检索数据
  • 存储内容: 代码、元数据、Token状态等
  
布尔电路 (Application Logic):
  • 角色: 用户定义的计算逻辑
  • 功能: 隐私保护的通用计算
  • 存储位置: 超图的Vertex中 (作为二进制blob)

OT (Communication Protocol):
  • 角色: MPC参与者间的安全通信
  • 功能: 不泄露选择的输入传输
  • 使用场景: Alice向Bob传输labels

类比:
  超图 : 布尔电路 : OT
    =
  硬盘 : 程序exe : 网络协议
  
  硬盘存储exe，网络传输数据
  超图存储电路，OT传输labels
```

### 问题：把超图编译为布尔电路？

**回答**:
```
✗ 不是! 它们不是编译关系

正确的关系链:
  QCL源代码 → (编译) → 布尔电路 → (存储) → 超图

1. QCL代码 (用户编写)
     ↓ Bedlam Compiler
2. 布尔电路 (编译产物)
     ↓ CodeDeployment Transaction
3. 超图Vertex (存储位置)
     ↓ MPC Execution (检索)
4. Alice/Bob本地 (执行)

超图不参与编译过程!
超图只是存储编译后的电路。
```

---

## 最终图解

```
╔═══════════════════════════════════════════════════════════╗
║               Quilibrium完整架构分层                       ║
╚═══════════════════════════════════════════════════════════╝

用户空间 (User Space):
┌─────────────────────────────────────────────────────────┐
│ QCL开发 → Bedlam编译 → 布尔电路                          │
│                                                         │
│ Alice & Bob → MPC执行 (乱码电路 + OT)                   │
│   • Garbling                                           │
│   • Oblivious Transfer                                 │
│   • Evaluation                                         │
│   • Decoding                                           │
└─────────────────────────────────────────────────────────┘
              ↓ 上传电路            ↑ 检索电路
              ↓ 记录执行            ↑ 查询元数据

节点空间 (Node Space):
┌─────────────────────────────────────────────────────────┐
│ Worker节点                                              │
│ ┌─────────────────────────────────────────────────────┐ │
│ │ 超图CRDT (数据库逻辑)                                 │ │
│ │ • AddVertex                                         │ │
│ │ • GetVertex                                         │ │
│ │ • UpdateVertex                                      │ │
│ │ • CRDT Merge                                        │ │
│ └─────────────────────────────────────────────────────┘ │
│         ↕ 读写                                          │
│ ┌─────────────────────────────────────────────────────┐ │
│ │ PebbleDB (持久化存储)                                 │ │
│ │ • Key-Value Store                                   │ │
│ │ • 电路二进制 (opaque blobs)                          │ │
│ │ • 元数据                                             │ │
│ └─────────────────────────────────────────────────────┘ │
│                                                         │
│ ┌─────────────────────────────────────────────────────┐ │
│ │ BlossomSub (P2P路由)                                 │ │
│ │ • 消息转发                                           │ │
│ │ • 不解密内容                                         │ │
│ │ • 提供网络基础设施                                   │ │
│ └─────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘

关键点:
  • 上下层次清晰分离
  • Worker不执行布尔电路
  • MPC在用户空间进行
  • 超图只负责存储和查询
```

这就是完整的答案！超图和布尔电路是两个独立的系统层，不冲突、不编译，各司其职。

