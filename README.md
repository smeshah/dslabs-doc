# dslabs-doc

## 1. Overview

This README explores the integration of coroutines and RPCs within a Raft consensus algorithm implementation using the labs framework.  Coroutines provide a structured approach to managing asynchronous operations like network communication and leader election, enhancing code readability and maintainability. RPCs form the backbone of communication within the distributed Raft cluster, enabling nodes to send requests and receive responses efficiently. This document outlines how these concepts are leveraged within the labs framework and provides insights into their effective usage for implementing distributed systems

## 2. Coroutines in Raft

Coroutines are lightweight functions that allow pausing and resuming execution at specific points, unlike traditional functions that run to completion in a single go.

### 2.1 Motivation

- Non-blocking IO: Network communication and disk access in consensus algorithms like Raft can introduce delays. Coroutines allow the system to handle these operations without blocking the main execution thread, ensuring that critical tasks like leader election and log replication happen without unnecessary stalls.
- Efficient Concurrency: Raft requires managing multiple concurrent tasks like handling client requests, heartbeats, and log replication. Coroutines provide a way to structure these tasks for better code readability and maintainability.
- Modular Design: Different components of Raft (leader election, log replication, etc.) can be implemented as separate coroutines, improving code organization and promoting clearer separation of concerns.

### 2.2 Implementation Overview

- Leader Election: A coroutine could handle the process of a server becoming a candidate, sending vote requests, and transitioning to a leader state based on responses.
- Log Replication: Coroutines can manage sending AppendEntries RPCs to followers and handling their responses.
- Client Interaction: Coroutines can handle incoming client requests, potentially delegating work to other coroutines for log updates and waiting for responses.


```
// Coroutine for Leader Election
Coroutine::CreateRun([this](){
  while (true){
    if (!isLeader()) {
      startLeaderElection(); 
      // Consider a sleep or yield here to avoid constant election attempts 
      // if not yet a leader.
    }
    Coroutine::Sleep(electionTimeout); 
  }
});

// Coroutine for Heartbeats (if you're a leader)
Coroutine::CreateRun([this](){
  while (true){
    if (isLeader()) {
      sendHeartbeats();  
    }
    Coroutine::Sleep(heartbeatInterval); 
  }
});

```

- `Coroutine::Sleep(interval)` is to pause the execution of the current coroutine for the duration specified by interval.

### 2.3 Important Considerations

- Synchronization: When multiple coroutines interact, ensure you have mechanisms to prevent race conditions and data inconsistencies.
- Error Handling: Implement robust error handling and recovery within coroutines since they might encounter timeouts, network issues, etc.

## 3. Events Overview

- Event system provides a mechanism for coordinating actions or notifications within an application.

### 3.1 SharedIntEvent

- Represents an event associated with an integer value. It allows components to:
    - Set the value and trigger notifications
    - Wait until the value meets specific conditions.

- Methods:
    
    1. Set(const int& v):
        - Updates the event's internal value.
        - Triggers notifications to any components waiting on this event if their conditions are now met.
        - Returns the previous value.

    2. Wait(uint64_t timeout):
        - The core wait mechanism for an event.
        - The calling component/coroutine is suspended until the event is considered 'ready' (`ev->Set(1)`) or the timeout expires.

    3. WaitUntilGreaterOrEqualThan(int x, int timeout):
        - A component uses this to wait until the event's value is greater than or equal to 'x'.
        - Will wait up to the specified timeout period.
        - Returns `false` if the value meets the condition before the timeout, `true` if a timeout occurs.
    

## 4. RPCs in Raft

Remote Procedure Calls (RPCs) are the primary communication mechanism within a Raft cluster. They are used for:
- Leader Election: Nodes send `RequestVote` RPCs to solicit votes during elections.
- Log Replication: Leaders send `AppendEntries` RPCs to replicate log entries to followers.

### 4.1 Types of RPCs in Raft

- Synchronous RPCs: These RPCs block the caller until a response is received from the remote node. Consider using synchronous RPCs in Raft when strong consistency guarantees are needed and blocking is acceptable. (Example: A candidate node waiting for vote responses during an election)
- Asynchronous RPCs: These RPCs allow the caller to continue with other tasks while waiting for responses. Use asynchronous RPCs  to optimize performance and avoid unnecessary blocking.  (Example: A leader sending `AppendEntries RPCs` to multiple followers without waiting for each response.)

### 4.2 Implementation Overview

#### 4.2.1 Synchronous RPC

Used for basic communication where the sender needs to wait for a direct response from each target server.

#### Mechanism 
The code (a demo present currently in lab) iterates through servers, sending individual RPC requests. Each request has a built-in waiting mechanism with a timeout for handling potential delays or failures.

#### 4.2.2 Asynchronous RPC (with Normal Events)

Improves efficiency by allowing the sender to continue other tasks while waiting for RPC responses.

#### Mechanism
- RPC requests are sent to servers without immediately blocking the sender.
- A regular "Event" object (or a similar mechanism) is likely associated with each RPC.
- When a response arrives, it triggers a callback function or signals the event to notify the sender.

```
Coroutine::CreateRun([this]() {
    std::vector<shared_ptr<IntEvent>> pending_events;

    for (int server_id = 0; server_id < NSERVERS; server_id++) {
        pending_events.push_back(commo()->SendString(0, server_id, "hello", nullptr));
    }

    while (!pending_events.empty()) {
        for (auto it = pending_events.begin(); it != pending_events.end(); ) {
            auto& event = *it;
            if (event->IsReady()) { // Replace with the appropriate check
                string res = event->GetResult();  // Assuming a way to get the result
                if (event->status_ == Event::TIMEOUT) {
                    Log_info("timeout happens");
                } else {
                    Log_info("rpc response is: %s", res.c_str()); 
                }
                it = pending_events.erase(it); // Remove completed event
            } else {
                ++it; 
            }
        }
        Coroutine::Sleep(50); // Adjust sleep interval as necessary
    }
});
```

#### 4.2.3 Asynchronous RPC with QuorumEvent

Optimized for scenarios requiring coordination based on responses from multiple servers, particularly when waiting for every server isn't necessary (e.g., leader elections).

#### Mechanism

- RPC requests are sent to multiple servers in parallel.
- A specialized `ReplyQuorumEvent` object tracks responses and maintains logic for determining when a majority quorum has been reached.
- The code can proceed once a majority quorum is reached, even if some servers haven't responded yet.

`server.cc`

```
Coroutine::CreateRun([this](){

    while(true){
      
        auto event = commo()->SendRequestVote(0, site_id_, request_params, response_params);
        event->Wait(electionTimeout);
        if (event->status_ == Event::TIMEOUT) {
            Log_info("failed to connect from server %d", site_id_);
            continue;
        }
        // process replies
        for (auto reply : event->replies){
            Log_info("Processing reply");
            Log_info("reply.ret %u", reply.ret);
        }

        if (event->Yes()) {
            currentState="leader";
        } 
    } 
});
```

`commo.h`

```
class Reply{
    public:
        uint64_t ret;
        bool_t vote_granted;
        Reply(uint64_t ret, bool_t vote_granted) : ret(ret), vote_granted(vote_granted) {}
        Reply() : ret(0), vote_granted(0) {}
};

class ReplyQuorumEvent : public QuorumEvent {

public:
    vector<Reply> replies;
    ReplyQuorumEvent(int n_total, int quorum) : QuorumEvent(n_total, quorum) {
        //replies.resize(n_total);
    }

    void VoteYes(Reply& reply){
        replies.push_back(reply);
        this->QuorumEvent::VoteYes();
    }
    
    void VoteNo(Reply& reply){
        replies.push_back(reply);
        this->QuorumEvent::VoteNo();
    }
};
```

`commo.cc`

```
shared_ptr<ReplyQuorumEvent> 
RaftCommo::SendRequestVote(parid_t par_id,
                            siteid_t site_id,
                            const uint64_t& arg1,
                            const uint64_t& arg2,
                            const uint64_t& arg3,
                            const uint64_t& arg4,
                            uint64_t* arg5,
                            bool_t* arg6
                            ) {
    auto proxies = rpc_par_proxies_[par_id];
    auto ev = Reactor::CreateSpEvent<ReplyQuorumEvent>(NSERVERS, NSERVERS/2);
    for (auto& p : proxies) {
        if (p.first != site_id) {
            RaftProxy *proxy = (RaftProxy*) p.second;
            FutureAttr fuattr;
            fuattr.callback = [ev](Future* fu) {
                /* This is a handler that will be invoked when the RPC returns */
                uint64_t ret1;
                bool_t vote_granted;
                /* Retrieve RPC return values in order */
                fu->get_reply() >> ret1;
                fu->get_reply() >> vote_granted;
                Reply reply(ret1, vote_granted);
                if (vote_granted == true) {
                    ev->VoteYes(reply);
                } else {
                    ev->VoteNo(reply);
                }
            };
            Call_Async(proxy, RequestVote, arg1, arg2, arg3, arg4, fuattr);
        }
    }
    return ev;
}
```