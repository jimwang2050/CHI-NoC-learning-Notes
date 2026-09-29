# B9 Error Handling
本章介绍错误处理相关要求。本章包含以下小节：

- B9.1 Packet level
- B9.2 Sub-packet level
- B9.3 Use of interface parity
- B9.4 Hardware and software error categories

### B9.1 Packet level

本节介绍数据包层面的错误。本节包含以下小节：

- B9.1.1 Error types
- B9.1.2 Error response fields

#### B9.1.1 Error types

数据包层面有两种错误报告类型。

数据包层面的错误报告类型包括：

Data Error，DERR：当访问的地址位置正确，但在数据内部检测到错误时使用。通常，在由纠错码（ECC）或奇偶校验检测到数据损坏时使用该类型。Data Error 报告由以下方式支持：

- DAT 数据包中的 RespErr、Poison 和 DataCheck 字段
- RSP 数据包中的 RespErr 字段

当 Home 收到的一个请求需要传播到从属节点，而该请求的处理产生 DERR 时，Home 必须不停止将该请求传播到从属节点。

> **注意**
>
> 从 Home 中逐出的数据出现错误，或作为该请求的结果在侦听响应中收到的数据出现错误，都是该请求产生 DERR 的示例。

Non-data Error，NDERR：当检测到与数据损坏无关的错误时使用。本规范未定义报告该错误类型的所有情形。通常，该错误类型在以下情况下报告：

- 试图访问不存在的位置。
- 非法访问，例如写入只读位置。
- 试图使用不支持的事务类型。

Non-data Error 报告由 RSP 和 DAT 数据包中的 RespErr 字段支持。当 Home 收到的一个请求的处理产生 NDERR 时，允许（但不要求）将该请求传播到从属节点。Home 必须在返回给请求方的响应中传回 NDERR。

#### B9.1.2 错误响应字段

RespErr 字段用于指示错误状况。RespErr 字段同时包含在 Response 数据包和 Data 数据包中。

表 B9.1 展示了 RespErr 字段的编码。有关 Exclusive Okay 响应的更多详细信息，请参见 B6.3.1 Responses to Exclusive requests。

表 B9.1：错误响应字段编码

RespErr[1:0] 名称 描述

0b00 OK Normal Okay。表示以下任一情况：

- 普通访问成功。对于 WriteNoSnpDef，

RespErr 必须与 Resp 结合使用，以确定确切的响应。参见表 B4.29。

- 独占访问失败。

| 0b01 | EXOK | Exclusive Okay。表示独占访问的读部分或写部分已成功。 |
| --- | --- | --- |
| 0b10 | DERR | 数据错误 |
| 0b11 | NDERR | 非数据错误 |

单个事务不允许混合使用 OK 和 EXOK 响应。

带有 Data 响应的事务，必须要么在所有 Data 数据包中都不包含 NDERR，要么在所有 Data 数据包中都包含 NDERR。

允许在单个事务内混合使用 OK 和 DERR 响应。

允许在单个事务内混合使用 EXOK 和 DERR 响应。

允许在单个事务内混合使用 OK 和 NDERR 响应，这种情况只能出现在同时具有 Data 响应和非数据响应的事务中。

不允许混合使用 EXOK 和 NDERR。

#### B9.1.3 错误与事务结构

所有事务都必须以符合协议的方式完成，即使其中包含错误响应。

使用 DMT 的事务的错误处理，与不使用 DMT 的同一请求的错误处理相同。

由于请求或侦听上不存在传播错误的机制，如果在互连处检测到错误，则请求必须不使用 DMT 或 DCT。

如果事务包含数据包，则数据包的源必须发送正确数量的数据包，但数据值不要求有效。

Resp 字段给出与事务关联的缓存状态，并且可能受错误状况影响。有关合法 Resp 字段取值的更多详细信息，请参见 B4.5 Response types。在带有 NDERR 指示的响应中，Resp 中编码的缓存状态可以是任何值，包括保留值。

无论是否存在错误状况，响应中的 Resp 字段对于 Data 消息的每个数据包都必须具有相同的值。

对于 Snoopable 请求，收到带有 NDERR 的响应的请求方必须：

- 对于 Allocating 事务：
- 当起始状态为 I 时，请求方必须不分配接收到的数据。
- 如果请求是从 Non-Invalid 状态发出的，则请求方必须保持缓存副本不变。

在两种情况下，缓存状态都必须不发生改变。

- Allocating 事务包括：
* ReadClean
* ReadNotSharedDirty * ReadShared * ReadUnique * ReadPreferUnique * MakeReadUnique * CleanUnique * MakeUnique
- 对于 deallocating 事务：
- 请求方必须继续正常执行，并以符合协议的方式进行。
- deallocating 事务包括：
* WriteBack * WriteEvictFull * Evict * WriteEvictOrEvict
- 对于不改变分配的 Other 事务：
- 请求方必须不上调缓存状态。
- 允许请求方下调缓存状态。

带有 NDERR 的 SnpResp 消息中的缓存状态必须为 I。完成方必须使该缓存行的本地缓存副本无效。此外，当对 Forwarding snoop 的响应导致 NDERR 时，Snoopee 必须不向请求方转发数据。因此，如果 CompData 消息已经发送给请求方，则发往归属节点的侦听响应必须不包含 NDERR。

#### B9.1.4 按事务类型使用错误响应

本节定义了每种事务类型对错误字段的允许使用方式（但并非必需）。

以下各表列出了与下列事务类型关联的 Data 和 Response 数据包：

- B9.1.4.1 读事务
- B9.1.4.2 无数据事务
- B9.1.4.3 写事务
- B9.1.4.4 Atomic 事务
- B9.1.4.5 其他事务
- B9.1.4.6 缓存暂存事务
- B9.1.4.7 侦听事务

各表使用以下键：

OK RespErr 字段必须包含取值为 0b00 的 OK RespErr。

Y 允许使用该 RespErr 值。

N 不允许使用该 RespErr 值。

- 该事务类型不使用 Data 或 Response 数据包。

##### B9.1.4.1 读事务

读事务可以包含多个 CompData 数据包。

已知损坏的读数据必须具有适当的错误指示，该错误可以是 Poison、DERR 或 NDERR。

当 RespSepData 包含 NDERR 时，所有对应的 DataSepResp 数据包都必须标记为 NDERR。

在对 Read 请求的 Data 响应中，NDERR 响应只允许出现在零个或全部数据响应数据包中。

表 B9.2 列出了读事务中与 Data 和 Response 数据包关联的合法 RespErr 字段值。

表 B9.2：读事务中与 Data 和 Response 数据包关联的合法 RespErr 字段值

| Read transaction | Associated Data and Response packets Read Receipt CompData OK EXOK DERR | NDERR | CompAck |
| --- | --- | --- | --- |
| ReadNoSnp | OK Y Y Y | Y | OK |
| ReadNoSnpSep | OK - - - | - | - |
| ReadOnce ReadOnceCleanInvalid ReadOnceMakeInvalid | OK Y N Y | Y | OK |
| ReadClean ReadNotSharedDirty ReadShared | - Y Y Y | Y | OK |
| ReadUnique ReadPreferUniquea MakeReadUniqueb | - Y N Y | Y | OK |

a 即使请求中的 Excl 位被置为 1，也不允许 EXOK 响应。返回的 data 响应中的 OK 响应不得被视为 ReadPreferUnique 的失败。独占序列的失败仅根据相应的 MakeReadUnique Excl 或 CleanUnique Excl 事务完成来确定。

b 仅当数据返回给请求方时适用。当数据未返回给请求方时，允许的错误参见表 B9.5。

表 B9.3 列出了读事务中与 Data-only 和 Non-data 数据包关联的合法 RespErr 字段值。

表 B9.3：读事务中与 Data-only 和 Non-data 数据包关联的合法 RespErr 字段值

| Read transaction | Associated Data-only and Non-data packets DataSepResp RespSepData OK EXOK DERR NDERR OK EXOK | DERR NDERR |
| --- | --- | --- |
| ReadNoSnp | Y N Y Y Y N | N Y |
| ReadNoSnpSep | Y N Y Y - - | - - |
| ReadOnce ReadOnceCleanInvalid ReadOnceMakeInvalid | Y N Y Y Y N | N Y |
|  |  | 下页续 |

表 B9.3 – 续上页

| Read transaction | Associated Data-only and Non-data packets DataSepResp RespSepData OK EXOK DERR NDERR OK EXOK | DERR | NDERR |
| --- | --- | --- | --- |
| ReadClean ReadNotSharedDirty ReadShared | Y N Y Y Y N | N | Y |
| ReadUnique ReadPreferUniquea MakeReadUniqueb | Y N Y Y Y N | N | Y |

a 返回的 data 响应中的 OK 响应不得被视为 ReadPreferUnique 的失败。独占序列的失败仅根据相应的 MakeReadUnique Excl 或 CleanUnique Excl 事务完成来确定。

b 仅当数据返回给请求方时适用。当数据未返回给请求方时，允许的错误参见表 B9.4。

表 B9.4 列出了 RespSepData 与 DataSepResp 相对其消息来源的合法组合。

表 B9.4：根据消息来源的 RespSepData 与 DataSepResp 合法组合

| RespSepData | DataSepResp | 当 DataSepResp 来自归属节点 / 来自从属节点时的合法组合 |
| --- | --- | --- |
| OK | OK | Y Y |
|  | NDERR | Y Y |
|  | DERR | Y Y |
| NDERR | NDERR | Y - |

##### B9.1.4.2 无数据事务

当另一个组件对该事务的处理遇到数据损坏错误时，可以为无数据事务报告 DERR。即使不发生数据传输，也可以将该 DERR 指示回发起方组件。

表 B9.5 显示了无数据事务数据包合法的 RespErr 字段取值。

表 B9.5：无数据事务合法的 RespErr 字段取值

无数据事务 关联响应数据包 CompAck

Comp Persist CompPersist NDERR NDERR NDERR EXOK EXOK EXOK DERR DERR DERR OK OK OK

CleanUnique Y Y Y Y - - - - - - - - OK

MakeReadUniquea Y N Y Y - - - - - - - - OK

MakeUnique Y N Y Y - - - - - - - - OK

CleanShared Y N Y Y - - - - - - - - -

CleanSharedPersist Y N Y Y - - - - - - - - -

CleanSharedPersistSep Y N Y Y Y N Y Y Y N Y Y -

CleanInvalid Y N Y Y - - - - - - - - -

CleanInvalidPoPA

CleanInvalidStorage

MakeInvalid

Evict Y N N Y - - - - - - - - -

a 仅当数据不返回给请求方时适用。有关数据返回给请求方时允许的错误，请参见表 B9.2 和表 B9.3。

表 B9.6 显示了对 StashOnce 事务的响应中允许的 RespErr 取值。

表 B9.6：无数据 Stash 事务合法的 RespErr 字段取值

无数据 Stash 事务 关联响应数据包

Comp StashDone CompStashDone NDERR NDERR NDERR EXOK EXOK EXOK DERR DERR DERR OK OK OK

StashOnceUnique Y N Y Y - - - - - - - -

StashOnceShared

StashOnceSepUnique Y N Y Y Y N Y Y Y N Y Y

StashOnceSepShared

##### B9.1.4.3 写事务

写事务可以包含 NDERR 或 DERR。错误可以在两个方向上发出信号：从请求方到完成方，以及从完成方返回请求方。

已知损坏的写数据必须带有适当的错误指示，该错误可以是 Poison 或 DERR。

对于写事务，完成方可以使用合并的 CompDBIDResp 或使用 Comp 响应，将错误发回给请求方。完成方甚至可以在观察到该事务的 WriteData 之前就发出错误信号，这是允许的，但不是必需的。当对事务的处理（例如缓存查找）遇到数据损坏错误时，就可能发生这种情况。

表 B9.7 显示了写事务响应数据包合法的 RespErr 字段取值。

表 B9.7：写事务合法的 RespErr 字段取值

写事务 关联响应数据包 CompAck

DBIDResp* Comp CompDBIDResp NDERR NDERR EXOK EXOK DERR DERR OK OK

| WriteNoSnp | OK | Y | Y | Y | Y | Y | Y | Y | Y | OK |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| WriteNoSnpDef | OK | Ya | N | Y | Y | Ya | N | Y | Y | - |
| WriteUnique | OK | Y | N | Y | Y | Y | N | Y | Y | OK |
| WriteNoSnpZero WriteUniqueZero | OK | Y | N | Y | Y | Y | N | Y | Y | - |
| WriteBack WriteCleanFull WriteEvictFull | - | Y | N | N | Y | Y | N | Y | Y | OK |
| WriteEvictOrEvict | - | Y | N | N | Y | Y | N | Y | Y | OK |

a 在 WriteNoSnpDef 事务中，与 Resp[2:0] 组合使用，以传达 Successful、Unsupported 和 Defer 响应。参见表 B4.29。

检测到待发送写数据中存在错误的请求方，可以在写数据包中附带错误指示。这表明该数据值已知是损坏的。

表 B9.8 显示了写事务数据包合法的 RespErr 字段取值。

表 B9.8：写事务数据包中合法的 RespErr 字段取值

写事务 关联数据包

WriteData WriteDataCancel NonCopyBackWriteDataCompAck NDERR NDERR NDERR EXOK EXOK EXOK DERR DERR DERR OK OK OK

| WriteNoSnp | Y | N | Y | N | Y | N | Y | N | Y | N | Y N |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| WriteNoSnpDef | Y | N | Y | N | Y | N | Y | N | Y | N | Y N |
| WriteUnique | Y | N | Y | N | Y | N | Y | N | Y | N | Y N |
|  |  |  |  |  |  |  |  |  |  |  | 下页续 |

表 B9.8 – 续上页

写事务 关联数据包

WriteData WriteDataCancel NonCopyBackWriteDataCompAck NDERR NDERR NDERR EXOK EXOK EXOK DERR DERR DERR OK OK OK

WriteBack Y N Y N - - - - - - - - WriteCleanFull WriteEvictFull WriteEvictOrEvict

表 B9.9 显示了当请求方接口上支持 MTE 时，写事务 TagMatch 数据包合法的 RespErr 字段取值。

表 B9.9：写事务 TagMatch 数据包中合法的 RespErr 字段取值

| 写事务 | 关联响应数据包 TagMatch OK EXOK DERR | NDERR |
| --- | --- | --- |
| WriteNoSnp WriteUnique | Y N Y | Y |

WriteBack - - - -

WriteCleanFull

WriteEvictFull

WriteEvictOrEvict

WriteNoSnpZero

WriteUniqueZero

WriteNoSnpDef

关于 Combined Write 事务响应中允许的 RespErr 字段取值：

- 写请求的对应响应参见表 B9.7。
- CMO 请求的对应响应参见表 B9.5。

CompCMO 中允许的错误响应与表 B9.5 中 Comp 的相同。

##### B9.1.4.4 Atomic 事务

完成方在收到与某个事务关联的全部写数据并执行完所需操作之前就给出 Comp 响应，这是允许的，但不是必需的。这种行为与希望对写数据发出 Data Error 信号的组件不兼容，此类组件必须使用延迟形式的 Comp 或 CompData 响应。

在 Atomic 事务中，已知损坏的读数据或写数据必须具有适当的错误指示。读数据错误可以是 Poison、DERR 或 NDERR。写数据错误可以是 Poison 或 DERR。

在事务内，可以在以下位置发出 DERR 或 NDERR 信号：

- 在 CompDBIDResp 响应中。
- 对于 AtomicStore 事务，在 Comp 响应中。
- 对于 AtomicLoad、AtomicSwap 和 AtomicCompare 事务，在 CompData 响应中。

对于无法完成的 Atomic 事务，必须使用 NDERR。事务结构，包括所有写数据传输、读数据传输和其他响应，仍必须发生。在此 NDERR 情形中，不需要 Snoop 事务，包括因置位 SnoopMe 而产生的返回到原始请求方的那些 Snoop 事务。

无需指定与原子操作执行（例如溢出）相关的错误。所有原子操作都针对所有输入组合做了完整规定。

一个事务既包含出站数据也包含入站数据，但只有一个 Error 字段。对于 Atomic 事务，允许 Error 字段指示写数据或读数据上的错误。事务内不支持用于区分错误的各种潜在不同原因的机制。故障日志或类似结构可能能够提供此类信息，但这不是 CHI 协议的要求。

Atomic 事务中允许的 RespErr 取值是读事务和写事务中允许取值的合并。

DERR 可以在不同数据包之间有所不同。

表 B9.10 显示了 Atomic 事务响应数据包合法的 RespErr 字段取值。

表 B9.10：Atomic 事务响应数据包中合法的 RespErr 字段取值

Atomic 事务 关联响应数据包

DBIDResp* Comp CompDBIDResp NDERR NDERR EXOK EXOK DERR DERR OK OK

| AtomicStore | OK | Y | N | Y | Y | Y | N | Y | Y |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| AtomicLoad AtomicSwap AtomicCompare | OK | - | - | - | - | - | - | - | - |

表 B9.11 显示了 Atomic 事务数据包合法的 RespErr 字段取值。

表 B9.11：Atomic 事务数据包中合法的 RespErr 字段取值

Atomic 事务 关联响应数据包

WriteData CompData NDERR NDERR EXOK EXOK DERR DERR OK OK

| AtomicStore | Y | N | Y | N | - | - | - | - |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| AtomicLoad AtomicSwap AtomicCompare | Y | N | Y | N | Y | N | Y | Y |

表 B9.12 显示了在支持 MTE 时，Atomic 事务 TagMatch 数据包合法的 RespErr 字段取值。

表 B9.12：Atomic 事务 TagMatch 数据包中合法的 RespErr 字段取值

Atomic 事务 关联响应数据包

TagMatch NDERR EXOK DERR OK

AtomicStore Y N Y Y

AtomicLoad

AtomicSwap

AtomicCompare

##### B9.1.4.5 其他事务

本节描述 DVMOp 和 PrefetchTgt 事务的错误处理要求。

###### B9.1.4.5.1 DVMOp

DVMOp 事务可以在 Comp 响应中包含 NDERR。互连可以合并某个 DVMOp 的所有侦听响应中的错误响应，并在发往请求方的最终 Comp 消息中包含单个错误响应。DBIDResp 数据包必须只使用 OK 响应。即使 WriteData 响应的发送方可能不使用 DERR，该数据包如果在传输过程中遇到错误，也可以被标记为 DERR。参见 B9.2.3 Interoperability of Poison and DataCheck。

表 B9.13 显示了 DVM 事务响应数据包合法的 RespErr 字段取值。

表 B9.13：DVM 事务响应数据包中合法的 RespErr 字段取值

DVM 事务 关联响应数据包

DBIDResp Comp CompDBIDResp NDERR NDERR EXOK EXOK DERR DERR OK OK

DVMOp OK Y N Y Y Y N Y Y

表 B9.14 显示了 DVM 事务数据包合法的 RespErr 字段取值。

表 B9.14：DVM 事务数据包中合法的 RespErr 字段取值

关联数据包

DVM 事务 NonCopyBackWriteData NDERR EXOK DERR OK

| DVMOp | Y | N | Y | N |
| --- | --- | --- | --- | --- |
| B9.1.4.5.2 PrefetchTgt 发往不支持该事务的地址的 PrefetchTgt 事务请求必须被丢弃。 |  |  |  |  |
| 注：允许（但不要求）组件记录并报告此类错误。 |  |  |  |  |

##### B9.1.4.6 缓存暂存事务

如果指定的暂存目标不支持接收暂存侦听（Stash snoop），则归属节点必须忽略暂存提示，并在不进行暂存的情况下完成该事务。此类暂存目标的示例包括 RN-I、RN-D、旧版 RN-F 或非请求节点。在这些情况下，归属节点不得向请求方发送错误信号。此类被错误指定的暂存目标可归因于软件错误。

如果归属节点不支持 Stash 请求，则必须以符合协议的方式完成该事务，且不得发送错误信号。

##### B9.1.4.7 侦听事务

包含数据的侦听事务响应可以指示 DERR。包含数据的侦听事务响应可以在同一事务内针对不同数据包混合使用 OK 和 DERR 响应。

对于已知损坏的侦听请求，其响应数据必须带有适当的错误指示，该错误可以是 Poison 或 DERR。

不包含数据的侦听事务响应可以指示 NDERR。

以 NDERR 响应的 Snoopee：

- 不得在响应中携带数据。
- 必须使任何本地缓存的副本失效。
- 必须将响应中的缓存状态设置为 Invalid。
- 不得针对 Forwarding snoop 向请求方转发 CompData 响应。因此，如果 CompData 消息已发送给请求方，则对归属节点的侦听响应不得包含 NDERR。

表 B9.15 显示了侦听请求响应数据包合法的 RespErr 字段取值。

表 B9.15：侦听事务的响应数据包中合法的 RespErr 字段取值

侦听事务关联的数据和响应数据包

SnpResp SnpRespData NDERR NDERR EXOK EXOK DERR DERR OK OK

SnpOnce Y N N Y Y N Y N

SnpClean

SnpNotSharedDirty

SnpShared

SnpUnique

SnpPreferUnique

SnpUniqueStash

SnpCleanShared

SnpCleanInvalid

SnpStashUnique Y N N Y - - - -

SnpStashShared

SnpMakeInvalid

SnpMakeInvalidStash

SnpQuery

SnpDVMOp

对于 Clean 缓存行上的 DERR，建议（但非必须）将其丢弃，既不将该错误传播到内存，也不将其包含在对请求方的响应中。

Dirty 缓存行上的 DERR 必须传播到内存，并包含在对请求方的响应中。

对 DataPull 请求的响应中的 DERR，预期不会传递到对 Stash 请求的 Comp 响应中。

对于 Forwarding snoop 事务，当同时向请求方转发数据并向归属节点返回数据时，如果其中一个响应未遇到该错误，则允许（但非必须）仅由一个响应包含 DERR 指示。

表 B9.16 显示了 Forwarding Snoop 响应数据包合法的 RespErr 字段取值。

表 B9.16：Forwarding Snoop 事务的响应数据包中合法的 RespErr 字段取值

侦听事务关联的响应数据包

SnpResp SnpRespFwded NDERR NDERR EXOK EXOK DERR DERR OK OK

SnpOnceFwd Y N N Y Y N Y N SnpCleanFwd SnpNotSharedDirtyFwd SnpSharedFwd SnpUniqueFwd SnpPreferUniqueFwd

表 B9.17 显示了 Forwarding Snoop 数据响应数据包合法的 RespErr 字段取值。

表 B9.17：Forwarding Snoop 事务的数据包中合法的 RespErr 字段取值

侦听事务关联的数据包

SnpRespData CompData

SnpRespDataFwded NDERR NDERR EXOK EXOK DERR DERR OK OK

SnpOnceFwd Y N Y N Y N Y N

SnpCleanFwd

SnpNotSharedDirtyFwd

SnpSharedFwd

SnpUniqueFwd

SnpPreferUniqueFwd

### B9.2 子数据包级别

子数据包级别有两种类型的错误报告：

- B9.2.1 Poison
- B9.2.2 Data Check

#### B9.2.1 Poison

Poison 位用于指示一组数据字节此前已被损坏。在 DAT 数据包中随数据一起传递 Poison 位，可使该数据未来的任何使用者都能被告知该数据已损坏。

当支持 Poison 时：

- DAT 数据包中每 64 位数据包含一个 Poison 位。
- 被标记为 poisoned 的数据：
- 不得被任何请求方使用。
- 若被标记为 poisoned，则允许（但非必需）将其存储在缓存和内存中。
- Poison 值一旦置位，就必须随数据一起传播。
- 当检测到 Poison 错误时，允许（但非必需）对数据过度标记 Poison（over poison）。
- 不支持对 MTE 标签使用 Poison。

如果 64 位块中存在任何有效字节，则 Poison 必须是准确的。这就是 Poison 粒度。当 64 位块中的全部 8 个字节均无效时，Poison 位可以取任意值。

Data_Poison 属性用于指示某个组件是否支持 Poison。

> **注意**
>
> 尽管不支持对标签使用 Poison，但实现方式可以选择执行以下操作之一：

- 与数据关联的 Poison 会导致该标签被标记为 Poison。取决于与标签关联的 poison 的粒度，可用于清除与数据关联的 poison 的相同技术可能无法清除标签上的 poison。
- 与数据关联的 Poison 不会导致该标签被标记为 Poison。这意味着被损坏的标签随后可能被用于 MTE Match 操作，从而可能错误地失败。这种情况的发生率应显著低于数据损坏的发生率。
- 可以采用多种方法的组合，具体取决于所使用的缓存或存储结构。

也可以采用其他实现方式。

#### B9.2.2 Data Check

DataCheck 字段用于检测 DAT 数据包中的数据错误（Data Error）。

当支持 Data Check 时：

- DAT 数据包中每 64 位数据携带 8 个 DataCheck 位。
- Data Check 位是一个奇偶校验位，用于生成 Odd Byte 奇偶校验。

> **注意**
>
> 接口奇偶校验（Interface parity）可选地扩展 DataCheck 字段在 DAT 通道上提供的错误检测。

Data_Check 和 Check_Type 属性用于指示 DAT 数据包中是否支持 DataCheck。

#### B9.2.3 Poison 与 DataCheck 的互操作性

如果 Data 数据包的接收方不支持 Poison 和 DataCheck 特性，则互连必须按需枚举 Poison 和 DataCheck 错误响应，并将其转换为 DAT 数据包中的 DERR。

如果某个接口上对 Poison 和 DataCheck 特性的支持情况不一致，则适用以下规则：

- 如果接口上不支持 Poison，则必须将 Poison 映射为 DataCheck 或 DERR。在此类接口上，如果支持 DataCheck，则期望（但不要求）将 Poison 映射为 DataCheck 而非 DERR。从 Poison 转换为 DataCheck 时，当某个 8 字节块被标记为 Poisoned 时，必须对该块对应的全部 8 个 DataCheck 位进行处理，以生成奇偶校验错误。
- 如果接口上不支持 DataCheck，则必须将 DataCheck 映射为 Poison 或 DERR。在此类接口上，如果支持 Poison，则期望（但不要求）将 DataCheck 映射为 Poison 而非 DERR。从 DataCheck 转换为 Poison 时，如果给定的 8 字节块中有一个或多个 DataCheck 位产生奇偶校验错误，则必须置位该块对应的 Poison 位。

> **注意**
>
> Poison 与 DERR 处理方式的区别在于：接收到的数据包中的 Poison 错误通常由接收方延迟处理，而 DERR 错误通常不由接收方延迟处理。

对于检测到 Poison 错误的数据包发送方，只需在 Poison 位中指示该错误即可。并不要求发送方将 RespErr 字段值设置为 DERR。

对于检测到 DataCheck 错误的数据包发送方，只需在 DataCheck 字段中指示该错误即可，不要求将 RespErr 字段值设置为 DERR。

由于 Poison 和 DataCheck 字段是独立设置的，因此一种错误类型不要求设置另一种。

在 RespErr 字段值设置为 DERR 或 NDERR 的 Data 数据包中，Poison 和 DataCheck 字段的值不适用，可以取任意值。

### B9.3 接口奇偶校验的使用

对于安全关键应用，必须能够检测并尽可能纠正在 SoC 内部各条独立连线上出现的瞬态错误和功能性错误。

系统组件中的错误可能传播，并在相连的组件中引发多个错误。错误检测与纠正（EDC）需要端到端地工作，覆盖从源到目的地的所有逻辑和连线。

实现端到端保护的一种方式是：在组件内部采用定制的 EDC 方案，同时在组件之间实现简单的错误检测方案。这些组件之间不存在逻辑，连接相对较短。本节介绍一种奇偶校验方案，用于检测组件间接口上的单比特错误。如果多比特错误出现在不同的奇偶校验信号组中，则可以检测出来。

图 B9.1 示出了可以使用奇偶校验的位置。

![图 p383](../images/fig_p0383_1.png)

图 B9.1：AMBA 中奇偶校验的使用

AMBA 奇偶校验可选地扩展 DAT 通道上由 DataCheck 字段提供的错误检测，使其覆盖所有通道上的完整 flit 和控制信号。接口上采用的保护方案由属性 Check_Type 定义。参见 B16.1 接口属性与参数。

#### B9.3.1 字节奇偶校验信号

以下属性是为字节奇偶校验接口保护而添加的所有校验信号所共有的：

- 使用奇校验。奇校验意味着向接口上的信号组添加校验信号，并

以如下方式驱动：使该组中置为 1 的比特数始终为奇数。

- 覆盖数据和载荷的奇偶校验信号定义为每组不超过 8 比特。这一

限制假定生成每个奇偶校验比特的时序预算中最多有 3 级逻辑可用。

- 覆盖关键控制信号的奇偶校验信号，其可用时序预算可能较小，因此

定义为使用单个奇校验比特。

- 校验信号的最低有效校验比特覆盖载荷的最低有效字节。
- 如果载荷中的比特未填满最高有效字节，则校验信号的最高有效比特覆盖

少于 8 比特。

- 在 Check Enable 项为 True 的每个周期中，校验信号都必须被正确驱动，参见表 B9.18。
- 奇偶校验信号必须根据相关联载荷中的所有比特进行相应驱动，无论

这些比特是否适用。

#### B9.3.2 错误检测行为

本规范不对检测到奇偶校验错误时的组件或系统行为作规定。取决于系统和受影响的信号，翻转的比特可能产生多种影响。翻转的比特可能：

- 无危害
- 导致性能问题
- 导致数据损坏
- 导致安全违规
- 导致死锁

此外，一个信号上的错误也可能导致其他信号上的奇偶校验失败。

当检测到错误时，接收方可选择：

- 终止或传播该事务。当事务被终止时，允许但不要求符合协议。
- 纠正奇偶校验信号，或传播该错误。
- 更新其内存或保持不变。允许但不要求将该位置标记为毒化。
- 通过其他方式发出错误响应信号，例如使用中断。

#### B9.3.3 接口奇偶校验信号

表 B9.18 示出了各通道上的奇偶校验信号及其属性。

表 B9.18：接口奇偶校验信号

| 通道 | 校验信号 | 覆盖的信号 | 校验信号宽度 | 校验粒度 | 校验使能 |
| --- | --- | --- | --- | --- | --- |
| Common | SYSCOREQCHK | SYSCOREQ | 1 | 1 | RESETn == 1 |
|  | SYSCOACKCHK | SYSCOACK | 1 | 1 | RESETn == 1 |
|  | TXSACTIVECHK | TXSACTIVE | 1 | 1 | RESETn == 1 |
|  | RXSACTIVECHK | RXSACTIVE | 1 | 1 | RESETn == 1 |
|  | TXLINKACTIVEREQCHK | TXLINKACTIVEREQ | 1 | 1 | RESETn == 1 |
|  | TXLINKACTIVEACKCHK | TXLINKACTIVEACK | 1 | 1 | RESETn == 1 |
|  | RXLINKACTIVEREQCHK | RXLINKACTIVEREQ | 1 | 1 | RESETn == 1 |
|  | RXLINKACTIVEACKCHK | RXLINKACTIVEACK | 1 | 1 | RESETn == 1 |
| REQ | REQFLITPENDCHK | REQFLITPEND | 1 | 1 | RESETn == 1 |
|  | REQFLITVCHK | REQFLITV | 1 | 1 | RESETn == 1 |
|  | REQFLITCHK | REQFLIT | ceil(R/8)a | 1 至 8 | REQFLITV == 1 |
|  | REQRPCHKb | REQFLITRP REQSHAREDCRD | 1 | 1 至 4 | REQFLITV == 1 |
|  | REQLCRDVCHK | REQLCRDV | Num_RP_REQ | 1 | RESETn == 1 |
|  | REQLCRDSHVCHKc | REQLCRDSHV | 1 | 1 | RESETn == 1 |
| RSP | RSPFLITPENDCHK | RSPFLITPEND | 1 | 1 | RESETn == 1 |
|  | RSPFLITVCHK | RSPFLITV | 1 | 1 | RESETn == 1 |
|  | RSPFLITCHK | RSPFLIT | ceil(T/8)d | 1 至 8 | RSPFLITV == 1 |
|  | RSPLCRDVCHK | RSPLCRDV | 1 | 1 | RESETn == 1 |

下页续

表 B9.18 – 续上页

| 通道 | 校验信号 | 覆盖的信号 | 校验信号宽度 | 校验粒度 | 校验使能 |
| --- | --- | --- | --- | --- | --- |
| SNP | SNPFLITPENDCHK | SNPFLITPEND | 1 | 1 | RESETn == 1 |
|  | SNPFLITVCHK | SNPFLITV | 1 | 1 | RESETn == 1 |
|  | SNPFLITCHK | SNPFLIT | ceil(S/8)e | 1 至 8 | SNPFLITV == 1 |
|  | SNPRPCHKf | SNPFLITRP SNPSHAREDCRD | 1 | 1 至 4 | SNPFLITV == 1 |
|  | SNPLCRDVCHK | SNPLCRDV | Num_RP_SNP | 1 | RESETn == 1 |
|  | SNPLCRDSHVCHKg | SNPLCRDSHV | 1 | 1 | RESETn == 1 |
| DAT | DATFLITPENDCHK | DATFLITPEND | 1 | 1 | RESETn == 1 |
|  | DATFLITVCHK | DATFLITV | 1 | 1 | RESETn == 1 |
|  | DATFLITCHK | DATFLIT | ceil(D/8)h | 1 至 8 | DATFLITV == 1 |
|  | DATLCRDVCHK | DATLCRDV | 1 | 1 | RESETn == 1 |

a R = 请求 flit 宽度。参见 B13.9.1 请求 flit。b 如果 REQFLITRP 和 REQSHAREDCRD 都不存在，则从接口中省略 REQRPCHK。c 如果 REQLCRDSHV 不存在，则从接口中省略 REQLCRDSHVCHK。d T = 响应 flit 宽度。参见 B13.9.2 响应 flit。e S = 侦听 flit 宽度。参见 B13.9.3 侦听 flit。f 如果 SNPFLITRP 和 SNPSHAREDCRD 都不存在，则从接口中省略 SNPRPCHK。g 如果 SNPLCRDSHV 不存在，则从接口中省略 SNPLCRDSHVCHK。h D = 数据 flit 宽度。参见 B13.9.4 数据 flit。

### B9.4 硬件与软件错误类别

本规范定义了两类错误：

- 基于软件的
- 基于硬件的

#### B9.4.1 基于软件的错误

当对同一位置进行多次访问，且使用了不正确或不匹配的 Snoopable 或 Memory 属性时，就会发生基于软件的错误。参见 B2.8.7 Mismatched Memory attributes。

基于软件的错误可能导致一致性丢失以及数据值损坏。对于基于软件的错误，要求系统不得死锁，并且事务始终能及时地在系统中推进。

对于在一个 4KB 内存区域内的访问，基于软件的错误不得导致其他 4KB 内存区域内的数据损坏。

对于保存在 Normal 内存中的位置，可以使用适当的存储操作和软件缓存维护，将内存位置恢复到已定义状态。

在访问外设设备时，无法保证外设的正确运行。唯一的要求是外设继续以符合协议的方式响应事务。将被错误访问的外设设备恢复到已知可用状态所需的事件序列，属于 IMPLEMENTATION DEFINED。

#### B9.4.2 基于硬件的错误

基于硬件的错误定义为任何不属于基于软件错误的协议错误。

> **警告**
>
> 如果发生基于硬件的错误，不能保证能够从该错误中恢复。系统可能崩溃、锁死，或者遭受其他不可恢复的故障。

Chapter B10
