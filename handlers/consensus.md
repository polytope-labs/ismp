# Consensus Handlers

ISMP exposes a set of handlers that allows users to manage the state of it's various consensus clients.

## `CreateConsensusClientMessage`

The is defined as:

```rust
/// We define the [`StateCommitment`] as the commitment to the global state trie at a given height.
pub struct StateCommitment {
    /// Timestamp in seconds
    pub timestamp: u64,
    /// Root hash of the request/response overlay tree.
    pub overlay_root: Option<H256>,
    /// Root hash of the global state trie.
    pub state_root: H256,
}

/// Since consensus systems may come to conensus about the state of multiple state machines, we
/// identify each state machine individually.
pub struct StateMachineId {
    /// The state machine identifier
    pub state_id: StateMachine,
    /// It's consensus client identifier
    pub consensus_client: ConsensusClientId,
}

/// Identifies a state machine at a given height
pub struct StateMachineHeight {
    /// The state machine identifier
    pub commitment: StateCommitment,
    /// the corresponding block height
    pub height: u64,
}

/// Statically defined consensus client Ids
pub type ConsensusClientId = [u8; 4];

/// The message that offchain parties send to initialize a consensus client.
pub struct CreateConsensusClientMessage {
    /// encoded consensus state
    pub consensus_state: Vec<u8>,
    /// Consensus client id
    pub consensus_client_id: ConsensusClientId,
    /// State machine commitments
    pub state_machine_commitments: Vec<(StateMachineId, StateMachineHeight)>,
}

```

This should be a subjectively chosen initial state for a consensus client. A sort of trusted setup for the initiated. Because it is subjectively chosen, it is recommended that this message is initiated either by the "admin" of the state machine or through a quorum of votes which allows the network to properly audit the contents of the initial consensus state. The handler for this message simply persists the consensus client and all of it's intermediate states as 


## `ConsensusMessage`

```rust
pub struct ConsensusMessage {
    /// Encoded Consensus Proof
    pub consensus_proof: Vec<u8>,
    /// Consensus client id
    pub consensus_client_id: ConsensusClientId,
}
```

The handler uses the `IsmpHost` to fetch the appropriate [`ConsensusClient`](../consensus-client.md) implementation, invoking it's `verify_consensus_proof` method with the provided proof, and a trusted consensus state. If consensus proof verification is successful, the handler should persist the latest state commitments for each state machine returned.


## `FraudProofMessage`

```rust
pub struct FraudProofMessage {
    /// The first consensus Proof
    pub proof_1: Vec<u8>,
    /// The second consensus Proof
    pub proof_2: Vec<u8>,
    /// Consensus client id
    pub consensus_client_id: ConsensusClientId,
}
```

The handler uses the `IsmpHost` to fetch the appropriate [`ConsensusClient`](../consensus-client.md) implementation, invoking it's `verify_fraud_proof` method with the provided proofs, and a trusted consensus state. If fraud proof verification is successful, the handler marks a consensus client as frozen, requiring an admin or network quorum to unfreeze the client.
 
