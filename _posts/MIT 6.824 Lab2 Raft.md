

## MIT 6.824 Lab2 Raft

### 简介

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;这个系列的实验会实现一个有容错的存储键值对的系统。而在这个实验中，要扩展Raft——一个复制的状态机协议。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;一个复制服务通过将状态复制到多个服务器上来达成容错。就算有失败，复制使得服务能继续运行。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Raft管理服务的状态，它还帮助服务选出失败以后应该到达的正确状态。Raft实现了一个有限状态机，它将客户请求组成一个叫做log的序列，确保所有的副本都同意log的内容。每个副本都按顺序处理log中客户的请求，将这些请求应用到副本服务状态的本地副本。因为所有可用的副本都看到相同的日志，它们就会同时处理相同的请求，因此会有相同的状态。如果一个服务器宕机有重启，Raft负责为这些服务器更新日志。只要有一个主服务器，Raft就会继续工作。

------

### Part 2A: Election

#### 简介

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在这个part中，需要实现Raft的选举算法，下面来详细介绍下Raft的领导者选举规则：

+ 整个服务器集群处在一个时间线上，时间以轮来计，每轮开始进行领导者选举，然后开始工作，直到领导者宕机，开始下一轮选举。
+ 若当前服务器在一定时间内没有收到领导者发来的heartbeat，它就认为此时没有领导者，它会发起一个选举，将term+1，并将自己作为候选人，然后使用RPC服务向其他服务器发送投票请求。这个timeout的时间是随机生成的，为了防止所有服务器同时超时，进入候选状态，最后没有领导者被选出。
+ 因为要成为领导者必须要有集群的全部日志，所以领导者的状态不能落后于跟随者，因此处于跟随者状态的服务器收到投票请求的时候，首先会比较自己和候选人的term，再比较最近命令的term和index，如果发现落后则拒绝，否则投票。
+ 若收到多个候选人的投票请求，按先到先投的原则。
+ 当本轮选举结束而没有领导者被选出时，各个服务器会分别timeout，进行下一轮选举，直到有一个领导者被选举出来。

#### 重要方法的实现

##### func (rf *Raft) electionDeamon()

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;这是领导者的选举线程，它一直循环，当发现跟随者超时时，将跟随者转换为候选者；当发现服务器的状态是候选者的时候，它将准备并向所有其他服务器发送投票请求，然后等待，检查结果。

```go
func (rf *Raft) electionDeamon() {
	for {
		switch rf.state {
		case FOLLOWER:
			select {
			case <-time.After(time.Duration(rand.Int63()%330+550) * time.Millisecond):
				rf.state = CANDIDATE
				println("follower->candidate")
			}
		case LEADER:
            //TODO 见下
			time.Sleep(HEARTBEAT)
		case CANDIDATE:
			rf.mu.Lock()
			defer rf.mu.Unlock()
			rf.currentTerm++
			rf.voteFor = rf.me
			rf.voteCount++
			rf.mu.Unlock()
			println("send vote request")
			go rf.sendAllVoteRequests()
			//println("start select")
			select {
			case <-time.After(time.Duration(rand.Int63()%333+550) * time.Millisecond):
			case <-rf.isLeader:
				println("there is a leader")
				rf.mu.Lock()
				rf.state = LEADER
				rf.nextIndex = make([]int, len(rf.peers))
				rf.matchIndex = make([]int, len(rf.peers))
				for i := range rf.peers {
					rf.nextIndex[i] = rf.getLastIndex() + 1
					rf.matchIndex[i] = 0
				}
				rf.mu.Unlock()
			}
		}
	}
}
```

---

#####func (rf *Raft) sendRequestVote(server int, args *RequestVoteArgs, reply *RequestVoteReply) bool

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;这是Raft的发送投票方法，它通过RPC调用RequestVote方法并检查结果。若候选者的轮落后于跟随者，则它将它的当前轮次加一，将自己转为跟随者；若成功获得选票，则它会统计获得选票，如果选票大于一半，则它就成为领导者。

```go
func (rf *Raft) sendRequestVote(server int, args *RequestVoteArgs, reply *RequestVoteReply) bool {
	ok := rf.peers[server].Call("Raft.RequestVote", args, reply)
	println("is vote: " + strconv.FormatBool(reply.VoteGranted))
	rf.mu.Lock()
	defer rf.mu.Unlock()
	if ok {
		term := rf.currentTerm
		if rf.state != CANDIDATE {
			return ok
		}
		if args.Term != term {
			return ok
		}
		if reply.Term > term {
			rf.currentTerm = reply.Term
			rf.state = FOLLOWER
			rf.voteFor = -1
			rf.persist()
		}
		if reply.VoteGranted {
			rf.voteCount++
			if rf.state == CANDIDATE && rf.voteCount > len(rf.peers)/2 {
				rf.state = FOLLOWER
				rf.isLeader <- true
			}
		}
	}
	return ok
}
```

---

##### func (rf *Raft) RequestVote(args *RequestVoteArgs, reply *RequestVoteReply)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;这个服务器处理投票请求的方法。它按照规则判断候选者是否落后于自己，并决定是否给出选票。

```go
func (rf *Raft) RequestVote(args *RequestVoteArgs, reply *RequestVoteReply) {
	// Your code here (2A, 2B).
	rf.mu.Lock()
	defer rf.mu.Unlock()
	println("args.Term: " + strconv.Itoa(args.Term))
	println("rf.Term: " + strconv.Itoa(rf.currentTerm))
	reply.VoteGranted = false
	if args.Term < rf.currentTerm {
		reply.VoteGranted = false
		reply.Term = rf.currentTerm
	}

	if args.Term > rf.currentTerm {
		rf.currentTerm = args.Term
		rf.state = FOLLOWER
	}
	reply.Term = rf.currentTerm
	if (args.CandidateId == rf.voteFor) || args.Term > rf.currentTerm || (rf.getLastTerm() < args.LastLogTerm ||
		(rf.getLastTerm() == args.LastLogTerm && rf.getLastIndex() <= args.LastLogIndex)) {
		reply.VoteGranted = true
		rf.voteFor = args.CandidateId
		rf.state = FOLLOWER
	}
}
```

### Part II: Single-worker word count

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;这个part需要实现一个统计文档中出现单词数目的简单例子，具体如下：

- mapF()方法将每个传入的内容分割为单词，用单词作为键，为每个单词创建一个KeyValue，值为"1”，表示这个单词出现了一次，将所有KeyValue作为一个切片返回。
- redecuF()方法传入一个键和键对应的值数组，这个方法需要将所以mapF()中统计的次数相加，返回值数组的长度即可。

### Part III: Distributing MapReduce tasks

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;这个part需要用分布式的思想实现上一个part的内容，具体采用的是RPC调用和go的并发编程来实现。

- schedule()在所给的阶段（map或reduce）启动和等待所有任。mapFIles参数决定map阶段输入文件的名字。nReduce参数是reduce任务的数量。

- 所有ntasks个任务都会在worker中被调度，当所有任务都成功结束，schedule()会返回。

- master线程在mapreduce中调用两次schedule()，map阶段一次，reduce()阶段一次。这个方法的任务是，把任务交给可用的工作线程。因为任务一般比工作线程多，所以schedule()方法要给每个工作线程一系列任务。

- schedule()方法从rigisterChan参数中知道有多少可用的工作线程。这个channel为每个worker产生一个字符串包含它的RPC地址。一些worker在schedule()方法被调用之前已经存在，另一些会在这个方法运行时生成，schedule()必须将这些worker充分利用。

- schedule()通过发送一个Worker.DoTask RPC给worker来告诉它要执行任务。

  这里踩了好几个坑，说明一下。一个是关于sync.WaitGroup的add和Done两个方法，add只能在主线程里面用，想要同步线程就要在其他线程里用Done。然后将worker从registerChan中读出来并结束一个任务以后要放回去（来自一个没怎么学go并发编程的兄弟）

### Part IV: Handling worker failures

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在这个part中，需要处理失败的线程。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;一个RPC调用失败不意味着这个线程没有在执行这个任务，这可能是因为返回值丢失或者master线程的RPC调用超时了。因此，这可能会发生于两个worker收到相同任务，处理他，然后生成输出。两个map或reduce调用必须对同一个输入生成相同的输出，所以如果后续处理有时读取一个输出并且有时读取另一个输出，则不会有不一致。除此之外，MapReduce框架确保map方法和reduce方法自动输出：输出文件可能不存在也可能包含map或reduce方法的单个执行的全部输出。



### 流程

初始化各个服务器，配置初始参数，读取以前的配置（如果有的话），启动选举线程，启动执行命令线程。

选举进程：若raft被关闭则关闭选举进程，若要重置时间则重置，若超时则启动投票线程。

投票线程：先装填投票请求参数，然后用rpc发送投票请求，判断得到票数是否能使之成为leader，若成为则启动心跳线程。

心跳线程：每隔一段时间启动一致性检查线程，然后休眠。

一致性检查线程：如果这一个服务器的nextIndex，发送心跳包（空的附加日志）。