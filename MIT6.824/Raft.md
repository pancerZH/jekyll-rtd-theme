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

## Rules of Vote

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