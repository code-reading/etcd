## Progress (复制进度)

Progress represents a follower’s progress in the view of the leader. Leader maintains progresses of all followers, and sends `replication message` to the follower based on its progress. 

> leader会存储所有 follower 对自身 log 数据的 progress（复制进度），leader 根据每个 follower 的 progress向其发送 replication message。

`replication message` is a `msgApp` with log entries.

> replication message 是 msgApp 外加上 log 数据。

A progress has two attribute: `match` and `next`. `match` is the index of the highest known matched entry. If leader knows nothing about follower’s replication status, `match` is set to zero. `next` is the index of the first entry that will be replicated to the follower. Leader puts entries from `next` to its latest one in next `replication message`.

> progress 有两个比较重要的属性：match 和 next。match 是 leader 知道的 follower 对自身数据的最新复制进度【或者说就是 follower 最新的 log entry sent index】，如果 leader 对 follower 的复制进度一无所知则这个值为 0。next 则是将要发送给 follower 的下一个 log entry sent 的序号。

A progress is in one of the three state: `probe`, `replicate`, `snapshot`. 

> progress 有三个状态：probe，replicate 和 snapshot。

```
                            +--------------------------------------------------------+          
                            |                  send snapshot                         |          
                            |                                                        |          
                  +---------+----------+                                  +----------v---------+
              +--->       probe        |                                  |      snapshot      |
              |   |  max inflight = 1  <----------------------------------+  max inflight = 0  |
              |   +---------+----------+                                  +--------------------+
              |             |            1. snapshot success                                    
              |             |               (next=snapshot.index + 1)                           
              |             |            2. snapshot failure                                    
              |             |               (no change)                                         
              |             |            3. receives msgAppResp(rej=false&&index>lastsnap.index)
              |             |               (match=m.index,next=match+1)                        
receives msgAppResp(rej=true)                                                                   
(next=match+1)|             |                                                                   
              |             |                                                                   
              |             |                                                                   
              |             |   receives msgAppResp(rej=false&&index>match)                     
              |             |   (match=m.index,next=match+1)                                    
              |             |                                                                   
              |             |                                                                   
              |             |                                                                   
              |   +---------v----------+                                                        
              |   |     replicate      |                                                        
              +---+  max inflight = n  |                                                        
                  +--------------------+                                                        
```

When the progress of a follower is in `probe` state, leader sends at most one `replication message` per heartbeat interval. The leader sends `replication message` slowly and probing the actual progress of the follower. A `msgHeartbeatResp` or a `msgAppResp` with reject might trigger the sending of the next `replication message`.

> 如果 follower 处于 probe 状态，则 leader 每个心跳包最多只发送一个 replication message。leader 会缓慢发送 replication message 并探测 follower 的处理速度。leader 收到 msgHeartbeatResp 或者收到 msgAppResp（其中 reject 值为 true）时，leader 会发送下 一个 replication message。

When the progress of a follower is in `replicate` state, leader sends `replication message`, then optimistically increases `next` to the latest entry sent. This is an optimized state for fast replicating log entries to the follower.

> 当 follower 处于 replicate 状态时，leader 会一次尽量多地把批量 replication message 发送给 follower，并把 next 取值为当前 log entry sent 的最大值，以让 follower 尽可能快地跟上 leader 的最新数据。

When the progress of a follower is in `snapshot` state, leader stops sending any `replication message`.

> 当 follower 处于 snapshot 状态时候，leader 不再发送 replication message 给 follower。

A newly elected leader sets the progresses of all the followers to `probe` state with `match` = 0 and `next` = last index. The leader slowly (at most once per heartbeat) sends `replication message` to the follower and probes its progress.

> 新当选的 leader 会把所有 follower 的 state 置为 probe，把 match 置为0，把 next 置为自身 log entry set 的最大值。

A progress changes to `replicate` when the follower replies with a non-rejection `msgAppResp`, which implies that it has matched the index sent. At this point, leader starts to stream log entries to the follower fast. The progress will fall back to `probe` when the follower replies a rejection `msgAppResp` or the link layer reports the follower is unreachable. We aggressively reset `next` to `match`+1 since if we receive any `msgAppResp` soon, both `match` and `next` will increase directly to the `index` in `msgAppResp`. (We might end up with sending some duplicate entries when aggressively reset `next` too low.  see open question)

> 当 follower 给 leader 的 msgAppResp 的 reject 为 false 的时候，它会被置为 replicate 状态，reject 为 false 就意味着 follower 能够跟上 leader 的发送速度。leader 会启动 stream 方式向以求最快的方式向 follower 发送 replication message。当 follower 与 leader 之间的连接断连或者 follower 给 leader 回复的 msgAppResp 的 reject 为 true 时，就会被重新置为 probe 状态，leader 当然也会把 next 置为 match+1。

A progress changes from `probe` to `snapshot` when the follower falls very far behind and requires a snapshot. After sending `msgSnap`, the leader waits until the success, failure or abortion of the previous snapshot sent. The progress will go back to `probe` after the sending result is applied.

> 当 follower 的 log entry set 与 leader 的 log entry sent 相差甚巨的时候，leader 会把 follower 的状态置为 snapshot，然后以 msgSnap 请求方式向其发送 snapshot 数据，发送完后 leader 就等待 follower 直到超时或者成功或者失败或者连接中断。当 follower 接收完毕 snapshot 数据后，就会回到 probe 状态。

### Flow Control

> leader 与 follower 之间进行数据同步的时候，可以通过下面两个步骤进行流量控制

1. limit the max size of message sent per message. Max should be configurable.
Lower the cost at probing state as we limit the size per message; lower the penalty when aggressively decreased to a too low `next`

> 1.限制 message 的 max size。这个值是可以通过相关参数进行限定的，限定后可以降低探测 follower 接收速度的成本；

2. limit the # of in flight messages < N when in `replicate` state. N should be configurable. Most implementation will have a sending buffer on top of its actual network transport layer (not blocking raft node). We want to make sure raft does not overflow that buffer, which can cause message dropping and triggering a bunch of unnecessary resending repeatedly. 

> 2.当 follower 处于 replicate 状态时候，限定每次批量发送消息的数目。leader 在网络层之上有一个发送 buffer，通过类似于 tcp 的发送窗口的算法动态调整 buffer 的大小，以防止 leader 由于发包过快导致 follower 大量地丢包，提高发送成功率。


> snapshot，故名思议，是某个时间节点上系统状态的一个快照，保存的是此刻系统状态数据，以便于让用户可以恢复到系统任意时刻的状态。 etcd-raft 中的 snapshot 代表了应用的状态数据，而执行 snapshot 的动作也就是将应用状态数据持久化存储，这样，在该 snapshot 之前的所有日志便成为无效数据，可以删除。
