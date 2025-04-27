---
title: "MapReduce"
description: "MIT-6.824实验一详解"
swiper_index: 3
date: 2024-08-25
tags:
  - MIT-6.824
  - 分布式系统
categories:
  - 分布式系统文章合集
cover: https://img.cczywyc.com/map-reduce/MapReduce_cover.jpg
---

MapReduce 是一种处理大量数据的编程模型以及实现，它使用 Map 和 Reduce 两个函数对于海量数据计算做了一次抽象，用于在集群上使用分布式算法处理和生成大数据集。MapReduce 最先由 Google 在 2004 年提出，并且在内部得到了大量的实践，简单说来，MapReduce 模型由一个 Map 函数处理一个基于 key/value pair 的数据集合，输出中间的基于 key/value pair 的数据集合，然后再由一个 Reduce 函数用来合并所有的具有相同中间 key 值的中间 value 值。

仔细分析发现，现实世界中有很多满足上述处理模型的例子，关于 MapReduce 的产生背景和详细的介绍本文不再详细叙述，强烈建议阅读 Google 的[论文](https://pdos.csail.mit.edu/6.824/papers/mapreduce.pdf)，一定会对 MapReduce 模型有更进一步的理解。

说起分布式系统，大家应该都知道 MIT 有一个明星课程 6.824，它是专门讲解分布式系统的，这个课程还设置了一系列非常有难度的实验来检验你对课程的掌握。为了进一步加深我对分布式系统的理解，前段时间我便开始完成 2024 年春季 6.824 的 5 个实验 lab（备注：6.824 以前是 4 个实验 lab，从 2024 年春季课程开始，拆分成了 5 个，但是 lab 的大体内容都是一样的，并且现在课程改名为 6.5840，其实都是一样的），本篇文章就是 MapReduce 实验的实现详解文章，以此记录实现步骤和思考过程。

6.824 历年所有的课程安排、实验 lab 和示例代码都可以在 [课程官网](https://pdos.csail.mit.edu/6.824/) 找到。

# 实验前准备

关于实验前的准备，这里我以 2024 春季课程为例。

1. 本实验是基于 Go 语言来完成的，首先需要安装 Go 的开发环境；
2. clone 实验代码，本实验运行的 main 函数已经编写好，我们只需要自己实现 Coordinator 和 Worker 以及 RPC 模块，代码仓库地址和实验标准也可以在 [官网](https://pdos.csail.mit.edu/6.824/labs/lab-mr.html) 找到。

# 实验详解

本实验中我们需要实现一个分布式的 MapReduce 来完成统计多个文件中每个单词出现的次数多任务。这个分布式的 MapReduce 是由一个 Coordinator 进程 和多个 Worker 进程共同完成任务。我们要完成这个任务分为两步，第一步是执行 Map 任务，第二步是执行 Reduce 任务。

## 执行概括

通过将 Map 调用的输入数据自动分割为 M 个数据片段的集合，Map 调用被分布到多台机器上执行。输入的数据片段能够在不同的机器上并行处理。使用分区函数将 Map 调用产生的中间 key 值分成 R 个不同分区（例如本实验中的 hash(key) mod R），Reduce 调用也被分布到多台机器上执行。分区数量（R）和分区函数由用户指定。在本实验中，R 从主函数中当作参数传入，为 10。下面是 MapReduce 执行概览图。

![](https://img.cczywyc.com/map-reduce/mapReduce_execution_overview.png)

在 MapReduce 模型中，有一个特殊的程序叫 Coordinator（也叫做 Master），其他的程序都是 Worker 程序，由 Coordinator 分配任务给 Worker。待分配的任务被分成了两类，一类是 Map 任务，一类是 Reduce 任务，Coordinator 先将 Map 任务分配给空闲的 Worker，并记录 Worker 进程和 Map 任务的相关信息和执行结果，等所有的 Map 任务都执行结束后开始分配 Reduce 任务给空闲的 Worker，直至所有的任务结束，程序退出。

## 程序数据结构设计

### Coordinator

Coordinator 进程只有一个，它负责任务的调度，并且记录每一个任务的状态（等待调度、执行中或已完成），同时 Cootdinator 还应该记录每个 Worker 机器（进程）的标识。

```go
type Phase int

const (
	MapPhase Phase = iota + 1
	ReducePhase
	Done
)

type TaskType int

const (
	MapTask    TaskType = iota + 1
	ReduceTask          // reduce task
)

type TaskStatus int

const (
	Waiting TaskStatus = iota + 1
	Running
	Finished
)

type Task struct {
	Id       int        // the map task or reduce task id
	WorkName string     // the worker name
	TaskType TaskType   // the task type, map or reduce
	Status   TaskStatus // the task state
	Input    []string   // task input files, map task is only one input file
	Output   []string   // task output files, reduce task is only one output file
}

type BaseInfo struct {
	nReduce   int // the total number of reduce tasks
	taskMap   map[TaskType][]*Task
	workerMap map[string]*WorkInfo
}

type WorkInfo struct {
	name           string
	lastOnlineTime time.Time
}

type Coordinator struct {
	phase    Phase
	baseInfo *BaseInfo
	timer    chan struct{}
	mutex    sync.Mutex
}
```

### Worker

Worker 进程有一个或者多个，在真实的 MapReduce 场景中，Worker 进程往往分布在不同的机器上执行以提高任务执行的效率，在本实验中所有的 Worker 进程都在一个机器上完成，可以通过起多个 Worker 进程来模拟，不过我的程序依然是按照多机器来设计的。

Worker 进程负责具体执行任务，它通过 RPC 和 Coordinator 通信。在本实验中，空闲的Worker 进程不断地向 Coordinator 发送请求获取任务，直至 MapReduce 程序退出。

```go
type WorkerS struct {
	name    string
	mapF    func(string, string) []KeyValue
	reduceF func(string, []string) string
	workDir string
}

// KeyValue is the key/value pire of the map functions
type KeyValue struct {
	Key   string
	Value string
}

// ByKey is for sorting by key
type ByKey []KeyValue

func (a ByKey) Len() int           { return len(a) }
func (a ByKey) Swap(i, j int)      { a[i], a[j] = a[j], a[i] }
func (a ByKey) Less(i, j int) bool { return a[i].Key < a[j].Key }
```

### RPC 设计

前面说到，Coordinator 和 Worker 通过 RPC 通信，官方给出的 RPC 代码中使用了 Unix domain socket 建立连接，这里需要了解一下 Unix domain socket。

Unix domain socket 或者 IPC socket 是一种终端，可以使同一台操作系统上的两个或多个进程数据通信。与管道相比，Unix domain sockets 既可以使用字节流也可以使用数据队列，而管道只能使用字节流。Unix domain socket 的接口和Internet socket 很像，区别在于它不使用网络底层协议来通信，而是使用系统文件的地址来作为自己的身份。它可以被系统进程引用，因此两个进程可以同时打开一个 Unix domain sockets 来进行通信，不过这种通信方式是发生在系统内核里而不会在网络里传播。关于这种通信方式，下面是一个用 golang 实现的简易的服务端和客户端通信的 demo。

```go
// server.go

package main

import (
    "fmt"
    "io"
    "log"
    "net"
    "os"
)

func main() {
    var file string = "test.sock" //用于 unix domain socket 的文件
start:
    lis, err := net.Listen("unix", file) //开始监听
    if err != nil {                      //如果监听失败，一般是文件已存在，需要删除它
            log.Println("UNIX Domain Socket 创 建失败，正在尝试重新创建 -> ", err)
            err = os.Remove(file)
            if err != nil { //如果删除文件失败 ，要么是权限问题，要么是之前监听不成功，不管是什么 都应该退出程序，不然后面 goto 就死循环了
                    log.Fatalln("删除 sock 文件失败！程序退出 -> ", err)
            }
            goto start //删除文件后重新执行一次创建
    } else { //监听成功会直接执行本分支
            fmt.Println("创建 UNIX Domain Socket 成功")
    }

    defer lis.Close() //虽然本次操作不会执行， 不过还是加上比较好

    for {
            conn, err := lis.Accept() //开始接 受数据
            if err != nil {
                    log.Println("请求接收错误 -> ", err)
                    continue //一个连接错误，不会影响整体的稳定性，忽略就好
            }
            go handle(conn) //开始处理数据
    }
}

func handle(conn net.Conn) {
    defer conn.Close()

    for {
            io.Copy(conn, conn) //把发送的数据 转发回去
    }
}
```

```go
// client.go

package main

import (
    "bufio"
    "fmt"
    "log"
    "net"
    "os"
)

func main() {
    file := "test.sock"
    conn, err := net.Dial("unix", file) //发起 请求
    if err != nil {
            log.Fatal(err) //如果发生错误，直接退出程序，因为请求失败所以不需要 close
    }
    defer conn.Close() //习惯性的写上

    input := bufio.NewScanner(os.Stdin) //创建 一个读取输入的处理器
    reader := bufio.NewReader(conn)     //创建 一个读取网络的处理器
    for {
            fmt.Print("请输入需要发送的数据: ")       //打印提示
            input.Scan()                    // 读取终端输入
            data := input.Text()            // 提取输入内容
            conn.Write([]byte(data + "\n")) // 将输入的内容发送出去，需要将 string 转 byte 加 \n  作为读取的分割符

            msg, err := reader.ReadString('\n') //读取对端的数据
            if err != nil {
                    log.Println(err)
            }
            fmt.Println(msg) //打印接收的消息
    }
}
```

我们再来看官方给的代码：

```go
// server start a thread that listens for RPCs from worker.go
func (c *Coordinator) server() {
	rpc.Register(c)
	rpc.HandleHTTP()
	//l, e := net.Listen("tcp", ":1234")
	sockname := coordinatorSock()
	os.Remove(sockname)
	l, e := net.Listen("unix", sockname)
	if e != nil {
		log.Fatal("listen error:", e)
	}
	go http.Serve(l, nil)
}

// call send an RPC request to the coordinator, wait for the response.
// usually returns true, returns false if something goes wrong.
func call(rpcname string, args interface{}, reply interface{}) bool {
	// c, err := rpc.DialHTTP("tcp", "127.0.0.1"+":1234")
	sockname := coordinatorSock()
	c, err := rpc.DialHTTP("unix", sockname)
	if err != nil {
		log.Fatal("dialing:", err)
	}
	defer c.Close()

	err = c.Call(rpcname, args, reply)
	if err == nil {
		return true
	}

	log.Println(err)
	return false
}

// Cook up a unique-ish UNIX-domain socket name
// in /var/tmp, for the coordinator.
// Can't use the current directory since
// Athena AFS doesn't support UNIX-domain sockets.
func coordinatorSock() string {
	s := "/var/tmp/5840-mr-"
	s += strconv.Itoa(os.Getuid())
	return s
}
```

可以看到正是基于上述的通信方式。

## 任务执行

接着我们再来谈任务的执行。Worker 进程向 Coordinator 发送请求获取任务，Coordinator 找到空闲的任务，将其返回给 Worker，Worker 根据任务的类型和任务的参数执行任务，待任务结束后，Worker 告知 Coordinator，Coordinator 修改任务状态，继续分配下一个任务。

```go
// worker.go

// Worker is called by main/mrworker.go
func Worker(mapf func(string, string) []KeyValue, reducef func(string, []string) string) {
	// get the current workspace path
	workDir, _ := os.Getwd()
	w := WorkerS{
		name:    "worker_" + strconv.Itoa(rand.Intn(100000)),
		mapF:    mapf,
		reduceF: reducef,
		workDir: workDir,
	}

	// send the RPC to the coordinator for asking the task in a loop
	for {
		reply := callGetTask(w.name)
		if reply.Task == nil {
			// can not get the task, maybe all map tasks or all reduce task are running but not be finished
			// waiting to the next phase
			time.Sleep(time.Second)
		}

		log.Printf("[Info]: Worker: Receive the task: %v \n", reply)
		var err error
		switch reply.Task.TaskType {
		case MapTask:
			err = w.doMap(reply)
		case ReduceTask:
			err = w.doReduce(reply)
		default:
			// worker exit
			log.Printf("[Info]: Worker name: %s exit.\n", w.name)
			return
		}
		if err == nil {
			callTaskDone(reply.Task)
		}
	}
}

// callGetTask send RPC request to coordinator for asking task
func callGetTask(workName string) *GetTaskReply {
	args := GetTaskArgs{
		WorkerName: workName,
	}
	reply := GetTaskReply{}
	ok := call("Coordinator.GetTask", &args, &reply)
	if !ok {
		log.Printf("[Error]: Coordinator.GetTask failed!\n")
		return nil
	}
	return &reply
}

func callTaskDone(task *Task) {
	args := TaskDoneArgs{
		Task: task,
	}
	reply := TaskDoneReply{}
	ok := call("Coordinator.TaskDone", &args, &reply)
	if !ok {
		log.Printf("[Error]: Coordinator.TaskDone failed!\n")
	}
}
```

首先是 Map 任务。它负责读取初始文件，根据文件内容生成一系列的临时中间文件，这些临时的中间文件内容是一系列的 key/value pair，其中 key 是单词，value 是次数，也就是 1。Map 任务的数量应当根据输入文件的多少适当拆分，本实验中为简单起见，约定一个文件一个 Map 任务，因此一共是 8 个 Map 任务。所有的 Map 任务执行成功后开始 Reduce 阶段，因此 MapReduce 程序需要标记当前处于 Map 阶段还是 Reduce 阶段，只有 Map 阶段结束，才会进入 Reduce 阶段。

### Map 任务

以本实验为例，Map 任务负责读取一个原始的文件的内容，调用 map 函数（注意 map 函数以插件的方式加载，本实验中不需要我们实现）生成 [word-1] 这种格式的临时文件。前面说到，为了方便起见，一个文件代表一个 Map 任务，每个 Map 任务生成的临时文件数量和分区数量 R 保持一致。关于 Map 产生的临时文件分配及命名规则，后面会单独描述，下面是 Worker 节点处理 Map 任务的逻辑。

```go
// doMap execute the map task
func (w *WorkerS) doMap(reply *GetTaskReply) error {
	task := reply.Task
	if len(task.Input) == 0 {
		log.Printf("[Error]: task number %d: No input!\n", task.Id)
		return errors.New("map task no input")
	}
	log.Printf("[Info]: Worker name: %s start execute number: %d map task \n", w.name, task.Id)

	fileName := task.Input[0]
	inputBytes, err := os.ReadFile(fileName)
	if err != nil {
		log.Printf("[Error]: read map task input file error: %v \n", err)
		return err
	}

	// kv2ReduceMap: key: reduce index, value: key/value list. split the same key into reduce
	kv2ReduceMap := make(map[int][]KeyValue, reply.NReduce)
	var output []string
	outputFileNameFunc := func(idxReduce int) string {
		return fmt.Sprintf("mr-%d-%d-temp-", task.Id, idxReduce)
	}

	// call the map function
	kva := w.mapF(fileName, string(inputBytes))
	for _, kv := range kva {
		idxReduce := ihash(kv.Key) % reply.NReduce
		kv2ReduceMap[idxReduce] = append(kv2ReduceMap[idxReduce], kv)
	}

	for idxReduce, item := range kv2ReduceMap {
		// write to the temp file
		oFile, _ := os.CreateTemp(w.workDir, outputFileNameFunc(idxReduce))
		encoder := json.NewEncoder(oFile)
		for _, kv := range item {
			err := encoder.Encode(kv)
			if err != nil {
				log.Printf("[Error]: write map task output file error: %v \n", err)
				_ = oFile.Close()
				break
			}
		}
		// rename
		index := strings.Index(oFile.Name(), "-temp")
		_ = os.Rename(oFile.Name(), oFile.Name()[:index])
		output = append(output, oFile.Name())
		_ = oFile.Close()
	}

	task.Output = output
	log.Printf("[Info]: Worker name: %s finished the map task number: %d \n", w.name, task.Id)
	return nil
}
```

### Reduce 任务

Reduce 任务发生在 Map 任务全部执行结束以后，负责执行 Reduce 操作。它处理 Map 任务产生的中间文件，调用 Reduce 函数输出结果到最终的文件中。以本实验为例，Reduce 任务找到其对应编号（具体规则下面会说）的临时文件，统计单词出现的次数，并将最终的结果输出到文件中，下面是 Worker 节点处理 Reduce 任务的逻辑。

```go
// doReduce execute the reduce task
func (w *WorkerS) doReduce(reply *GetTaskReply) error {
	task := reply.Task
	var kva ByKey
	for _, fileName := range task.Input {
		open, _ := os.Open(fileName)
		decoder := json.NewDecoder(open)
		for {
			var kv KeyValue
			if err := decoder.Decode(&kv); err != nil {
				log.Printf("[Error]: read reduce task input file error: %v \n", err)
				_ = open.Close()
				break
			}
			kva = append(kva, kv)
		}
	}

	// sort
	sort.Sort(kva)

	// write to a temp file
	tempFile := fmt.Sprintf("mr-out-%d-temp-", task.Id)
	oFile, _ := os.CreateTemp(w.workDir, tempFile)

	i := 0
	for i < len(kva) {
		j := i + 1
		for j < len(kva) && kva[j].Key == kva[i].Key {
			j++
		}
		var values []string
		for k := i; k < j; k++ {
			values = append(values, kva[k].Value)
		}
		result := w.reduceF(kva[i].Key, values)
		_, _ = fmt.Fprintf(oFile, "%v %v\n", kva[j].Key, result)

		i = j
	}

	// rename the reduce task output
	index := strings.Index(oFile.Name(), "-temp")
	_ = os.Rename(oFile.Name(), oFile.Name()[:index])
	_ = oFile.Close()

	task.Output = []string{oFile.Name()}
	log.Printf("[Info]: Worker name: %s finished the reduce task number: %d \n", w.name, task.Id)
	return nil
}
```

## 产生的文件

Map 任务和 Reduce 任务产生的文件这里单独来说。在 MIT 实验里对产生的文件名作了具体的要求：

1. A reasonable naming convention for intermediate files is `mr-X-Y`, where X is the Map task number, and Y is the reduce task number.
2. The worker implementation should put the output of the X'th reduce task in the file `mr-out-X`.

前面我们知道，每个 Map 都有对应的任务 ID，每一个原始文件对应一个 Map 任务，根据实验要求，每个 Map 分别输出 R 个临时文件，这里的 R 就是分区数量，即 Reduce 任务的数量。在 Map 任务中，对每个 key 作 hash，并与分区数量 R 取余，从而将不同的 key 散列到不同的 Reduce 任务上；Reduce 任务根据自身任务 ID 和散列的文件 ID 找到它需要处理的临时文件，这样就实现了相同的 key 必定由同一个 Reduce 处理的效果，从而确保了统计的准确性。最后每个 Reduce 根据其自身编号输出对应的文件。

![](https://img.cczywyc.com/map-reduce/MapReduce_map_file.png)

假设一共有 4 个文本需要处理，对应 4 个 Map 任务，分区数量 R 是 9。那么按照规则编号为 1 的 Map 任务生成的中间临时文件便是 mr-1-1、mr-1-2、mr-1-3 ...... mr-1-9，同样地，2 号 Map 任务生成的中间临时文件是 mr-2-1、mr-2-2、mr-2-3 ...... mr-2-9。

![](https://img.cczywyc.com/map-reduce/MapReduce_reduce_file.png)

可以看到，Reduce 阶段，编号为 1 的 Reduce 任务需要处理 Y = 1 的临时文件，其他 Reduce 也是如此。

以上我们就把 Map 任务和 Reduce 产生的文件处理规则说清楚了。

## 容错

最后来说一下错误处理。

### Master 故障

论文中提到，处理 Master 失败一个简单的解决办法就是让 Master 节点周期性的将其数据结构写入磁盘，即形成一个检查点，如果 Master 节点异常，则可以从最后一个检查点重新启动 Master 进程，从而完成故障恢复。实验原因，本次并未实现 Master 节点的故障恢复。

### Worker 故障

处理 Worker 节点故障可以单独起一个线程，让 Master 节点周期性地 ping 每个 Worker 节点，如果在一个约定的时间范围内 Master 没有收到 Worker 节点的心跳信息，则将其标记为失效。所有由这个失效的 worker 完成的 Map 任务被重设为初始状态，重新分配给其他的 Worker 节点。同样的，Worker 失效时正在运行的 Map 或 Reduce 任务也将被重置为空闲状态，重新等待分配执行。

当 Worker 故障时，由于已经完成的 Map 任务的输出存储在这台机器上，Map 任务的输出已经不可访问了，因此必须重新执行。而已经完成的 Reduce 任务的输出存储在全局文件系统上（例如 GFS），因此不需要再次执行。

```go
// workerTimer create a timer that checks the worker online status every 1 second
func (c *Coordinator) workerTimer() {
	go func() {
		ticker := time.NewTicker(time.Second)
		defer ticker.Stop()

		for range ticker.C {
			select {
			case <-c.timer:
				log.Printf("[Info]: Worker timer exit. \n]")
				return
			default:
				c.mutex.Lock()

				for _, workInfo := range c.baseInfo.workerMap {
					if time.Now().Sub(workInfo.lastOnlineTime) <= 10 {
						continue
					}
					// According to the MapReduce paper, in a real distributed system,
					// since the intermediate files output by the Map task are stored on their respective worker nodes,
					// when the worker is offline and cannot communicate, all map tasks executed by this worker,
					// whether completed or not, should be reset to the initial state and reallocated to other workers,
					// while the files output by the reduce task are stored on the global file system (GFS),
					// and only unfinished tasks need to be reset and reallocated.
					if c.phase == MapPhase {
						mapTasks := c.baseInfo.taskMap[MapTask]
						for _, task := range mapTasks {
							if task.WorkName == workInfo.name {
								task.Status = Waiting
								task.WorkName = ""
								task.Output = []string{}
							}
						}
					} else if c.phase == ReducePhase {
						reduceTasks := c.baseInfo.taskMap[ReduceTask]
						for _, task := range reduceTasks {
							if task.WorkName == workInfo.name && task.Status == Running {
								task.Status = Waiting
								task.WorkName = ""
								task.Output = []string{}
							}
						}
					}
					delete(c.baseInfo.workerMap, workInfo.name)
				}
				c.mutex.Unlock()
			}
		}
	}()
}

```



注：本实验完整代码见我的 [Github 仓库](https://github.com/cczywyc/mit-6.5840)。

(全文完)

# Refrence

http://nil.csail.mit.edu/6.5840/2024/labs/lab-mr.html

https://pdos.csail.mit.edu/6.824/papers/mapreduce.pdf



