# Introduction

The past 81 days were one of the most rewarding journey for me in studying computer science. I couldn't praise more about MIT OpenCourseWare, and all the teaching staff in MIT 6.824 Distributed Systems, thank you all so much.

In this article, I will present some of the bugs I encountered in each lab and the mistakes I made, I will focus on lab 4 and the more general lessons I have learned from this series of labs. No code according to the lab requirement.

# Part 2 Raft

Raft is the foundation of this series of labs. [The Raft paper](https://pdos.csail.mit.edu/6.824/papers/raft-extended.pdf) and [Student Guide](https://thesquareplanet.com/blog/students-guide-to-raft/) explain the idea thoroughly. If one feels like one has a lot of questions when reading the materials, it is 100% normal, because there is lot of detail only revealing itself in implementation, I guess the most important thing is to keep coding and keep debugging.

1.  Sleep()

        Sleep() after any code that might get executed repeatedly and precipitate chaos in the system e.g. when an election is called by the candidate, it needs to Sleep(), the time is the time estimated to finish the election, otherwise the server will send out many unnecessary VoteRequest RPC.

2.  Never follower->leader

        Before server changing state to leader after winning the election, double check if the current state is candidate because it may have reverted back to follower.

3.  Term

        Always check term and reply with term. Term exchange should be omnipresent.

4.  Disregard stale RPC reply

        Make sure each RPC reply matches current term, if it was sent from previous term, then it should be discarded.

5.  Optimization on log conflict

        I implemented both ConflictIndex&Term and only ConflictIndex. I find the latter easier to understand and to debug, this will benefit greatly in Part 3.

6.  RPC reply is passed by reference

        Be sure to use new reply{} for every RPC otherwise pointers for each server will point back to the same underlying reply{} causing chaos.

7.  votedFor = nil

        In follower's AppendEntryRPC() handler, somewhere you'd probably reset the server's votedFor, don't set it to -1 (nil) because it might cause electing two leaders under the same term, setting it to rf.me will ensure it can start an election but won't electing two leaders under the same term. Scenario:
                S1 was leader(Term 1), it got rebooted.
                S2, S3, S4 enter Term 2, elected S2 (Term 2) as the leader.
                S2 got isolated, but it managed to send RPCs to S3 and S4, so S3 and S4 won't start an election soon.
                Now S3 and S4's votedFor are reset to -1.
                No entry has been appended during this period.
                S1 rebooted and enter Term 2, starts an election and win.
                S2 joins the network, now S1 and S2 are both leaders.

# Part 3

I have lost all my Part 3 notes, what I recall from this lab is that the implementation of the whole structure was rather smooth, thanks to the well-written lab description. However I did remember spending a lot of time on mutex management, resolving deadlocks.

One thing stood out from memory is that at the end I refactored Get() by implementing what is mentioned in Raft paper: 1) the leader must talk to the majority before serving data 2) after being elected the leader commits a nil entry to commit all the previous entries. This greatly simplified things, however it did render some Raft tests impossible to pass, but commenting out some code should do it, the implementation was well worth it.

# Part 4A

My algorithm for re-balancing partitions:

        NShards = avgShardsPerGroup * nGroups + remainder
        y = k * x + b

        The current config is y1, k1, x1, b1
        new config is y2, k2, x2, b2

        There will be (b) groups with (k+1) shards, (x-b) groups with k shards
        e.g. 10 shards, 3 groups
        y  = k * x + b
        10 = 3 * 3 + 1
        1 group with (3+1)=4 shards, (3-1)=2 groups with 3 shards
        b              k+1            x-b   		 k

        Steps:
        1. calculate new y=kx+b
        2. remove shards from old groups, put them into a pool
        3. assign shards from the pool to groups:
                if "join", assign shards to new groups
                if "leave", assign shards to exsiting groups
        4. minimize communication overhead when assigning shards, for example if a group currently has 1 shard, and it should either have 2 or 3 shards in the new config, pick 3, transfer as much data as possible in a single round of RPCs.

# Part 4B Sharded Key/Value Server

        Step 1: kvserver probes a new config from the shardmaster

The probing is straightforward: kvserver has a goroutine that periodically calls the master clerk to fetch the latest config.

1.  kvservers responds to RPC before it has probed the master for nextConfig at all

        This is bad because the nextConfig is an important token for kvservers to recognize if each other is up to date.

        I added a field in ShardKV, kv.probed. kvservers only respond if kv.probed is true.

2.  Rapid change of configs

        Instead of waiting for one config to finish applying before the system could applying the next config, which is quite a waste of time, it is better to add an aborting mechanism in the system.

        The configProber periodically probes for new config, stores it in a field, kv.nextConfig. At every step of converging towards a new config, keep checking whether the config being applied is the same one as kv.nextConfig, if not, abort.

---

        Step 2: kvserver converges towards the new config regardless of its current state

When does a server stop serving a shard? When does the next server start serving the shard? Are transfers actively initiated by the current keeper or are they requested by the next keeper then got sent over?

It is hard to answer all these questions at the beginning of the lab all at once and expected to come up with a perfect design. What is more logical is to have a coarse design then implement, then improve, AKA heuristics.

After many iterations of design, the final design I came up with follows:

        1. G1 detects a new config
        2. G1 finds the missing shards for the new config
        3. G1 sends out RequestShardRPC to all the other groups with its missing shards attached
        4. Receiver of RequestShardRPC, G2, replies with all the shards it has and the shards it will send to the sender, also G2 stops serving the shards it will send
        5. G2 sends out TransferDataRPC to the G1 to transfer data
        6. G1 waits for all data needed for the new config to come in, then it submits the config change through Raft and start serving the new shards
        7. G1 sends out DeleteShardRPC to all the other groups
        8. G2 deletes data

Just like any design, the devil is in the detail.

1.  why do receivers of RequestShardRPCs need to reply all the shards it has?

        This is a way to detect unclaimed shards. When the system just got started, all shards are unclaimed, thus it is safe for a server not to wait for data transfer about unclaimed shards.

        I tried a different approach to solve the unclaimed shards problem using master to maintain a 'pool' that holds all unclaimed shards, but it was more complicated to build a fault-tolerant system involving the master.

2.  LastServingConfigNum

        Along with all the shards the receivers of RequestShardRPCs have, receivers also attach a field in the RequestShardRPC reply: LastServingConfigNum - what the last config number the shard served was. This is used to distinguish which server's data is more up to date when a conflict occurs, and conflicts of shards do occur often.

3.  Why not just transfer data in the reply of RequestShardRPC instead of having a new TransferDataRPC?

        Actively sending data is necessary for fault-tolerance. The sender has better control against uncertainties such as unreliable network, change of the receiver's leadership etc.

4.  Why is there a separate DeleteDataRPC instead of deleting after the data sender receives confirmation from the receiver?

        The latter approach assumes the receiver of data will apply data and start serving the shards immediately upon receipt. However this is ideal, frequent shard conflict — that is multiple groups claim to have a same shard — demands a mechanism of resolution, thus requiring a staging area — receivers of data need to collect the information from all groups: which group has what shard and what data, whose data is more up to date. Only when this process completes, the config change is finished then DeleteDataRPCs will be broadcasted.

        There are two mechanisms to ensure data safety and no shard is being served by two groups at the same time:
        1) DeleteDataRPCs are broadcasted by the receiver of data transfer only when the new data has been applied and the shards are being served. The previous keeper of shards stopped serving when it received the RequestShardRPC.
        2) In the reply of RequestShardRPC, every group responds with the LastServingConfigNum of each shard, and the sender of RequestShardRPC will do a check only to keep the up-to-date ones, it initiates a self-delete process if other groups have more up-up-date data.

5.  State of a shard

        1. Unclaimed: when all servers just started, no shard is assigned to any group
        2. Serving: the shard is being served by a group
        3. HavingNotServing: the shard and its data are held by one group but it is not serving the shard, usually resulted from receiving RequestShardRPCs. The data are not necessarily up-to-date.

6.  Abort change

        In order to be agile in the config change process, constantly check if the config waiting to be applied is the same as the newest config probed from the shardmaster. Once a newer config is detected, servers should abandon the current change and start over for the newest config.

7.  Raft's ready state

        In lab 3 it was recommended to submit Get() requests through Raft to ensure the server is ready to serve up-to-date data. However I found this approach counterproductive in lab 4. After implementing what is recommended on the Raft paper, 1) Only serve after the leader has talked to the majority 2) Commit a nil command after a new leader is elected, I personally feel it has greatly simplified the design workload.

        The implementation guarantees the leading server is always up to date (all entries are applied, definitely not isolated), so judgement about whose shard is more up to date made by the kvserver is always based on fresh information.

8.  Proof of 'up-to-dateness'

        'Term', 'lastLogIndex' and 'lastLogTerm' in Raft design are clever authentication tokens for servers to distinguish stale or fresh RPCs. In this sharded kvserver design I intuitively picked ConfigNum as the authentication token. However depending on your design, this may not be sufficient.

9.  all-to-all communication and relevant groups

        My initial design included targeted communication — that is each group figures out which group currently holds which shards and only send out RPCs to the necessary groups — this significantly lowers network traffic. However uncertainties make this 'figuring out' process super complicated: some groups may have abandoned the last config change; some groups may have not deleted data due to unreliable network etc. Thus all-to-all communication was embraced.

        Even all-to-all communication was chosen, it doesn't make sense to send RPC to group 105 when only group 100-102 are involved in the last few configurations, only relevant groups are sufficient. Relevant groups refer to all groups involved from the last completed config to the latest config inclusively. A completed config means all the groups involved in this config are serving the shards assigned to them, they are the last sure keeper of shards, from there as groups join/leave, shards get passed around.

        This is implemented by having each group report to the shardmaster that a config has been applied in this group, and the shardmaster concludes that a config is completed when all groups involved have reported so.

10. Tester's shutdown & restart and killing goroutines

        This bug took me a while to notice. The test setup shuts down servers by isolation, not by real 'kill', so StartServer() is called again when restarting a server, whatever long-existing goroutines spawned by StartServer() will have duplicates e.g. configProber(), there will be two of them probing the shardmaster for new config and causing unexpected race condition. Adding a 'dead' field and modifying Kill() solved this problem.

11. Duplicate check

        Whatever is used to check duplicated commands, it is important to transfer the necessary data (clerkID and its OpID) to the new keeper of shards along with storage data.

12. All data change must go through Raft

        As the linchpin of this series of labs, underlying Raft servers provide fault-tolerance, that means all data change must go through Raft e.g. update storage data, start/stop serving shards, update clerk's opID, delete data...literally any data change.

        I personally feel one of the most important lessons lab 4 has taught me is that it instilled a way of thinking that data change is only fault-tolerant when it has passed the underlying consensus module.

# More General Lessons Learned

1.  Trial and error

        As an inexperienced programmer, especially in distributed system design, I think there is a fine balance between how much time you should spend on designing and how soon you should start implementing. Insufficient design results in disorientation, too much detail beforehand builds up uncertainty. The fine line is drawn on how much you know about the subject: only design detail that you have internalized, leave ambiguity to exploration while implementing.

        What is even more important is not to hesitate to give up a draft when you clearly know you are in a dead end or a rabbit hole. Design - code - redesign - code, so on so forth.

2.  Write logs

        Write down everything you have thought about, this helps you DRASTICALLY in every possible way, I promise.

        Do not start writing code right away trying to fix a bug unless the bug is trivially simple. Any bug that you might say to yourself 'I need to think about it', you should write down the reasoning. Not only this helps you to solve the problem but it's also great for backtracking, which is significant when you come back to retrace why you did what you did. We don't necessarily remember the code we wrote, but we should be able to track the reasoning.

3.  Minimize non-deterministic code

        For example, don't use time.Sleep() as a way to guess how long some code may finish then some other code can then execute. Not only this puts a fixed overhead on performance, but also you can never be sure where the threshold will be even after a thousand tests. Using deterministic approach such as mutex, conditional variable, channel and so on.

4.  Test MANY MANY MANY times

        Distributed systems have an non-deterministic nature, passing a test 10 times is not merely enough, test it as many times as necessary to give you a level of confidence that you can call it 'business/production ready'. Every failed test will reveal a minor bug, it is a lengthy yet satisfying rigmarole towards perfection. I felt somewhat safe after all tests passed 400+ times.

5.  Never rush forward

        Don't move on if you know there is a bug in your code but you are too tired to fix it because you have already fixed so many bugs. Debugging will become increasingly harder if you add more untested code, it will slow you down in the long run.

6.  Use shell script to facilitate tests

        Shell script helps a lot in running tests hundreds of times.

7.  Keep print messages clean and use basic Unix commands

        For each major method, the printing message should be identifiable e.g. "Raft 1-2-AppendEntries:" is Raft 1 in Term 2 in AppendEntriesRPC handler. This makes debugging exponentially easier.

        Also, learning basic unix commands such as grep will make debugging much smoother too. MIT's Missing Semester has a lecture about Data Wrangling explaining this thoroughly https://missing.csail.mit.edu/2020/data-wrangling/

8.  The test setup

        Finally, I believe it is important to acknowledge that without a proper test environment, this series of labs won't be near as good as it is now. Test-Driven Development (TDD) was a term I have heard many times before but only deeply appreciated after I have finished this series of labs.

        How the test environment is set up in the labs is simply amazing, it is the fruit of years of hard work by many MIT staff. It would be amiss not to study it. After all it is up to oneself to set up tests for one's own project in the future.

        Each test creates a config, the config consists of a tester, a network, all the servers and all the necessary states of statistics.

        The network simulates RPC through the following steps:

        1. use gob to encode and decode args and reply
        2. add randomness to simulate real network
        3. call the exported method, return the reply

        Each config in each lab is tailored with the functions to do necessary checks.
