# ISMP Dispatcher

The dispatcher is the public interface which modules use to create requests and responses.
The dispatcher should be the only public interface through which modules interact with the protocol.
The dispatcher should emit relevant events after any dispatch
The interface for the Ismp Dispatcher is:

```rust
    pub trait IsmpDispatcher {
    /// This function should accept requests from modules and commit them to the state
    /// It should emit the `Request` event after a successful dispatch
    fn dispatch_request(&self, request: DispatchRequest) -> Result<(), Error>;

    /// This function should accept responses from modules and commit them to the state
    /// It should emit the `Response` event after a successful dispatch
    fn dispatch_response(&self, response: PostResponse) -> Result<(), Error>;
}
```

### Events

Events should be emitted for state machine updates and when requests are responses are dispatched.This allows offchain components
to be notified when new messages need to be relayed.  
The structure of ISMP events is described below:

```rust
pub enum Event {
    /// Emitted when a state machine is successfully updated to a new height
    StateMachineUpdated {
        /// State machine id
        state_machine_id: StateMachineId,
        /// Latest height
        latest_height: u64,
    },
    /// Emitted when a challenge period has begun for a consensus client
    ChallengePeriodStarted {
        /// Consensus client id
        consensus_client_id: ConsensusClientId,
        /// Tuple of previous height and latest height
        state_machines: BTreeSet<(StateMachineHeight, StateMachineHeight)>,
    },
    /// Emitted for an outgoing response
    Response {
        /// Chain that this response will be routed to
        dest_chain: StateMachine,
        /// Source Chain for this response
        source_chain: StateMachine,
        /// Nonce for the request which this response is for
        request_nonce: u64,
    },
    /// Emitted for an outgoing request
    Request {
        /// Chain that this request will be routed to
        dest_chain: StateMachine,
        /// Source Chain for request
        source_chain: StateMachine,
        /// Request nonce
        request_nonce: u64,
    },
}
```