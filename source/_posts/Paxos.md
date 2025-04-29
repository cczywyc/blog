---
title: "Paxos"
description: "分布式共识算法-paxos"
Swiper_index: 5
date: 2024-10-02
tags:
  - MIT-6.824
  - 分布式系统
categories:
  - 分布式系统文章合集
cover: https://img.cczywyc.com/post-cover/paxos_cover.jpg
---

# Paxos 背景和介绍

进入正文之前先来简单说一下几个概念和 Paxos 的由来。

### 分布式一致性问题

在分布式计算领域，一个核心且基础的挑战是如何让一组独立的计算机（或称为节点、进程）在可能发生故障（如节点崩溃、网络消息丢失或延迟）的环境下，就某个值或状态达成一致 。这个问题被称为共识（Consensus）。达成共识的能力对于构建可靠的分布式系统至关重要，这些系统需要在多个副本之间维护数据一致性，例如分布式数据库、分布式锁服务、以及实现状态机复制（State Machine Replication）等场景 。状态机复制是一种强大的技术，它允许将任何单机上的确定性算法转化为一个容错的、分布式的实现，确保即使部分副本发生故障，整个系统对外也能表现得像一个永不宕机的单体服务 。

一个健壮的共识算法通常需要满足几个关键属性：

1. 有效性（Validity）：最终被选定的值必须是某个进程实际提议过的值。
2. 一致性（Agreement/Consistency）：所有做出决定的正常进程必须对同一个值达成一致 。这是共识最核心的保证。
3. 完整性（Integrity）：每个进程最多只能决定一个值。
4. 可终止性（Termination/Liveness）：所有正常的进程最终都能做出决定。
5. 容错性（Fault Tolerance）：算法必须能够在一定数量的进程失败或网络异常的情况下继续运行并达成共识。

分布式一致性问题可以简单描述为：如何确保分布式系统中的多个节点对某个值达成一致，即使在部分节点故障或网络不可靠的情况下。

### Paxos 的诞生

Paxos 算法由 Leslie Lamport 于 1989 年首次提出，并在 1998 年发表论文《The Part-Time Parliament》。由于这篇论文使用了一个古希腊 Paxos 岛上的议会作为比喻，晦涩难懂，直到 2001 年 Lamport 又发表了《Paxos Made Simple》，才使这个算法被更广泛地理解和应用。

Paxos 算法解决的核心问题是：在一个可能发生节点宕机或网络异常的分布式系统中，如何就某个值达成一致，使得系统满足一致性（所有节点最终达成相同的决定）和安全性（一旦某个值被选定，就不会被改变）。

Paxos 算法之所以如此重要，是因为它为分布式系统中的一个核心问题提供了第一个经过严格证明的、可在异步网络模型下容忍非拜占庭故障（non-Byzantine failures，即节点只会崩溃或丢失消息，不会恶意发送错误信息）的解决方案。

* 保证一致性（Safety）：Paxos 最核心的贡献在于它提供了一种机制，确保即使在节点崩溃、消息丢失、延迟或乱序的情况下，系统中的所有节点最终也能就某个值达成唯一的、一致的决定 。它保证了“只选出最多一个值”（Agreement）和“被选出的值必须是提议过的值”（Validity）这两个关键的安全属性。
* 容错性（Fault Tolerance）：Paxos 算法通过其核心的“法定人数”（Quorum）机制，能够容忍少数节点（通常是少于半数）的失效。只要大多数节点（一个 Quorum）能够正常工作并相互通信，系统就能继续就新的值达成共识 。
* 分布式系统的基石：Paxos 不仅仅是一个理论算法，它为构建实际的、高可用的分布式系统奠定了基础。许多关键的工业级系统，如 Google 的 Chubby 锁服务、微软的 Autopilot 集群管理系统，以及亚马逊、IBM 等公司的许多内部系统，都使用了 Paxos 或其变种来实现核心的协调和一致性保证 。同时，Paxos 的思想也深刻影响了后续共识算法（如 Raft ）的设计。

Paxos 的出现，标志着分布式系统理论的一个重大突破，它成功地将严谨的理论证明与解决实际工程挑战（如构建容错数据库、文件系统等）的需求结合起来。它证明了在充满不确定性的分布式环境中，实现可靠的一致性是可能的，为现代大规模分布式计算系统的发展铺平了道路。

### 应用场景

Paxos 算法在现代分布式系统中有广泛应用：

1. 分布式数据库 ：如 Google 的 Spanner、Cockroach DB 等
2. 分布式锁服务 ：如 Google 的 Chubby
3. 配置管理 ：如 ZooKeeper（虽然 ZooKeeper 使用的是 Paxos 的变种 ZAB 协议）
4. 共识系统 ：如区块链中的共识机制

# Paxos 算法原理和详细实现过程

### Paxos 中的角色
Paxos 算法中定义了三种角色：

1. Proposer（提议者） ：提出提案，提案包括提案编号和提案的值。
2. Acceptor（接受者） ：接收提案并决定是否接受。
3. Learner（学习者） ：不参与决议，只学习被批准的提案。

在实际实现中，一个节点可以同时扮演多个角色。这种角色的分离，特别是 Acceptor 作为核心状态维护者的设计，为 Paxos 提供了灵活性。例如，Proposer 的选举（通常为了提高效率而选出一个 Leader）可以独立于核心共识逻辑进行，而 Learner 的实现方式也可以多样化。然而，Acceptor 的正确性和状态持久性是保证整个协议安全性的基石。任何对 Acceptor 状态的访问和修改都需要严格的并发控制和持久化保证。

### 算法流程

在深入之前，有必要再强调一下提案编号（Proposal Number）的关键作用：

* 唯一性与全序性：每个 Proposer 发起的每一次提议尝试都必须使用一个比它之前使用过的任何编号都大的、全局唯一的编号 。这通常通过组合一个单调递增的计数器（或高精度时间戳）和 Proposer 自身的唯一标识符来实现。
* 建立优先级：提议编号的大小决定了提议的优先级。Acceptor 只会响应（Promise）或接受（Accept）编号大于等于其已承诺的最高编号的提议。这确保了协议的进展总是朝着更新（编号更大）的提议方向进行。
* 保证一致性：提议编号是 Paxos 安全性的核心。通过要求 Proposer 在第二阶段选择的值必须基于第一阶段从多数 Acceptor 处了解到的、具有最高编号的已接受值，Paxos 巧妙地利用提议编号的顺序来确保一旦某个值被选定，后续更高编号的提议也只能选定相同的值。



Paxos 算法分为两个阶段：

#### 阶段一：准备（Prepare）与承诺（Promise）

这个阶段的目标是让一个 Proposer 获得“领导权”（即一个被多数 Acceptor 承诺接受的提议编号），并发现是否存在任何可能已经被选定的值。

* （1a）Prepare 消息：
    * Proposer 想要提议一个值（比如 v），它首选选择一个新的、唯一的、比它之前用过的都大的提议编号 `n`
    * Proposer 向一个 **法定人数（Quorum）** 的 Acceptor 发送 `Prepare(n)` 消息。
* （1b）Promise 响应：
    * 当一个 Acceptor 收到 `Prepare(n)` 消息时，它会比较 `n` 和它已经记录的已响应过的最高 Prepare 请求编号 `max_prepare`。
    * 情况 1：`n` 大于该 Acceptor 已经响应过的所有 Prepare 请求的编号，则该 Acceptor 必须：
        1. 将 max_prepare 更新为 `n`，并持久化这个状态。
        2. 向 Proposer 返回一个 `Promise(n, prev_accepted_n, prev_accepted_v)` 消息。
            - `prev_accepted_n` 是这个 Acceptor 之前接受过的提议中编号最高的那个提议编号（如果没有接受过任何提议，则为空或者置为特殊值，如 0 或 -1）。
            - `prev_accepted_v` 是对应 `prec_accepted_n` 的值（如果 `prev_accepted_n` 为空，则此值也为空）。
        3. 这个 `Promise` 响应意味着 Acceptor 承诺：它未来不会再接受任何编号小于 `n` 的提议，也不会再响应任何编号小于 `n` 的 Prepare 请求。
    * 情况 2：`n` 小于 max_prepare，这表示收到了一个过时的 Prepare 请求（可能来自网络延迟，或者有其他的 Proposer）已经发起了更高编号的提议，Acceptor 可以选择：
        1. 直接忽略这个 `Prepare(n)` 消息。
        2. 或者可以优化一下，回复一个 `Nack(n, max_prepare)` 拒绝的消息，告知 Proposer 它的提议编号太旧了，让 Proposer 可以更快放弃或重试。

#### 阶段二：接受（Accept）与已接受（Accepted）

如果 Proposer 成功从多数 Acceptor 那里获得了对提议编号 `n` 的 Promise，它就可以进入第二阶段，尝试让一个具体的值被接受。

* （2a）Accept 消息
    * Proposer 收集来自 Quorum 的 `Promise(n, prev_n, prev_v)` 响应。
    * 关键决策：选择要提议的值 `v_proposal` 。
        1. Proposer 检查所有收到的 `Promise` 响应中携带的 `(prev_n, prev_v)`。
        2. 如果**存在**任何 `prev_v` 不为空的响应（即至少有一个 Acceptor 之前接受过某个值），Proposer **必须**选择所有报告的 `(prev_n, prev_v)` 中具有**最高** `prev_n` 的那个 `prev_v` 作为它本次要提议的值 `v_proposal`。
        3. 如果所有收到的 `Promise` 响应中的 `prev_v` 都为空（即这个 Quorum 中的 Acceptor 之前都没有接受过任何值），那么 Proposer **可以自由选择**它最初想要提议的值 `v_original` 作为 `v_proposal`。
    * 一旦确定了 `v_{proposal}`，Proposer 就向之前回复了 `Promise` 的那个 Quorum（或者另一个 Quorum，通常是同一个）发送 `Accept(n, v_proposal)` 消息。
* （2b）Accepted 响应：
    * 当一个 Acceptor 收到 `Accept(n, v)` 消息时，它会检查 `n` 是否**大于或等于**它记录的 `max_prepare`。**注意**：这里允许等于 `n`，因为 Acceptor 在阶段 1b 已经承诺了不接受小于 `n` 的提议，但接受等于 `n` 的提议是允许的，并且是协议正确运行所必需的。
    * 情况 1：`n` > `max_prepare`。Acceptor 接受这个提议：
        1. 记录下接受的提议编号 `accepted_proposal_number = n` 和接受的值 `accepted_value = v`（**并持久化这些状态**）。
        2. 向 Proposer 和所有 Learner 发送 `Accepted(n, v)` 消息。
    * 情况 2：`n` < `max_prepare`。这表示 Acceptor 在回复 `Promise(n)` 之后，又收到了一个更高编号的 `Prepare(n')` 并回复了 `Promise(n')`。根据承诺，它不能再接受编号为 `n` 的提议了。因此，它会忽略这个 `Accept(n, v)` 消息。

**共识达成**：当一个值 `v` 被一个 Quorum（多数）的 Acceptor 接受（即它们都发送了 `Accepted(n, v)` 消息，对于某个特定的 `n`），那么这个值 `v` 就被 Paxos 协议**选定（chosen）**了。Learner 通过收集这些 `Accepted` 消息来得知结果。

这个两阶段过程的核心机制在于：**阶段一的 Prepare/Promise 交互强制任何想要成为“领导者”（获得批准的 Proposer）的进程，必须先去了解并尊重可能已经发生的“历史”（即被 Quorum 接受的值），然后才能在阶段二提出自己的（可能被修正过的）主张。** 如果 Quorum 中的 Acceptor 报告了之前接受的值，新的 Proposer 就失去了自由选择值的权利，必须延续那个已被接受（或趋向于被接受）的值。正是这种机制，保证了即使有多个 Proposer 并发尝试，或者 Proposer 在过程中失败，最终也只会有一个值被稳定地选定。

### 算法特性

法定人数（Quorum）的概念：保证多数一致

Quorum 在 Paxos 中扮演着至关重要的角色，它是保证算法安全性和容错性的数学基础。

* **定义**：一个 Quorum 是 Acceptor 集合的一个子集，其大小通常定义为**严格超过半数**。在一个包含 `N` 个 Acceptor 的系统中，Quorum 的大小通常是 ⌊N/2⌋+1。例如，在有 5 个 Acceptor 的系统中，Quorum 大小是 3；在有 3 个 Acceptor 的系统中，Quorum 大小是 2 。
* **Quorum 交集属性**：Quorum 的核心特性是**任意两个 Quorum 必须至少有一个共同的成员** 。这是集合论的一个简单结果（如果两个子集都包含超过半数的元素，它们的交集不可能是空的）。
* **保证安全性（Safety）**：Paxos 利用 Quorum 交集属性来确保一致性。
    - 在阶段一，Proposer 必须从一个 Quorum 的 Acceptor 处获得 Promise。
    - 在阶段二，Proposer 必须将 Accept 请求发送给一个 Quorum 的 Acceptor，并且一个值只有被一个 Quorum 的 Acceptor 接受才算被选定。
    - 假设一个值 `v` 在提议 `m` 中被一个 Quorum `Q_1` 接受了。现在，如果另一个 Proposer 发起了更高编号的提议 `n > m`，它必须在阶段一从某个 Quorum `Q_2` 获得 Promise。由于 `Q_1` 和 `Q_2` 必然有交集，Proposer `n` 在阶段一必定会联系到至少一个接受了 `(m, v)` 的 Acceptor。这个 Acceptor 会在 `Promise` 响应中报告 `(m, v)`。根据阶段二的规则，Proposer `n` 必须选择具有最高编号的已接受值，因此它也只能提议值 `v`（或者是在 `m` 和 `n` 之间被选定的某个值，通过归纳法可知也必须是 `v`）。这样就保证了不会有与 `v` 不同的值在更高的提议编号中被选定。
* **提供容错性（Fault Tolerance）**：Quorum 机制使得 Paxos 能够容忍少数 Acceptor 的失败。在一个有 `N = 2F + 1` 个 Acceptor 的系统中，只要至少有 `F + 1` 个 Acceptor（即一个 Quorum）能够正常工作并相互通信，系统就能继续运行 Prepare 和 Accept 阶段，从而达成共识 。系统可以容忍最多 `F` 个 Acceptor 同时发生故障。
* **动态 Quorum**：虽然 Basic Paxos 通常使用静态的、基于多数的 Quorum，但在更高级的系统中，有时会考虑动态调整 Quorum 配置（例如，增减节点）。但这本身就是一个需要通过共识来安全完成的操作，以避免在配置变更期间引入不一致性。

### 算法难点与挑战
1. 活锁问题 ：多个 Proposer 可能会不断地相互抢占，导致没有提案能被选定。解决方法是引入随机延迟或选举一个主 Proposer。
2. 恢复问题 ：节点故障恢复后如何获取最新状态。
3. 多轮决议 ：实际系统中需要连续对多个值达成一致，需要引入实例编号。

# 代码实现

下面是一个基于 Golang 版本的 Paxos 算法的简易实现。

## 基础数据结构定义

```go
// ProposalID 表示提案编号
type ProposalID struct {
    Number int
    NodeID string
}

// Value 表示提案的值
type Value interface{}

// PrepareRequest 表示准备请求
type PrepareRequest struct {
    ProposalID ProposalID
}

// PrepareResponse 表示准备响应
type PrepareResponse struct {
    ProposalID    ProposalID
    AcceptedID    *ProposalID
    AcceptedValue Value
    Ok            bool
}

// AcceptRequest 表示接受请求
type AcceptRequest struct {
    ProposalID ProposalID
    Value      Value
}

// AcceptResponse 表示接受响应
type AcceptResponse struct {
    ProposalID ProposalID
    Ok         bool
}
```

## Acceptor 实现

Acceptor 是 Paxos 算法中的关键角色，负责接收和响应 Proposer 的请求：

```go
package paxos

// Acceptor represents a node that can accept proposals in the Paxos protocol
type acceptor struct {
    promisedID    *ProposalID
    acceptedID    *ProposalID
    acceptedValue Value
}

// NewAcceptor creates a new Acceptor instance
func NewAcceptor() Acceptor {
    return &acceptor{}
}

// HandlePrepare processes a prepare request from a proposer
func (a *acceptor) HandlePrepare(request PrepareRequest) PrepareResponse {
    // If we haven't promised anyone yet, or this proposal is higher than our promise
    if a.promisedID == nil || (request.ProposalID.Number > a.promisedID.Number) {
        a.promisedID = &request.ProposalID
        return PrepareResponse{
            ProposalID:    request.ProposalID,
            AcceptedID:    a.acceptedID,
            AcceptedValue: a.acceptedValue,
            Ok:           true,
        }
    }

    // Reject if the proposal number is less than what we've promised
    return PrepareResponse{
        ProposalID: request.ProposalID,
        Ok:         false,
    }
}

// HandleAccept processes an accept request from a proposer
func (a *acceptor) HandleAccept(request AcceptRequest) AcceptResponse {
    // Accept only if the proposal ID is greater than or equal to what we've promised
    if a.promisedID == nil || request.ProposalID.Number >= a.promisedID.Number {
        a.promisedID = &request.ProposalID
        a.acceptedID = &request.ProposalID
        a.acceptedValue = request.Value
        return AcceptResponse{
            ProposalID: request.ProposalID,
            Ok:         true,
        }
    }

    // Reject if the proposal number is less than what we've promised
    return AcceptResponse{
        ProposalID: request.ProposalID,
        Ok:         false,
    }
}
```

在上面的代码中，`promisedId` 是 Acceptor 承诺过的最高提案编号，`acceptedId` 和 `acceptedValue` 分别是已接受的提案编号和值。

## Proposer 实现

Proposer 负责发起提案并协商整个一致性过程：

```go
package paxos

import (
    "fmt"
    "math/rand"
    "time"
)

// Proposer represents a node that proposes values in the Paxos protocol
type proposer struct {
    nodeID        string
    proposalNum   int
    acceptors     []Acceptor
    quorumSize    int
}

// NewProposer creates a new Proposer instance
func NewProposer(nodeID string, acceptors []Acceptor) Proposer {
    return &proposer{
        nodeID:      nodeID,
        proposalNum: 0,
        acceptors:   acceptors,
        quorumSize:  len(acceptors)/2 + 1,
    }
}

// Propose attempts to get a value accepted by a majority of acceptors
func (p *proposer) Propose(value Value) (bool, Value) {
    for {
        // Generate a new proposal ID
        p.proposalNum++
        proposalID := ProposalID{
            Number: p.proposalNum,
            NodeID: p.nodeID,
        }
        
        // Phase 1: Prepare
        prepareResponses := p.sendPrepareRequests(proposalID)
        
        // Check if we got a majority of OKs
        if len(prepareResponses) < p.quorumSize {
            // Wait a bit before retrying to avoid live-lock
            time.Sleep(time.Duration(rand.Intn(100)) * time.Millisecond)
            continue
        }
        
        // If any acceptor already accepted a value, we must use the value
        // with the highest proposal number
        highestAcceptedID := p.findHighestAcceptedProposal(prepareResponses)
        valueToPropose := value
        
        if highestAcceptedID != nil {
            for _, resp := range prepareResponses {
                if resp.AcceptedID != nil && 
                   resp.AcceptedID.Number == highestAcceptedID.Number &&
                   resp.AcceptedID.NodeID == highestAcceptedID.NodeID {
                    valueToPropose = resp.AcceptedValue
                    break
                }
            }
        }
        
        // Phase 2: Accept
        acceptResponses := p.sendAcceptRequests(proposalID, valueToPropose)
        
        // Check if we got a majority of OKs
        if len(acceptResponses) >= p.quorumSize {
            return true, valueToPropose
        }
        
        // Wait a bit before retrying to avoid live-lock
        time.Sleep(time.Duration(rand.Intn(100)) * time.Millisecond)
    }
}

// sendPrepareRequests sends prepare requests to all acceptors
func (p *proposer) sendPrepareRequests(proposalID ProposalID) []PrepareResponse {
    var okResponses []PrepareResponse
    
    for _, acceptor := range p.acceptors {
        resp := acceptor.HandlePrepare(PrepareRequest{ProposalID: proposalID})
        if resp.Ok {
            okResponses = append(okResponses, resp)
        }
    }
    
    return okResponses
}

// sendAcceptRequests sends accept requests to all acceptors
func (p *proposer) sendAcceptRequests(proposalID ProposalID, value Value) []AcceptResponse {
    var okResponses []AcceptResponse
    
    for _, acceptor := range p.acceptors {
        resp := acceptor.HandleAccept(AcceptRequest{
            ProposalID: proposalID,
            Value:      value,
        })
        if resp.Ok {
            okResponses = append(okResponses, resp)
        }
    }
    
    return okResponses
}

// findHighestAcceptedProposal finds the highest proposal ID among accepted proposals
func (p *proposer) findHighestAcceptedProposal(responses []PrepareResponse) *ProposalID {
    var highest *ProposalID
    
    for _, resp := range responses {
        if resp.AcceptedID != nil {
            if highest == nil || 
               resp.AcceptedID.Number > highest.Number ||
               (resp.AcceptedID.Number == highest.Number && 
                resp.AcceptedID.NodeID > highest.NodeID) {
                highest = resp.AcceptedID
            }
        }
    }
    
    return highest
}
```

## Learner 实现

Learner 不参与提案，它只学习被批准的提案。Learner 接收来自 Acceptor 的消息，当收到来自多数（一个 Quorum）Acceptor 的关于同一个提议（相同编号和值）的 Accepted 消息时，Learner 就知道这个值已经被选定，随之可以将选定的值应用于本地状态或通知客户端。

注意：Learner 的失败通常不影响 Paxos 协议达成共识的过程，但可能会影响最终结果被知晓的范围。

```go
// Learner represents a node that learns the agreed-upon value in the Paxos protocol
type learner struct {
    mu            sync.RWMutex
    learnedValue  Value
    learnedID     *ProposalID
}

// NewLearner creates a new Learner instance
func NewLearner() Learner {
    return &learner{}
}

// Learn records a value that has been accepted by a quorum of acceptors
func (l *learner) Learn(proposalID ProposalID, value Value) {
    l.mu.Lock()
    defer l.mu.Unlock()

    // Only learn if this is a newer proposal than what we've already learned
    if l.learnedID == nil || proposalID.Number > l.learnedID.Number {
        l.learnedValue = value
        l.learnedID = &proposalID
    }
}

// GetLearnedValue returns the most recently learned value
func (l *learner) GetLearnedValue() Value {
    l.mu.RLock()
    defer l.mu.RUnlock()
    return l.learnedValue
```

# 总结一下

Paxos 算法是分布式计算领域的一项里程碑式的成就，它为在不可靠网络和节点故障环境中实现数据一致性这一核心挑战提供了第一个经过严格证明的解决方案。通过其精巧的两阶段协议和 Quorum 机制，Paxos 保证了即使在部分失败的情况下，系统也能就某个值达成唯一的共识，这对于构建可靠的分布式数据库、配置管理系统和状态机复制至关重要。

然而，Paxos 的强大功能也伴随着其众所周知的复杂性。其原始论文《The Part-Time Parliament》独特的叙事风格，以及算法本身在并发和故障场景下的微妙交互，使得 Paxos 的理解和正确实现都相当困难。正如本文在实现部分所强调的，状态管理（尤其是 Acceptor 状态的持久化）和并发控制是实现 Paxos 时需要特别注意的关键点，任何疏忽都可能破坏其安全性保证。

Basic Paxos 在理论上优先保证安全性，但牺牲了活性的保证，这导致在实践中几乎必须引入 Leader 选举机制来避免活锁并提高性能。这种对 Leader 的依赖，以及 Basic Paxos 处理一系列决策时的效率问题（每轮决策都需要完整的两阶段），催生了 Paxos 的多种变种：

* Multi-Paxos：通过选举稳定 Leader 并允许 Leader 跳过 Prepare 阶段来优化连续决策的效率，是实践中最常用的形式。
* Fast Paxos：旨在减少达成共识所需的网络延迟，允许在某些条件下（无冲突时）跳过 Prepare 阶段，用一轮消息完成共识，但可能增加冲突处理的复杂性。
* Cheap Paxos：着眼于减少维持系统容错性所需的活跃 Acceptor 数量，通过使用备份 Acceptor 来降低常规操作的资源消耗。

这些变种的存在本身就说明了原始 Paxos 设计中固有的权衡（例如，简单性 vs 效率，理论保证 vs 实践性能）。此外，对 Paxos 理解难度的普遍认同，也直接推动了旨在提供类似一致性保证但更易于理解和实现的替代算法的开发，其中最著名的就是 Raft。Raft 通过更明确地定义 Leader 角色、状态转换和日志复制机制，降低了工程师理解和实施共识算法的门槛。（后面我可能会做一下 6.824 中 Raft 的 lab，到时候可能会专门写一篇 Raft 的文章，先挖个坑吧）

总而言之，Paxos 算法以其开创性的理论贡献和在构建健壮分布式系统中的实际应用价值，奠定了其在计算机科学中的重要地位。尽管其复杂性促使了变种和替代方案的发展，但理解 Paxos 的核心原理——角色、阶段、Quorum 和安全保证——对于任何深入研究或构建分布式系统的工程师和研究人员来说，仍然是不可或缺的基础。

(全文完)

## Refrence

1. Lamport, L. (1998). The Part-Time Parliament. ACM Transactions on Computer Systems (TOCS), 16(2), 133-169.
2. Lamport, L. (2001). Paxos Made Simple. ACM SIGACT News (Distributed Computing Column), 32(4), 51-58.
3. Fischer, M. J., Lynch, N. A., & Paterson, M. S. (1985). Impossibility of distributed consensus with one faulty process. Journal of the ACM (JACM), 32(2), 374-382.
4. Ongaro, D., & Ousterhout, J. (2014). In Search of an Understandable Consensus Algorithm. 2014 USENIX Annual Technical Conference (USENIX ATC 14), 305-319.  (Raft 论文，作为 Paxos 的重要对比和后续发展)
5. Schneider, F. B. (1990). Implementing fault-tolerant services using the state machine approach: A tutorial. ACM Computing Surveys (CSUR), 22(4), 299-319.  (关于状态机复制的重要背景文献)
