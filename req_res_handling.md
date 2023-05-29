### Message Handling

Messages are processed in batches, therefore proofs supplied need to be batch proofs.  
If it's impossible to produce batch proofs, then messages need to be sent individually.
The message types for ISMP are described in the snippet below:

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

To handle incoming request messages, the consensus client and state machine are validated using the algorithm described
in the previous section.
The request timeout is also verified to ensure the request has not expired.
The proof is verified and the router is called to dispatch the requests to the destination modules.
A receipt of this request is stored to prevent any duplicate from being handled in the future.

```rust
pub fn handle<H>(host: &H, msg: RequestMessage) -> Result<MessageResult, Error>
    where
        H: IsmpHost,
{
    let state_machine = validate_state_machine(host, msg.proof.height)?;
    // Verify membership proof
    let state = host.state_machine_commitment(msg.proof.height)?;

    state_machine.verify_membership(
        host,
        RequestResponse::Request(msg.requests.clone()),
        state,
        &msg.proof,
    )?;

    let router = host.ismp_router();
    // If a receipt exists for any request then it's a duplicate and it is not dispatched
    let result = msg
        .requests
        .into_iter()
        .filter(|req| host.request_receipt(req).is_none() || req.timed_out(host.timestamp()))
        .map(|request| {
            let res = router.handle_request(request.clone());
            host.store_request_receipt(&request)?;
            Ok(res)
        })
        .collect::<Result<Vec<_>, _>>()?;

    Ok(MessageResult::Request(result))
}
```

## Response Handler

To handle incoming response messages, the consensus client and state machine are validated using the algorithm described
in the previous section.
The proof is verified and the router is called to dispatch the responses to the destination modules.
A receipt of this response is stored to prevent any duplicate from being handled in the future.
When verifying a batch of GET requests, the proof height must be equal to the retrieval height specified in the
requests.

```rust
/// Validate the state machine, verify the response message and dispatch the message to the router
pub fn handle<H>(host: &H, msg: ResponseMessage) -> Result<MessageResult, Error>
    where
        H: IsmpHost,
{
    let state_machine = validate_state_machine(host, msg.proof().height)?;
    for request in &msg.requests() {
        // For a response to be valid a request commitment must be present in storage
        let commitment = host.request_commitment(request)?;

        if commitment != hash_request(request) {
            return Err(Error::RequestCommitmentNotFound {
                nonce: request.nonce(),
                source: request.source_chain(),
                dest: request.dest_chain(),
            });
        }
    }

    let state = host.state_machine_commitment(msg.proof().height)?;

    let result = match msg {
        ResponseMessage::Post { responses, proof } => {
            // Verify membership proof
            state_machine.verify_membership(
                host,
                RequestResponse::Response(responses.clone()),
                state,
                &proof,
            )?;

            let router = host.ismp_router();

            responses
                .into_iter()
                .filter(|res| host.response_receipt(res).is_none())
                .map(|response| {
                    let res = router.handle_response(response.clone());
                    host.store_response_receipt(&response)?;
                    Ok(res)
                })
                .collect::<Result<Vec<_>, _>>()?
        }
        ResponseMessage::Get { requests, proof } => {
            // Ensure the proof height is greater than each retrieval height specified in the Get
            // requests
            sufficient_proof_height(&requests, &proof)?;
            // Since each get request can  contain multiple storage keys, we should handle them
            // individually
            requests
                .into_iter()
                .map(|request| {
                    let keys = request.keys().ok_or_else(|| {
                        Error::ImplementationSpecific("Missing keys for get request".to_string())
                    })?;
                    let values =
                        state_machine.verify_state_proof(host, keys.clone(), state, &proof)?;

                    let router = host.ismp_router();
                    let res = router.handle_response(Response::Get {
                        get: request.get_request()?,
                        values: keys.into_iter().zip(values.into_iter()).collect(),
                    });
                    Ok(res)
                })
                .collect::<Result<Vec<_>, _>>()?
        }
    };

    Ok(MessageResult::Response(result))
}

```

## Timeout handler

Timeouts are handled differently for Post and Get requests. For post requests the timeout is evaluated relative to the
destination chain's timestamp along side a proof of non-membership.
Get request timeouts are processed without any proof, the timeout is just evaluated relative to the source chain's
timestamp just, this is because Get requests are never delivered to the counterparty chain, but are processed offchain
by interested parties.

```rust
pub fn handle<H>(host: &H, msg: TimeoutMessage) -> Result<MessageResult, Error>
    where
        H: IsmpHost,
{
    let results = match msg {
        TimeoutMessage::Post { requests, timeout_proof } => {
            let state_machine = validate_state_machine(host, timeout_proof.height)?;
            let state = host.state_machine_commitment(timeout_proof.height)?;
            for request in &requests {
                // Ensure a commitment exists for all requests in the batch
                let commitment = host.request_commitment(request)?;
                if commitment != hash_request(request) {
                    return Err(Error::RequestCommitmentNotFound {
                        nonce: request.nonce(),
                        source: request.source_chain(),
                        dest: request.dest_chain(),
                    });
                }

                if !request.timed_out(state.timestamp()) {
                    Err(Error::RequestTimeoutNotElapsed {
                        nonce: request.nonce(),
                        source: request.source_chain(),
                        dest: request.dest_chain(),
                        timeout_timestamp: request.timeout(),
                        state_machine_time: state.timestamp(),
                    })?
                }
            }

            let key = state_machine.state_trie_key(requests.clone());

            let values = state_machine.verify_state_proof(host, key, state, &timeout_proof)?;

            if values.into_iter().any(|val| val.is_some()) {
                Err(Error::ImplementationSpecific("Some Requests not timed out".into()))?
            }

            let router = host.ismp_router();
            requests
                .into_iter()
                .map(|request| {
                    let res = router.handle_timeout(request.clone());
                    host.delete_request_commitment(&request)?;
                    Ok(res)
                })
                .collect::<Result<Vec<_>, _>>()?
        }
        TimeoutMessage::Get { requests } => {
            for request in &requests {
                let commitment = host.request_commitment(request)?;
                if commitment != hash_request(request) {
                    return Err(Error::RequestCommitmentNotFound);
                }

                // Ensure the get timeout has elapsed on the host
                if !request.timed_out(host.timestamp()) {
                    Err(Error::RequestTimeoutNotElapsed)?
                }
            }
            let router = host.ismp_router();
            requests
                .into_iter()
                .map(|request| {
                    let res = router.handle_timeout(request.clone());
                    host.delete_request_commitment(&request)?;
                    Ok(res)
                })
                .collect::<Result<Vec<_>, _>>()?
        }
    };

    Ok(MessageResult::Timeout(results))
}

```