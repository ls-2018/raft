- 假设：我们现在有[A,B,C,D,E,F,G] 7个节点,全部都是Voter,并且按照顺序存储到了`r.configurations.latest.Servers`
- 前提：集群刚刚搭建 
  - 因此数据库中并没有存储 `keyLastVoteTerm` 值。

如果现在重新发生了选举,以7号节点为例:
`raft.go:1736`
- 1、会先对每一个非本机节点发送`rpcRequestVote`请求 
- 2、并且会将最新的候选任期写入到`bolt`
```go
for _, server := range r.configurations.latest.Servers {
    if server.Suffrage == Voter {
        if server.ID == r.localID {
            // Persist a vote for ourselves
            if err := r.persistVote(req.Term, req.Candidate); err != nil {
                r.logger.Error("failed to persist vote", "error", err)
                return nil
            }
            // Include our own vote
            respCh <- &voteResult{
                RequestVoteResponse: RequestVoteResponse{
                    RPCHeader: r.getRPCHeader(),
                    Term:      req.Term,
                    Granted:   true,
                },
                voterID: r.localID,
            }
        } else {
            askPeer(server)
        }
    }
}

```

远端节点
```go
func (r *Raft) requestVote(rpc RPC, req *RequestVoteRequest) {
	...
	lastVoteTerm, err := r.stable.GetUint64(keyLastVoteTerm)
	if err != nil && err.Error() != "not found" {
		r.logger.Error("failed to get last vote term", "error", err)
		return
	}
	...
```
初始时，所有节点的任期都是一致的且都没有写入数据库
```
那么这里的 lastVoteTerm err!=nil
就会直接返回了

实则不然 因为 err.Error() == "not found" 会继续执行
如果忽略这一条，继续按照现在我的思路往下走
```
方案1 ：当次申请，因为所有节点都没有`keyLastVoteTerm`会失败，但是 最后一次 会将`keyLastVoteTerm`写入本地
        那么，当所有的node都执行够一次之后，那么就可以正常投票了

方案2 ：忘记了