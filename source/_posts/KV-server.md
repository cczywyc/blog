---
title: "KV Server"
description: "MIT-6.824实验二详解"
date: 2024-09-17
tags:
  - MIT-6.824
  - 分布式系统
categories:
  - 分布式系统文章合集
cover: https://img.cczywyc.com/post-cover/kv_server_cover.jpg
---

MIT 6.824 课程的第二个 lab 是实现一个 key/value server，这里是官网的 [实验说明](http://nil.csail.mit.edu/6.5840/2024/labs/lab-kvsrv.html)。本篇文章是关于 key/value server 实验设计思路及实现过程。

# 实验概述

本实验要求实现一个单机版的 kv server，支持 `Put`、`Append` 和 `Get` 三种操作。该服务保证所有的操作都必须是线性化（linearizable）的，确保操作顺序符合实时顺序，并且能够处理网络故障（例如消息丢失），保证操作只执行一次。实验目标是能够满足不同客户端以及不可靠网络等场景，确保服务器在并发和故障情况下都能够正常工作。

# 设计

* 线性化：通过使用 Go 的互斥锁（mutex）保护共享数据，确保并发访问时操作顺序一致。
* 网络故障处理：客户端为每个请求分配唯一序列号（sequence number），服务器通过客户端唯一标识（Client ID）和请求序列号检测重复请求，确保幂等性。
* 操作定义：
    * Put(key, value): 将键 `key` 的值设置为 `value`，覆盖原有值。需确保线程安全，并记录序列号以防重复。
    * Append(key, arg): 向键对应的值增加内容，并返回旧值，若键不存在则视为对空字符串的追加。
    * Get(key): 获取键的值，若键不存在返回空字符串。由于 Get 是只读操作，重复执行不会影响状态，无需特别处理序列号。

# 实现

## 核心数据结构设计

### 键值对存储

键值对存储核心的数据结构选择 `map`，并配合互斥锁（`sync.Mutex`）保证并发安全

```go
type KVServer struct {
	mu         sync.Mutex
	store      map[string]string
	clientSeqs map[int64]int // to track the latest sequence number for each client
}

```

* store: 存储键值对
* clientSeqs：记录每个客户端已经处理过的序列号，用于检测重复请求。

### 操作原子性

通过互斥锁保护共享资源的访问：

```go
// Get fetches the current value for the key.
// A Get for a non-existing key should return an empty string.
func (kv *KVServer) Get(args *GetArgs, reply *GetReply) {
	kv.mu.Lock()
	defer kv.mu.Unlock()
  // 处理 Get 逻辑
}

```

每个操作在处理前先获取锁，确保同一时刻只有一个协程修改数据。

### 处理重复请求

为每个客户端分配唯一 `ClientId`，每次请求携带递增的 Seq。服务器通过 `clientSeqs` 记录已处理的请求：

```go
	if args.SeqNumber <= kv.clientSeqs[args.ClientId] {
		// Repeat the request and return the historical value
		reply.PreviousValue = kv.store[args.Key]
		return
	}

	......

	// update the request sequence number for the client
	kv.clientSeqs[args.ClientId] = args.SeqNumber

```

## 关键步骤代码

### Get

```go
// client.go

func (ck *Clerk) Get(key string) string {
	args := GetArgs{
		Key:       key,
		ClientId:  ck.clientId,
		SeqNumber: ck.seqNumber,
	}
	reply := GetReply{}
	ck.seqNumber++

	// send the Get RPC request
	ok := ck.server.Call("KVServer.Get", &args, &reply)
	if ok {
		return reply.Value
	}
	return ""
}
```

```go
// server.go

// Get fetches the current value for the key.
// A Get for a non-existing key should return an empty string.
func (kv *KVServer) Get(args *GetArgs, reply *GetReply) {
	kv.mu.Lock()
	defer kv.mu.Unlock()

	value, exists := kv.store[args.Key]
	if !exists {
		reply.Value = ""
	} else {
		reply.Value = value
	}
}
```

### Put && Append

```go
// client.go

func (ck *Clerk) PutAppend(key string, value string, op string) string {
	args := PutAppendArgs{
		Key:       key,
		Value:     value,
		Op:        op,
		ClientId:  ck.clientId,
		SeqNumber: ck.seqNumber,
	}
	reply := PutAppendReply{}
	ck.seqNumber++

	// send the RPC request
	ok := ck.server.Call("KVServer."+op, &args, &reply)
	if ok {
		return reply.PreviousValue
	}
	return ""
}

func (ck *Clerk) Put(key string, value string) {
	ck.PutAppend(key, value, "Put")
}

// Append value to key's value and return that value
func (ck *Clerk) Append(key string, value string) string {
	return ck.PutAppend(key, value, "Append")
}
```

```go
// server.go

// Put installs or replaces the value for a particular key in the map.
func (kv *KVServer) Put(args *PutAppendArgs, reply *PutAppendReply) {
	kv.mu.Lock()
	defer kv.mu.Unlock()

	if args.SeqNumber <= kv.clientSeqs[args.ClientId] {
		// Repeat the request and return the historical value
		reply.PreviousValue = kv.store[args.Key]
		return
	}

	kv.store[args.Key] = args.Value
	reply.PreviousValue = kv.store[args.Key]

	// update the request sequence number for the client
	kv.clientSeqs[args.ClientId] = args.SeqNumber
}

// Append appends arg to key's value and returns the old value.
// An Append to a non-existing key should act as if the existing value were a zero-length string.
func (kv *KVServer) Append(args *PutAppendArgs, reply *PutAppendReply) {
	kv.mu.Lock()
	defer kv.mu.Unlock()

	if args.SeqNumber <= kv.clientSeqs[args.ClientId] {
		reply.PreviousValue = kv.store[args.Key]
		return
	}

	// find the key if exist
	value, exists := kv.store[args.Key]
	if !exists {
		// a zero-length string
		value = ""
	}
	// append the value
	kv.store[args.Key] = value + args.Value
	reply.PreviousValue = value

	// update the request sequence number for the client
	kv.clientSeqs[args.ClientId] = args.SeqNumber
}
```

# 总结

本实验是一个比较简单的分布式系统的入门实验。它初步提出了分布式系统中的数据一致性问题以及容错处理，在实现过程中需要考虑线性化、客户端的重复请求和网络故障处理等场景，实现较为简单，因此本文篇幅也相对较短。完整代码见本项目 [Github 仓库](https://github.com/cczywyc/mit-6.5840/tree/main/src/kvsrv)。

# Refrence

1. http://nil.csail.mit.edu/6.5840/2024/labs/lab-kvsrv.html

