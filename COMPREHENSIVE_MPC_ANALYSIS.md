# Quilibrium MPC System - Comprehensive Analysis
# 问题全面解答

## 目录

1. [布尔电路与乱码电路等价性](#1-布尔电路与乱码电路等价性)
2. [超图与计算的关系](#2-超图与计算的关系)
3. [实际例子分析](#3-实际例子分析)
4. [系统架构总结](#4-系统架构总结)
5. [关键发现](#5-关键发现)

---

## 1. 布尔电路与乱码电路等价性

### 核心问题回答：为什么它们等价？

**简短回答**: 乱码电路通过加密保存了布尔电路的所有真值表信息，但以一种只能"单向遍历"的方式呈现。

### 数学表达

```
布尔电路: f: {0,1}^n → {0,1}^m
乱码电路: GC_f: Labels^n → Labels^m

等价性: Decode(GC_f(Encode(x))) = f(x)

其中:
- Encode(x): 将输入bits映射到对应的labels
- GC_f: 使用labels评估加密的电路
- Decode: 将输出labels解码回bits
```

### 示例：2*32+5-4/33

```
1. 表达式分解为布尔运算:
   ├─ 乘法: 2 × 32 → ~200个门
   ├─ 加法: result + 5 → ~80个门
   ├─ 除法: 4 / 33 → ~240个门
   └─ 减法: result - quotient → ~80个门
   总计: ~600个门

2. 每个门的加密:
   AND门 (a,b→c) 的真值表:
   
   明文:              加密版本:
   a=0,b=0 → c=0     L_a0, L_b0 → 加密表 → L_c0
   a=0,b=1 → c=0     L_a0, L_b1 → 加密表 → L_c0
   a=1,b=0 → c=0     L_a1, L_b0 → 加密表 → L_c0
   a=1,b=1 → c=1     L_a1, L_b1 → 加密表 → L_c1
   
   关键: Bob只能解密他拥有的label对应的行！

3. 为何等价:
   ✓ 结构同构: 每个布尔门 ↔ 一个加密门
   ✓ 语义保持: 真值表逻辑保留在加密中
   ✓ 功能完整: 正确输入 → 正确输出
   ✓ 信息隐藏: 无法探索其他执行路径
```

### 安全性证明素描

```
假设: AES是安全的伪随机函数

定理: 在诚实但好奇(honest-but-curious)模型下，
      乱码电路不泄露除输出外的任何信息。

证明思路:
1. Labels不可区分: L0, L1 ≈ Random(128bit)
2. 未使用的行: 加密密钥未知 → 无法解密
3. OT安全性: 发送方不知接收方选择
4. 组合: 即使看到所有加密表，也无法推断:
   - 哪些label代表0或1
   - 其他输入会产生什么输出
   - 电路的语义含义
```

---

## 2. 超图与计算的关系

### 关键澄清：超图 ≠ 计算查询引擎

```
┌─────────────────────────────────────────────┐
│ 常见误解 ✗                                   │
├─────────────────────────────────────────────┤
│ "超图查询执行计算"                            │
│ "数值计算转换为超图查询"                      │
│ "OT用于查询超图"                              │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│ 正确理解 ✓                                   │
├─────────────────────────────────────────────┤
│ 超图 = 持久化存储层                          │
│   • 存储代码(电路二进制)                      │
│   • 存储元数据(执行记录)                      │
│   • 不执行计算                                │
│                                             │
│ 乱码电路 = 计算执行层                        │
│   • P2P直接通信                              │
│   • 内存中执行                                │
│   • 不持久化                                  │
│                                             │
│ OT = 安全输入传输                            │
│   • 用于MPC参与者间传递labels                │
│   • 不涉及超图查询                            │
└─────────────────────────────────────────────┘
```

### 超图的四大作用

```
1. 代码仓库 (Code Repository)
   ┌─────────────────────────────────┐
   │ Vertex:                         │
   │   Address: Hash(circuit)        │
   │   Data: Circuit binary          │
   │   Metadata: {                   │
   │     creator,                    │
   │     timestamp,                  │
   │     input_types,                │
   │     output_types                │
   │   }                             │
   └─────────────────────────────────┘
   
   查询: GetVertexData(code_address)
   → 检索电路用于garbling

2. 执行注册表 (Execution Registry)
   ┌─────────────────────────────────┐
   │ Vertex:                         │
   │   Address: Hash(exec_id)        │
   │   Data: {                       │
   │     code_ref,                   │
   │     rendezvous,                 │
   │     participants,               │
   │     payment_proof               │
   │   }                             │
   └─────────────────────────────────┘
   
   查询: GetVertexData(exec_id)
   → 发现rendezvous点进行P2P连接

3. 审计日志 (Audit Trail)
   ┌─────────────────────────────────┐
   │ Hyperedge:                      │
   │   Source: [CodeVertex]          │
   │   Target: [ExecutionVertex]     │
   │   Label: "executed_at"          │
   │   Timestamp: block_number       │
   └─────────────────────────────────┘
   
   查询: GetHyperedges(code_address)
   → 追踪代码的所有执行历史

4. 结果存根 (Result Stubs)
   ┌─────────────────────────────────┐
   │ Vertex:                         │
   │   Address: Hash(result)         │
   │   Data: {                       │
   │     commitment,                 │
   │     proof,                      │
   │     encrypted_result            │
   │   }                             │
   └─────────────────────────────────┘
   
   查询: GetVertexData(result_id)
   → 验证计算完成(不含明文)
```

### 完整的数据流

```
                    Hypergraph              MPC Protocol
                    (存储)                  (计算)
                       │                       │
Step 1: Deploy        │                       │
  Code ───────────────►│                       │
  (Write)              │                       │
                       │                       │
Step 2: Request        │                       │
  Execution ──────────►│                       │
  (Write)              │                       │
                       │                       │
Step 3: Discover       │                       │
  Participants ◄───────┤                       │
  (Read)               │                       │
                       │                       │
Step 4: Retrieve       │                       │
  Code ◄───────────────┤                       │
  (Read)               │                       │
                       │                       │
Step 5: Execute        │                       │
                       │  ┌────────────────────┤
                       │  │ P2P Channel        │
                       │  │ • Garble           │
                       │  │ • OT               │
                       │  │ • Evaluate         │
                       │  │ (不接触超图!)        │
                       │  └────────────────────┤
                       │                       │
Step 6: Record         │                       │
  Result ─────────────►│                       │
  (Write, Optional)    │                       │
                       │                       │

关键点: 超图像"硬盘"，MPC像"CPU"
       数据存储 vs 数据处理
```

---

## 3. 实际例子分析

### 例子1: 字符串包含 - "abcde" contains "bc"

#### 布尔电路设计

```
输入:
  haystack: "abcde" = 5 bytes = 40 bits + 8 bits length = 48 bits (实际256*8+8)
  needle: "bc" = 2 bytes = 16 bits + 8 bits length = 24 bits

电路结构:
  1. 长度检查: haystack.len >= needle.len
     └─ 8-bit comparator: 20 gates
  
  2. 滑动窗口比较 (4个位置):
     Position 0: "ab" vs "bc" → false
       ├─ 'a'(0x61) == 'b'(0x62)? → 8-bit comparator: 16 gates
       └─ 'b'(0x62) == 'c'(0x63)? → 16 gates
     Position 1: "bc" vs "bc" → true ✓
       ├─ 'b'(0x62) == 'b'(0x62)? → true
       └─ 'c'(0x63) == 'c'(0x63)? → true
     Position 2: "cd" vs "bc" → false
     Position 3: "de" vs "bc" → false
     
     Each position: 32 gates
     Total: 128 gates
  
  3. OR树 (any match?):
     └─ 4-input OR: 3 gates

总计: ~535 gates
乱码大小: ~8.7 KB
```

#### MPC执行时序

```
Alice (has "abcde"):              Bob (has "bc"):
  ↓                                 ↓
[Setup: 300ms]                    [Setup: 300ms]
  • Load circuit                    • Connect P2P
  • Generate labels                 • Receive garble key
  ↓                                 ↓
[Garbling: 200ms]                 [Waiting]
  • Encrypt gates                   
  • Send to Bob ──────────────────→ 
  ↓                                 ↓
[OT: 150ms] ◄─────────────────────► [OT: 150ms]
  • Send label pairs                • Choose based on "bc"
  ↓                                 ↓
                                  [Evaluation: 180ms]
                                    • Decrypt gates
                                    • Compute output
                                    • Send result ─────→
  ↓                                 ↓
[Decoding: 10ms]
  • Receive label
  • Decode: true ✓
  
Total: ~850ms
Data: ~10 KB transferred

Result: true (contains)
Privacy: 
  ✓ Bob不知道haystack是"abcde"
  ✓ Alice不知道needle是"bc" (如果Bob作为evaluator输入)
```

### 例子2: 快速排序 - ["zebra","apple","mango","banana","cherry"]

#### 完整流程(5.1-5.9 & 6.1-6.7)

##### 5.1 源代码语言: QCL

```qcl
// quicksort.qcl - Quilibrium Circuit Language
// 类C语法，专为电路设计

func quicksort(arr *StringArray) {
    quicksort_rec(arr, 0, arr.len-1)
}

func partition(arr *StringArray, low, high uint8) uint8 {
    pivot := arr.items[high]
    i := low
    for j := low; j < high; j++ {
        if strcmp(arr.items[j], pivot) <= 0 {
            swap(arr, i, j)
            i++
        }
    }
    swap(arr, i, high)
    return i
}
```

**关键点**: 
- QCL不是Python/JavaScript，是专门的DSL
- 编译为布尔电路，不是虚拟机字节码
- 限制: 循环必须展开，递归必须限制深度

##### 5.2 如何传入Quilibrium网络

```bash
# 方法1: 通过客户端命令行
$ qclient compute deploy \
    --code quicksort.circ \
    --domain [random_32_bytes] \
    --payment 1000QUIL

# 方法2: 通过gRPC API
client = QuilibriumClient(endpoint)
tx = client.createCodeDeployment({
    circuit: readFile("quicksort.circ"),
    inputTypes: ["StringArray"],
    outputTypes: ["StringArray"],
    domain: randomBytes(32)
})
client.broadcast(tx)

# 方法3: 通过REST API
POST /api/v1/compute/deploy
{
    "circuit": "<base64_encoded_circuit>",
    "input_types": ["StringArray"],
    "output_types": ["StringArray"],
    "domain": "0xabcd...",
    "signature": "<ed448_signature>"
}
```

**网络传播**:
```
Client
  ↓ gRPC/REST
Node A (接收)
  ↓ 验证签名、支付
Master Process
  ↓ 广播到consensus
All Nodes (P2P gossip)
  ↓ 达成共识
Shard-Owning Worker
  ↓ 持久化
Hypergraph Storage
```

##### 5.3 & 5.4 存储位置与格式

```
物理存储:
=========

节点文件系统:
/home/errord/chain/monorepo/.config/
├── store/
│   ├── worker_0/          # Shard 0x00-0x3F
│   ├── worker_1/          # Shard 0x40-0x7F  
│   ├── worker_2/          # Shard 0x80-0xBF ← quicksort在这里!
│   │   └── data/
│   │       ├── 000123.sst      # SSTable文件
│   │       ├── 000124.sst
│   │       ├── MANIFEST
│   │       └── CURRENT
│   └── worker_3/          # Shard 0xC0-0xFF

PebbleDB内部 (worker_2/data/000123.sst):
──────────────────────────────────────
Key: 0x01|cccc...cccc|7f3a9c1d...|00
     ↑   ↑            ↑           ↑
     │   │            │           └─ Discriminator (VERTEX_ADDS)
     │   │            └───────────── Code address
     │   └────────────────────────── Domain (COMPUTE_INTRINSIC)
     └────────────────────────────── Prefix (VERTEX_DATA)

Value: (Snappy compressed VectorCommitmentTree)
  Raw size: 156,842 bytes
  Compressed: 87,234 bytes
  Contains:
    ├─ Index 0: Circuit binary (gates, wires, metadata)
    └─ Index 1: Deployment metadata JSON

逻辑表示 (Hypergraph CRDT):
───────────────────────────
vertexAdds[ShardKey{
    L1: [3]byte{0x84, 0x19, 0xA2},  // Bloom indices
    L2: COMPUTE_INTRINSIC_DOMAIN
}][0x7f3a9c1d...] = VectorCommitmentTree{
    root: 0x9a8b7c6d...,
    leaves: [...]
}
```

**关键点**:
- 每个worker只存储其负责的shard
- 使用CRDT保证多副本最终一致性
- PebbleDB提供高性能读写
- Snappy压缩节省空间

##### 5.5 用户提交排序数据

```python
# Python客户端示例
from quilibrium import QuilibriumClient

client = QuilibriumClient(
    grpc_endpoint="localhost:8337",
    private_key=load_key("~/.config/keys/ed448.key")
)

# 准备数据
data_to_sort = ["zebra", "apple", "mango", "banana", "cherry"]

# 序列化为电路格式
serialized_input = serialize_string_array(data_to_sort)
# 输出格式:
# [0:255]   : "zebra" padded to 32 bytes
# [256:511] : "apple" padded to 32 bytes
# [512:767] : "mango" padded to 32 bytes
# [768:1023]: "banana" padded to 32 bytes
# [1024:1279]: "cherry" padded to 32 bytes
# [1280:2567]: empty padding
# [2568:2575]: length=5 (uint8)
# Total: 2576 bits = 322 bytes

# 创建执行请求
execution_request = {
    "code_address": "0x7f3a9c1d...",
    "domain": COMPUTE_INTRINSIC_DOMAIN,
    "rendezvous": os.urandom(32),  # 随机会合点
    "inputs_hash": sha3_256(serialized_input),  # 输入承诺
    "payment": create_bulletproof(50)  # 50 QUIL
}

# 签名并提交
tx = client.sign_and_submit(execution_request)
print(f"Execution requested: {tx.hash}")

# Alice本地保留序列化输入
# 等待Bob连接进行MPC
```

**数据流**:
```
Python Client
  ↓ Serialize data (322 bytes)
  ↓ Create transaction (inputs_hash only, not data!)
gRPC API
  ↓ Validate signature
Node (Master Process)
  ↓ Verify payment proof
  ↓ Order in block
Blockchain
  ↓ Emit event
All Workers
  ↓ Check shard match
Worker 2 (shard owner)
  ↓ Store execution metadata
Hypergraph

注意: 实际输入数据不上链! 只有hash上链
      数据在MPC时P2P传输
```

##### 5.6 谁来接收数据？

```
数据接收者: MPC参与方 (Alice & Bob)，不是区块链节点

场景1: Alice有数据，Bob辅助计算
  Alice: 
    • 作为Garbler
    • 本地保留: serialized_input (322 bytes)
    • 发送: 加密电路 + Alice的输入labels
  
  Bob:
    • 作为Evaluator
    • 接收: 加密电路 + garble_key
    • 通过OT: 获取自己输入的labels (如果有)
    • 不接收: Alice的原始数据!

场景2: Alice和Bob各有数据
  Alice: 输入前5个字符串
  Bob: 输入后5个字符串
  
  Alice序列化: items[0..4]
  Bob序列化: items[5..9]
  
  MPC协议:
    Alice发送: 前2568 bits的labels
    Bob通过OT: 获取后2568 bits的labels

区块链节点的角色:
  • 不接收原始数据
  • 只接收: metadata, hash, payment
  • 存储: 代码和执行记录
  • 不参与: MPC计算
```

##### 5.7 3节点网络中谁计算？

```
网络拓扑:
=========

Node A (IP: 192.168.1.10)
  Master Process (Core 0):
    └─ Global consensus, all shards coordinator
  Worker Process (Core 1):
    └─ Shard: 0x00-0x7F

Node B (IP: 192.168.1.11)
  Master Process (Core 0):
    └─ Global consensus, all shards coordinator
  Worker Process (Core 1):
    └─ Shard: 0x80-0xBF ← quicksort code stored here!

Node C (IP: 192.168.1.12)
  Master Process (Core 0):
    └─ Global consensus, all shards coordinator
  Worker Process (Core 1):
    └─ Shard: 0xC0-0xFF

角色分配:
=========

区块链层面 (3节点都参与):
  ✓ Node A: 参与共识，验证区块
  ✓ Node B: 参与共识，验证区块，存储quicksort代码
  ✓ Node C: 参与共识，验证区块

MPC计算层面 (链外P2P):
  ✗ Node A, B, C: 都不直接参与MPC!
  ✓ Alice: 可能运行在任何节点，或独立客户端
  ✓ Bob: 可能运行在任何节点，或独立客户端

实际计算流程:
  1. Alice提交CodeExecute交易
     → 3个节点都接收并验证
     → Node B存储执行metadata (shard owner)
  
  2. Alice和Bob都监听rendezvous
     → Alice: 可能通过Node A的BlossomSub
     → Bob: 可能通过Node C的BlossomSub
     → 节点只是路由消息，不解密内容
  
  3. Alice和Bob建立直接P2P通道
     → 加密连接
     → 可能经过节点中继，但节点看不到明文
  
  4. MPC在Alice和Bob之间执行
     → Node A, B, C只提供网络路由
     → 不参与计算，不能看到数据
  
  5. 结果返回Alice
     → (可选) Alice提交result commitment到链上
     → Node B再次存储 (shard owner)

谁控制计算？
  → Alice控制 (她是Garbler，决定何时开始)
  → Bob协助 (他是Evaluator，必须在线配合)
  → 节点不控制，只提供基础设施

形象比喻:
  节点 = 云存储 + 邮局 + 公证处
    (存代码，路由消息，记录交易)
  
  Alice & Bob = 计算机用户
    (真正执行计算的人)
```

##### 5.8 & 6.6 具体计算细节

```
Atomic-Level Execution Timeline:
================================

T=0ms ┌─── Alice: Load circuit ───┐
      │ Syscall: open("000123.sst", O_RDONLY)
      │ Syscall: read(fd, buf, 87234)
      │ Decompress: Snappy(buf) → 156,842 bytes
      │ Parse: Circuit.Unmarshal()
      │   ├─ Gates: 45,231
      │   ├─ Wires: 47,799
      │   └─ I/O mapping
      └───────────────────────────┘

T=10ms ┌─── Alice: Generate labels ───┐
       │ For wire in 0..47,799:
       │   L0[wire] = getrandom(16 bytes, GRND_NONBLOCK)
       │   L1[wire] = getrandom(16 bytes, GRND_NONBLOCK)
       │   L0[wire].lsb = 0
       │   L1[wire].lsb = 1
       │ Total: 1,528,768 bytes of randomness
       └──────────────────────────────┘

T=60ms ┌─── P2P Connection ───┐
       │ Alice: BlossomSub.Publish({
       │   topic: "/quilibrium/compute/9b8a...",
       │   data: {role:"garbler", pubkey:[...]}
       │ })
       │ 
       │ Bob: BlossomSub.Publish({
       │   topic: "/quilibrium/compute/9b8a...",
       │   data: {role:"evaluator", pubkey:[...]}
       │ })
       │
       │ Routing through Node B:
       │   ├─ Receive Alice's message
       │   ├─ Forward to Bob (via BlossomSub mesh)
       │   └─ Establish direct TCP channel:
       │       Alice:12345 ↔ Node B:8336 ↔ Bob:23456
       └───────────────────────────┘

T=160ms ┌─── ECDH Key Agreement ───┐
        │ Alice_PrivKey = ed448_private (57 bytes)
        │ Bob_PubKey = ed448_public (57 bytes)
        │ 
        │ shared_secret = ScalarMult(Alice_PrivKey, Bob_PubKey)
        │               = ScalarMult(Bob_PrivKey, Alice_PubKey)
        │               = [32 bytes]
        │
        │ k_AB = HKDF-Expand(shared_secret, "alice_to_bob", 32)
        │ k_BA = HKDF-Expand(shared_secret, "bob_to_alice", 32)
        │
        │ AES_AB = AES256.new(k_AB)
        │ AES_BA = AES256.new(k_BA)
        └──────────────────────────┘

T=180ms ┌─── Alice: Encode inputs ───┐
        │ input_array = ["zebra", "apple", "mango", "banana", "cherry"]
        │
        │ For string_idx in 0..4:
        │   s = input_array[string_idx]
        │   For char_idx in 0..len(s)-1:
        │     byte = s[char_idx]
        │     For bit_idx in 0..7:
        │       wire = string_idx*256 + char_idx*8 + bit_idx
        │       bit_value = (byte >> bit_idx) & 1
        │       if bit_value == 0:
        │         alice_labels[wire] = L0[wire]
        │       else:
        │         alice_labels[wire] = L1[wire]
        │
        │ Result: alice_labels = [2568 labels]
        └────────────────────────────┘

T=185ms ┌─── Alice→Bob: Send input labels ───┐
        │ For label in alice_labels:
        │   encrypted = AES_AB.Encrypt(label)
        │   TCP_Send(bob_socket, encrypted)
        │
        │ Packet structure:
        │   [2 bytes length] [16 bytes encrypted_label]
        │   × 2568 iterations
        │
        │ TCP details:
        │   ├─ Socket: Alice:12345 → Node B:8336
        │   ├─ Fragmented into ~30 TCP packets
        │   ├─ Each packet: ~1400 bytes (MTU)
        │   └─ Total: 41,088 bytes sent
        └─────────────────────────────────────┘

T=235ms ┌─── Bob: Receive labels ───┐
        │ While count < 2568:
        │   length = TCP_Recv(alice_socket, 2)
        │   encrypted = TCP_Recv(alice_socket, length)
        │   label = AES_BA.Decrypt(encrypted)
        │   wire_labels[count] = label
        │   count++
        │
        │ Kernel operations:
        │   ├─ recv() syscalls: 2568 × 2 = 5136
        │   ├─ Buffer copies: userspace ← kernel
        │   └─ AES decryptions: 2568 (AES-NI)
        └───────────────────────────┘

T=250ms ┌─── Alice: Garble & stream ───┐
        │ garble_key = Random(16 bytes)
        │ Send_Encrypted(garble_key)
        │
        │ aes_garble = AES128.new(garble_key)
        │
        │ For gate_id in 0..45,230:
        │   gate = circuit.Gates[gate_id]
        │   
        │   IF gate.Op == XOR:
        │     // Free XOR - no table!
        │     garbled_table = []
        │   
        │   ELSE IF gate.Op == AND:
        │     // Half-Gates
        │     L_a0 = L0[gate.Input0]
        │     L_a1 = L1[gate.Input0]
        │     L_b0 = L0[gate.Input1]
        │     L_b1 = L1[gate.Input1]
        │     L_c0 = L0[gate.Output]
        │     
        │     j0 = gate_id * 2
        │     j1 = gate_id * 2 + 1
        │     
        │     // CPU operations:
        │     TG = aes_garble.Encrypt(L_a0 || j0)  // AES-NI: ~5 cycles
        │     if L_a1.select_bit:
        │       TG = TG ⊕ L_c0  // XOR: ~1 cycle
        │     
        │     TE = aes_garble.Encrypt(L_b0 || j1)
        │     if L_b1.select_bit:
        │       TE = TE ⊕ (L_c0 ⊕ L_a0)
        │     
        │     garbled_table = (TG, TE)  // 32 bytes
        │   
        │   // Stream immediately
        │   packet = {gate_id, gate.Op, garbled_table}
        │   encrypted_packet = AES_AB.Encrypt(packet)
        │   TCP_Send(bob_socket, encrypted_packet)
        │   
        │   // Pipelining: garble next gate while TCP sends
        │
        │ Total CPU: 45,231 gates
        │   - XOR: 23,456 (skip)
        │   - AND: 15,234 × 2 AES = 30,468 AES ops
        │   - OR: 4,567 × 4 AES = 18,268 AES ops
        │   - INV: 1,974 × 2 AES = 3,948 AES ops
        │   Total: 52,684 AES operations
        │
        │ CPU time: 52,684 × 5 cycles ÷ 3.5 GHz ≈ 75 µs
        │ Network time: 752 KB ÷ 100 Mbps ≈ 60ms
        │ Total: ~800ms (network-bound)
        └───────────────────────────────┘

T=1050ms ┌─── Bob: Evaluate gates ───┐
         │ wire_labels[0..2567] = alice_labels (received)
         │
         │ For gate_id in 0..45,230:
         │   // Receive garbled gate
         │   packet = TCP_Recv()
         │   gate = Decrypt(packet)
         │   
         │   label_a = wire_labels[gate.Input0]
         │   label_b = wire_labels[gate.Input1]
         │   
         │   SWITCH gate.Op:
         │     CASE XOR:
         │       // Free XOR
         │       wire_labels[gate.Output] = label_a ⊕ label_b
         │       // CPU: 1 XOR instruction (0.3ns)
         │     
         │     CASE AND:
         │       TG, TE = gate.garbled_table
         │       j0 = gate_id * 2
         │       j1 = gate_id * 2 + 1
         │       
         │       // Decrypt generator half
         │       wg = aes_garble.Decrypt(label_a || j0)
         │       if label_a.select_bit:
         │         wg = wg ⊕ TG
         │       
         │       // Decrypt evaluator half
         │       we = aes_garble.Decrypt(label_b || j1)
         │       if label_b.select_bit:
         │         we = we ⊕ TE ⊕ label_a
         │       
         │       wire_labels[gate.Output] = wg ⊕ we
         │       // CPU: 2 AES + 3 XOR (15 cycles)
         │     
         │     CASE OR:
         │       // Point-and-permute
         │       row = (label_a.select_bit << 1) | label_b.select_bit
         │       encrypted = gate.garbled_table[row-1]
         │       wire_labels[gate.Output] = 
         │         DoubleDecrypt(label_a, label_b, gate_id, encrypted)
         │       // CPU: 1 AES (5 cycles)
         │   
         │   // Pipeline: decrypt next gate while CPU computes
         │   
         │   IF gate_id % 1000 == 0:
         │     Progress: gate_id / 45,231 (2.2%, 4.4%, ...)
         │
         │ Total CPU operations:
         │   - XOR: 23,456 × 0.3ns = 7 µs
         │   - AND: 15,234 × 4.3ns = 65 µs
         │   - OR: 4,567 × 1.4ns = 6 µs
         │   - INV: 1,974 × 1.4ns = 3 µs
         │   Total: 81 µs (CPU time)
         │
         │ Wall time: ~600ms
         │   (dominated by network receive, not CPU!)
         └────────────────────────────┘

T=1650ms ┌─── Bob→Alice: Send output ───┐
         │ output_labels = []
         │ For i in 0..2567:
         │   output_wire = 47797 + i
         │   output_labels.append(wire_labels[output_wire])
         │
         │ For label in output_labels:
         │   encrypted = AES_BA.Encrypt(label)
         │   TCP_Send(alice_socket, encrypted)
         │
         │ Data: 41,088 bytes
         │ Time: ~50ms
         └───────────────────────────────┘

T=1700ms ┌─── Alice: Decode result ───┐
         │ received_labels = Receive()
         │
         │ output_bits = [0] * 2568
         │ For i in 0..2567:
         │   received = received_labels[i]
         │   expected_L0 = L0[47797+i]
         │   expected_L1 = L1[47797+i]
         │   
         │   // Constant-time comparison (16 bytes)
         │   if memcmp(received, expected_L0, 16) == 0:
         │     output_bits[i] = 0
         │   elif memcmp(received, expected_L1, 16) == 0:
         │     output_bits[i] = 1
         │   else:
         │     PANIC("Invalid label!")
         │
         │ // Decode bits to strings
         │ sorted_array = []
         │ For string_idx in 0..4:
         │   bytes = []
         │   For char_idx in 0..31:
         │     byte = 0
         │     For bit_idx in 0..7:
         │       bit_pos = string_idx*256 + char_idx*8 + bit_idx
         │       byte |= (output_bits[bit_pos] << bit_idx)
         │     if byte != 0:  // Ignore padding
         │       bytes.append(chr(byte))
         │   sorted_array.append(''.join(bytes))
         │
         │ Result: ["apple", "banana", "cherry", "mango", "zebra"]
         │ ✓ Sorted correctly!
         │
         │ Verification:
         │   strcmp("apple", "banana") = -1 ✓
         │   strcmp("banana", "cherry") = -1 ✓
         │   strcmp("cherry", "mango") = -1 ✓
         │   strcmp("mango", "zebra") = -1 ✓
         └────────────────────────────┘

T=1705ms ┌─── P2P Channel Close ───┐
         │ Alice: TCP_Close(bob_socket)
         │ Bob: TCP_Close(alice_socket)
         │
         │ Cleanup:
         │   - Free wire_labels memory (1.5 MB)
         │   - Free circuit memory (156 KB)
         │   - Close AES contexts
         │   - Unsubscribe from BlossomSub topic
         └─────────────────────────┘

═══════════════════════════════════════
Total Execution Time: 1.7 seconds
═══════════════════════════════════════

Performance Breakdown:
  Setup & Circuit Load:      250ms  (15%)
  Garbling & Streaming:      800ms  (47%)
  Evaluation:                600ms  (35%)
  Result Decoding:            50ms  (3%)

Data Transfer:
  Alice → Bob:  793 KB
  Bob → Alice:   41 KB
  Total:        834 KB

CPU Usage:
  Alice (Garbler):    ~80 µs CPU, 1050ms wall (mostly I/O)
  Bob (Evaluator):    ~80 µs CPU,  600ms wall (mostly I/O)

Privacy Achieved:
  ✓ Bob cannot determine original array order
  ✓ Network nodes cannot see computation details
  ✓ Only encrypted data transmitted
  ✓ No plaintext storage in hypergraph
```

##### 5.9 & 6.7 结果收集与返回

```
结果传递链:
==========

Bob (Evaluator)
  ├─ Computed: output_labels[0..2567]
  ├─ Encrypted: AES_BA(output_labels)
  └─ Send: TCP → Node B → Alice
            │
            ▼
Alice (Garbler)
  ├─ Received: encrypted_output_labels
  ├─ Decrypted: labels
  ├─ Decoded: sorted_array = ["apple", "banana", "cherry", "mango", "zebra"]
  └─ Verified: ✓

可选: 链上记录结果
──────────────────
Alice创建ComputeFinalize交易:
  {
    execution_id: 0x9b8a7c6d...,
    result_commitment: SHA3(sorted_array),
    proof: Optional(ZK-SNARK of correct execution),
    signature: Ed448(Alice_PrivKey, [above])
  }

广播到网络:
  Alice → Node A → All Nodes (P2P gossip)
  
共识后写入Hypergraph:
  NEW VERTEX:
    Domain: COMPUTE_INTRINSIC_DOMAIN
    Address: SHA3(execution_id || result_commitment)
    Data: {
      "execution_ref": "0x9b8a7c6d...",
      "result_commitment": "0x8a7b6c5d...",
      "timestamp": 1729425896,
      "verified": true
    }
  
  Shard owner (Node B Worker 1):
    └─ Persists to PebbleDB

用户查询结果:
─────────────
// Alice本地已有明文结果

// 第三方验证 (需要Alice授权):
client.getExecutionResult("0x9b8a7c6d...")
  → Returns: {
      commitment: "0x8a7b6c5d...",
      status: "completed",
      timestamp: 1729425896
    }
  
  // 明文不在链上，需要Alice主动分享

// Alice可以选择性披露:
client.shareResult("0x9b8a7c6d...", 
                   recipient_pubkey,
                   encrypted_result)
```

---

## 4. 系统架构总结

### 关键组件职责

```
┌────────────────────────────────────────────────────────┐
│ Component         │ Responsibility                     │
├────────────────────────────────────────────────────────┤
│ QCL Compiler      │ Source code → Boolean circuit     │
│ (Bedlam)          │ Optimization, SSA transformation  │
├────────────────────────────────────────────────────────┤
│ Hypergraph (CRDT) │ Code storage, Metadata registry   │
│                   │ NOT computation execution         │
├────────────────────────────────────────────────────────┤
│ Blockchain        │ Consensus, Ordering, Payment      │
│ (Master Process)  │ NOT data storage or computation   │
├────────────────────────────────────────────────────────┤
│ Workers           │ Shard-specific state management   │
│ (Cores 1+)        │ Hypergraph persistence            │
├────────────────────────────────────────────────────────┤
│ BlossomSub (P2P)  │ Message routing, Topic gossip     │
│                   │ Encrypted channel relay           │
├────────────────────────────────────────────────────────┤
│ Garbled Circuits  │ Privacy-preserving computation    │
│ (Alice & Bob)     │ Off-chain, Direct P2P             │
├────────────────────────────────────────────────────────┤
│ FERRET OT         │ Secure input label transfer       │
│                   │ Oblivious to sender's view        │
├────────────────────────────────────────────────────────┤
│ Bulletproofs      │ Payment proofs (range proofs)     │
│                   │ NOT for computation verification  │
└────────────────────────────────────────────────────────┘
```

### 数据流图

```
Code Lifecycle:
───────────────
Developer → QCL → Bedlam Compiler → Circuit Binary
                                      ↓
                                  Hypergraph (persistent)
                                      ↓
                              Retrieved by Garbler
                                      ↓
                                Used in MPC execution

Computation Lifecycle:
──────────────────────
User → CodeExecute Tx → Blockchain (consensus)
                            ↓
                        Hypergraph (metadata)
                            ↓
                        P2P Discovery (rendezvous)
                            ↓
                    Alice ↔ Bob (MPC off-chain)
                            ↓
                    Result → Alice (local)
                            ↓
                    (Optional) → Blockchain (commitment)

Data Location:
──────────────
┌──────────────┬─────────────────────────────────────┐
│ Phase        │ Where is Data?                      │
├──────────────┼─────────────────────────────────────┤
│ Development  │ Developer's machine (QCL source)    │
├──────────────┼─────────────────────────────────────┤
│ Deployment   │ Hypergraph (circuit binary)         │
│              │ All shard-owning nodes               │
├──────────────┼─────────────────────────────────────┤
│ Request      │ Blockchain (tx metadata + payment)  │
│              │ User's machine (input data)          │
├──────────────┼─────────────────────────────────────┤
│ Execution    │ P2P channel (encrypted)             │
│              │ Alice & Bob's memory (labels)        │
├──────────────┼─────────────────────────────────────┤
│ Result       │ Alice's machine (plaintext)         │
│              │ Blockchain (optional commitment)     │
└──────────────┴─────────────────────────────────────┘
```

---

## 5. 关键发现

### 发现1: 三层分离架构

```
存储层 (Storage)     ←→  Hypergraph + PebbleDB
  • 代码持久化
  • 元数据注册
  • CRDT同步

共识层 (Consensus)   ←→  Blockchain + VDF
  • 交易排序
  • 支付验证
  • 状态转换

计算层 (Compute)     ←→  Garbled Circuits + OT
  • 隐私计算
  • P2P直连
  • 链外执行

这三层互相独立但协同工作!
```

### 发现2: 超图不是查询引擎

```
常见误解:
  "超图像数据库，查询执行计算"
  
实际情况:
  超图 = 版本化键值存储 + CRDT
  用途 = 存储代码和元数据
  查询 = GetVertex/GetHyperedge (简单读取)
  
计算发生在:
  MPC参与者之间 (Alice ↔ Bob)
  使用乱码电路协议
  完全在链外!
```

### 发现3: OT的精确作用

```
OT不是:
  ✗ 查询超图的方式
  ✗ 计算的执行方式
  ✗ 数据库检索协议
  
OT是:
  ✓ MPC中的安全输入传输
  ✓ Bob选择labels，Alice不知道选了哪个
  ✓ 乱码电路的必要组件
  
场景:
  Alice有: (L0, L1) 两个labels
  Bob有: bit ∈ {0,1} 他的输入位
  OT让: Bob获得L_bit，Alice不知道bit
```

### 发现4: 节点不执行MPC

```
3节点网络中:
  Node A, B, C:
    ✓ 存储代码和元数据
    ✓ 达成共识
    ✓ 路由P2P消息
    ✗ 不执行MPC计算
    ✗ 不看到明文数据
    ✗ 不知道计算结果
  
  Alice & Bob:
    ✓ 可能运行在节点上，也可能是独立客户端
    ✓ 执行MPC (直接P2P通道)
    ✓ 看到计算结果
    
节点提供基础设施，用户执行计算!
```

### 发现5: 隐私保证的多层次

```
Level 1: 网络层隐私
  • BlossomSub只看到加密消息
  • 节点无法解密P2P通道
  
Level 2: 协议层隐私
  • 乱码电路隐藏执行路径
  • OT隐藏输入选择
  
Level 3: 应用层隐私
  • 输入数据不上链
  • 结果只有参与者知道
  • 可选择性披露
  
Level 4: 存储层隐私
  • 代码是公开的 (circuit binary)
  • 元数据是公开的 (execution record)
  • 输入/输出是私密的 (不在链上)
```

### 发现6: 性能瓶颈

```
从原子操作分析可见:

CPU时间: 
  • Garbling: ~80µs (可忽略)
  • Evaluation: ~80µs (可忽略)
  
网络时间:
  • Circuit transfer: ~800ms (主要瓶颈!)
  • Label transfer: ~100ms
  
优化方向:
  1. 电路大小优化 (编译器改进)
  2. 流式传输 (已实现)
  3. 网络压缩 (可考虑)
  4. Free-XOR最大化 (编译器优化)
  
结论: 当前系统是网络绑定，不是CPU绑定
```

### 发现7: 数据不离开用户

```
传统云计算:
  用户 → 上传数据 → 云服务器 → 计算 → 返回结果
       (数据暴露!)

Quilibrium MPC:
  Alice → 保留数据本地 → 编码为labels → MPC → 结果
         (数据从不上传!)
  
  Bob → 保留数据本地 → OT获取labels → MPC → 结果
       (数据从不上传!)
  
链上只有:
  • 代码hash
  • 执行请求
  • 支付证明
  • (可选) 结果承诺
  
链上没有:
  ✗ 输入数据
  ✗ 中间值
  ✗ 明文结果
```

---

## 6. 总结与展望

### 系统优势

```
1. 真正的去中心化
   • 无可信第三方
   • P2P直接计算
   • 节点只提供基础设施

2. 完整的隐私保护
   • 输入隐私 (OT)
   • 计算隐私 (乱码电路)
   • 输出隐私 (选择性披露)

3. 通用计算能力
   • 任意布尔电路
   • 类C语言编程
   • 支持复杂算法

4. 经济激励
   • Token支付
   • 费用市场
   • 可持续运营

5. 可审计性
   • 链上元数据
   • 执行记录
   • (可选) 零知识证明
```

### 适用场景

```
✓ 隐私竞拍
✓ 安全投票
✓ 联合统计
✓ 私密匹配
✓ 医疗数据分析
✓ 金融风控
✓ 机器学习推理
```

### 技术限制

```
⚠ 电路大小
  • 复杂算法 → 大电路 → 慢
  • 目前适合中等复杂度计算

⚠ 双方在线
  • 需要Alice和Bob同时在线
  • 不支持离线计算

⚠ 诚实但好奇模型
  • 当前实现假设参与者遵守协议
  • 未防御主动攻击者
  • (可扩展为恶意安全)
```

### 未来方向

```
1. 性能优化
   • SIMD并行化
   • GPU加速
   • 更优的编译器

2. 安全增强
   • 恶意安全模型
   • ZK-SNARK验证
   • 防止中止攻击

3. 功能扩展
   • 多方MPC (>2)
   • 离线/在线混合
   • 状态持久化

4. 开发者工具
   • IDE集成
   • 调试器
   • 性能分析器
```

---

## 附录：快速参考

### 命令行操作

```bash
# 编译电路
$ bedlam compile program.qcl -o program.circ

# 部署代码
$ qclient compute deploy \
    --code program.circ \
    --domain 0x$(openssl rand -hex 32) \
    --payment 1000QUIL

# 执行计算
$ qclient compute execute \
    --code-address 0x7f3a9c... \
    --inputs inputs.json \
    --payment 50QUIL

# 查询结果
$ qclient compute result 0x9b8a7c...
```

### 关键文件位置

```
bedlam/compiler/compiler.go    - QCL编译器
bedlam/circuit/garble.go       - Garbling实现
bedlam/circuit/eval.go         - Evaluation实现
bedlam/ot/ferret.go            - FERRET OT
hypergraph/hypergraph.go       - 超图CRDT
node/execution/intrinsics/compute/ - Compute内置操作
client/cmd/                    - 客户端命令
```

### 核心数据结构

```go
// 电路
type Circuit struct {
    NumGates int
    NumWires int
    Gates    []Gate
    Inputs   []IOArg
    Outputs  []IOArg
}

// 门
type Gate struct {
    Op      Operation  // XOR, AND, OR, INV
    Input0  Wire
    Input1  Wire
    Output  Wire
}

// 标签
type Label [16]byte  // 128-bit
```

这份文档全面回答了您的所有问题！🎉

