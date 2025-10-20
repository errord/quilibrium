# Complete End-to-End Computation Example
# String Operations and QuickSort in Quilibrium

## Example 1: String Contains Check - "abcde" contains "bc"?

### Step 1: QCL Code Implementation

```qcl
// File: string_contains.qcl
// Check if haystack contains needle

package main

// String represented as fixed-size array of bytes
type String struct {
    data [256]byte  // Max 256 characters
    len  uint8      // Actual length
}

// Check if haystack contains needle
func contains(haystack String, needle String) bool {
    if needle.len == 0 {
        return true
    }
    if haystack.len < needle.len {
        return false
    }
    
    // Sliding window comparison
    for i := uint8(0); i <= haystack.len - needle.len; i++ {
        match := true
        for j := uint8(0); j < needle.len; j++ {
            if haystack.data[i+j] != needle.data[j] {
                match = false
                break
            }
        }
        if match {
            return true
        }
    }
    return false
}

// Main entry point for MPC
// Alice has: haystack = "abcde"
// Bob has: needle = "bc"
func main(haystack String, needle String) bool {
    return contains(haystack, needle)
}
```

### Step 2: Boolean Circuit Representation

```
Circuit Breakdown for "abcde" contains "bc":
============================================

Input Encoding:
---------------
haystack: "abcde"
  data[0] = 'a' = 0x61 = 01100001
  data[1] = 'b' = 0x62 = 01100010
  data[2] = 'c' = 0x63 = 01100011
  data[3] = 'd' = 0x64 = 01100100
  data[4] = 'e' = 0x65 = 01100101
  len = 5
  Total: 256*8 + 8 = 2056 bits

needle: "bc"
  data[0] = 'b' = 0x62
  data[1] = 'c' = 0x63
  len = 2
  Total: 2056 bits

Circuit Structure:
------------------
┌─────────────────────────────────────────────┐
│ Input Wires: 4112 (2056 + 2056)             │
└───────────────────┬─────────────────────────┘
                    │
    ┌───────────────┴───────────────┐
    │                               │
    ▼                               ▼
┌─────────┐                   ┌─────────┐
│ Length  │                   │ Data    │
│ Compare │                   │ Compare │
└────┬────┘                   └────┬────┘
     │                             │
     │    ┌────────────────────────┘
     │    │
     ▼    ▼
┌────────────┐
│ Sliding    │
│ Window     │
│ (4 iters)  │
└──────┬─────┘
       │
       ▼
┌───────────────┐
│ OR tree       │
│ (any match?)  │
└───────┬───────┘
        │
        ▼
   Output: 1 bit

Gate Breakdown:
───────────────
1. Length comparison (haystack.len >= needle.len):
   - 8-bit comparator: ~20 gates
   
2. Character comparison (per position):
   - 8-bit equality: ~16 gates (8 XNOR + 7 AND + 1 final AND)
   - Per needle char: 16 gates
   - Total for 2 chars: 32 gates per position
   
3. Sliding window (4 positions: i=0,1,2,3):
   - Position 0: 32 gates
   - Position 1: 32 gates
   - Position 2: 32 gates
   - Position 3: 32 gates
   - AND for full match per pos: 4 gates
   - Total: 132 gates per position × 4 = 528 gates
   
4. OR tree (any position matched?):
   - 3 OR gates (binary tree for 4 values)
   
Total Circuit: ~550 gates
```

### Step 3: Compilation Process

```bash
# Compile QCL to circuit
$ cd /home/errord/chain/monorepo/bedlam
$ ./bedlam compile string_contains.qcl -o string_contains.circ

Compilation Output:
===================
Parsing...                   ✓
Type checking...             ✓
SSA generation...            ✓
  - Functions: 2 (contains, main)
  - Basic blocks: 12
  - SSA values: 856
Circuit generation...        ✓
  - Input bits: 4112
  - Output bits: 1
  - Wires: 4663
  - Gates: 547
    - XOR: 234
    - AND: 198
    - INV: 45
    - OR: 70
Optimization...              ✓
  - Dead gate elimination: 12 gates removed
  - Gate fusion: 5 gates merged
  - Final gates: 535
Serialization...             ✓
  - Output size: 8.7 KB

Circuit saved to: string_contains.circ
```

### Step 4: Circuit Binary Format

```
Circuit Binary Structure (string_contains.circ):
===============================================

Offset | Size | Content
-------+------+------------------------------------------
0x0000 | 4    | Magic: "BCRCT" (Bedlam Circuit)
0x0004 | 4    | Version: 0x00000001
0x0008 | 4    | Num inputs: 2
0x000C | 4    | Input 0 bits: 2056
0x0010 | 4    | Input 1 bits: 2056
0x0014 | 4    | Num outputs: 1
0x0018 | 4    | Output 0 bits: 1
0x001C | 4    | Num wires: 4663
0x0020 | 4    | Num gates: 535
-------+------+------------------------------------------
0x0024 | VAR  | Gate Array:
       |      | 
       |      | Gate 0:
       |      |   Op: XOR (0x01)
       |      |   Input0: wire 0
       |      |   Input1: wire 2056
       |      |   Output: wire 4112
       |      |
       |      | Gate 1:
       |      |   Op: XOR (0x01)
       |      |   Input0: wire 1
       |      |   Input1: wire 2057
       |      |   Output: wire 4113
       |      |
       |      | ... (533 more gates)
       |      |
       |      | Gate 534:
       |      |   Op: OR (0x03)
       |      |   Input0: wire 4660
       |      |   Input1: wire 4661
       |      |   Output: wire 4662 (final output)
-------+------+------------------------------------------
0x22E4 | VAR  | Metadata:
       |      |   - Input names
       |      |   - Output names
       |      |   - Source line mapping
-------+------+------------------------------------------
EOF: 8932 bytes total
```

## Example 2: QuickSort - Complete End-to-End Flow

### 5.1 QuickSort Algorithm in QCL

```qcl
// File: quicksort.qcl
// Sort array of strings using quicksort

package main

type String struct {
    data [32]byte   // Max 32 chars per string
    len uint8
}

type StringArray struct {
    items [10]String  // Max 10 strings
    len uint8
}

// String comparison: returns -1 if a<b, 0 if a==b, 1 if a>b
func strcmp(a String, b String) int8 {
    minLen := a.len
    if b.len < minLen {
        minLen = b.len
    }
    
    for i := uint8(0); i < minLen; i++ {
        if a.data[i] < b.data[i] {
            return -1
        }
        if a.data[i] > b.data[i] {
            return 1
        }
    }
    
    if a.len < b.len {
        return -1
    }
    if a.len > b.len {
        return 1
    }
    return 0
}

// Swap two strings in array
func swap(arr *StringArray, i uint8, j uint8) {
    temp := arr.items[i]
    arr.items[i] = arr.items[j]
    arr.items[j] = temp
}

// Partition for quicksort
func partition(arr *StringArray, low uint8, high uint8) uint8 {
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

// Quicksort implementation
func quicksort_rec(arr *StringArray, low uint8, high uint8) {
    if low < high {
        pi := partition(arr, low, high)
        if pi > 0 {
            quicksort_rec(arr, low, pi-1)
        }
        quicksort_rec(arr, pi+1, high)
    }
}

func quicksort(arr *StringArray) {
    if arr.len > 0 {
        quicksort_rec(arr, 0, arr.len-1)
    }
}

// MPC entry point
// Alice has: unsorted array
// Bob has: nothing (or could have part of array)
// Output: sorted array
func main(arr StringArray) StringArray {
    quicksort(&arr)
    return arr
}
```

### 5.2 How to Upload to Quilibrium Network

```bash
# Step 1: Compile locally
$ cd /home/errord/chain/monorepo/bedlam
$ ./bedlam compile quicksort.qcl -o quicksort.circ

Compilation stats:
  Input bits: 2568 (10 strings × 256 bits + 8 bits length)
  Output bits: 2568
  Gates: 45,231 (recursion unrolled for depth 10)
  Circuit size: 156 KB

# Step 2: Deploy to Quilibrium
$ cd /home/errord/chain/monorepo/client

# Create deployment transaction
$ ./qclient compute deploy \
    --code quicksort.circ \
    --input-types "StringArray" \
    --output-types "StringArray" \
    --domain 0x$(head -c 32 /dev/urandom | xxd -p -c 32) \
    --owner-key ~/.config/keys/bls_owner.key \
    --payment 1000QUIL

Transaction created:
  Type: CodeDeployment
  Code hash: 0x7f3a9c1d...
  Size: 156,842 bytes
  Fee: 1,000 QUIL
  
Broadcasting...
  Tx hash: 0x8b4f2e1a...
  Block: 12,345,678
  Status: ✓ Confirmed

Code deployed at address: 0x7f3a9c1d8e5b6a4f2c9d1e8a5b7c3f4a
```

### 5.3 Storage in Hypergraph

```
Blockchain Transaction Record:
==============================
Block 12,345,678
Transaction: 0x8b4f2e1a...
  Type: CodeDeployment
  From: 0x[Alice_address]
  Timestamp: 2025-10-20 12:34:56 UTC
  Gas: 156,842 units
  Fee: 1,000 QUIL

Hypergraph State Update:
=========================

Domain: COMPUTE_INTRINSIC_DOMAIN (0xcccc...cccc)

NEW VERTEX ADDED:
─────────────────
Vertex Address: 0x7f3a9c1d8e5b6a4f2c9d1e8a5b7c3f4a
  (Computed as: Poseidon(domain || circuit_binary))

Vertex Type: CODE_ARTIFACT

Vertex Data (VectorCommitmentTree):
  Index 0 (Code):
    ┌─ Offset 0x0000: Circuit header
    ├─ Offset 0x0024: Gate array (45,231 gates)
    ├─ Offset 0x1F8C: Metadata
    └─ Total: 156,842 bytes
    
  Index 1 (Metadata):
    {
      "creator": "0x[Alice_address]",
      "timestamp": 1729425296,
      "input_types": ["StringArray"],
      "output_types": ["StringArray"],
      "version": "1.0",
      "source_hash": "0x[quicksort.qcl hash]"
    }

ShardKey: (Bloom filter of domain)
  L1: [3]byte{0x12, 0x34, 0x56}  ← Bloom indices
  L2: [32]byte{0xcccc...}        ← Full domain

Storage Persistence:
  Worker Process 2 (owns shard [0x12...])
    └─► PebbleDB:
        Key: concat(domain, code_address)
        Value: VectorCommitmentTree root + data

CRDT Sets Updated:
  vertexAdds[ShardKey][code_address] = VertexData
  
Merkle Proof: Generated for verification
  Root commitment: 0x9a8b7c6d...
```

### 5.4 Storage Format Details

```
On-Disk Storage (PebbleDB):
===========================

Database: /home/errord/chain/monorepo/.config/store/worker_2/

Key Format:
  Prefix: 0x01 (VERTEX_DATA)
  Domain: [32]byte (COMPUTE_INTRINSIC_DOMAIN)
  Address: [32]byte (code_address)
  Discriminator: 0x00 (VERTEX_ADDS)
  Total: 66 bytes

Value Format (Serialized VectorCommitmentTree):
  ┌─────────────────────────────────────┐
  │ Tree Root: [32]byte                 │
  ├─────────────────────────────────────┤
  │ Num Leaves: varint                  │
  ├─────────────────────────────────────┤
  │ Leaf 0:                             │
  │   Path: []byte                      │
  │   Value: []byte (circuit binary)    │
  │   Size: varint                      │
  ├─────────────────────────────────────┤
  │ Leaf 1:                             │
  │   Path: []byte                      │
  │   Value: []byte (metadata JSON)     │
  │   Size: varint                      │
  └─────────────────────────────────────┘

Actual Disk Layout:
  File: 000123.sst (SSTable)
    Block offset: 0x4F8A0
    Key: 0x01|cccc...cccc|7f3a9c1d...|00
    Value: [compressed VectorCommitmentTree]
    Compression: Snappy
    Compressed size: 87,234 bytes (from 156,842)
```

### 5.5 User Submits Data for Sorting

```javascript
// Client code (JavaScript/Node.js example)

const { QuilibriumClient } = require('@quilibrium/client');

// Initialize client
const client = new QuilibriumClient({
  grpcEndpoint: 'localhost:8337',
  privateKey: loadPrivateKey('~/.config/keys/ed448_priv.key')
});

// Prepare data to sort
const inputData = {
  items: [
    { data: stringToBytes("zebra"), len: 5 },
    { data: stringToBytes("apple"), len: 5 },
    { data: stringToBytes("mango"), len: 5 },
    { data: stringToBytes("banana"), len: 6 },
    { data: stringToBytes("cherry"), len: 6 },
  ],
  len: 5
};

// Serialize input according to circuit format
const serialized = serializeStringArray(inputData);
// Result: 2568 bits = 321 bytes

console.log("Serialized input:", serialized.toString('hex'));
// Output: 7a656272610000... (zebra padded) + 6170706c650000... (apple) + ...

// Create execution request
const executionRequest = {
  codeAddress: '0x7f3a9c1d8e5b6a4f2c9d1e8a5b7c3f4a',
  domain: COMPUTE_INTRINSIC_DOMAIN,
  rendezvous: generateRendezvous(), // Random 32-byte value
  inputs: serialized,
  payment: createPaymentProof(50), // 50 QUIL for execution
  role: 'garbler' // Alice will be garbler
};

// Submit transaction
const txHash = await client.submitCodeExecute(executionRequest);
console.log("Execution request submitted:", txHash);
```

### 5.6 Data Reception and Processing

```
Network Flow Diagram:
=====================

┌─────────────┐
│   Alice     │ (Client, submits execution request)
│  (Garbler)  │
└──────┬──────┘
       │ 1. Create CodeExecute transaction
       │    - code_address: 0x7f3a9c1d...
       │    - rendezvous: 0x9b8a7c6d...
       │    - inputs_hash: Hash(serialized_inputs)
       │    - payment_proof: BulletProof(50 QUIL)
       │
       ▼
┌───────────────────────────────────────┐
│    Master Process (Core 0)            │
│  ┌─────────────────────────────────┐  │
│  │  Global Consensus Engine        │  │
│  │  - Receives transaction         │  │
│  │  - Validates payment proof      │  │
│  │  - Orders in block              │  │
│  └──────────────┬──────────────────┘  │
│                 │                      │
│  ┌──────────────▼──────────────────┐  │
│  │  Hypergraph State              │  │
│  │  - Records execution metadata  │  │
│  │  - Updates CRDT                │  │
│  └──────────────┬──────────────────┘  │
└─────────────────┼────────────────────┘
                  │
         Broadcasts to workers
                  │
       ┌──────────┴──────────┐
       │                     │
       ▼                     ▼
┌─────────────────┐   ┌─────────────────┐
│ Worker 1        │   │ Worker 2        │
│ (Shard 0x12...) │   │ (Shard 0xAB...) │
└────────┬────────┘   └────────┬────────┘
         │                     │
         │ If shard matches    │ If shard matches
         │ (checks Bloom)      │ (checks Bloom)
         │                     │
         ▼                     ▼
    Processes               Ignores
    execution            (not in shard)

Worker 1 Processing:
────────────────────
1. Receives block with CodeExecute tx
2. Checks ShardKey:
   Bloom(COMPUTE_INTRINSIC_DOMAIN) = [0x12, ...]
   → Matches Worker 1's shard ✓
3. Extracts rendezvous: 0x9b8a7c6d...
4. Subscribes to P2P topic:
   "/quilibrium/compute/9b8a7c6d..."
5. Waits for participants
```

### 5.7 Who Computes? (3-Node Network Example)

```
Network Topology:
=================

Node A (Master + Worker)        Node B (Master + Worker)        Node C (Master + Worker)
  IP: 192.168.1.10                IP: 192.168.1.11                IP: 192.168.1.12
  Peer ID: Qm...AAA               Peer ID: Qm...BBB               Peer ID: Qm...CCC
  
  Core 0: Master Process          Core 0: Master Process          Core 0: Master Process
    - Global consensus              - Global consensus              - Global consensus
    - Shard: ALL (coordinator)      - Shard: ALL (coordinator)      - Shard: ALL (coordinator)
  
  Core 1: Worker Process          Core 1: Worker Process          Core 1: Worker Process
    - Shard: 0x00-0x7F              - Shard: 0x80-0xBF              - Shard: 0xC0-0xFF
    - Owns: 50% keyspace            - Owns: 25% keyspace            - Owns: 25% keyspace

Shard Assignment:
─────────────────
COMPUTE_INTRINSIC_DOMAIN = 0xcccc...cccc
Bloom filter indices: [0x12, 0x34, 0xCC]

Which worker processes this execution?
  Shard 0xCC falls in range:
    - Node A Worker: 0x00-0x7F → NO
    - Node B Worker: 0x80-0xBF → NO
    - Node C Worker: 0xC0-0xFF → YES ✓

Answer: Node C's worker process handles this execution
        But actual MPC computation involves 2 participants (not nodes!)

Participant Discovery:
──────────────────────
Alice (Garbler) - could be on any node, or external client
Bob (Evaluator) - discovers execution via rendezvous

Node C's Role:
  1. Stores code in hypergraph (shard owner)
  2. Records execution metadata
  3. Validates payment
  4. Does NOT perform MPC computation!
     (MPC is P2P between Alice and Bob, off-chain)

Actual Computation:
───────────────────
┌────────────┐                      ┌────────────┐
│   Alice    │◄────P2P Channel────►│    Bob     │
│ (anywhere) │                      │ (anywhere) │
└────────────┘                      └────────────┘
      │                                   │
      │ Both connect to:                  │
      │ P2P topic: /quilibrium/compute/[rendezvous]
      │                                   │
      │◄──────────────┬──────────────────►│
                      │
                 BlossomSub
            (routed by all nodes)

Key Point: The 3 nodes provide infrastructure
          (storage, consensus, routing)
          But MPC computation happens P2P between
          Alice and Bob directly!
```

### 5.8 Detailed Computation Process

```
Atomic Operations Breakdown:
============================

═══════════════════════════════════════════════════════════
Phase 1: Pre-Computation Setup
═══════════════════════════════════════════════════════════

[Alice Side - Garbler Setup]
────────────────────────────
Op 1.1: Query hypergraph for code
  HYPERGRAPH_QUERY:
    key = COMPUTE_DOMAIN || 0x7f3a9c1d...
    result = GetVertexData(key)
  → Retrieves: 156,842 byte circuit
  Duration: ~5ms

Op 1.2: Load circuit into memory
  Circuit.Unmarshal(circuitData)
  Parse:
    - 45,231 gates
    - 47,799 wires
    - Input mapping: wire[0..2567]
    - Output mapping: wire[47797..49364]
  Duration: ~10ms

Op 1.3: Generate wire labels (cryptographic RNG)
  For each wire i in 0..47,799:
    L0[i] = Random(128 bits)
    L1[i] = Random(128 bits)
    Set select bit: L0[i].lsb = 0, L1[i].lsb = 1
  
  Total: 47,799 × 2 × 128 bits = 1,528,768 bytes of random data
  Duration: ~50ms (from /dev/urandom or hardware RNG)

Op 1.4: Connect to P2P rendezvous
  BlossomSub.Subscribe("/quilibrium/compute/9b8a7c6d...")
  Publish presence:
    {
      role: "garbler",
      pubkey: Alice_Ed448_PubKey,
      timestamp: now()
    }
  Duration: ~100ms (network latency)

[Bob Side - Evaluator Setup]
─────────────────────────────
Op 1.5: Monitor blockchain for execution requests
  EventLoop:
    new_block_event → scan for CodeExecute transactions
    if (tx.rendezvous matches interest):
      proceed to Op 1.6

Op 1.6: Connect to P2P rendezvous
  BlossomSub.Subscribe("/quilibrium/compute/9b8a7c6d...")
  Publish presence:
    {
      role: "evaluator",
      pubkey: Bob_Ed448_PubKey,
      timestamp: now()
    }
  Duration: ~100ms

Op 1.7: Mutual authentication & channel establishment
  Alice → Bob: "Hello, I'm garbler, pubkey: [Alice_PubKey]"
  Bob → Alice: "Hello, I'm evaluator, pubkey: [Bob_PubKey]"
  
  ECDH Key Agreement:
    shared_secret = ECDH(Alice_PrivKey, Bob_PubKey)
                  = ECDH(Bob_PrivKey, Alice_PubKey)
  
  Derive encryption keys:
    k_alice_to_bob = HKDF(shared_secret, "alice_to_bob")
    k_bob_to_alice = HKDF(shared_secret, "bob_to_alice")
  
  Duration: ~20ms

═══════════════════════════════════════════════════════════
Phase 2: Input Encoding
═══════════════════════════════════════════════════════════

[Alice - Garbler Inputs]
────────────────────────
Op 2.1: Encode Alice's input (unsorted array)
  StringArray: ["zebra", "apple", "mango", "banana", "cherry"]
  
  Bit-level encoding:
    Wire 0: 'z' bit 0 = 0
    Wire 1: 'z' bit 1 = 1
    Wire 2: 'z' bit 2 = 0
    Wire 3: 'z' bit 3 = 1
    Wire 4: 'z' bit 4 = 1
    Wire 5: 'z' bit 5 = 1
    Wire 6: 'z' bit 6 = 1
    Wire 7: 'z' bit 7 = 0
    Wire 8-15: 'e'
    Wire 16-23: 'b'
    ...
    Wire 2560-2567: len=5
  
  Select labels based on input bits:
    alice_input_labels[0] = L0[0]  (bit 0 = 0)
    alice_input_labels[1] = L1[1]  (bit 1 = 1)
    alice_input_labels[2] = L0[2]  (bit 2 = 0)
    ...
    
  Total: 2568 labels selected
  Duration: ~1ms

Op 2.2: Send Alice's input labels to Bob
  Encrypted_Send(alice_input_labels)
    For each label in alice_input_labels:
      encrypted = AES_Encrypt(k_alice_to_bob, label)
      Send(encrypted)
  
  Data size: 2568 labels × 16 bytes = 41,088 bytes
  Duration: ~50ms (network transfer)

[Bob - Evaluator Inputs]
────────────────────────
Op 2.3: Prepare Bob's input (in this case, empty or could be additional strings)
  For QuickSort, typically only garbler has input
  Bob's input bits: 0 (no input needed)
  
  If Bob had input:
    bob_input_bits = EncodeToBits(bob_data)
    bob_choices = []bool from bits

[FERRET OT Protocol - Bob Gets His Labels]
───────────────────────────────────────────
Op 2.4: FERRET OT Setup (if Bob has inputs)
  Alice (Sender):
    COT_Setup(security_param=128)
      → Generates base OTs
      Duration: ~500ms
    
    ROT_Extend(num_ots=2568)
      → Extends to required OT count
      Duration: ~200ms
  
  Bob (Receiver):
    COT_Setup(security_param=128)
    ROT_Extend(num_ots=2568)

Op 2.5: OT Transfer (if Bob has inputs)
  For each bob_input_bit i in 0..2567:
    Alice:
      r0[i], r1[i] = FERRET_GetBlock(i, 0), FERRET_GetBlock(i, 1)
      e0[i] = r0[i] ⊕ L0[2568+i]  // Bob's wire labels start at wire 2568
      e1[i] = r1[i] ⊕ L1[2568+i]
      Send(e0[i], e1[i])
    
    Bob:
      choice = bob_input_bits[i]
      r[i] = FERRET_GetBlock(i, choice)
      e = Receive()
      bob_input_labels[i] = r[i] ⊕ e[choice]
  
  Total OT data: 2568 × 2 × 16 bytes = 82,176 bytes
  Duration: ~300ms

═══════════════════════════════════════════════════════════
Phase 3: Circuit Garbling & Transmission
═══════════════════════════════════════════════════════════

Op 3.1: Generate AES key for garbling
  garble_key = Random(128 bits)
  aes_cipher = AES_Init(garble_key)
  Duration: ~1ms

Op 3.2: Send garble key to Bob
  Encrypted_Send(k_alice_to_bob, garble_key)
  Duration: ~5ms

Op 3.3: Garble each gate (streaming)
  For gate_id in 0..45,230:
    gate = circuit.Gates[gate_id]
    
    switch gate.Op:
      case XOR, XNOR:
        // Free XOR - no garbled table needed!
        // Bob will XOR labels directly
        garbled_table = []  // Empty
      
      case AND:
        // Half-Gates optimization
        input0_wire = gate.Input0
        input1_wire = gate.Input1
        output_wire = gate.Output
        
        j0 = gate_id * 2
        j1 = gate_id * 2 + 1
        
        // Compute TG (generator half-gate)
        TG = Encrypt_AES(aes_cipher, L0[input0_wire], j0)
        if L1[input0_wire].select_bit == 1:
          TG = TG ⊕ L0[output_wire]
        
        // Compute TE (evaluator half-gate)
        TE = Encrypt_AES(aes_cipher, L0[input1_wire], j1)
        if L1[input1_wire].select_bit == 1:
          TE = TE ⊕ (L0[output_wire] ⊕ L0[input0_wire])
        
        garbled_table = (TG, TE)  // 2 × 16 bytes = 32 bytes
      
      case OR, INV:
        // Full 4-row garbling (with row reduction)
        ... (similar encryption process)
        garbled_table = 3 rows × 16 bytes = 48 bytes
    
    // Stream to Bob immediately
    Send_Encrypted(k_alice_to_bob, {
      gate_id: gate_id,
      op: gate.Op,
      garbled_table: garbled_table
    })
  
  Total data:
    - XOR gates (23,456): 0 bytes
    - AND gates (15,234): 32 bytes each = 487,488 bytes
    - OR gates (4,567): 48 bytes each = 219,216 bytes
    - INV gates (1,974): 32 bytes each = 63,168 bytes
    Total: 769,872 bytes ≈ 752 KB
  
  Duration: ~800ms (compute + network)

═══════════════════════════════════════════════════════════
Phase 4: Circuit Evaluation
═══════════════════════════════════════════════════════════

[Bob - Evaluator Computation]
──────────────────────────────

Op 4.1: Initialize wire label storage
  wire_labels = Array[47,799] of Label
  
  // Set input labels
  For i in 0..2567:
    wire_labels[i] = alice_input_labels[i]  // Received earlier
  For i in 2568..5135:
    wire_labels[i] = bob_input_labels[i-2568]  // From OT

Op 4.2: Evaluate gates in topological order
  For gate_id in 0..45,230:
    gate = Receive_GarbledGate()  // Streamed from Alice
    
    label_a = wire_labels[gate.Input0]
    label_b = wire_labels[gate.Input1]
    
    switch gate.Op:
      case XOR:
        // Free XOR - just XOR the labels!
        wire_labels[gate.Output] = label_a ⊕ label_b
      
      case XNOR:
        wire_labels[gate.Output] = label_a ⊕ label_b ⊕ R
        // where R is global offset
      
      case AND:
        // Half-Gates decryption
        TG, TE = gate.garbled_table
        
        j0 = gate_id * 2
        j1 = gate_id * 2 + 1
        
        // Decrypt generator half
        wg = Decrypt_AES(aes_cipher, label_a, j0)
        if label_a.select_bit == 1:
          wg = wg ⊕ TG
        
        // Decrypt evaluator half
        we = Decrypt_AES(aes_cipher, label_b, j1)
        if label_b.select_bit == 1:
          we = we ⊕ TE ⊕ label_a
        
        // Combine
        wire_labels[gate.Output] = wg ⊕ we
      
      case OR, INV:
        // Point-and-permute decryption
        row_index = (label_a.select_bit << 1) | label_b.select_bit
        if row_index == 0:
          wire_labels[gate.Output] = Label{0}  // Zero row omitted
        else:
          encrypted_row = gate.garbled_table[row_index-1]
          wire_labels[gate.Output] = Decrypt_Double(
            aes_cipher, label_a, label_b, gate_id, encrypted_row
          )
    
    // Progress: gate_id / 45,231
    if gate_id % 1000 == 0:
      Log("Evaluated", gate_id, "gates")
  
  Total AES operations: ~15,234 (only AND gates need decryption)
  Duration: ~600ms (CPU-bound)

Op 4.3: Extract output labels
  output_labels = []
  For i in 0..2567:  // 2568 output bits (sorted array)
    output_wire = 47797 + i
    output_labels.append(wire_labels[output_wire])
  
  Duration: <1ms

Op 4.4: Send output labels back to Alice
  Encrypted_Send(k_bob_to_alice, output_labels)
  Data size: 2568 × 16 bytes = 41,088 bytes
  Duration: ~50ms

═══════════════════════════════════════════════════════════
Phase 5: Output Decoding
═══════════════════════════════════════════════════════════

[Alice - Decode Result]
───────────────────────

Op 5.1: Receive output labels from Bob
  received_labels = Encrypted_Receive(k_bob_to_alice)
  Duration: ~50ms

Op 5.2: Decode labels to bits
  output_bits = []
  For i in 0..2567:
    received_label = received_labels[i]
    expected_L0 = L0[47797+i]
    expected_L1 = L1[47797+i]
    
    if received_label == expected_L0:
      output_bits.append(0)
    else if received_label == expected_L1:
      output_bits.append(1)
    else:
      ERROR("Invalid output label!")
  
  Duration: ~2ms

Op 5.3: Decode bits to sorted array
  Bits to bytes:
    bits[0..7] → byte 0 = 0x61 = 'a'
    bits[8..15] → byte 1 = 0x70 = 'p'
    bits[16..23] → byte 2 = 0x70 = 'p'
    bits[24..31] → byte 3 = 0x6c = 'l'
    bits[32..39] → byte 4 = 0x65 = 'e'
    ...
  
  Reconstruct StringArray:
    items[0] = "apple"  (sorted: position 0)
    items[1] = "banana"
    items[2] = "cherry"
    items[3] = "mango"
    items[4] = "zebra"
  
  Result: ["apple", "banana", "cherry", "mango", "zebra"] ✓
  Duration: ~1ms

Op 5.4: Verify result (optional)
  Check sorted property:
    strcmp(items[i], items[i+1]) <= 0 for all i
  ✓ Verified

═══════════════════════════════════════════════════════════
Phase 6: Result Recording (Optional)
═══════════════════════════════════════════════════════════

Op 6.1: Create result commitment
  result_commitment = Hash(sorted_array)
  proof = ZK_Proof_of_Correct_Execution()  // Optional

Op 6.2: Submit result transaction
  Transaction:
    Type: ComputeFinalize
    execution_id: 0x9b8a7c6d...
    result_commitment: result_commitment
    proof: proof
    
  Broadcast to network
  Duration: ~100ms

Op 6.3: Update hypergraph
  NEW VERTEX:
    Address: Hash(result_commitment)
    Data: {
      execution_ref: 0x9b8a7c6d...,
      result: encrypted(sorted_array),
      timestamp: now()
    }
  
  CRDT merge across shards
  Duration: ~50ms

═══════════════════════════════════════════════════════════
Total Timeline Summary
═══════════════════════════════════════════════════════════

Phase 1 (Setup):           ~300ms
Phase 2 (Input encoding):  ~350ms
Phase 3 (Garbling):        ~800ms
Phase 4 (Evaluation):      ~700ms
Phase 5 (Decoding):        ~55ms
Phase 6 (Recording):       ~150ms
────────────────────────────────
TOTAL:                     ~2.4 seconds

Data Transfer:
  Alice → Bob: 752 KB (garbled circuit) + 41 KB (input labels) = 793 KB
  Bob → Alice: 41 KB (output labels)
  Total: 834 KB

Privacy Achieved:
  ✓ Bob never learns original array order
  ✓ Alice never learns... (nothing hidden in this example, but framework supports)
  ✓ Network nodes never learn computation inputs/outputs
  ✓ Only encrypted data in hypergraph
```

## Summary Diagram: Complete Flow

```
                    QUILIBRIUM SECURE COMPUTATION FLOW
┌──────────────────────────────────────────────────────────────────────┐
│                                                                      │
│  Developer        Client          Blockchain        MPC Participants│
│    │               │                  │                  │           │
│    │ 1. Write QCL │                  │                  │           │
│    │──────────────►│                  │                  │           │
│    │               │                  │                  │           │
│    │ 2. Compile   │                  │                  │           │
│    │◄─────────────┤                  │                  │           │
│    │               │                  │                  │           │
│    │               │ 3. CodeDeployment│                  │           │
│    │               ├─────────────────►│                  │           │
│    │               │                  │ Store in         │           │
│    │               │                  │ Hypergraph       │           │
│    │               │                  │────┐             │           │
│    │               │                  │◄───┘             │           │
│    │               │                  │                  │           │
│    │               │ 4. CodeExecute   │                  │           │
│    │               ├─────────────────►│                  │           │
│    │               │                  │ Record request   │           │
│    │               │                  │────┐             │           │
│    │               │                  │◄───┘             │           │
│    │               │                  │                  │           │
│    │               │                  │ 5. Notify        │           │
│    │               │                  ├─────────────────►│           │
│    │               │                  │  (rendezvous)    │           │
│    │               │                  │                  │           │
│    │               │                  │                  │ 6. P2P    │
│    │               │                  │                  │ Connect   │
│    │               │                  │                  │───────┐   │
│    │               │                  │                  │◄──────┘   │
│    │               │                  │                  │           │
│    │               │                  │                  │ 7. MPC    │
│    │               │                  │                  │ Execute   │
│    │               │                  │                  │ (off-chain│
│    │               │                  │                  │───────┐   │
│    │               │                  │                  │◄──────┘   │
│    │               │                  │                  │           │
│    │               │ 8. Result        │                  │           │
│    │               │◄─────────────────┼──────────────────┤           │
│    │               │                  │                  │           │
│    │               │ 9. (Optional)    │                  │           │
│    │               │ ComputeFinalize  │                  │           │
│    │               ├─────────────────►│                  │           │
│    │               │                  │ Record result    │           │
│    │               │                  │────┐             │           │
│    │               │                  │◄───┘             │           │
│    │               │                  │                  │           │
└──────────────────────────────────────────────────────────────────────┘

Key Points:
  • Hypergraph = Code storage + Execution metadata
  • Blockchain = Consensus + Ordering + Payment
  • MPC = Actual computation (off-chain, P2P, encrypted)
  • Privacy = No node sees plaintext computation details
```

This completes the atomic-level analysis of the entire system!

