## MIT 6.824
### lab2


Raft算法中服务器有三种角色
1. Follower
2. Candidate
3. Leader

每个服务器上都会存储的**持久**状态:
1. currentTerm: 当前节点所能看到的最大的term值, 初始化为0并单调递增
2. votedFor: 当前term里将票投给对象的candidateId, 如果尚未投票则为空(我实现时置为-1)
3. log[]: 日志条目(每条日志条目包含命令和任期), 会按顺序作用于状态机, 第一个索引Index为1

每个服务器上都会存储的**易失**状态:
1. commitIndex: 当前服务器最高的被提交的日志的索引, 初始化为0并单调递增
2. lastApplied: 当前服务器最高的被应用于状态机的日志的索引, 初始化为0并单调递增

在状态为Leader的服务器上会额外存储的**易失**状态:
1. nextIndex[]: 针对每个其他节点, 下一个需要发送的日志条目的索引, 初始化为leader最后一个日志索引+1
2. matchIndex[]: 针对每个其他节点, 当前所知的和Leader匹配的最高日志索引, 初始化为0并单调递增

Raft中RPC的种类
一. RequestVote
> candidate节点请求其他节点投票给自己

请求参数:
1. term: 当前candidate节点的term值
2. candidateId: 当前candidate节点的编号
3. lastLogIndex: 当前candidate节点最后一个日志的索引
4. lastLogTerm: 当前candidate节点最后一个日志的term值

返回值:
1. term: 接受投票节点的term值, 主要用来更新当前candidate节点的term值
2. voteGranted: 是否给该申请节点投票

一个节点（无论当前是什么状态）在接收到RequestVote(term, candidateId, lastLogIndex, lastLogTerm)消息时, 其会做如下判断(条件从上往下依次判断):
1. 如果参数携带的term < currentTerm, 则返回currentTerm并拒绝投票请求: (currentTerm, false), 并保持当前节点状态不变
2. 如果参数携带的term = currentTerm, 如果投票节点的votedFor不为空并且不等于参数携带的candidateId则返回当前term并拒绝投票请求: (currentTerm, false),
并保持当前节点状态不变; 如果voteFor为空或者等于candidateId则同意投票请求: (currentTerm, true), 并转换当前节点为Follower, 重置定时器, voteFor置为candidateId
3. 如果参数携带的term > currentTerm, 则首先设置投票节点currentTerm = term, voteFor置为空并转换为Follower状态, 重置定时器, 进入下一步判断:
    1) 如果投票节点最后一条日志的term > 参数携带的lastLogTerm, 则返回**更新后的term**并拒绝投票请求: (currentTerm, false)
    2) 如果投票节点最后一条日志的term = 参数携带的lastLogTerm, 则比较该节点最后一条日志的Index和参数携带的lastLogIndex大小; 如果该节点最后一条日志的Index > lastLogIndex,
    则返回**更新后的term**并拒绝投票请求: (currentTerm, false), 否则令voteFor = candidateId并同意投票请求: (currentTerm, true)
    3) 如果投票节点最后一条日志的term < 参数携带的lastLogTerm, 令voteFor = candidateId并同意投票请求: (currentTerm, true)

区分2和3的原因是paper中提到同一term期间每个节点最多给一个candidate投票, 在情况2中可能存在某个节点已经给别的candidate投过票的情况, 而情况3中所有投票节点会先更新为Follower状态, voteFor也会置为空, 所以不用再判断上述投过票的情况

疑问点: 在情况2中voteFor为空或者等于candidateId时(感觉这种情况很少), 要不要将lastLogIndex和lastLogTerm纳入判断
updated: 疑问解决, 非常有必要! 见raft.go中相关comments

二. AppendEntries
> leader节点使用该消息向其他节点同步日志, 或者发送空消息作为心跳包以维持leader的统治地位

请求参数:
1. term: 当前leader节点的term值
2. leaderId: 当前leader节点的编号
3. prevLogIndex: 当前发送的日志的前面一个日志的索引
4. prevLogTerm: 当前发送的日志的前面一个日志的term值
5. entries[]: 需要各个节点存储的日志条目(用作心跳包时为空, 可能会出于效率发送超过一个日志条目)
6. leaderCommit: 当前leader节点最高的被提交的日志的索引(就是leader节点的commitIndex)

返回值:
1. term: 接收日志节点的term值, 主要用来更新当前leader节点的term值
2. success: 如果接收日志节点的log[]结构中prevLogIndex索引处含有日志并且该日志的term等于prevLogTerm则返回true, 否则返回false

一个节点（无论当前是什么状态）接收到AppendEntries(term, leaderId, prevLogIndex, prevLogTerm, entries[], leaderCommit)消息时, 其会做如下判断(条件从上往下依次判断):
1. 如果参数携带的term < currentTerm, 则返回当前term并返回: (currentTerm, false), 并保持当前节点状态不变
2. 如果参数携带的term >= currentTerm, 则设置currentTerm = term, voteFor = leaderId, 转换当前节点为Follower状态, 重置随机定时器, 进入下一步判断:
    1) 如果当前节点log[]结构中prevLogIndex索引处不含有日志, 则返回(currentTerm, false)
    2) 如果当前节点log[]结构中prevLogIndex索引处含有日志但该日志的term不等于prevLogTerm, 则返回(currentTerm, false)
    3) 如果当前节点log[]结构中prevLogIndex索引处含有日志并且该日志的term等于prevLogTerm, 则**执行存储日志, 然后应用日志到状态机**并返回(currentTerm, true)
    
        * 存储日志:
            
            1. 如果新的日志条目和已经存在的日志条目产生冲突, 则将这个已经存在的日志条目以及它之后的日志条目全部删除, 然后将参数携带的entries[]中的条目加到当前节点log[]中
            
            2. 如果新的日志条目和已经存在的日志条目不产生冲突, 不要删除prevLogIndex之后的日志, 因为可能是当前节点接收到了过期的同步日志请求
        
        * 应用日志到状态机:
            1. 如果参数携带的leaderCommit > 当前节点的commitIndex, 则设置commitIndex = min(leaderCommit, index of last new entry), 并且将当前节点(lastApplied, commitIndex]区间内的日志应用到状态机中
           
Raft算法的事件和状态转换

一. Raft集群启动时各个节点:
1. 可以获取整个Raft集群的所有节点的连接信息 
2. 进行一系列初始化(currentTerm置为0, votedFor置为-1, 初始状态为Follower)

二. Followers状态
1. 回复来自leaders或者candidates的消息
2. 如果选举时间超过也没接收到**当前leader**发来的AppendEntries消息或者**同意candidate当选**的RequestVote消息, 则转换为Candidate状态

三. Candidates状态
1. currentTerm自增1
2. 给自己投一票（votedFor设置为当前节点）
3. 重置随机定时器
4. 向所有其他节点并行发送RequestVote消息

Candidates会一直保持candidate状态直到下面三种情况之一发生(见paper):
1. 得到了超过半数节点的同意（voteGranted为true）则该节点当选leader, 转换自己的状态到Leader
2. 接收到另一节点发来的AppendEntries消息表明自己是leader, 如果当前candidate节点认为该leader是合法的(消息携带参数中的term > currentTerm), 则将自己转换到Follower状态
3. 定时器超时前没收到大多数投票, 则开始一轮新的选举

Candidate接收到RequestVote的返回结果(依据上一条的论述):
1. 如果返回消息中的term > currentTerm则设置currentTerm = term, 重置随机定时器, 并将自己转换为Follower状态
2. 如果得到了超过半数节点的同意（voteGranted为true）则该节点当选leader, 转换自己的状态到Leader
3. 定时器超时前没收到大多数投票, 则开始一轮新的选举

四. Leaders状态
1. 初始化nextIndex[](全部置为自己最后一条日志的Index + 1)和matchIndex[(全部置为0)
2. 定期发送AppendEntries消息(心跳包)给每个节点, 以此来维持自己的leader地位(发送周期小于随机定时器最小超时时间, 最好超时周期为发送周期的三倍左右)
3. 如果leader节点的最后一条日志索引 >= 某个follower节点的nextIndex(存储在leader的nextIndex[]中), AppendEntries消息中发送的日志条目从leader节点的nextIndex算起

Leader接收到AppendEntries的返回结果:
1. 如果返回消息中的term > currentTerm则设置currentTerm = term, 重置随机定时器, 并将自己转换为Follower状态
2. 如果返回消息中的success为false(由于日志不一致造成)则将该节点在nextIndex[]对应的值减1并重新发送消息
3. 如果返回消息中的success为true, 则更新该节点在的nextIndex和matchIndex值
4. 如果存在一个N > commitIndex, 并且大多数节点的matchIndex值都 >= N, 同时log[N-1].term(这里和paper中不一样的原因和我的实现有关, 实现中N-1是日志在log[]内的位置, 其对应的Index索引为N) = currentTerm, 则更新commitIndex为N
5. 将Leader节点(lastApplied, commitIndex]区间内的日志应用到状态机中

注: 最初实现的时候很容易忽略情况4(见paper中Figure 8的情形), 会实现成一旦接收到大多数Follower的成功反馈, 就更新对应的commitIndex并提交该条AppendEntries所同步的所有日志和此之前的所有日志. 由于Leader只允许提交当前term的日志, 不允许提交之前term的日志, 所以采用情况4的做法来达到正确提交日志到状态机的目的


测试用例详解:
1. TestInitialElection2A
    1) 创建包含3台server的Raft Group(网络为reliable)
    2) 首先检查leader是否被正常选举出来, 并检查每个任期是否只有不超过一个leader
    3) 最后检查在网络不出错的情况下sleep一段时间任期是否发生变化
    
2. TestReElection2A
    1) 创建包含3台server的Raft Group(网络为reliable)
    2) 首先检查leader是否被正常选举出来, 并检查每个任期是否只有不超过一个leader(这里选举出来的leader后面统称为leader1)
    3) 将2中的leader1断开连接, 检查是否有新的leader重新被选举出来(这里选举出来的leader后面统称为leader2)
    4) 重新连接leader1, 并不影响leader2
    5) 断开2台server的连接, 没有leader会被选举出来
    6) 逐一恢复2台server的连接, 每次都有leader被选举出来
    
3. TestBasicAgree2B
    1) 创建包含5台server的Raft Group(网络为reliable)
    2) 三轮迭代, 每轮start一个命令, 并检查命令是否已经正确复制到所有5台server上
    
4. TestFailAgree2B
    1) 创建包含3台server的Raft Group(网络为reliable)
    2) start一个命令, 检查命令是否已经正确复制到所有3台server上
    3) 断开某台server的连接, start一系列命令, 检查命令是否已经正确复制到所有已连接的2台server上
    4) 重新连接之前断开连接的server, start一系列命令, 检查命令是否已经正确复制到所有已连接的3台server上

5. TestFailNoAgree2B
    1) 创建包含5台server的Raft Group(网络为reliable)
    2) start一个命令, 检查命令是否已经正确复制到所有5台server上
    3) 断开其中3台server的连接, start一个命令, 检查命令有没有被复制到其他server上(命令正常来说无法提交, 因为Raft Group当前只有2台已连接server)
    4) 重新连接之前断开连接的3台server, start一个命令, 检查命令是否已经正确复制到所有已连接的5台server上

6. TestConcurrentStarts2B
    1) 创建包含3台server的Raft Group(网络为reliable)
    2) 五轮迭代, 每轮先start一个命令, 然后并发start另外五个命令, 检查所有命令是否已经正确复制到所有server上, 一旦检查通过一次就退出迭代

7. TestRejoin2B
    1) 创建包含3台server的Raft Group(网络为reliable)
    2) start一个命令, 检查命令是否已经正确复制到所有3台server上
    3) 将2中的leader1断开连接, 并在leader1中start三个命令(是无效的, 不会被提交)
    4) start一个命令, 检查命令是否已经正确复制到已连接的2台server上
    5) 将5中的leader2断开连接, 并重新连接2中的leader1
    6) start一个命令, 检查命令是否已经正确复制到已连接的2台server上(这里会出现一种情况: 首次执行Start的server可能是是错误的leader, 因为leader1刚恢复连接肯能还是Leader状态, 但后续会由于term小变为Follower, 然后重新选举出新leader, 这种情况会在一段时间后重新找到正确的leader进行Start操作, 具体见config.go里one函数的实现)
    7) 新连接5中的leader2并start一个命令, 检查命令是否已经正确复制到已连接的3台server上(这里首次执行start也可能会产生上述情况)
    
8. TestBackup2B

    和TestRejoin2B类似, 条件更加harsh
    
9. TestCount2B

    主要测RPC的数量, 不能请求过多
        
10. TestPersist12C

    主要测servers crash后重启能否恢复crash之前的状态, 不持久状态的话恢复后状态都为空, start命令会导致和committed log不一致(因为可能crash之前已经有日志被提交, restart后日志会从Index=1算起)
   
11. TestPersist22C

    和TestPersist12C类似, 条件更加harsh
    
12. TestPersist32C

    和TestPersist12C类似, 条件更加harsh
    
13. TestFigure82C

    模拟paper中Figure8的场景, 其他和TestPersist12C类似
    
14. TestUnreliableAgree2C
    1) 创建包含5台server的Raft Group(网络为unreliable, unreliable主要体现在RPC请求可能会被推迟或拒绝)
    2) 50轮迭代, 每轮都要start几个命令, 检查所有命令是否已经正确复制到server上(这里每轮迭代中的expectedServers都设置为1, 并没要求所有5个server都包含命令, 但实际上我测下来好像最后都是5个server复制成功的样子)
    
15. TestFigure8Unreliable2C

    创建包含5台server的Raft Group(网络为unreliable), 其他测试基本和TestFigure82C一致
    
16. TestReliableChurn2C && TestUnreliableChurn2C

    共用一个函数, 一个网络为reliable一个为unreliable, 反正就是一堆更harsh的操作...
    

ref:
1. [Students' Guide to Raft](https://thesquareplanet.com/blog/students-guide-to-raft/)
2. [Raft Q&A](https://thesquareplanet.com/blog/raft-qa/)
3. [分布式一致性算法-Raft学习笔记](https://blog.csdn.net/wang_wbq/article/details/80376310)