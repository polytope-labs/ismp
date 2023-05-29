## Message Handling

Messages are processed in batches, therefore proofs supplied need to be batch proofs.  
If it's impossible to produce batch proofs, then messages need to be sent individually

```rust
pub struct RequestMessage {
    /// Requests from source chain
    pub requests: Vec<Request>,
    /// Membership batch proof for these requests
    pub proof: Proof,
}

pub enum TimeoutMessage {
    Post {
        /// Request timeouts
        requests: Vec<Request>,
        /// Non membership batch proof for these requests
        timeout_proof: Proof,
    },
    /// There are no proofs for Get timeouts, we only need to
    /// ensure that the timeout timestamp has elapsed on the host
    Get {
        /// Requests that have timed out
        requests: Vec<Request>,
    },
}

pub enum ResponseMessage {
    Post {
        /// Responses from sink chain
        responses: Vec<Response>,
        /// Membership batch proof for these responses
        proof: Proof,
    },
    Get {
        /// Request batch
        requests: Vec<Request>,
        /// State proof
        proof: Proof,
    },
}
```

## Validating state machine

Before messages are processed the consensus client and state machine need to be validated, there needs to be a check
for expired and frozen clients, as well as a check that ensures the challenge period for the state commitment referred
to by the proof has elapsed.
The client validation algorithm is described as follows:

```rust
/// This function checks to see that the challenge period configured on the host chain
/// for the state machine has elapsed.
fn verify_challenge_period_elapsed<H>(host: &H, proof_height: StateMachineHeight) -> Result<(), Error>
    where
        H: IsmpHost,
{
    let update_time = host.consensus_update_time(proof_height.id.consensus_client)?;
    let delay_period = host.challenge_period(proof_height.id.consensus_client);
    let current_timestamp = host.timestamp();
    if current_timestamp - update_time > delay_period {
        Ok(())
    } else {
        Err(Error::ChallengePeriodNotElapsed {
            consensus_id: proof_height.id.consensus_client,
            current_time: host.timestamp(),
            update_time: host.consensus_update_time(proof_height.id.consensus_client)?,
        })
    }
}

/// This function does the preliminary checks for a request or response message
/// - It ensures the consensus client is not frozen
/// - It ensures the state machine is not frozen
/// - Checks that the challenge period configured for the state machine has elapsed.
fn validate_state_machine<H>(
    host: &H,
    proof_height: StateMachineHeight,
) -> Result<(), Error>
    where
        H: IsmpHost,
{
    // Ensure consensus client is not frozen
    let consensus_client_id = proof_height.id.consensus_client;
    // Ensure client is not frozen
    host.is_consensus_client_frozen(consensus_client_id)?;

    // Ensure state machine is not frozen
    host.is_state_machine_frozen(proof_height)?;

    // Ensure challenge period has elapsed
    verify_challenge_period_elapsed(host, proof_height)?;

    Ok(())
}
```

## Request Handler

## Response Handler

## Timeout handler