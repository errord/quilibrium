# Hypergraph Computation Mapping Analysis

## How Computation Maps to Hypergraph Queries

### Core Concept: Computation ≠ Hypergraph Query

**Important Clarification**: 
- Hypergraph is for **STATE STORAGE**, not computation execution
- OT (Oblivious Transfer) is for **SECURE INPUT TRANSFER** in MPC
- Computation happens in **Garbled Circuits**, not in hypergraph queries

### Architecture Layers

```
┌─────────────────────────────────────────────────────────────┐
│                    User Application Layer                    │
│  (Submit: Code + Inputs + Payment)                          │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│              Blockchain / Consensus Layer                    │
│  - Record transactions                                       │
│  - Validate proofs                                           │
│  - Manage state transitions                                  │
└───────────────────────┬─────────────────────────────────────┘
                        │
           ┌────────────┼────────────┐
           │            │            │
           ▼            ▼            ▼
    ┌──────────┐ ┌──────────┐ ┌──────────┐
    │ Storage  │ │ Compute  │ │  Token   │
    │  Layer   │ │  Layer   │ │  Layer   │
    └──────────┘ └──────────┘ └──────────┘
         │            │            │
         ▼            ▼            ▼
    Hypergraph   Garbled        Bulletproofs
     (CRDT)      Circuits        (Payment)
```

### 1. Hypergraph's Role in Computation

#### 1.1 What Hypergraph Stores

```
Hypergraph Structure:
====================

Vertex: Computational Artifact
  - Code deployment (circuit binary)
  - Execution metadata
  - Access control info
  - Result commitments

Hyperedge: Relationships
  - Code dependencies
  - Data flow connections
  - Permission links
  - Computation lineage

Example Storage:
───────────────
Domain: 0xabcd... (Compute Intrinsic Domain)

Vertex 1: Code Artifact
  Address: Hash(circuit_binary)
  Data: {
    circuit: <binary_data>,
    input_types: ["int64", "int64"],
    output_types: ["bool"],
    metadata: {
      creator: 0x1234...,
      timestamp: 1234567890,
      version: "1.0"
    }
  }

Vertex 2: Execution Record
  Address: Hash(execution_id)
  Data: {
    code_ref: Vertex1.Address,
    rendezvous: 0x5678...,
    participants: [Alice_pubkey, Bob_pubkey],
    result_commitment: Hash(result),
    proof: <bulletproof>
  }

Hyperedge: Code→Execution
  Source: [Vertex1]
  Target: [Vertex2]
  Label: "executed_by"
```

#### 1.2 Hypergraph Query Operations

```go
// Example: Retrieve deployed code
func RetrieveCode(domain [32]byte, codeAddress [32]byte) ([]byte, error) {
    // Construct 64-byte key: domain + codeAddress
    key := [64]byte{}
    copy(key[:32], domain[:])
    copy(key[32:], codeAddress[:])
    
    // Query hypergraph for vertex data
    vertexData, err := hypergraph.GetVertexData(key)
    if err != nil {
        return nil, err
    }
    
    // Extract circuit from VectorCommitmentTree
    circuit, err := vertexData.Get([]byte{0 << 2}) // Index 0
    return circuit, err
}

// NOT a computation query! Just data retrieval.
```

### 2. Computation Execution Flow

```
Step-by-Step: Where Each Component Acts
========================================

┌─ Step 1: Code Deployment ─────────────────────────────────┐
│                                                            │
│  Developer writes QCL code                                 │
│      ↓                                                     │
│  Compile to circuit (LOCAL)                                │
│      ↓                                                     │
│  Create CodeDeployment transaction                         │
│      ↓                                                     │
│  [HYPERGRAPH]: Store circuit in vertex                     │
│      └─► Query later: GetVertexData(code_address)         │
│                                                            │
└────────────────────────────────────────────────────────────┘

┌─ Step 2: Execution Request ───────────────────────────────┐
│                                                            │
│  User creates CodeExecute transaction                      │
│      ↓                                                     │
│  [BLOCKCHAIN]: Record request, payment proof               │
│      ↓                                                     │
│  [HYPERGRAPH]: Store execution metadata                    │
│      └─► Query: GetVertex(execution_id)                   │
│                                                            │
└────────────────────────────────────────────────────────────┘

┌─ Step 3: Participant Discovery ───────────────────────────┐
│                                                            │
│  Both parties query hypergraph:                            │
│      [HYPERGRAPH QUERY]:                                   │
│          GetVertexData(execution_id)                       │
│          → retrieves rendezvous point                      │
│      ↓                                                     │
│  Connect via P2P (BlossomSub topic)                        │
│      ↓                                                     │
│  Establish encrypted channel                               │
│                                                            │
└────────────────────────────────────────────────────────────┘

┌─ Step 4: Circuit Retrieval ───────────────────────────────┐
│                                                            │
│  Garbler queries:                                          │
│      [HYPERGRAPH QUERY]:                                   │
│          GetVertexData(code_address)                       │
│          → retrieves circuit binary                        │
│      ↓                                                     │
│  Load circuit into memory                                  │
│  Parse gates and wires                                     │
│                                                            │
└────────────────────────────────────────────────────────────┘

┌─ Step 5: MPC Execution (OFF-CHAIN!) ──────────────────────┐
│                                                            │
│  [P2P CHANNEL]: All computation happens here               │
│      ↓                                                     │
│  Garbler: Generate wire labels (random)                    │
│  Garbler: Encrypt truth tables                             │
│  Garbler: Send encrypted circuit to Evaluator              │
│      ↓                                                     │
│  [OT Protocol]: Evaluator obtains input labels             │
│      ↓                                                     │
│  Evaluator: Compute gate-by-gate                           │
│  Evaluator: Send output labels back                        │
│      ↓                                                     │
│  Garbler: Decode output                                    │
│                                                            │
│  NOTE: No hypergraph queries during computation!           │
│        Hypergraph was only for code storage/retrieval      │
│                                                            │
└────────────────────────────────────────────────────────────┘

┌─ Step 6: Result Recording (Optional) ─────────────────────┐
│                                                            │
│  Create result commitment                                  │
│      ↓                                                     │
│  [HYPERGRAPH]: Store in vertex                             │
│      └─► Query later: GetVertexData(result_id)            │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

### 3. OT's Role in Computation

```
OT (Oblivious Transfer) Purpose:
================================

Scenario: Bob needs his input labels, but Alice shouldn't know which ones

Without OT (INSECURE):
  Alice: "Bob, your input is 5 (binary: 101)"
  Alice: "Here are labels: bit0=L1, bit1=L0, bit2=L1"
  → Alice learns Bob's input! ✗

With OT (SECURE):
  Alice: Prepares pairs for each bit:
    bit0: (L_0, L_1)  ← Bob will choose one
    bit1: (L_0, L_1)
    bit2: (L_0, L_1)
    
  Bob: "My input is 5 (binary: 101)"
        Bit 0 = 1 → OT retrieves L_1 from pair 0
        Bit 1 = 0 → OT retrieves L_0 from pair 1
        Bit 2 = 1 → OT retrieves L_1 from pair 2
        
  Alice: Only knows she sent 3 pairs
         Doesn't know which Bob chose! ✓

OT Protocol (FERRET) Details:
─────────────────────────────

Alice (Sender):               Bob (Receiver):
Has: (L0, L1) pairs          Has: choice bits (c_i)

1. COT Setup
   └─► Generate correlated OT base
   
2. ROT Phase  
   └─► Extend to many OTs efficiently
   
3. For each bit i:
   Alice:                     Bob:
   r0_i, r1_i = Random()      r_i = GetBlock(c_i)
   e0 = r0_i ⊕ L0_i           
   e1 = r1_i ⊕ L1_i           
   Send(e0, e1) ──────────►   
                              L_i = r_i ⊕ e_{c_i}
                              
Result: Bob gets L_{c_i}, Alice doesn't learn c_i
```

### 4. State vs Computation Separation

```
┌────────────────────────────────────────────────────────────┐
│                  State (Hypergraph)                        │
├────────────────────────────────────────────────────────────┤
│  What it stores:                                           │
│    • Code artifacts (circuits)                             │
│    • Execution metadata                                    │
│    • Access permissions                                    │
│    • Result commitments                                    │
│                                                            │
│  What it does NOT do:                                      │
│    ✗ Execute computations                                  │
│    ✗ Process inputs                                        │
│    ✗ Garble/evaluate circuits                              │
│                                                            │
│  Query operations:                                         │
│    • GetVertexData(address)                                │
│    • AddVertex(domain, address, data)                      │
│    • CreateTraversalProof(keys)                            │
│    • VerifyTraversalProof(proof)                           │
└────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────┐
│              Computation (Garbled Circuits)                │
├────────────────────────────────────────────────────────────┤
│  Where it happens:                                         │
│    • Off-chain P2P channels                                │
│    • Direct node-to-node communication                     │
│    • Memory (not persistent storage)                       │
│                                                            │
│  What it does:                                             │
│    ✓ Garble circuits (generate labels, encrypt)            │
│    ✓ Evaluate gates (decrypt, compute)                     │
│    ✓ OT protocol (secure input transfer)                   │
│    ✓ Result decoding                                       │
│                                                            │
│  Computation operations:                                   │
│    • Garble(circuit, inputs)                               │
│    • OT_Send(labels) / OT_Receive(choices)                 │
│    • Evaluate(encrypted_gates, labels)                     │
│    • Decode(output_labels)                                 │
└────────────────────────────────────────────────────────────┘
```

### 5. Why This Architecture?

```
Design Rationale:
=================

1. Hypergraph for Persistence
   └─► Code needs to survive across sessions
   └─► Results may need verification later
   └─► CRDT properties enable distributed consensus
   
2. Off-chain Computation
   └─► MPC is communication-intensive (MBs of data)
   └─► On-chain storage is expensive
   └─► Privacy: computation details stay private
   
3. Separation of Concerns
   ┌─────────────┬──────────────┬──────────────┐
   │ Hypergraph  │  Blockchain  │  P2P+MPC     │
   ├─────────────┼──────────────┼──────────────┤
   │ State store │  Consensus   │  Execution   │
   │ CRDT merge  │  Validation  │  Privacy     │
   │ Persistence │  Ordering    │  Performance │
   └─────────────┴──────────────┴──────────────┘
```

## Example: Computing "2*32+5-4/33"

```
Timeline of Operations:
=======================

T0: Developer deploys code
    └─► [HYPERGRAPH WRITE]: Store circuit at address 0xABCD
        └─► AddVertex(COMPUTE_DOMAIN, 0xABCD, circuit_data)

T1: Alice requests computation with Bob
    └─► [HYPERGRAPH WRITE]: Store execution request
        └─► AddVertex(COMPUTE_DOMAIN, exec_id, {
              code_ref: 0xABCD,
              rendezvous: 0x1234,
              participants: [Alice, Bob]
            })

T2: Alice & Bob discover each other
    └─► [HYPERGRAPH READ]: Query execution metadata
        └─► GetVertexData(exec_id) → rendezvous point

T3: Alice retrieves circuit
    └─► [HYPERGRAPH READ]: Load circuit
        └─► GetVertexData(0xABCD) → circuit binary

T4: MPC execution (OFF-CHAIN, P2P)
    └─► [NO HYPERGRAPH INVOLVED]
        ├─► Alice garbles circuit
        ├─► OT transfer
        ├─► Bob evaluates
        └─► Result computed

T5: (Optional) Record result
    └─► [HYPERGRAPH WRITE]: Store commitment
        └─► AddVertex(COMPUTE_DOMAIN, result_id, commitment)
```

## Conclusion

### Hypergraph is NOT for computation queries!

**Hypergraph Role**: 
- 📦 Persistent storage for code and metadata
- 🔍 Data retrieval by address
- 📜 Audit trail of executions
- 🔐 Access control via CRDT

**Computation Role** (Garbled Circuits + OT):
- 🔒 Privacy-preserving execution
- 🚀 Off-chain performance
- 🤝 Direct P2P interaction
- 🎯 Secure input handling (via OT)

**The hypergraph is the "hard drive", garbled circuits are the "CPU"!**

