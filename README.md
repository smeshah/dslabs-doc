
## Coroutine
- Handling asynchronous operations ( I/O operations, networking, latency-bound tasks) without blocking the main execution thread.
- Allows functions to be suspended and resumed at certain points.
- Allows multiple operations to proceed concurrently without waiting for each one to complete sequentially.


### Implementation 
- Implementing distinct operations such as leader election, sending append entries, and other tasks as separate coroutines.
- Enhances the modularity and readability of the code, allowing each coroutine to handle a specific part of the system's functionality.
- To ensure effective concurrency in coroutines, especially those running infinite loops (`while (true)`), it's crucial to incorporate `await` for async operations or `sleep` pauses. 

```
Coroutine::CreateRun([this](){
    while (true){
        // Handle Election        
        Coroutine::Sleep(timeoutA);
    }
});

Coroutine::CreateRun([this](){
    while (true){
        // Handle Append Entries        
        Coroutine::Sleep(timeoutB);
    }
});
```

### RPC (Remote Procedure Call)

- Enables a computer program to execute a procedure (or subroutine) in another address space (commonly on another computer on a shared network), without the programmer needing to explicitly code for this interaction.
- asnyc rpc: client initiates the call and then continues with its own processing without waiting for the server's response.
- sync rpc:  client initiates the call to the server and then waits for the server to finish processing the call and return the result before proceeding. 

### Implementation

- RPC will be used to send requests from the client to the leader.
- RPC will be utilized for communication among a set of server replicas for purposes such as leader election, transmitting entries from the leader to the followers, and other coordination tasks.

Class RaftCommo (commo.cc) is meant to handle sending RPC requests.

#### Example of sync RPC

- using sync rpc \
    \
    `server.cc` 
    ```
    Coroutine::CreateRun([this](){

        for (int server_id = 0; server_id < NO_OF_SERVERS; server_id++){
            if (server_id == loc_id_)
                continue;
            else{
                string res;
                auto event = commo()->SendString(0, /* partition id is always 0 for lab1 */
                                                server_id, "hello", &res);
                event->Wait(RPC_TIMEOUT); 
                if (event->status_ == Event::TIMEOUT) {
                Log_info("timeout happens");
                } else {
                Log_info("rpc response is: %s", res.c_str()); 
                }
            }

        }
    });
    ```
    `commo.cc` 
    ```
    shared_ptr<IntEvent> 
    RaftCommo::SendString(parid_t par_id, siteid_t site_id, const string& msg, string* res) {
        auto proxies = rpc_par_proxies_[par_id];
        auto ev = Reactor::CreateSpEvent<IntEvent>();
        for (auto& p : proxies) {
            if (p.first == site_id) {
            RaftProxy *proxy = (RaftProxy*) p.second;
            FutureAttr fuattr;
            fuattr.callback = [res,ev](Future* fu) {
                fu->get_reply() >> *res;
                ev->Set(1);
            };
            /* wrap Marshallable in a MarshallDeputy to send over RPC */
            Call_Async(proxy, HelloRpc, msg, fuattr);
            }
        }
        return ev;
    }
    ```

    Using synchronous RPC and waiting for each server's response can lead to inefficiencies. 

- Using async operation


    The QuorumEvent feature in the framework allows us to make asynchronous calls to multiple servers. It collects responses from these servers and stores them. The main advantage is that once a majority of responses are received, it doesn't have to wait for the rest, which is particularly useful during leader elections. \
    \
    `server.cc`

    ```
    Coroutine::CreateRun([this](){

        while(true){
          
            auto event = commo()->SendRequestVote(0, site_id_, request_params, response_params);
            event->Wait(1000000);
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