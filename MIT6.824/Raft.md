---
sort: 3
---

# Raft

This part is all based on the paper [Raft](https://pdos.csail.mit.edu/6.824/papers/raft-extended.pdf), which is absolutely worth carefully reading.

## What is Raft

Raft is a consensus algorithm for managing a replicated log. Compared with PAXIOS, it is much easier to understand, and also provides powerful functions to support consistency of managed logs, as well as fault tolerance.

## Roles in Raft

Raft is implemented on several distributed severs to provide functions. There are three kinds of roles defined in Raft:

1. **Leader**: it is used to manage logs on itself as well as on all other servers. Services would communicate with it to send logs and apply states. Logs of the Raft leader are all valid, and conflicts on other servers, if any, are invalid. Besides communicate with services, it also communicates with other servers to synchronize its logs. The final goal is to reach an agreement: all the logs on different servers are the same, and the states applied on the state machine are the same, which is called *committed* logs.
2. **Follower**: it is mainly used as replicate of leader. Imagine that: if there were only one server, the leader, logs and states could also be valid and recorded correctly without failures beyond exceptions. To avoid this, replicates are very useful and necessary, because we need the availability. Followers all keep a replicate of logs on leader locally, and once the leader crashes, one of the followers would become next leader.
3. **Candidate**: just as described above, once the leader crashes, a follower would become the next leader, but not any of followers could become leader. A follower wishing to become leader shall become candidate firstly, admitted by the majority of all the servers then, and finally it could become the leader. The middle status of follower and leader is called candidate.

The conversion rule between the three roles is simple: follower could convert to candidate, and candidate could also convert back to follower; candidate could convert to leader, but leader could only convert to follower.

What's more, we can see that Raft is not a purely distributed consistency algorithm, because it still needs a leader. But on the other hand, this leader is not selected manually on purpose, but  selected from a group of similar servers. It is kind of centric, but not totally.

## Rules for Vote

### Introduction

The conversion from follower to leader has rule, which is called **vote**.

The background is that the leader would always send heartbeat message to all the followers regularly, but if the follower does not hear from the leader for a while, it believes the leader is dead, and a new leader must be elected. At this time, this follower would convert to **candidate**. Notice: this conversion is not driven by someone external, but the follower itself, with the help of a timer inside.

The new candidate then sends vote request to all other servers. Other servers, after receiving the vote request, must determine whether it should grant vote to it. So there requires a detailed rule. This rule must guarantee:

1.  The candidate's logs should be as new and complete as the receiver's
2. The receiver has not voted for other candidates after the former leader crashed

In this way, we introduces a concept named **term**, to indicate the which leader hosts; and the **index** of log is also recorded to show how new log the server contains. By following the consistency principle, we could determine a log by term and index, and the first critical rule is:

**A log is identified by its term and index**

This means by confirming a pair of term and index, we can identify a certain single log, and no other possibilities. In this way, we could let the candidate send vote request with these messages, and receivers would compare it with its own. To be more specific, the last log of its record, this is because of the second critical rule:

**All logs are ordered by its index in ascending order**

This is guaranteed by the fact that each time the leader receives a new log, it appends the log to its record with index incremented, and because of consistency, the index increments regularly, even on different leaders in different time. Because each leader would guarantee the rule from its start point, and former logs are also guaranteed by former leaders.

If the last log of the receiver is newer than the candidate according to the message received, it proves that the receiver contains logs that do not exist on candidate, and the candidate should not be leader in fact: we should let the follower with newest, which is equal to the longest on logs' length, logs become the leader.

Another point is to make it apart from former vote, because the receiver of course voted for someone in last term. In this way, the candidate should increase its current term, and includes this information in the vote request, and receivers would take it into consideration. The receiver would not vote for any others if it has voted for someone. This is called *first-come-first-serve* principle.

The vote request and respond could be like:

```go
// vote request
{
    Term int			// candidate’s term
    CandidateId int		// candidate requesting vote
    LastLogIndex int	// index of candidate’s last log entry
    LastLogTerm int		// term of candidate’s last log entry
}

// vote response
{
    Term int 			// currentTerm, for candidate to update itself
    voteGranted bool 	// true means candidate received vote
}
```

rules:

> 1. Reply false if term < currentTerm
> 2.  If votedFor is null or candidateId, and candidate’s log is at least as up-to-date as receiver’s log, grant vote

There are some variables saved in server's state, and you could refer to the paper for more details.

### Lab 2A

Lab 2A is based on vote, and the goal is to implement Raft's vote mechanism. It must work correctly in different scenarios, including initial setup, leader crashes, network failures and so on.

As I have described above, the initial setup could be easy. But to avoid split vote, which means a lot of followers become candidates and start up votes at the same time, and it may not make a candidate win a majority of servers to elect a leader, the expiration time, which means the threshold of not receiving heartbeat message from leader, should be randomized. And then, it reduces the possibility of followers staring up votes simultaneously.

By comparing the term of  vote request with their own, receivers could know whether this request is out of date or a valid one. This also applies to later other RPCs.

Once a leader has been elected, it should send heartbeat message to all other servers to avoid another election starting. For the heartbeat message, the same check would be applied, and expired heartbeat message should not reset the timer of election timeout. If the message received contains a term newer than receiver's own, the receiver would update its current term. 

In this way, as we can see, this becomes a loop: with the help of vote rules, each time the leader absent, a new leader with the most complete logs would be elected and work as its former, as long as there are more than a half of servers are still alive, for example, 3 of 5.

These rules also guarantee that each leader must be in different terms in an ascending order, though inconsistent perhaps. Because in one term, all servers would be synchronized to the same term. Even some servers' term fails to be updated to the newest term, in the next round of election, the term contained in vote request would only allows those candidates having the newest term to be elected. As for the lagging servers, even if they start an election, their term is out of date and index lagging than others' in fact, and it would be impossible for them to be elected as the leader. So that Raft's term would never roll back but increase forever.

## Rules for Log replication

The replication process of logs is the most important part in Raft, and the most complicated part. The definition of this process looks smooth and reasonable, but it becomes pretty hard for implementation, because a lot of corner cases must be taken into consideration.

In this part, I would go through from the very start status, also the most ideal status.

### Ideal status

Leader elected after startup send log append RPC to followers successfully without any delay. In this way, the replication on followers could always be synchronized with leader. In this scenario, everything works smoothly.

### If leader crashes

Leader could crash in the middle of synchronization, and in this scenario, a new leader should be elected immediately. And because of all followers' logs are the same as former leader's, nothing else is needed.

### If delay exists

If there are some delays between leader and followers, things could be complicated. As we discussed above, only followers with newest log could be elected as new leader. And even the new leader has been elected, it still needs to figure out each followers' status, like which log should be sent to them separately. 

To apply this, each leader should maintain two data structures: an array of next index of log of each follower, and an array of index of log that could confirmed being replicated on each follower. When initialized, the records of nextIndex array should all be assigned as leader's commit index + 1, and lastApplied should be all set as 0. This difference indicates that the records in nextIndex are only estimations of followers' status, or am eager guess; and for records in lastApplied, they are solid and reliable. These records should both be updated by later communication between leader and followers.

The follower should also check the validation of each log it receives, there are some steps:

1. Locate the index used to mark start point. The leader would send two variables: preLogTerm and preLogIndex, to record the term and index of the start log's preceding log.
2. After finding out the preceding log by index, the follower should check whether the term of this log is the same as preLogTerm. Because one log could be identified by term and index, once the two variables agree, the follower could confirm the log needing appended is valid and should be accepted. But if the check fails, it should be rejected.
3. The logs are then appended after the preceding log.

### If quick synchronization required

We could also implement the feature that leader could send a batch of logs to followers for replication. And in this way, followers should go though the log array received, and check each log's validation. But be careful to avoid out of index error when implementing.

### If network delay exists

A common delay is just lagging of leader's status, but network delay is a real headache. Imagine that: some RPCs are sent in an order but received in a different order. If the follower does not check, a disaster would be caused finally, and it must be a total chaos.

#### For Follower:

##### When receiving request in an earlier term

The follower should refuse the request directly with its current term assigned respond.

##### When receiving request in a later term

The follower should update its own term as new as the request's, and the check for the log append is still necessary.

##### When receiving request with conflict preceding log information

The follower should delete logs from the conflict point.

##### When receiving request with existing logs

Sometimes followers could receive requests containing logs already existing in their records, and their job is to check these log, if conflicts happen, they should truncate their own records from the conflicting point.

#### For leader:

##### When receiving respond with a later term

This indicates the leader is out of date and it should immediately covert back to a follower.

##### When receiving respond with an earlier term

This indicates the respond is out of date, and the leader should ignore it and never update its records according to the respond. In fact, to guarantee leader's safety, the leader should **confirm the terms of the request it sent, the respond it received, and its own current term are the same, and only then should it accepts the respond as a valid one**.

##### When receiving respond with success flag

The follower then should update its records of this follower sent the respond according to the message. For example, it should update the nextIndex and lastApplied for the follower, and currently, the leader just set the nextIndex value as the last log's index it sent to the follower in this communication plus 1, and lastApplied value as the same as this last log's index.

### Optimization

There are some optimizations possible for Raft, and one has been mentioned above, that is to send multi logs in a single RPC, instead of one by one. Another is used when conflict happens, the follower should not just reply with a false flag, but it should provide the conflict log's term, as well as the first index of this very **term** it stores. In this way, by searching the term in its own log records, the leader could find out the preceding log of this whole term's logs, and use its information as preLogIndex and preLogTerm, and send logs following it the next time. In this way, the leader could bypass a whole term in one single communication, instead of one index per communication. But once the leader fails to find logs with this term, it should set the nextIndex value of this follower as conflict index directly.

## Rules for Commit

Once a log has been applied to the state machine, it is called *committed*, and it should be determined by the leader. The leader does it by confirming this very log has been replicated on a majority of servers (including itself), and this is to avoid losing committed logs. Once a log has been replicated on a majority of servers, it would never be lost unless the whole Raft system crashes.

When the leader has determined the log should be applied and committed, it increase its commitIndex variable to the index of the log. In practice, it is equal to plus 1 to it. For followers, they should check the commitIndex in log append RPC it received from leader, and if this value is larger than its own, the follower knows it should commit this log, too.

But it is possible that the follower does not contain the log indicated by the leader's commitIndex, then the follower should compare the commitIndex from leader, and the index of the last log appended in this RPC. The follower would choose the smaller one as the target it should commit itself to.

## Lab 2B & Lab 2C

In the two labs, our job is to implement a fully functional Raft with fault-tolerance ability, mainly I described above. The key points I have also discussed above, but during the process of implementation,  I did not figure it out so well, and I had to struggle with a lot of wired errors and bugs. To be honest, however, some bugs looked wired at first glance, but by analyzing the detailed log with code logic, finally I could find out why this happened and how to solve them. In this way, a detailed and well-formatted logs are very important and necessary, and I organized it like this:

```bash
[server 1, term=1, commit idnex=1] something described heer
```

This helped me a lot when testing.

Another lesson is that we must check the validation of requests and responds both. At first, I only checked when receiving requests, but not for responds, and finally outdated replies ruin all the system.

Again, the checking rule:

**request's term = respond's term = server's current term**

This could be another critical rule.

A big challenge of this series labs is the random failure when testing. Because there are a bunch of random-based tests, your code could pass in one round bur fail in another, and multi tests are required to test the correctness of your code. In this way, I wrote a bash script to help this process. It could run tests in parallel by running multi tasks at the same time, and it could record how many times your code fails in these tests, and persist the printed log of this round.