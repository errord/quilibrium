# Boolean Circuit to Garbled Circuit Transformation Analysis

## Example: 2*32+5-4/33

### Step 1: Expression to Boolean Circuit

```
Expression: 2*32+5-4/33
Inputs: All constants in this example
Let's use variables: (a*b+c-d/e) where a=2, b=32, c=5, d=4, e=33

Binary representation (8-bit for simplicity):
a = 2  = 00000010
b = 32 = 00100000
c = 5  = 00000101
d = 4  = 00000100
e = 33 = 00100001
```

### Step 2: Decompose to Boolean Operations

```
Multiplication (a*b):
┌────────────────────────────────┐
│ Binary Multiplication Circuit  │
│                                │
│  a[7:0] ─┐                     │
│          ├─► AND gates         │
│  b[7:0] ─┘    │                │
│               ├─► Adder tree   │
│               │                │
│           product[15:0] ────►  │
└────────────────────────────────┘

Gates needed:
- 8×8 = 64 AND gates (partial products)
- 7-level adder tree (carry-save adders)
- Total: ~200 gates

Addition (result+c):
┌────────────────────────────────┐
│ Ripple Carry Adder             │
│                                │
│  [15:0] ─┐                     │
│          ├─► Full Adders (16)  │
│  c[7:0] ─┘                     │
│                                │
│       sum[15:0] ────►          │
└────────────────────────────────┘

Gates needed:
- 16 full adders
- Each full adder: 2 XOR + 2 AND + 1 OR = 5 gates
- Total: ~80 gates

Division (d/e):
┌────────────────────────────────┐
│ Division Circuit (Restoring)   │
│                                │
│  d[7:0] ─┐                     │
│          ├─► Shift-Subtract    │
│  e[7:0] ─┘    iterations (8)   │
│                                │
│    quotient[7:0] ────►         │
└────────────────────────────────┘

Gates needed:
- 8 iterations of subtract + compare
- Each: ~30 gates
- Total: ~240 gates

Subtraction (sum-quotient):
┌────────────────────────────────┐
│ Subtractor (Adder+NOT)         │
│                                │
│  sum[15:0] ─┐                  │
│             ├─► Full Adders    │
│  quot[7:0]─┘                   │
│                                │
│    result[15:0] ────►          │
└────────────────────────────────┘

Total: ~80 gates
```

### Step 3: Complete Boolean Circuit

```
Total Circuit Structure:
========================

Inputs: a[7:0], b[7:0], c[7:0], d[7:0], e[7:0] = 40 wires
Gates: ~600 gates total
Output: result[15:0] = 16 wires

Circuit DAG (Directed Acyclic Graph):
                 
Input Wires (40) ─┬─► Mult Circuit (200 gates)
                  │         │
                  │      product
                  │         │
                  ├─► Add Circuit (80 gates)
                  │         │
                  │      sum
                  │         │
                  ├─► Div Circuit (240 gates)
                  │         │
                  │     quotient
                  │         │
                  └─► Sub Circuit (80 gates)
                            │
                         result (16)
```

### Step 4: Why Garbled Circuit is Equivalent?

#### A. Truth Table Preservation

For each gate (e.g., AND gate):
```
Boolean Circuit:
Input A | Input B | Output C
   0    |    0    |    0
   0    |    1    |    0
   1    |    0    |    0
   1    |    1    |    1

Garbled Circuit:
Label_A0 | Label_B0 | Label_C0  (encrypts to row 0,0→0)
Label_A0 | Label_B1 | Label_C0  (encrypts to row 0,1→0)
Label_A1 | Label_B0 | Label_C0  (encrypts to row 1,0→0)
Label_A1 | Label_B1 | Label_C1  (encrypts to row 1,1→1)
```

**Key Insight**: Each label pair (L0, L1) encodes 0 and 1, but:
- Evaluator can't tell which label means 0 or 1
- Only sees one label per wire
- Can only decrypt ONE row per gate

#### B. Encryption Scheme (Half-Gates)

```
For AND gate with inputs A, B → output C:

Garbler (Alice):
1. Generate random labels:
   L_A0, L_A1 (for wire A)
   L_B0, L_B1 (for wire B)
   L_C0, L_C1 (for wire C)
   
2. Compute garbled table:
   j0 = gate_id * 2
   j1 = gate_id * 2 + 1
   
   TG = Encrypt_AES(L_A0, j0) ⊕ (if A=1 then L_C0 else 0)
   TE = Encrypt_AES(L_B0, j1) ⊕ (if B=1 then (L_C0 ⊕ L_A) else 0)
   
3. Send (TG, TE) to evaluator

Evaluator (Bob):
1. Received: label_a (either L_A0 or L_A1)
              label_b (either L_B0 or L_B1)
              
2. Compute:
   wg = Decrypt_AES(label_a, j0)
   if (label_a.select_bit == 1):
       wg = wg ⊕ TG
   
   we = Decrypt_AES(label_b, j1)
   if (label_b.select_bit == 1):
       we = we ⊕ TE ⊕ label_a
   
   label_c = wg ⊕ we
   
3. Result: label_c is exactly what Alice intended for this A,B combination
```

#### C. Mathematical Equivalence Proof

**Theorem**: For any boolean function f: {0,1}^n → {0,1}^m, the garbled circuit GC_f produces the same output as f for any input.

**Proof Sketch**:
```
1. Induction on circuit depth:
   
   Base case (depth 0): Input wires
   - Alice assigns labels based on actual input bits
   - Bob receives correct labels via OT
   - Bijection: bits ↔ labels maintained
   
   Inductive step (depth k→k+1):
   - Assume: All wires at depth k have correct labels
   - For gate g at depth k+1:
     * Inputs: labels l_a, l_b (correspond to bits a, b)
     * Alice encrypted: C = g(a,b) using l_a, l_b
     * Bob decrypts: l_c using l_a, l_b
     * By construction: l_c encodes g(a,b)
   - Bijection maintained
   
   Result: Output labels encode f(input) correctly
```

#### D. Information-Theoretic Analysis

```
What Bob learns from Garbled Circuit:
=====================================

Given:
- Garbled tables for all gates
- One label per input wire (his inputs via OT)
- One label per intermediate wire (computed)

Bob can compute:
✓ Output value (when Alice reveals label meanings)

Bob CANNOT compute:
✗ Alternative execution paths (other table rows encrypted with unknown keys)
✗ Input values he didn't choose (OT hides Alice's other labels)
✗ Semantic meaning of intermediate wires (random labels)
✗ Circuit structure (sees only encrypted gates, not logic)

Why equivalent?
- Computational: Breaking AES-128 is infeasible
- Cryptographic: Labels are indistinguishable from random
- Correctness: Decryption recovers correct output for actual inputs
```

### Step 5: Concrete Example Walkthrough

```
Simplified: 2-bit multiplication (a*b where a=2=10, b=3=11)

Boolean Circuit:
┌─────────────────────────────────┐
│  a[1] a[0]   b[1] b[0]          │
│   │    │      │    │            │
│   │    └──────┼────┴─► AND → p0 │
│   │           │                 │
│   └───────────┴──────► AND → p1 │
│               │                 │
│               └──────► AND → p2 │
│                                 │
│  p0 ─┐                          │
│      ├─► Half Adder ─► sum[0]  │
│  p1 ─┘      │                   │
│          carry                  │
│             │                   │
│          ┌──┴──┐                │
│  p2 ───►│ HA  ├──► sum[1]      │
│          └─────┘│                │
│             carry → sum[2]      │
└─────────────────────────────────┘

Result: 10 * 11 = 110 (6 in decimal)

Garbled version:
===============

Wire labels (128-bit each, shown as 4-char for brevity):
a[0]=0: AAAA,  a[0]=1: AAAB
a[1]=0: BBBA,  a[1]=1: BBBB
b[0]=0: CCCA,  b[0]=1: CCCB
b[1]=0: DDDA,  b[1]=1: DDDB

Alice's input: a=2=10 → sends (AAAA, BBBB)
Bob's input: b=3=11 → OT gets (CCCB, DDDB)

Gate 1 (a[0] AND b[0]):
Encrypted table: 
  Enc(AAAA,CCCB,0) = E1
  
Bob computes: Decrypt(AAAA,CCCB,E1) = XXXX (label for 0)

Gate 2 (a[0] AND b[1]):
Bob computes: YYYY (label for 1)

Gate 3 (a[1] AND b[1]):
Bob computes: ZZZZ (label for 1)

Half Adder for (XXXX ⊕ YYYY):
Bob computes: sum[0]=WWWW (label for 1), carry=VVVV (label for 1)

Half Adder for (ZZZZ ⊕ VVVV):
Bob computes: sum[1]=UUUU (label for 1), sum[2]=TTTT (label for 0)

Bob sends to Alice: (WWWW, UUUU, TTTT)
Alice decodes: 0b110 = 6 ✓

Equivalence verified!
```

## Why They're Equivalent: Core Principles

### 1. Functional Equivalence
```
Boolean Circuit: f(x) = y
Garbled Circuit: GC_f(Labels(x)) = Labels(y)
Decoding: Decode(Labels(y)) = y

∴ Decode(GC_f(Labels(x))) = f(x)
```

### 2. Security via Obfuscation
- Labels appear random → no semantic information
- Encryption hides unused paths → no alternative branches visible
- OT hides choices → no input inference

### 3. Correctness via Cryptographic Binding
- AES ensures decryption yields intended label
- Label propagation maintains bit values through circuit
- Output decoding recovers actual result

## Complexity Analysis

```
Original Expression: 2*32+5-4/33

Circuit Complexity:
- Gates: O(n²) for n-bit multiplication (dominant)
- Depth: O(log n) for adder trees
- Wires: O(n²)

Garbled Circuit:
- Communication: 2×128 bits per AND gate
- Computation: 1 AES per gate for Bob
- Storage: 2×128 bits per AND gate

For n=32 bits:
- ~10,000 gates
- ~2.5 MB garbled circuit
- ~10,000 AES operations
- Time: ~10ms on modern CPU
```

## Conclusion

Garbled circuits are equivalent to boolean circuits because:

1. **Structural Isomorphism**: Every boolean gate maps to exactly one garbled gate
2. **Semantic Preservation**: Truth tables are preserved under encryption
3. **Computational Soundness**: Decryption recovers correct values (assuming honest execution)
4. **Information-Theoretic Security**: No information leaks beyond the output (in honest-but-curious model)

The "magic" is that encryption provides a one-way function: easy to evaluate forward (with correct labels), computationally infeasible to reverse or explore alternative paths.

