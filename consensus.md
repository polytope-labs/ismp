## Consensus Client

ISMP's messaging abstraction is built on top of the `ConsensusClient`. We call it a consensus client because that is exactly what it should be.
A client that observes a blockchains consensus messages in order for it to come to a conclusion about what is canonical on the network.

Recall that the consensus mechanism is responsible for coming to consensus about the canoncial series of state transitions on the network. Through knowledge of the canonical state transitions of a counter party state machine, ISMP builds the request-response abstraction by leveraging state proofs.

In this document, we formally define the `ConsensusClient` which serves as an oracle for the canonical state of a state machine and the corresponding `ConsensusMessage` which is used to advance the state of the consensus client. 

## `ConsensusClient`

The consensus client can be seen as one half of a full blockchain client. In that it verifies only consensus proofs in order to advance it's view of the blockchain network. Where full nodes would verify both consensus proofs and the state transition function of the network. This in effect makes consensus clients suitable for resource-constrained environments like blockchains, allowing it become interoperable with another state machine in a trust-free manner.

The search for a mechanism by which a consensus client may observe and come to conclusions about the canonical state of another blockchain leads us to understand the concept of safety on the topic of safety in distributed systems. We elaborate more on this in our research article on consensus proofs $^{[1]}$. Succintly we show that safety in on-chain light clients will require the use of a challenge window, even after consensus proof verification. This allows us to detect potential byzantine behaviour that may arise without this challenge window in place.


```rust

/// We define the consensus client as a module that handles logic for consensus & fraud proof verification,
pub trait ConsensusClient {
    /// Verify the associated consensus proof, using the trusted consensus state.
    fn verify_consensus(
        &self,
        host: &dyn IsmpHost,
        trusted_consensus_state: Vec<u8>,
        proof: Vec<u8>,
    ) -> Result<(Vec<u8>, Vec<IntermediateState>), Error>;

    /// Given two distinct consensus proofs, verify that they're both valid and represent conflicting views of the network.
    /// returns Ok(()) if they're both valid.
    fn verify_fraud_proof(
        &self,
        host: &dyn IsmpHost,
        trusted_consensus_state: Vec<u8>,
        proof_1: Vec<u8>,
        proof_2: Vec<u8>,
    ) -> Result<(), Error>;

    /// Return the unbonding period (i.e the time it takes for a validator's deposit to be unstaked from the network)
    fn unbonding_period(&self) -> Duration;
}

```

The `verify_consensus` client is responsible for decoding the provided consensus state bytes, as well as the proof according the structures it's internal algorithm requires. If verification is successful, it should update the trusted consensus state and return it along side any new state transitions finalized by the consensus proof. These new state transitions cannot be used to prove new requests and responses immediately. Instead they are kept in a pending state that extends for the configured `challenge_period` on the `IsmpHost`. This allows honest network participants to ascertain if they indeed describe valid state transitions on the counterparty network. In the unfortunate case that they don't, this challenge period enables them to submit a fraud proof, in the form of consensus proofs for two distinct state transitions that are valid for the consensus client's verification algorithm. The consensus client exposes a way to verify the validity of this fraud proof through the `verify_fraud_proof` method. Finally there's the issue of long fork attacks, we mitigate this by keeping track of the timestamp at which a consensus client was successfully updated. The next time a consensus client receives a new update, we can check that the difference between the current time and the last time the consensus client was updated does not exceed the client's unbonding period. This in effect mitigates any potential long fork attacks that may arise as a result of a loss of liveness of consensus clients.



