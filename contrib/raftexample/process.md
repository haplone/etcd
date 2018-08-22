
main()                                  main.go
  |-make(chan string)                   新建proposeC管道，用来将用户层数据发送给RAFT协议层
  |-make(chan raftpb.ConfChange)        新建confChangeC管道，用来将配置修改信息发送给RAFT协议层
  |-newRaftNode()                       raft.go 返回结构体会作为底层RAFT协议与上层应用的中间结合体
  | |                                       同时会返回commitC errorC管道，用来接收请求和错误信息
  | |-raftNode()                        <<<1>>>新建raftNode对象，重点proposeC
  | |-raftNode.startRaft()              [协程] 启动示例中的代码
  |   |-os.Mkdir()                      如果snapshot目录不存在则创建
  |   |-snap.New()                      snap/snapshotter.go 只是实例化一个对象，并设置其中的dir成员
  |   |-wal.Exist()                     wal/util.go 判断WAL日志目录是否存在，用来确定是否是第一次启动
  |   |-raftNode.replayWAL()            raft.go 开始读取WAL日志，并赋值到raftNode.wal中
  |   | |-raftNode.loadSnapshot()
  |   | | |-snap.Snapshotter.Load()     snap/snapshotter.go 中的Load()函数
  |   | |   |-Snapshotter.snapNames()   会打开目录并遍历目录下的文件，按照文件名进行排序
  |   | |   |-Snapshotter.loadSnap()
  |   | |     |-Snapshotter.Read()      会读取文件并计算CRC校验值，判断文件是否合法
  |   | |-raftNode.openWAL()            打开snapshot，如果不存在则创建目录，否则加载snapshot中保存的值并尝试加载
  |   | | |-wal.Open()                  wal/wal.go 打开指定snap位置的WAL日志，注意snap需要持久化到WAL中才可以
  |   | |   |-wal.openAtIndex()         打开某个snapshot处的日志，并读取之后
  |   | |     |-readWalNames()          wal/util.go读取日志目录下的文件，会检查命名格式
  |   | |     |-searchIndex()           查找指定的index序号
  |   | |-ReadAll()                     读取所有日志，真正开始读取WAL
  |   | |-raft.NewMemoryStorage()       使用ETCD中的内存存储
  |   | |-raft.NewMemoryStorage()       raft/storage.go 新建内存存储
  |   | |--->>>                         从这里开始的三步操作是文档中启动节点前要求的
  |   | |-MemoryStorage.ApplySnapshot() 这里实际上只更新snapshot和新建ents成员，并未做其它操作
  |   | |-MemoryStorage.SetHartState()  更新hardState成员
  |   | |-MemoryStorage.Append()        添加到ents中
  |   | |-raftNode.lastIndex            更新成员变量
  |   |
  |   |-raft.Config{}                   raft/raft.go 构建RAFT核心的配置项，详细可以查看源码中的定义
  |   |-raft.RestartNode()              raft/node.go 如果已经安装过WAL则直接重启Node，这最常见场景
  |   |  |-raft.newRaft()               raft/raft.go
  |   |  | |-raft.becomeFollower()      启动后默认先成为follower 【became follower at term】
  |   |  | |                            返回新建对象 【newRaft】
  |   |  |-newNode()                    raft/node.go 这里只是实例化一个node对象，并未做实际初始化操作
  |   |  |-node.run()                   启动一个后台协程开始运行
  |   |
  |   |-raft.StartNode()                第一次安装，则重新部署
  |   |
  |   |-rafthttp.Transport{}            传输层的配置参数
  |   |-transport.Start()               rafthttp/transport.go 启动HTTP服务
  |   |  |-newStreamRoundTripper()      如下的实现是对http库的封装，保存在pkg/transport目录下
  |   |  | |-NewTimeoutTransport()
  |   |  |   |-NewTransport()
  |   |  |     |-http.Transport{}       调用http库创建实例
  |   |  |-NewRoundTripper()
  |   |-transport.AddPeer()             rafthttp/transport.go 添加对端服务，如果是三个节点，会添加两个
  |   | |-startPeer()                   rafthttp/peer.go 【starting peer】
  |   | | |-pipeline.start()            rafthttp/pipeline.go
  |   | | | |-pipeline.handle()         这里会启动一个协程处理
  |   | | |--->                        【started HTTP pipelining with peer】
  |   | | |-peer{}                      新建对象
  |   | | | |-startStreamWriter()       会启动两个streamWriter
  |   | | |   |-streamWriter.run()      启动协程处理 【started streaming with peer (writer)】
  |   | | |     |  <<<cw.connc>>>
  |   | | |     |-cw.status.active()    与对端已经建立链接【peer 1 became active】
  |   | | |     |--->                  【established a TCP streaming connection with peer (... writer)】
  |   | | |-streamReader.start()        这里会启动msgAppV2Reader、msgAppReader两个streamReader读取
  |   | |   |-streamReader.run()        启动协程处理，这里是一个循环处理 【started streaming with peer (... reader)】
  |   | |--->                          【started peer】
  |   |
  |   |-raftNode.serveRaft()            [协程] 主要是启动网络监听
  |   |-raftNode.serveChannels()        [协程] raft.go 真正的业务处理，在协程中监听用户请求、配置等命令
  |   | |-raftStorage.Snapshot()        获取snapshot
  |   | |-raft.Node.Propose()           阻塞等待该用户请求被RAFT状态机接受
  |   | |-raft.Node.ProposeConfChange() 发送配置请求
  |   |
  |   |-raft.Node.Tick()                另外的协程处理RAFT组件的同步信息
  |
  |-newKVStore()                        kvstore.go 创建内存KV存储结构
  | |-kvstore{}                         实例化KVStore存储对象
  | |-kvstore.readCommits()             会启动一个协程，也是存储的核心，用于读取已经提交的数据
  |   |                                    这里实际上调用了两次，第一次是函数内调用，第二次是协程
  |   |-snapshot.Load()                 第一次commitC中没有数据，实际上是加载snapshot
  |   |-recoverFromSnapshot()           从snapshot中恢复
  |   |                                 接下来是协程的处理
  |   |-gob.NewDecoder()                反序列化
  |   |-kvStore[]                       保存到内存中
  |
  |-serveHttpKVAPI()                    启动对外提供服务的HTTP端口
    |-srv.ListenAndServe()              真正启动客户端的监听服务

一般是定时器超时
raft.Step()
 | <<<pb.MsgHup>>>
 |- 【is starting a new election at term】
 |-raft.campaign()
   |-raft.becomeCandidate() 进入到选举状态，也可以是PreCandidate
   |-raft.poll() 首先模拟收到消息给自己投票
   |-raft.quorum() 因为集群可能是单个节点，这里会检查是否满足条件，如果是
   | |-raft.becomeLeader() 如果满足则成为主
   |-raft.send() 发送选举请求，消息类型可以是MsgPreVote或者MsgVote 【sent MsgVote request】

raft.stepCandidate()
 |-raft.poll() 【received MsgVoteResp from】
 | |-raft.becomeLeader() 如果满足多数派
 | | |-raft.appendEntry() 添加一个空日志，记录成为主的事件
 | | |---> 【became leader at term】
 | |-raft.bcastAppend() 广播发送
 |   |-raft.sendAppend()
 |---> 【has received 2 MsgVoteResp votes and 0 vote rejections】

node.run()
 |---> 【raft.node ... elected leader at term ...】


 [ETCD 示例源码](!https://jin-yang.github.io/post/golang-raft-etcd-example-sourcode-details.html)