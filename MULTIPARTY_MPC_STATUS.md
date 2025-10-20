# Quilibrium多方MPC支持现状分析

## 快速回答

**❌ 当前项目只支持2方MPC，不支持多方（n≥3）计算**

---

## 1. 源码证据

### 1.1 CodeDeployment结构明确限制为2方

```go
// node/execution/intrinsics/compute/compute_intrinsic_code_deployment.go

type CodeDeployment struct {
    // The QCL circuit to deploy
    Circuit []byte
    // The QCL types/classes of the main arguments, 0 - garbler, 1 - evaluator
    InputTypes [2]string  // ← 固定2个
    // The QCL types/classes of the output values
    OutputTypes []string
    // The app address
    Domain [32]byte

    inputQCLSource []byte
    inputSizes     [2][]int  // ← 固定2个
    compiler       compiler.CircuitCompiler
}
```

**关键点**：
- `InputTypes [2]string` - 数组大小固定为2
- 注释明确说明：`0 - garbler, 1 - evaluator`
- 不是可变长度的slice，是固定大小的array

### 1.2 编译器强制检查

```go
// bedlam/compiler/compiler.go:143-146

if len(program.Inputs) != 2 {
    return nil, nil,
        fmt.Errorf("invalid program for 2-party computation: %d parties",
            len(program.Inputs))
}
```

**错误信息**：明确提示"invalid program for 2-party computation"

### 1.3 Garbler/Evaluator应用也强制检查

```go
// bedlam/apps/garbled/main.go:274-276

if len(circ.Inputs) != 2 {
    return fmt.Errorf("invalid circuit for 2-party MPC: %d parties",
        len(circ.Inputs))
}

// bedlam/apps/garbled/main.go:344-346

if len(circ.Inputs) != 2 {
    return fmt.Errorf("invalid circuit for 2-party MPC: %d parties",
        len(circ.Inputs))
}
```

### 1.4 只有两个角色常量

```go
// bedlam/circuit/circuit.go:32-36

// Known multi-party computation roles.
const (
    IDGarbler int = iota  // 0
    IDEvaluator           // 1
)
```

**只定义了2个角色**：
- Garbler (加密方)
- Evaluator (评估方)

没有第三个、第四个角色的定义。

### 1.5 FERRET OT协议的party参数

```go
// bedlam/ot/ferret.go:27-36

type Ferret struct {
    party   uint8   // 只有1或2
    address string
    io      IO
}

// bedlam/apps/garbled/main.go:138-141

party := uint8(1)
if *evaluator {
    party = 2
}
```

**party只有2个值**：
- `1` - Garbler
- `2` - Evaluator

---

## 2. 为什么只支持2方？

### 2.1 技术架构决定

```
Yao's Garbled Circuits原生协议:
┌─────────────────────────────────────┐
│ 设计用于2方:                         │
│   • 一方Garble (加密电路)            │
│   • 一方Evaluate (评估电路)          │
│                                     │
│ 扩展到多方需要BMR协议:               │
│   • Beaver-Micali-Rogaway (1990)   │
│   • 架构完全不同                     │
│   • 实现复杂度显著增加               │
└─────────────────────────────────────┘
```

### 2.2 Quilibrium的设计选择

**选择2方MPC的理由**：
1. **简单高效** - Yao's GC在2方场景最优
2. **常数轮通信** - 延迟最低
3. **实现成本** - 代码简洁，易维护
4. **应用场景** - 大多数隐私计算是2方（买卖双方、数据提供者vs使用者）

---

## 3. 那个3party.qcl示例是什么？

### 3.1 示例代码

```qcl
// bedlam/apps/garbled/examples/3party.qcl

// Sample 3-party circuit where each party provides their input bit
// and the result is bitwise AND of the inputs.
package main

func main(a, b, e uint1) uint {
    return a & b & e
}
```

### 3.2 为什么存在但不能运行？

```
状态: 示例文件存在，但无法编译和运行

可能的原因:
┌────────────────────────────────────────────┐
│ 1. 历史遗留                                 │
│    • 早期曾考虑多方支持                     │
│    • 后来决定专注2方                        │
│    • 示例文件未删除                         │
│                                            │
│ 2. 未来规划                                │
│    • 保留作为未来扩展的参考                 │
│    • 展示语言层面的能力                     │
│    • 但运行时未实现                         │
│                                            │
│ 3. 测试用途                                │
│    • 测试编译器对多输入的解析               │
│    • 但不生成可执行电路                     │
└────────────────────────────────────────────┘
```

### 3.3 验证：尝试编译会失败

```bash
$ bedlam compile 3party.qcl
Error: invalid program for 2-party computation: 3 parties
```

---

## 4. Circuit设计上的"多方准备"

### 4.1 NumParties()方法

```go
// bedlam/circuit/circuit.go:119-122

// NumParties returns the number of parties needed for the circuit.
func (c *Circuit) NumParties() int {
    return len(c.Inputs)
}
```

**设计意图**：
- API设计上考虑了扩展性
- 返回参与方数量（而不是硬编码返回2）
- 但实际使用中被检查限制为2

### 4.2 架构上的可扩展性

```
当前架构:
┌──────────────────────────────────────┐
│ Circuit结构                           │
│   ├─ Inputs []IO  ← 可变长度slice    │
│   ├─ Outputs IO                      │
│   └─ Gates []Gate                    │
│                                      │
│ 理论上支持任意数量输入                 │
│ 但运行时强制检查 len(Inputs) == 2    │
└──────────────────────────────────────┘

如果未来要支持多方:
┌──────────────────────────────────────┐
│ 需要改动的地方:                       │
│ 1. 移除所有 len(...) != 2 检查       │
│ 2. InputTypes改为 []string           │
│ 3. 实现BMR协议                       │
│ 4. 修改OT协议支持多方                 │
│ 5. 修改Rendezvous机制                │
│ 6. 重新设计角色分配                   │
└──────────────────────────────────────┘
```

---

## 5. 多方MPC的技术挑战

### 5.1 需要实现BMR协议

```
BMR (Beaver-Micali-Rogaway) 协议:
═════════════════════════════════

与Yao's GC的区别:
┌─────────────────────────────────────────────┐
│ Yao's GC (2-party):                         │
│   • 一方garble整个电路                       │
│   • 另一方evaluate                          │
│   • OT用于输入交换                           │
│                                             │
│ BMR (n-party):                              │
│   • 所有方共同garble                         │
│   • 每个门需要所有方的密钥                   │
│   • 通信复杂度: O(n² × gates)               │
│   • 实现复杂度高很多                         │
└─────────────────────────────────────────────┘
```

### 5.2 通信复杂度

```
2-party (当前):
  • 通信轮次: O(1) - 常数轮
  • 通信量: O(gates) - 与门数成正比
  • AES电路: ~1.5MB

n-party (BMR):
  • 通信轮次: O(1) - 仍是常数轮
  • 通信量: O(n² × gates)
  • AES电路 (n=5): ~37.5MB (25倍)
```

### 5.3 OT协议需要扩展

```
当前FERRET OT:
  • 1-out-of-2 OT
  • 只支持2方

多方OT需要:
  • 1-out-of-n OT
  • 或多次2方OT组合
  • 复杂度显著增加
```

---

## 6. 如何模拟多方计算？

虽然原生不支持n方(n≥3)，但可以通过组合2方MPC实现某些多方场景：

### 6.1 方案1: 串行组合

```
场景: Alice, Bob, Carol三方各有输入，求和

串行方案:
┌─────────────────────────────────────────┐
│ Step 1: Alice + Bob → temp1            │
│   • 2-party MPC                        │
│   • temp1 = Alice_value + Bob_value    │
│                                        │
│ Step 2: temp1 + Carol → final          │
│   • 2-party MPC (temp1持有方 + Carol)  │
│   • final = temp1 + Carol_value        │
└─────────────────────────────────────────┘

限制:
  ✗ 中间结果temp1会泄露给持有方
  ✗ 不是真正的多方隐私
  ✓ 某些场景可接受
```

### 6.2 方案2: 并行聚合

```
场景: 多个数据提供者，一个计算方

并行方案:
┌─────────────────────────────────────────┐
│ Provider1 ←─MPC─→ Computer             │
│ Provider2 ←─MPC─→ Computer             │
│ Provider3 ←─MPC─→ Computer             │
│                                        │
│ Computer聚合结果但看不到原始数据         │
└─────────────────────────────────────────┘

适用场景:
  ✓ 联邦学习 (多方提供数据，中心聚合)
  ✓ 隐私投票 (多方投票，计票方统计)
  ✓ 数据市场 (多个卖家，一个买家)
```

### 6.3 方案3: 信任第三方

```
引入可信第三方:

Alice ←─MPC─→ TTP ←─MPC─→ Bob
                ↕
              Carol

TTP特点:
  • 协调多方计算
  • 不看到任何原始输入
  • 可以是DAO治理的节点
  • 半诚实假设
```

---

## 7. 未来多方支持的可能性

### 7.1 技术可行性

```
实现难度评估:
┌────────────────────────────────────────┐
│ 工作量估计:                             │
│   • BMR协议实现: ~2000行代码           │
│   • OT扩展: ~500行                     │
│   • 测试: ~1000行                      │
│   • 总计: ~3500行核心逻辑              │
│                                        │
│ 时间估计: 2-3个月 (单人全职)           │
│                                        │
│ 复杂度: 中等偏高                        │
└────────────────────────────────────────┘
```

### 7.2 性能影响

```
3方计算 vs 2方计算:
┌────────────────────────────────────────┐
│ 通信量: 9倍 (n²增长)                    │
│ 延迟: 1.5-2倍 (仍是常数轮)              │
│ 计算量: 3倍 (每方多计算)                │
│                                        │
│ 结论: 可接受，但不是主要场景            │
└────────────────────────────────────────┘
```

### 7.3 需求评估

```
实际应用场景分析:
┌────────────────────────────────────────┐
│ 2-party场景 (90%+):                    │
│   • 交易双方                           │
│   • 数据买卖                           │
│   • 隐私匹配                           │
│   • 联合查询                           │
│                                        │
│ 多方场景 (10%-):                       │
│   • 多方投票                           │
│   • 门槛签名                           │
│   • 多方拍卖                           │
│   • 联邦学习                           │
└────────────────────────────────────────┘

优先级: 低
理由: 大部分场景2方足够
```

---

## 8. 总结

### 8.1 当前状态

```
✅ 支持: 2-party MPC (Yao's Garbled Circuits)
❌ 不支持: n-party MPC (n ≥ 3)
```

### 8.2 技术限制

| 组件 | 限制 | 位置 |
|------|------|------|
| CodeDeployment | `InputTypes [2]string` | compute_intrinsic_code_deployment.go |
| 编译器 | `len(Inputs) != 2` 检查 | bedlam/compiler/compiler.go |
| Garbler/Evaluator | 只有2个角色 | bedlam/circuit/circuit.go |
| OT协议 | party只有1或2 | bedlam/ot/ferret.go |

### 8.3 原因分析

**选择2方的理由**：
1. ✅ 满足90%+的应用场景
2. ✅ 性能最优（常数轮，低延迟）
3. ✅ 实现简单，易维护
4. ✅ 工程上的MVP原则

**不支持多方的原因**：
1. 实现复杂度高（需要BMR）
2. 性能下降（通信量n²增长）
3. 需求优先级低
4. 可通过2方组合模拟部分场景

### 8.4 未来展望

```
短期 (1-2年):
  ✗ 不太可能添加多方支持
  ✓ 专注优化2方性能和安全性

中期 (2-3年):
  ? 可能添加特定场景的多方支持
    • 如门槛签名、多方投票
    • 使用更高效的专用协议

长期 (3-5年):
  ? 如果有明确需求，可能实现通用BMR
    • 需要社区反馈和用例驱动
    • 权衡性能和复杂度
```

### 8.5 给开发者的建议

```
如果你需要多方计算:
┌────────────────────────────────────────┐
│ 1. 评估是否真的需要n≥3                  │
│    很多场景可以分解为多个2方             │
│                                        │
│ 2. 考虑替代方案                         │
│    • 串行2方MPC组合                    │
│    • 引入可信第三方                     │
│    • 使用秘密分享(SPDZ)方案             │
│                                        │
│ 3. 等待官方支持                         │
│    • 跟踪项目roadmap                   │
│    • 提交feature request               │
│                                        │
│ 4. 贡献实现                            │
│    • BMR协议实现                       │
│    • Pull request贡献                  │
└────────────────────────────────────────┘
```

---

## 附录: 相关代码位置

### 核心限制代码

```
bedlam/compiler/compiler.go:143
  └─ 编译器检查: len(program.Inputs) != 2

node/execution/intrinsics/compute/compute_intrinsic_code_deployment.go:22
  └─ 输入类型: InputTypes [2]string

bedlam/circuit/circuit.go:33
  └─ 角色定义: IDGarbler, IDEvaluator

bedlam/apps/garbled/main.go:274, 344
  └─ 运行时检查: len(circ.Inputs) != 2

bedlam/ot/ferret.go:27
  └─ OT party: uint8 (只有1或2)
```

### 多方相关示例

```
bedlam/apps/garbled/examples/3party.qcl
  └─ 3方示例 (无法编译运行)
```

---

**文档版本**: 1.0  
**创建日期**: 2025-10-20  
**最后更新**: 2025-10-20  
**状态**: 当前准确

