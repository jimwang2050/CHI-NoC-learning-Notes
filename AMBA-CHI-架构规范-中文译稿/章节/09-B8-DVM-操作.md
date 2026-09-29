# B8 DVM 操作
本章描述协议用于管理虚拟内存的分布式虚拟内存（DVM）操作。本章包含以下各节：

- B8.1 DVM 事务简介
- B8.2 DVM 事务流程
- B8.3 DVMOp 字段取值限制
- B8.4 DVM 消息

### B8.1 DVM 事务简介

DVM 事务是一种可选特性，用于传递支持虚拟内存系统维护的消息。

DVM 事务支持以下操作：

- Non-sync 事务流程，包括：
- TLB Invalidate
- Branch Predictor Invalidate
- Physical Instruction Cache Invalidate
- Virtual Instruction Cache Invalidate
- Sync 事务流程，包括：
- Synchronization

DVM 事务仅对只读结构（如指令缓存、分支预测器和 TLB）进行操作，因此只需要失效操作。清理的概念不适用于只读结构。这意味着，失效的条目多于 DVM 消息所要求的条目在功能上是正确的，尽管额外的失效可能影响性能。

接口上对 DVM 操作的支持由 DVM_Support 属性定义。有关更多信息，参见 B16.1.22 DVM_Support。

### B8.2 DVM 事务流程

以下各节描述 Non-sync 与 Sync DVM 事务流程以及流控：

- B8.2.1 Non-sync 类型 DVM 事务流程
- B8.2.2 Sync 类型 DVM 事务流程
- B8.2.3 流控

#### B8.2.1 Non-sync 类型 DVM 事务流程

图 B8.1 展示了 Non-sync 类型 DVM 事务中的各个步骤。

请求方 0 ICN

RN-F0 MN

DVMOp(Non-sync)

DBIDResp

NonCopyBackWriteData

SnpDVMOp_P1

SnpDVMOp_P2

SnpResp_I

Comp

只有在所有先前的 DVMOp 事务都收到 Comp 响应之后，才能发送 DVMOp(Sync)。

RN-F0 MN

![Figure p341](images/fig_p0341_1.png)

![Figure p341](images/fig_p0341_2.png)

图 B8.1：Non-sync 类型 DVM 事务流程

图 B8.1 所示的必需步骤如下：

1. RN-F0 使用与 DVMType 相应的写语义，向杂项节点发送 DVMOp(Non-sync)。
2. 杂项节点接受 DVMOp(Non-sync) 请求，并提供 DBIDResp 响应。
3. RN-F0 在数据通道上发送一个 8 字节数据包。
4. 杂项节点向系统中其余的 RN-F 和 RN-D 节点广播 SnpDVMOp 侦听请求。杂项节点允许（但非必需）向发出原始 DVMOp 的 RN 发送 SnpDVMOp。SnpDVMOp 在侦听通道上发送，并且需要两个侦听请求。SnpDVMOp 的两个部分通过后缀 _P1 和 _P2 来标记。

> **注意**
>
> 消息的两个部分必须携带相同的 TxnID。请求节点必须有可用资源来接受 SnpDVMOp。参见 B8.2.3 流控。

5. 完成所需操作后，SnpDVMOp 的每个接收方都向杂项节点发送单个 SnpResp 响应。

> **注意**
>
> 发送 SnpResp 意味着目标请求节点已将 SnpDVMOp 转发到所需的请求节点结构，并且已释放接受另一个 DVM 操作所需的资源。发送 SnpResp 并不意味着所请求的 DVM 操作已经完成。参见 B8.2.2 Sync 类型 DVM 事务流程。

6. 收到所有 SnpResp 响应之后，杂项节点向请求节点发送 Comp 响应。

##### B8.2.1.1 Non-sync DVMOp 的 DVM 提前 Comp

互连中的杂项节点允许在无需等待完成对请求节点所需的侦听的情况下，为 Non-sync DVMOp 发送 Comp。杂项节点必须将 Non-sync DVMOp 相对于稍后从同一源收到的任何 Sync DVMOp 进行排序。如果杂项节点无法提供这种排序保证，则在为 Non-sync DVMOp 发送 Comp 响应之前，侦听必须已完成。

对于被允许为 Non-sync DVMOp 发送提前 Comp 的杂项节点，允许其择机将 Comp 和 DBIDResp 响应合并为单个 CompDBIDResp 响应。

> **注意**
>
> 提前为 Non-sync DVMOp 发送 Comp 响应可减少 DVMOp 完成的往返时延。这使单个源可以流水化更多 DVMOp 事务。

这种提前完成也能使 Sync DVMOp 得以继续，该 Sync DVMOp 正在等待同一请求节点此前发送的所有相关 DVMOp 事务的完成。

对于 Sync DVMOp，杂项节点仍必须等待收到该 Sync DVMOp 的侦听响应，之后才能向请求方发送该 Sync DVMOp 的 Comp。

#### B8.2.2 Sync 类型 DVM 事务流程

图 B8.2 展示了 Sync DVM 事务中的流程。

请求方 0  ICN

RN-F0 MN

DVMOp(Sync)

DBIDResp

NonCopyBackWriteData

Comp

RN-F0 MN

![Figure p343](images/fig_p0343_1.png)

图 B8.2：Sync DVM 事务流程

图 B8.2 所示的必需步骤如下：

1. RN-F0 向杂项节点发送 DVMOp(Sync)。

> **注意**
>
> 所有需要由 DVMOp(Sync) 保证其完成的先前 DVMOp 请求，必须在请求节点可以发送 DVMOp(Sync) 之前已收到 Comp 响应。

2. 杂项节点接受 DVMOp(Sync) 请求，并向请求方发送 DBIDResp 响应。
3. RN-F0 在数据通道上发送一个数据包，数据大小为 8 字节。
4. 杂项节点向 RN-F1 发送 SnpDVMOp。杂项节点允许（但非必需）向发起原始 DVMOp 的 RN 发送 SnpDVMOp。SnpDVMOp 在侦听通道上发送，并且需要两个 Snoop 请求。SnpDVMOp 的两个部分通过后缀 _P1 和 _P2 来标记。
5. 完成 DVM Sync 操作后，RN-F1 向杂项节点发送 SnpResp 响应。

> **注意**
>
> 发送 SnpResp 意味着所有与 DVM 相关的操作都已在请求节点的结构中完成，并且目标请求节点已释放接受另一个 SnpDVMOp 所需的资源。

6. 收到 SnpResp 后，杂项节点向 RN-F0 发送 Comp 响应。

#### B8.2.3 流控

本节描述 DVMOp 和 SnpDVMOp 的流控要求。

##### B8.2.3.1 DVMOp

DVMOp 的流控要求包括：

- DVMOp 可以收到来自杂项节点的 RetryAck 响应。
- 收到 RetryAck 响应的 DVMOp 必须等待来自具有相应 PCrdType 的杂项节点的 PCrdGrant 响应。
- 所有需要由 DVMOp(Sync) 保证其完成的先前 DVMOp 请求，必须在请求节点可以发送 DVMOp(Sync) 之前已收到 Comp 响应。
- 对于 DVMOp(Non-sync) 操作，互连必须保证前向进展。这要求杂项节点中至少有一个 tracker 条目保留给 DVMOp(Non-Sync)。
- 如果并不要求 DVMOp(Sync) 保证 DMVOp(Non-sync) 的完成，则允许来自同一请求节点的 DVMOp(Non-sync) 与 DVMOp(Sync) 重叠。

##### B8.2.3.2 SnpDVMOp

SnpDVMOp 的流控要求包括：

- 每个 SnpDVMOp 事务需要两个 SnpDVMOp 请求包。
- 对应于单个事务的两个 SnpDVMOp 请求包：
* 必须使用相同的 TxnID。* 必须使用相同的 SNP 通道 RP。有关更多信息，参见 B14.2.1.2 Flow control with Resource Planes。
* 可以按任意顺序发送或接收。
- 为了防止因使用侦听通道的两部分 SnpDVMOp 请求而产生死锁，只有当接收方请求节点已预分配资源以接受 SnpDVMOp 事务的两个部分时，才可以发送 SnpDVMOp 事务。
- 只有当对应于某个 SnpDVMOp 事务的两个 SnpDVMOp 请求包都已收到后，请求节点才必须为该事务提供响应。
- 只有当能够接受来自杂项节点的下一个 SnpDVMOp 时，请求节点才必须为某个 SnpDVMOp 提供响应。
- 系统中的每个 RN-F 和 RN-D 都会指定其可以并发接受的 SnpDVMOp 事务数量。
- 系统中的每个 RN-F 和 RN-D，除能接受 SnpDVMOp(Sync) 事务外，必须至少还能接受一个 SnpDVMOp(Non-Sync) 事务。
- 必须并发接受的 SnpDVMOp 事务的最小数量为两个。对于未指定数量的请求节点，这是默认数量。
- 对于已发出 StashOnceSep 请求的请求节点，其 SnpDVMOp(Sync) 操作必须等待，直到收到所有使用已失效页表项的 StashOnceSep 请求的未完成 StashDone 响应。

对于 SnpDVMOp(Non-sync)：

- 从杂项节点可以有多个 SnpDVMOp(Non-sync) 事务处于未完成状态。
- 请求节点在响应 SnpDVMOp(Non-Sync) 时，不得依赖任何事务的前向进展。

对于 SnpDVMOp(Sync)：

- 从杂项节点发往请求节点的 SnpDVMOp(Sync) 只能有一个处于未完成状态。
- 在响应 SnpDVMOp(Sync) 时，请求节点可以依赖任何事务的前向进展，但未完成的 DVMOp(Sync) 请求的完成除外。
- 即使请求节点持续收到更多 DVM 失效操作，它也必须及时完成 SnpDVMOp(Sync)。

### B8.3 DVMOp 字段取值限制

DVMOp 事务期间的字段取值限制如下所示：

- 表 B8.1 中的 Request 消息
- 表 B8.2 中的 Response 消息、DBIDResp、Comp 和 SnpResp
- 表 B8.3 中的 Snoop 消息
- 表 B8.4 中的 Data 消息

#### B8.3.1 Request DVMOp 字段取值限制

表 B8.1 给出了 DVMOp 事务的 Request 消息字段取值限制。

表 B8.1：DVMOp 的 Request 消息字段取值限制

| 字段名 | 限制 |
| --- | --- |
| QoS | 无限制。可以取任意值。 |
| TgtID | 预期为杂项节点的节点 ID。互连可以将其重新映射为正确的 TgtID。 |
| SrcID | 发起该 DVM 消息的请求方的源 ID。 |
| TxnID | 由请求方生成的 ID。必须遵循与任何其他事务相同的规则。 |
| ReturnNID StashNID DataTarget | 必须全为零。 |

StashNIDValid 必须为零。

Endian

Deep

PrefetchTgtHint

ReturnTxnID 必须全为零。

StashLPIDValid

StashLPID

Opcode 必须为 DVMOp。

MultiReq 必须为零。

NumReq 必须指示事务 Size 为 8 字节。

Size

Req_Addr_Width。参见 B8.4 DVM messages。Addr

PAS 必须全为零。

LikelyShared 必须为零。

下页续

表 B8.1 – 续上页

| 字段名 | 限制 |
| --- | --- |
| AllowRetry | 可以取任意值，因为 DVMOp 可能被给予 Retry。 |
| Order | 必须全为零。 |
| PCrdType | 如果 AllowRetry 为 1，则必须全为零，否则为 Credit 类型值。 |
| MemAttr | 必须全为零。 |
| SnpAttr DoDWT | 无限制。可以取任意值。参见 DVM domain。 |
| PGroupID StashGroupID TagGroupID | 不适用。必须全为零。 |
| LPID | 无限制。可以取任意值。 |
| Excl SnoopMe CAH | 必须为零。 |
| ExpCompAck | 必须为零。 |
| TagOp | 必须为零。 |
| TraceTag | 无限制。 |
| MPAM | 必须全为零。 |
| PBHA | 必须全为零。 |
| MECID StreamID | 必须全为零。 |
| SecSID1 | 必须为零。 |
| RSVDC | 无限制。可以取任意值。 |

#### B8.3.2 Response DVMOp 字段取值限制

表 B8.2 给出了 DVMOp 事务期间 Response 的 DBIDResp、SnpResp 以及 Comp 和 CompDBIDResp 消息字段取值限制。

表 B8.2：DVMOp 期间响应消息字段取值的限制

| 字段 | DBIDResp 消息 | Comp 和 CompDBIDResp 消息 | SnpResp 消息 |
| --- | --- | --- | --- |
| QoS | 无限制。可以取任意值。 | 无限制。可以取任意值。 | 无限制。可以取任意值。 |
|  |  |  | 下页续 |

表 B8.2 – 续上页

| 字段 | DBIDResp 消息 | Comp 和 CompDBIDResp 消息 | SnpResp 消息 |
| --- | --- | --- | --- |
| TgtID | 必须为原始请求方的 ID | 必须为原始请求方的 ID | 必须为正在处理 DVMOps 的杂项节点的 ID |
| SrcID | 必须为正在处理 DVMOps 的杂项节点的 ID。 | 必须为正在处理 DVMOps 的杂项节点的 ID。 | 必须为响应 snoop 的节点 ID |
| TxnID | 必须与原始请求的 TxnID 匹配 | 必须与原始请求的 TxnID 匹配 | 必须与 SnpDVMOp snoop 请求的 TxnID 匹配 |
| Opcode | 必须为 DBIDResp | 必须为 Comp 或 CompDBIDResp | 必须为 SnpResp |
| RespErr | 必须全为零 | 必须为 0b00、0b10 或 0b11 | 必须为 0b00 或 0b11 |
| Resp | 必须全为零 | 必须全为零 | 必须全为零 |
| FwdState DataPull | 必须全为零 | 必须全为零 | 必须全为零 |
| CBusy | 预期由完成方填充 | 预期由完成方填充 | 预期由完成方填充 |
| DBID PGroupID StashGroupID TagGroupID | 由正在处理 DVMOps 的杂项节点生成 | 由正在处理 DVMOps 的杂项节点生成 | 无限制。可以取任意值。 |
| PCrdType | 必须全为零 | 必须全为零 | 必须全为零 |
| TagOp | 必须全为零 | 必须全为零 | 必须全为零 |
| TraceTag | 无限制 | 无限制 | 无限制 |
| CacheLineID | 必须全为零 | 必须全为零 | 必须全为零 |

#### B8.3.3 Snoop DVMOp 字段取值限制

表 B8.3 给出了 DVMOp 事务期间 Snoop 消息字段（SnpDVMOp）的取值限制。

表 B8.3：DVMOp 的 Snoop 消息字段取值限制

| 字段名 | 限制 |
| --- | --- |
| QoS | 无限制。可以取任意值。 |
| SrcID | 必须为 Miscellaneous Node 的节点 ID。 |
| TxnID | 由 Miscellaneous Node 生成的 ID。 |

FwdNID 无限制。用作 Range 和 Num[4:0] 字段。参见表 B8.14。

下页续

表 B8.3 – 续上页

| 字段名 | 限制 |
| --- | --- |
| PBHA |  |
| FwdTxnID StashLPIDValid StashLPID | 不适用。必须为零。 |
| VMIDExt | 在 SnpDVMOp 请求的 Part 1 中必须用于 VMID[15:8]。在 SnpDVMOp 请求的 Part 2 中无限制，可以取任意值。 |
| Opcode | 必须为 SnpDVMOp。 |
| Addr | (Req_Addr_Width) - 3。参见 B8.4 DVM messages。 |
| PAS | 必须全为零。 |
| DoNotGoToSD | 必须为零。 |
| RetToSrc | 必须为零。 |
| TraceTag | 无限制。 |
| MPAM | 必须全为零。 |
| MECID | 必须全为零。 |

对应于单个 DVMOp 的两个 SnpDVMOp 请求数据包，在以下字段中必须具有相同的值：

- TxnID
- Opcode
- SrcID

#### B8.3.4 Data DVMOp 字段取值限制

表 B8.4 给出了 DVMOp 事务的 Data 消息字段取值限制。

表 B8.4：DVMOp 的 Data 消息字段取值限制

| 字段名 | 限制 |
| --- | --- |
| QoS | 无限制。可以取任意值。 |
| TgtID | 必须与 DBIDResp 响应中返回的 SrcID 相同。 |
| SrcID | 必须为原始 Requester 的 ID。 |
| TxnID | 必须与 DBIDResp 响应的 DBID 相同。 |
|  | 下页续 |

表 B8.4 – 续上页

| 字段名 | 限制 |
| --- | --- |
| HomeNID MismatchedMECID PBHA | 必须全为零。 |
| Opcode | 必须为 NonCopyBackWriteData。 |
| RespErr | 必须为 0b00 或 0b10。 |
| Resp | 必须全为零。 |
| FwdState DataSource | 必须全为零。 |
| DataPull | 必须为零。 |
| CBusy | 必须全为零。 |
| MECID DBID | 公共字段的最高有效 4 位必须全为零。无限制。可以取任意值。 |
| CCID | 必须全为零。 |
| DataID | 必须全为零。 |
| CacheLineID | 必须全为零。 |
| TagOp | 必须全为零。 |
| Tag | 必须为零。 |
| TU | 必须为零。 |
| TraceTag | 无限制。 |
| CAH | 必须为零。 |
| NumDat | 必须全为零。 |
| Replicate | 必须为零。 |
| RSVDC | 无限制。 |
| BE | 仅 BE[7:0] 必须为 1。 |
| Data | 对于 Data[63:0]，未使用的位必须为零；Data [n:64] = 可以取任意值。 |
| DataCheck | 必须为 Data 字段的相应值。 |
| Poison | 无限制。可以取任意值。 |

### B8.4 DVM messages

本节提供有关各种 DVM 消息、其关联字段以及 flit 打包的补充信息。它包含以下小节：

- B8.4.1 DVM message payload
- B8.4.2 DVM message packing
- B8.4.3 TLB Invalidate
- B8.4.4 Branch Predictor Invalidate
- B8.4.5 Instruction Cache Invalidate
- B8.4.6 Synchronization

#### B8.4.1 DVM message payload

从 Request Node 到 Miscellaneous Node 的 DVM 操作的 payload 分配在以下位置：

- Request Node 发出的 DVM 请求中的 Addr 字段。
- NonCopyBackWriteData 数据包中 Data 的低 8 字节。

从 Miscellaneous Node 到 Request Node 的 DVM 操作的 payload 使用 Addr 字段分布在两个 SnpDVMOp 请求数据包中。

建议（但并非必需）SnpDVMOp 消息的 payload 与其所源自的 DVMOp 消息的 payload 相匹配。

表 B8.5 给出了 payload 中的各种字段及其编码。

表 B8.5：DVMOp 字段与编码

| 字段 | 位 | 功能 |
| --- | --- | --- |
| AddrV | 1 | 指示消息是否包含地址。0b0 不包含地址 0b1 包含地址 |

Virtual Index（VI）有效。VIV 2 0b00 VI 无效

0b01 保留

0b10 保留

0b11 VI 有效

0b1 指示 Virtual Machine Identifier（VMID）有效。VMIDV 1

0b1 指示 Address Space Identifier（ASID）有效。ASIDV 1

Security 2 指示该无效操作适用于哪个安全状态。

各 DVMType 的编码参见表 B8.8。

Exception 2 指示该事务适用于：0b00 Hypervisor 和所有 Guest OS

EL3a 0b01

0b10 Guest OS

0b11 Hypervisor

下页续

表 B8.5 – 续上页

字段 位 功能

DVMType 3 指示 DVM 操作类型，如下：0b000 TLB Invalidate（TLBI）

0b001 Branch Predictor Invalidate（BPI）

0b010 Physical Instruction Cache Invalidate（PICI）

0b011 Virtual Instruction Cache Invalidate（VICI）

0b100 Synchronization

0b101-0b111 保留

VMID 8 Virtual Machine Identifier VMID[7:0]

ASID 16 Address Space Identifier

Stage 2 指示 Stage 1 或 Stage 1 无效：0b00 DVMv7：任意事务 DVMv8：Stage 1 和 Stage 2 均无效

0b01 仅 Stage 1 无效

0b10 仅 Stage 2 无效

0b11 Granule Protection Table（GPT）

Leaf 1 指示是否仅无效叶子条目：0b0 无效所有关联的转换。

0b1 仅无效叶子条目。

Rangeb 1 Range 可以取任意值。当 Range = 0 时，该事务为基于非范围的 TLBI 操作。对于非 TLBI 的 DVM 事务，Range 不适用且必须为零。

当 Range = 1 时，该事务为基于范围的 TLBI 操作。

Numb 5 在范围计算中用作常量乘数。所有二进制值均有效。

Scaleb 2 在地址范围指数计算中用作常量。所有二进制值均有效。

TTLb 包含待无效地址的 Translation Table Level（TTL）提示。2 详见 表 B8.6 和 表 B8.7。

下页续

表 B8.5 – 续上页

字段 位 功能

TGb Translation Granule（TG） 2 对于非范围的 TLB Invalidations，TG 和 TTL 指示表层级提示，参见表 B8.7。对于按范围的 TLB Invalidations，TG 指示颗粒大小：0b00 保留

0b01 4K

0b10 16K

0b11 64K

BaseAddr 37-41 范围的移位后基地址，根据 TG 进行移位：4K 时 BaseAddr 为 VA[MaxVA:12]

16K 时 BaseAddr 为 VA[MaxVA:14]，VA[13:12] 必须为零

64K 时 BaseAddr 为 VA[MaxVA:16]，VA[15:12] 必须为零

VA 或 49-53 Virtual Address。此字段还用于为任何 TLBI by IPA 操作传输 Intermediate Physical Address（IPA）。

PA 44-52 Physical Address。此字段用于为 GPT TLBI by PA 和 PICI by PA 操作传输 Physical Address。

VI 16 Virtual Index

VMIDExt 8 Virtual Machine Identifier VMID[15:8]

下页续

表 B8.5 – 续上页

字段 位 功能

ISc 用于 GPT TLBI by PA 操作的 Invalidation Size（IS）编码，4 包括仅 Leaf：0b0000 4K

0b0001 16K

0b0010 64K

0b0011 2MB

0b0100 32MB

0b0101 512MB

0b0110 1GB

0b0111 16GB

0b1000 64GB

0b1001 512GB

0b1010-0b1111 保留

a 仅 DVMv8。b 在非 TLBI 的 DVM 操作中不适用且必须为零。关于这些字段在 TLBI 操作中如何使用，参见 B8.4.3.1 TLB Invalidate by Range 和 B8.4.3.2 Level Hint in TLBI operations。c 在非 GPT TLBI by PA 或 GPT TLBI by PA, Leaf only 的 DVM 操作中不适用且必须为零。参见 B8.4.3.3 Invalidation Size in GPT TLBI by PA operations。

##### TTL 与 TG 字段

对于按地址范围执行的 TLB 无效（TLB Invalidation），TTL 字段可以指示翻译表遍历（translation table walk）的哪一级持有被无效地址的叶条目（leaf entry）。其编码如表 B8.6 所示。

表 B8.6：基于范围的 TLB 无效的叶条目提示

| TTL | 含义 |
| --- | --- |
| 0b00 | 无层级提示信息。 |
| 0b01 | 叶条目位于翻译表遍历的第 1 级。 |
| 0b10 | 叶条目位于翻译表遍历的第 2 级。 |
| 0b11 | 叶条目位于翻译表遍历的第 3 级。 |

对于按非范围地址执行的 TLB 无效，TTL 和 TG 字段指示翻译表遍历的哪一级持有被无效地址的叶条目。

其编码如表 B8.7 所示。

表 B8.7：非范围 TLB 无效的叶条目提示

| TG | TTL | 含义 |
| --- | --- | --- |
| 0b00 | 0b00 | 无层级提示 |
|  | 0b01 | 保留 |
|  | 0b10 | 保留 |
|  | 0b11 | 保留 |
| 0b01 | 0b00 | 叶条目位于翻译表遍历的第 0 级。 |
|  | 0b01 | 叶条目位于翻译表遍历的第 1 级。 |
|  | 0b10 | 叶条目位于翻译表遍历的第 2 级。 |
|  | 0b11 | 叶条目位于翻译表遍历的第 3 级。 |
| 0b10 | 0b00 | 保留 |
|  | 0b01 | 叶条目位于翻译表遍历的第 1 级。 |
|  | 0b10 | 叶条目位于翻译表遍历的第 2 级。 |
|  | 0b11 | 叶条目位于翻译表遍历的第 3 级。 |
| 0b11 | 0b00 | 保留 |
|  | 0b01 | 叶条目位于翻译表遍历的第 1 级。 |
|  | 0b10 | 叶条目位于翻译表遍历的第 2 级。 |
|  | 0b11 | 叶条目位于翻译表遍历的第 3 级。 |

##### Security 字段

Security 字段的含义随 DVMType 的不同而不同，如表 B8.8 所示。

表 B8.8：各 DVMType 的 Security 字段编码

| Security TLBI BPI PICI Invalidation All | Invalidation by PA | VICI |
| --- | --- | --- |
| 00 Realm Secure 和 Non-secure Root、Realm、Secure 和 Non-secure | Root | Secure 和 Non-secure |
| 01 来自 Secure 上下文的 Non-secure 地址 Reserved Realm 和 Non-secure | Realm | Reserved |
| 10 Secure Reserved Secure 和 Non-secure | Secure | Secure |
| 11 Non-secure Reserved Non-secure | Non-secure | Non-secure |
| ASID 字段 ASID 字段包含 8 位或 16 位地址空间标识符（Address Space Identifier）。 |  |  |

- Armv7 支持 8 位 ASID。
- 从 Armv8 起，支持 8 位和 16 位 ASID。

无法从一条 DVM 消息判断该消息使用的是 8 位 ASID 还是 16 位 ASID。

所有 8 位 ASID 消息都必须将 ASID[15:8] 位设置为零。

预计大多数系统会在整个系统中使用单一的 ASID 宽度，即 8 位 ASID 或 16 位 ASID。

在同时包含 8 位 ASID 和 16 位 ASID 组件的系统中，预计所有维护操作都由使用 16 位 ASID 的代理完成。这样可确保该代理能够对 8 位 ASID 和 16 位 ASID 组件都执行维护。

互操作性要求包括：

- 对于由 8 位 ASID 代理向 16 位 ASID 代理发送消息的情况，消息表现为 16 位 ASID，

其高 8 位被设置为零。

- 对于由 16 位 ASID 代理向 8 位 VMID 代理发送消息的情况：
- 如果高 8 位为零，则该消息被正确接收。
- 如果高 8 位非零，则会发生过度无效（over-invalidation），因为 8 位 ASID 代理会忽略高 8

位。

##### VMID 字段

VMID 字段包含 8 位或 16 位的虚拟机标识符（Virtual Machine Identifier）。

- Armv7 和 Armv8 支持 8 位 VMID。
- 从 Armv8.1 起，支持 8 位和 16 位 VMID。

无法从 DVM 消息判断该消息使用的是 8 位还是 16 位 VMID。

所有 8 位 VMID 消息都必须将 VMID[15:8] 字段置零。

预期大多数系统在整个系统中使用单一的 VMID 大小，即 8 位 VMID 或 16 位 VMID。

在同时包含 8 位 VMID 和 16 位 VMID 组件的系统中，预期所有维护都由使用 16 位 VMID 的代理完成。这确保该代理能够对 8 位 VMID 和 16 位 VMID 组件都执行维护。

互操作性要求包括：

- 对于 8 位 VMID 代理向 16 位 VMID 代理发送消息的情况，该消息表现为 16 位 VMID，其高 8 位被置零。
- 对于 16 位 VMID 代理向 8 位 VMID 代理发送消息的情况：
- 如果高 8 位为零，则该消息被正确接收。
- 如果高 8 位非零，则由于 8 位 VMID 代理忽略高 8 位，会发生过度无效化（over-invalidation）。

从 Armv8.1 起，使用 VMIDExt 传输 16 位 VMID 的高字节。

##### DVM 域

DVM 请求中的 SnpAttr 位用于区分 Inner 域和 Outer 域。

表 B8.9 给出了 DVM 事务中 SnpAttr 的取值编码。

表 B8.9：DVM 事务中 SnpAttr 取值编码

| SnpAttr | 域取值 |
| --- | --- |
| 0 | Inner 域 |
| 1 | Outer 域 |

还定义了两个额外的可选接口广播引脚：BROADCASTTLBIINNER 和 BROADCASTTLBIOUTER。它们决定互连中 TLBI 操作的广播。

#### B8.4.2 DVM 消息打包

表 B8.10 给出了来自请求节点的 DVMOp 请求中载荷的分布（采用 8 字节写语义），以及来自杂项节点的 SnpDVMOp 请求中载荷的分布。

在 DVMOp 中，请求中的地址字段与 8 字节写数据的组合承载完整载荷。请求中不使用 REQ.Addr[3]，其必须为零。

在两个 SnpDVMOp 请求中，两个地址字段的组合承载完整载荷。在 SnpDVMOp 请求中使用 SNP.Addr[0] 来指示正在传输的是载荷的哪一部分。

Maximum PA（MPA）与 Maximum VA（MVA）地址位的合法组合为：

- MPA = 44 : MVA = 49
- MPA = 45 : MVA = 51
- MPA = 46 至 52 : MVA = 53

参见 B8.4.3.1 TLB Invalidate by Range 和 B8.4.3.2 Level Hint in TLBI operations。

表 B8.10：DVMOp 与 SnpDVMOp 请求载荷

DVMOp REQ 中的 X DVMOp DAT SnpDVMOp 中的 X SnpDVMOp REQ.Addr[x] SNP.Addr[x] 请求第 1 部分 请求第 2 部分 位数 位数 DAT.Data[x]

Num[2:0]a 2:0 3 - 0 1 0 1 Num[3]a 3 1 0

4 1 AddrV Scale[0] 1 1 AddrV Scale[0] PA[6] PA[6] VA[6] VA[6]

5 1 VMIDV Scale[1] 2 1 VMIDV Scale[1] VIV[0] PA[7] VIV[0] PA[7] VA[7] VA[7]

6 1 ASIDV IS[0] 3 1 ASIDV IS[0] VIV[1] TTL[0] VIV[1] TTL[0] PA[8] PA[8] VA[8] VA[8]

7 1 Security[0] IS[1] 4 1 Security[0] IS[1] TTL[1] TTL[1] PA[9] PA[9] VA[9] VA[9]

8 1 Security[1] IS[2] 5 1 Security[1] IS[2] TG[0] TG[0] PA[10] PA[10] VA[10] VA[10]

下页续

表 B8.10 – 续上页

DVMOp REQ 中的 X DVMOp DAT SnpDVMOp 中的 X SnpDVMOp REQ.Addr[x] SNP.Addr[x] 请求第 1 部分 请求第 2 部分 位数 位数 DAT.Data[x]

9 1 Exception[0] IS[3] 6 1 Exception[0] IS[3] TG[1] TG[1] PA[11] PA[11] VA[11] VA[11]

10 1 Exception[1] VA[12] 7 1 Exception[1] VA[12] PA[12] PA[12]

13:11 3 DVMType[2:0] VA[15:13] 10:8 3 DVMType[2:0] VA[15:13] PA[15:13] PA[15:13]

21:14 8 VMID[7:0] VA[23:16] 18:11 8 VMID[7:0] VA[23:16] VI[27:20] PA[23:16] VI[27:20] PA[23:16]

37:22 16 ASID[15:0] VA[39:24] 34:19 16 ASID[15:0] VA[39:24] VI[19:12]b VI[19:12]b PA[39:24] PA[39:24]

39:38 2 Stage VA[41:40] 36:35 2 Stage VA[41:40] PA[41:40] PA[41:40]

40 1 Leaf VA[42] 37 1 Leaf VA[42] PA[42] PA[42] Rangec 41 1 VA[43] 38 1 VA[46] VA[43] PA[43] PA[43] Num[4]a 42 1 VA[44] 39 1 VA[47] VA[44] PA[44] PA[44]

43 1 - VA[45] 40 1 VA[48] VA[45] PA[45] PA[45]

44 1 - VA[46] 41 1 VA[50] VA[49] PA[46] PA[46]

45 1 - VA[47] 42 1 VA[52] VA[51] PA[47] PA[47]

48:46 3 - VA[50:48] 45:43 3 - PA[50:48] PA[50:48]

49 1 - VA[51] 46 1 - PA[51] PA[51]

50 1 - VA[52] 47 1 - -

55:51 5 - - 48 1 - - VMID[15:8]d 63:56 8 -

a 对于 SnpDVMOp 请求的第 2 部分，来自相应 DVMOp 请求的 Num[4:0] 随 Addr 字段一起承载在侦听 flit 的 FwdNID 字段上。参见表 B8.14。 b 当用作虚拟索引 VA（VI VA）时，REQ.Addr[37:30]、DAT.Data[37:30] 和 SNP.Addr[34:27] 可以取任意值。 c 对于 SnpDVMOp 请求的第 1 部分，来自相应 DVMOp 请求的 Range 随 Addr 字段一起承载在侦听 flit 的 FwdNID 字段上。参见表 B8.14。 d 对于 SnpDVMOp 请求的第 1 部分，来自相应 DVMOp 请求的 VMID[15:8] 随 Addr 字段一起承载在侦听 flit 的 VMIDExt 字段上。参见表 B13.8。

#### B8.4.3 TLB Invalidate

本节详细介绍 TLB Invalidate（TLBI）消息。

对于 TLBI 消息，部分字段具有固定取值，如表 B8.11 所示

表 B8.11：TLBI 消息的固定字段取值

| 字段 | 取值 | 状态 |
| --- | --- | --- |
| DVMType | 0b000 | TLBI |

TLBI 必须作用于哪些条目取决于消息中的字段。

表 B8.12 列出了所有支持的 TLBI 操作。

Arm 列指示支持该消息所需的最低 Arm 架构版本。

表 B8.12：TLBI 消息

| 操作 | Arm | Exception | Security | VMIDV | ASIDV | Leaf | Stage | AddrV |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| EL3 TLBI all | v8 | 0b01 | 0b10 | 0b0 | 0b0 | 0b0 | 0b00 | 0b0 |
| EL3 TLBI by VA | v8 | 0b01 | 0b10 | 0b0 | 0b0 | 0b0 | 0b00 | 0b1 |
| EL3 TLBI by VA, Leaf only | v8 | 0b01 | 0b10 | 0b0 | 0b0 | 0b1 | 0b00 | 0b1 |
| Secure Guest OS TLBI by Non-secure IPA | v8.4 | 0b10 | 0b01 | 0b1 | 0b0 | 0b0 | 0b10 | 0b1 |
| Secure Guest OS TLBI by Non-secure IPA, Leaf only | v8.4 | 0b10 | 0b01 | 0b1 | 0b0 | 0b1 | 0b10 | 0b1 |
| Secure TLBI all | v7 | 0b10 | 0b10 | 0b0 | 0b0 | 0b0 | 0b00 | 0b0 |
| Secure TLBI by VA | v7 | 0b10 | 0b10 | 0b0 | 0b0 | 0b0 | 0b00 | 0b1 |
| Secure TLBI by VA, Leaf only | v8 | 0b10 | 0b10 | 0b0 | 0b0 | 0b1 | 0b00 | 0b1 |
| Secure TLBI by ASID | v7 | 0b10 | 0b10 | 0b0 | 0b1 | 0b0 | 0b00 | 0b0 |
| Secure TLBI by ASID and VA | v7 | 0b10 | 0b10 | 0b0 | 0b1 | 0b0 | 0b00 | 0b1 |
| Secure TLBI by ASID and VA, Leaf only | v8 | 0b10 | 0b10 | 0b0 | 0b1 | 0b1 | 0b00 | 0b1 |
| Secure Guest OS TLBI all | v8.4 | 0b10 | 0b10 | 0b1 | 0b0 | 0b0 | 0b00 | 0b0 |
| Secure Guest OS TLBI by VA | v8.4 | 0b10 | 0b10 | 0b1 | 0b0 | 0b0 | 0b00 | 0b1 |
| Secure Guest OS TLBI all, Stage 1 only | v8.4 | 0b10 | 0b10 | 0b1 | 0b0 | 0b0 | 0b01 | 0b0 |
| Secure Guest OS TLBI by Secure IPA | v8.4 | 0b10 | 0b10 | 0b1 | 0b0 | 0b0 | 0b10 | 0b1 |
| Secure Guest OS TLBI by VA, Leaf only | v8.4 | 0b10 | 0b10 | 0b1 | 0b0 | 0b1 | 0b00 | 0b1 |
| Secure Guest OS TLBI by Secure IPA, Leaf only | v8.4 | 0b10 | 0b10 | 0b1 | 0b0 | 0b1 | 0b10 | 0b1 |
| Secure Guest OS TLBI by ASID | v8.4 | 0b10 | 0b10 | 0b1 | 0b1 | 0b0 | 0b00 | 0b0 |
| Secure Guest OS TLBI by ASID and VA | v8.4 | 0b10 | 0b10 | 0b1 | 0b1 | 0b0 | 0b00 | 0b1 |
| Secure Guest OS TLBI by ASID and VA, Leaf only | v8.4 | 0b10 | 0b10 | 0b1 | 0b1 | 0b1 | 0b00 | 0b1 |
| All OS TLBI all | v7 | 0b10 | 0b11 | 0b0 | 0b0 | 0b0 | 0b00 | 0b0 |
| Guest OS TLBI all, Stage 1 and 2 | v7 | 0b10 | 0b11 | 0b1 | 0b0 | 0b0 | 0b00 | 0b0 |

下页续

表 B8.12 – 续上页

| 操作 | Arm | Exception | Security | VMIDV | ASIDV | Leaf | Stage | AddrV |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Guest OS TLBI by VA | v7 | 0b10 | 0b11 | 0b1 | 0b0 | 0b0 | 0b00 | 0b1 |
| Guest OS TLBI all, Stage 1 only | v8 | 0b10 | 0b11 | 0b1 | 0b0 | 0b0 | 0b01 | 0b0 |
| Guest OS TLBI by IPA | v8 | 0b10 | 0b11 | 0b1 | 0b0 | 0b0 | 0b10 | 0b1 |
| Guest OS TLBI by VA, Leaf only | v8 | 0b10 | 0b11 | 0b1 | 0b0 | 0b1 | 0b00 | 0b1 |
| Guest OS TLBI by IPA, Leaf only | v8 | 0b10 | 0b11 | 0b1 | 0b0 | 0b1 | 0b10 | 0b1 |
| Guest OS TLBI by ASID | v7 | 0b10 | 0b11 | 0b1 | 0b1 | 0b0 | 0b00 | 0b0 |
| Guest OS TLBI by ASID and VA | v7 | 0b10 | 0b11 | 0b1 | 0b1 | 0b0 | 0b00 | 0b1 |
| Guest OS TLBI by ASID and VA, Leaf only | v8 | 0b10 | 0b11 | 0b1 | 0b1 | 0b1 | 0b00 | 0b1 |
| Secure Hypervisor TLBI all | v8.4 | 0b11 | 0b10 | 0b0 | 0b0 | 0b0 | 0b00 | 0b0 |
| Secure Hypervisor TLBI by VA | v8.4 | 0b11 | 0b10 | 0b0 | 0b0 | 0b0 | 0b00 | 0b1 |
| Secure Hypervisor TLBI by VA, Leaf only | v8.4 | 0b11 | 0b10 | 0b0 | 0b0 | 0b1 | 0b00 | 0b1 |
| Secure Hypervisor TLBI by ASID | v8.4 | 0b11 | 0b10 | 0b0 | 0b1 | 0b0 | 0b00 | 0b0 |
| Secure Hypervisor TLBI by ASID and VA | v8.4 | 0b11 | 0b10 | 0b0 | 0b1 | 0b0 | 0b00 | 0b1 |
| Secure Hypervisor TLBI by ASID and VA, Leaf only | v8.4 | 0b11 | 0b10 | 0b0 | 0b1 | 0b1 | 0b00 | 0b1 |
| Hypervisor TLBI all | v7 | 0b11 | 0b11 | 0b0 | 0b0 | 0b0 | 0b00 | 0b0 |
| Hypervisor TLBI by VA | v7 | 0b11 | 0b11 | 0b0 | 0b0 | 0b0 | 0b00 | 0b1 |
| Hypervisor TLBI by VA, Leaf only | v8 | 0b11 | 0b11 | 0b0 | 0b0 | 0b1 | 0b00 | 0b1 |
| Hypervisor TLBI by ASID | v8.1 | 0b11 | 0b11 | 0b0 | 0b1 | 0b0 | 0b00 | 0b0 |
| Hypervisor TLBI by ASID and VA | v8.1 | 0b11 | 0b11 | 0b0 | 0b1 | 0b0 | 0b00 | 0b1 |
| Hypervisor TLBI by ASID and VA, Leaf only | v8.1 | 0b11 | 0b11 | 0b0 | 0b1 | 0b1 | 0b00 | 0b1 |
| Realm TLBI all | v9.2 | 0b10 | 0b00 | 0b0 | 0b0 | 0b0 | 0b00 | 0b0 |
| Realm Guest OS TLBI all, Stage 1 only | v9.2 | 0b10 | 0b00 | 0b1 | 0b0 | 0b0 | 0b01 | 0b0 |
| Realm Guest OS TLBI all, Stage 1 and 2 | v9.2 | 0b10 | 0b00 | 0b1 | 0b0 | 0b0 | 0b00 | 0b0 |
| Realm Guest OS TLBI by VA | v9.2 | 0b10 | 0b00 | 0b1 | 0b0 | 0b0 | 0b00 | 0b1 |
| Realm Guest OS TLBI by VA, Leaf only | v9.2 | 0b10 | 0b00 | 0b1 | 0b0 | 0b1 | 0b00 | 0b1 |
| Realm Guest OS TLBI by ASID | v9.2 | 0b10 | 0b00 | 0b1 | 0b1 | 0b0 | 0b00 | 0b0 |
| Realm Guest OS TLBI by ASID and VA | v9.2 | 0b10 | 0b00 | 0b1 | 0b1 | 0b0 | 0b00 | 0b1 |
| Realm Guest OS TLBI by ASID and VA, Leaf only | v9.2 | 0b10 | 0b00 | 0b1 | 0b1 | 0b1 | 0b00 | 0b1 |
| Realm Guest OS TLBI by IPA | v9.2 | 0b10 | 0b00 | 0b1 | 0b0 | 0b0 | 0b10 | 0b1 |
| Realm Guest OS TLBI by IPA, Leaf only | v9.2 | 0b10 | 0b00 | 0b1 | 0b0 | 0b1 | 0b10 | 0b1 |

下页续

表 B8.12 – 续上页

| 操作 | Arm | Exception | Security | VMIDV | ASIDV | Leaf | Stage | AddrV |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Realm Hypervisor TLBI 全部 | v9.2 | 0b11 | 0b00 | 0b0 | 0b0 | 0b0 | 0b00 | 0b0 |
| Realm Hypervisor 按 VA 的 TLBI | v9.2 | 0b11 | 0b00 | 0b0 | 0b0 | 0b0 | 0b00 | 0b1 |
| Realm Hypervisor 按 VA 的 TLBI，仅叶条目 | v9.2 | 0b11 | 0b00 | 0b0 | 0b0 | 0b1 | 0b00 | 0b1 |
| Realm Hypervisor 按 ASID 的 TLBI | v9.2 | 0b11 | 0b00 | 0b0 | 0b1 | 0b0 | 0b00 | 0b0 |
| Realm Hypervisor 按 ASID 和 VA 的 TLBI | v9.2 | 0b11 | 0b00 | 0b0 | 0b1 | 0b0 | 0b00 | 0b1 |
| Realm Hypervisor 按 ASID 和 VA 的 TLBI，仅叶条目 | v9.2 | 0b11 | 0b00 | 0b0 | 0b1 | 0b1 | 0b00 | 0b1 |
| 按 PA 的 GPT TLBI | v9.2 | 0b01 | 0b10 | 0b0 | 0b0 | 0b0 | 0b11 | 0b1 |
| 按 PA 的 GPT TLBI，仅叶条目 | v9.2 | 0b01 | 0b10 | 0b0 | 0b0 | 0b1 | 0b11 | 0b1 |
| GPT TLBI 全部 | v9.2 | 0b01 | 0b10 | 0b0 | 0b0 | 0b0 | 0b11 | 0b0 |

##### B8.4.3.1 按范围的 TLB 无效操作

当 DVM_Support 为 DVM_v8.4 或更高版本时，按 IPA 或 VA 执行的 TLBI 操作可以选择对某个地址范围进行操作，条件是 Range 字段为 0b1。

基于范围的 TLBI 操作除了包含非基于范围的 TLBI 操作中的载荷字段外，还包含范围专用的载荷。

###### B8.4.3.1.1 按 VA 和 IPA 的 TLBI 操作的基于范围的载荷打包

按 VA 和 IPA 的 TLBI 操作的基于范围的字段包括：

- Range
- BaseAddr
- TG
- TTL
- Scale
- Num

> **注意**
>
> 对于按 VA 或 IPA 的非基于范围的 TLBI 操作，TG 和 TTL 字段可用作 Level 提示。参见 B8.4.3.2 Level Hint in TLBI operations。

对于按 VA 或 IPA 的 TLBI 操作，当 Range 字段置为 1 时，需要无效的地址范围使用以下公式计算，其中以字节为单位的 Translation_Granule_Size 由 TG 值确定，如表 B8.13 所示：

(Num + 1) × 2(5×Scale+1) × Translation_Granule_Size   BaseAddr ≤AddressRange < BaseAddr+

表 B8.13：适用于地址范围的 TLB 指令中的 TG 字段编码

| TG | 翻译粒度大小 | Translation_Granule_Size |
| --- | --- | --- |
| 00 | 保留 | 不适用 |
| 01 | 4KB 翻译粒度 | 4096 |
| 10 | 16KB 翻译粒度 | 16384 |
| 11 | 64KB 翻译粒度 | 65536 |

这些范围参数按表 B8.10 所示放置在请求数据包中。

Range 和 Num 值与相应 Snoop 事务的 Addr 字段一起，承载在 Snoop flit 中的 FwdNID 字段上。SnpDVMOp 中 FwdNID 的未使用位必须为零。

表 B8.14 示出了 Range 和 Num 值在 SnpDVMOp 载荷中的放置位置。

表 B8.14：SnpDVMOp 载荷中 Num 值的放置位置

FwdNID 位 SnpDVMOp 请求

第 1 部分 第 2 部分

0 Range Num[0]

1 0 Num[1]

2 0 Num[2]

3 0 Num[3]

4 0 Num[4]

5 0 0

6 0 0

##### B8.4.3.2 TLBI 操作中的 Level 提示

按 VA 或 IPA 的非基于范围的 TLBI 操作允许使用 TG 和 TTL 字段。这些操作使用 TG 和 TTL 来指示页表遍历的层级，该层级的叶条目标识了正在无效的地址。对于此类操作，适用以下条件：

- Range 字段必须为零。
- Num 和 Scale 字段不适用，并且必须为零。

##### B8.4.3.3 按 PA 的 GPT TLBI 操作中的无效大小

按 PA 的 GPT TLBI 操作执行基于范围的无效操作：从 PA 开始，在 IS 字段所指定的范围内无效 TLB 条目。

按 PA 的 GPT TLBI 操作的基于范围的字段为 IS 和 PA。如果 PA 未按 IS 值对齐，则不要求无效任何 TLB 条目。

IS 字段仅适用于按 PA 的 GPT TLBI 操作。

对于按 PA 的 GPT TLBI 操作，Range 字段适用，并且必须为 1。

对于 GPT TLBI 全部操作，Range 字段适用，并且必须为 0。

对于 GPT TLBI 操作，Num 和 Scale 字段不适用，并且必须为 0。

#### B8.4.4 Branch Predictor Invalidate

本节介绍 Branch Predictor Invalidate（BPI，分支预测器无效化）操作。

BPI 消息用于从分支预测器中无效化虚拟地址。

BPI 消息的固定字段取值如表 B8.15 所示。

表 B8.15：Branch Predictor Invalidate 消息的固定字段取值

| Field | Value | Status |
| --- | --- | --- |
| DVMType | 0b001 | Branch Predictor Invalidate |
| Part Num | - | 见表 B8.10 |
| VMIDV | 0b0 | VMID 字段无效 |
| ASIDV | 0b0 | ASID 字段无效 |
| Security | 0b00 | 同时适用于 Secure 和 Non-secure |
| Exception | 0b00 | 适用于所有 Guest OS 和 Hypervisor |
| VMID VMIDExt | 0xXX | 未指定 VMID |
| ASID | 0xXXXX | 未指定 ASID |
| Stage | 0b00 | 保留，置为 0 |
| Leaf | 0b0 | 保留，置为 0 |
| 注 不支持 BPI 与 16 位 ASID 一起使用。 |  |  |

表 B8.16 列出了 BPI 支持的操作。

表 B8.16：Branch Predictor Invalidate 操作

| Operation | Arm | AddrV |
| --- | --- | --- |
| Branch Predictor Invalidate all | v7 | 0b0 |
| Branch Predictor Invalidate by VA | v7 | 0b1 |

#### B8.4.5 Instruction Cache Invalidate

指令缓存可以使用 PA 或 VA 来标记其包含的数据。一个系统中可能同时包含这两种形式的缓存。

DVM 协议既包含使用物理地址（Physical Address）的指令缓存无效化操作，也包含使用虚拟地址（Virtual Address）的指令缓存无效化操作。

接收 DVM 消息的组件必须同时支持这两种形式的报文，与所实现的指令缓存类型无关。当收到的某条消息格式不是该缓存类型的原生格式时，可能需要进行过度无效化（over-invalidation）。

##### B8.4.5.1 Physical Instruction Cache Invalidate

本节介绍 DVM 消息所支持的 Physical Instruction Cache Invalidate（PICI，物理指令缓存无效化）操作。该消息类型也用于虚拟索引物理标记（Virtually Indexed Physically Tagged，VIPT）的指令缓存。

PICI 消息的固定字段取值如表 B8.17 所示。

表 B8.17：Physical Instruction Cache Invalidate 消息的固定字段取值

| Field | Value | Status |
| --- | --- | --- |
| DVMType | 0b010 | Physical Instruction Cache Invalidate |
| Part Num | - | 见表 B8.10 |
| Exception | 0b00 | 适用于所有 Guest OS 和 Hypervisor |
| Stage | 0b00 | 保留，置为 0 |
| Leaf | 0b0 | 保留，置为 0 |

所有支持的 PICI 操作如表 B8.18 所示。

表 B8.18：Physical Instruction Cache Invalidate 操作

| Operation | Arm | Security | VIV | AddrV |
| --- | --- | --- | --- | --- |
| PICI all Root, Realm, Secure, and Non-secure | v9.2 | 0b00 | 0b00 | 0b0 |
| PICI by PA without Virtual Index, Root only | v9.2 | 0b00 | 0b00 | 0b1 |
| PICI by PA with Virtual Index, Root only | v9.2 | 0b00 | 0b11 | 0b1 |
| PICI all Realm and Non-secure | v9.2 | 0b01 | 0b00 | 0b0 |
| PICI by PA without Virtual Index, Realm only | v9.2 | 0b01 | 0b00 | 0b1 |
| PICI by PA with Virtual Index, Realm only | v9.2 | 0b01 | 0b11 | 0b1 |
| PICI all Secure and Non-secure | v7 | 0b10 | 0b00 | 0b0 |
| PICI by PA without Virtual Index, Secure only | v7 | 0b10 | 0b00 | 0b1 |
| PICI by PA with Virtual Index, Secure only | v7 | 0b10 | 0b11 | 0b1 |
| PICI all, Non-secure only | v7 | 0b11 | 0b00 | 0b0 |
| PICI by PA without Virtual Index, Non-secure only | v7 | 0b11 | 0b00 | 0b1 |
| PICI by PA with Virtual Index, Non-secure only | v7 | 0b11 | 0b11 | 0b1 |
| 注 当 VIV 为 0b11 时，VI[19:12] 被用作 PA 的一部分。 |  |  |  |  |

##### B8.4.5.2 Virtual Instruction Cache Invalidate

本节介绍 Virtual Instruction Cache Invalidate（VICI）操作。

VICI 消息的固定字段取值如表 B8.19 所示。

表 B8.19：Virtual Instruction Cache Invalidate 消息的固定字段取值

| 字段 | 取值 | 状态 |
| --- | --- | --- |
| DVMType | 0b011 | Virtual Instruction Cache Invalidate |
| Part Num | - | 见 Table B8.10 |
| Stage | 0b00 | 保留，置为 0 |
| Leaf | 0b0 | 保留，置为 0 |

表 B8.20 给出了 VICI 支持的操作。

表 B8.20：Virtual Instruction Cache Invalidate 操作

| Operation | Arm | Exception | Security | VMIDV | ASIDV | AddV |
| --- | --- | --- | --- | --- | --- | --- |
| Hypervisor and all Guest OS VICI all, Secure and Non-secure | v7 | 0b00 | 0b00 | 0b0 | 0b0 | 0b0 |
| Hypervisor and all Guest OS VICI all, Non-secure only | v7 | 0b00 | 0b11 | 0b0 | 0b0 | 0b0 |
| All Guest OS VICI by ASID and VA, Secure only | v7 | 0b10 | 0b10 | 0b0 | 0b1 | 0b1 |
| All Guest OS VICI by VMID, Secure only | v8.4 | 0b10 | 0b10 | 0b1 | 0b0 | 0b0 |
| All Guest OS VICI by ASID, VA and VMID, Secure only | v8.4 | 0b10 | 0b10 | 0b1 | 0b1 | 0b1 |
| All Guest OS VICI by VMID, Non-secure only | v7 | 0b10 | 0b11 | 0b1 | 0b0 | 0b0 |
| All Guest OS VICI by ASID, VA and VMID, Non-secure only | v7 | 0b10 | 0b11 | 0b1 | 0b1 | 0b1 |
| Hypervisor VICI by VA, Non-secure only | v7 | 0b11 | 0b11 | 0b0 | 0b0 | 0b1 |
| Hypervisor VICI by ASID and VA, Non-secure only | v8.1 | 0b11 | 0b11 | 0b0 | 0b1 | 0b1 |

#### B8.4.6 Synchronization

本节介绍 DVMSync 操作。

当请求方需要知道此前所有失效操作何时完成时，会使用 Synchronization（Sync）消息。

Sync 消息的固定字段取值如表 B8.21 所示。

表 B8.21：Sync 操作固定取值

| 字段 | 取值 | 状态 |
| --- | --- | --- |
| DVMType | 0b100 | Synchronization 消息。 |
| Part Num | - | 见 Table B8.10。 |
| AddrV | 0b0 | 无地址信息可用。 |
| VMIDV | 0b0 | 无 VMID 信息可用。 |
|  |  | 下页续 |

表 B8.21 – 续上页

| 字段 | 取值 | 状态 |
| --- | --- | --- |
| ASIDV | 0b0 | 无 ASID 信息可用。 |
| Security | 0b00 | 安全信息不适用。 |
| Exception | 0b00 | 异常信息不适用。 |
| VMID VMIDExt | 0xXX | 未指定 VMID。 |
| ASID | 0xXXXX | 未指定 ASID。 |
| Stage | 0b00 | Stage 信息不适用。 |
| Leaf | 0b0 | Leaf 信息不适用。 |

Chapter B9
