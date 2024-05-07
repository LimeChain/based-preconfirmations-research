# Overview
This document outlines suggested proof of concept (PoC) scope and plan to implement Vanilla Based Sequencing and Preconfirmations. The scope will use [Taiko](https://github.com/taikoxyz) rollup as foundation and will aim to prove the viability for it to support vanilla based sequencing and preconfirmations. 

The high level scope includes:

- Launching a devnet environment to showcase the sequencing selection with two separate networks - L1 based on geth with GMEV-Boost setup and another for the modified Taiko nodes. The modified Relayer and Builder are also part of the infrastructure.
- Implementation of GMEV-Boost based on MEV-boost - Implement two separate pipelines for block building, one responsible for the L1 block building and another for the Taiko blocks, which should be compatible with the current MEV-Boost
- Modifying Rollup node to support the new transaction types - execution state tx + inclusion tx
- Implement SequencerRegistry smart contract without verification of the proposer
- Modifying TaikoL1 smart contracts to support the primary and secondary selection of sequencer
- Implement 2 new endpoints in the Relay
    - GET  `/relay/v1/builder/conditions` - used by the Builder
    - POST `/eth/v1/builder/conditions/{slot}/{parent_hash}/{pubkey}` - store the conditions
- Implement `/relay/v1/builder/conditions` endpoint querying on the Builder to get the conditions for block building from the Relayer

# **Project Plan**

To set up the entire pre-confirmation system, we need to make several changes at different levels. The following section briefly overviews the system components and how they connect.

## Steps by Software component

**Rollup Node**

- **End-user interaction**: Implement a handler for `eth_sendRawTransaction` for Inclusion of pre-confirmation transaction. On preconfirmation request rejection, it must return zero hash and non-zero `tx_hash` otherwise
- **End-user interaction**: Implement handler for `eth_sendRawTransaction` for Execution state preconfirmation transaction. On preconfirmation request rejection, it must return zero hash and non-zero `tx_hash` otherwise
- Pipelines API: Modify Rollup node to query `/gmev/v1/validators` from GMEV-Boost on startup to get a list of its registered validators
- Pipelines API: the Rollup node will have its endpoint to the beacon chain, querying scheduled proposers for the next epoch, cross-checking the ones that belong to the Rollup Node operator. Based on when the L2 Sequencer has a monopoly over the L1 slot (primary selection) or not (secondary selection), it will either post its conditions (preconfed transactions) to GMEV-Boost through POST `/gmev/v1/conditions` endpoint or both the former and the L1 mempool

**L1 Smart contract**

- The `ISequencerRegistry` interface is to be implemented, which maintains a list of active preconfirmation service providers. The `register` and `activate` methods are crucial part of the Rollup Node’s (acting as L2 Sequencer) initialisation process. The implementing contract needs to set a threshold for the required activation stake and the withdrawal challenge period. **Тhe implementation will use mock signature verifications and SSZ multiproofs**, only ensuring there is a present L2 sequencer in the contract with known in advance metadata and a address.
- The `TaikoL1` smart contract needs to store a mapping that associates the proposer to the proposed Block in the `proposeBlock` method, so it can be verified the proposer is an active L1 validator in the `verifyBlock` method.
- The `TaikoL1` smart contract needs to check through an utility smart contract that `ISequencerRegistry.isEligibleSigner` for the `msg.sender` in it’s `verifyBlock` method to verify the L2 sequencer is eligible.
    - In case there is no eligible L1 proposer to act as L2 sequencer for the given slot, The `TaikoL1` needs to further implement a fallback mechanism. It can get it’s fallback proposer index by getting the `X`-th parent block hash (x < 256) modulo `ISequencerRegistry.eligibleCountAt` and then getting the sequencer metadata using `ISequencerRegistry.sequencerByIndex`,
- `TaikoL1` smart contract or helper contract needs to store L2 sequencer stake, activation threshold, withdrawal challenge period. It needs to call `ISequencerRegistry.activate` when eligibility L2 sequencer is above threshold.

**GMEV-Boost**

- Fork the latest MEV-boost implementation
- Modify MEV-boost with and additional optional `conditions_version` field, signalizing support for Conditions API to the Relay
- Implement`/gmev/v1/validators` endpoint handler
- Implement `/gmev/v1/conditions` endpoint handler
- Implementing the Conditions API:
    - Calling a **Relay** through POST endpoint `/eth/v1/builder/conditions/{slot}/{parent_hash}/{pubkey}`
        - DTOs and example request body:
        
        ```jsx
        class ValidatorConditionsV1(Container):
            top: List[Transaction, MAX_TOP_TRANSACTIONS], //Transaction-hex RLP encoded preconf tx
            rest: List[Transaction, MAX_REST_TRANSACTIONS]
            
        class SignedValidatorConditionsV1(Container):
            message: ValidatorConditionsV1,
            conditions_hash: Hash32,
            signature: BLSSignature
            
        "message": {
        	"top": [
        		"0x02f878831469668303f51d843b9ac9f9843b9aca0082520894c93269b73096998db66be0441e836d873535cb9c8894a19041886f000080c001a031cc29234036afbf9a1fb9476b463367cb1f957ac0b919b69bbc798436e604aaa018c4e9c3914eb27aadd0b91e10b18655739fcf8c1fc398763a9f1beecb8dddddd"
        	],
        	"rest": [
        		"0x02f878831469668303f51d843b9ac9f9843b9aca0082520894c93269b73096998db66be0441e836d873535cb9c8894a19041886f000080c001a031cc29234036afbf9a1fb9476b463367cb1f957ac0b919b69bbc798436e604aaa018c4e9c3914eb27aadd0b91e10b18655739fcf8c1fc398763a9f1beecb8eeeee",
        	]
        },
        "conditions_hash": "0xcf8e0d4e9587369b2301d0790347320302cc0943d5a1884560367e8208d920f2"
        "signature": "0x1b66ac1fb663c9bc59509846d6ec05345bd908eda73e670af888da41af171505cc411d61252fb6cb3fa0017b679f8bb2305b26a285fa2737f175668d0dff91cc1b66ac1fb663c9bc59509846d6ec05345bd908eda73e670af888da41af171505"
        ```
        

**Relay**

Develop a dummy Relay

- Conditions API: POST `/eth/v1/builder/conditions/{slot}/{parent_hash}/{pubkey}`  endpoint handler
    - storing latest `conditions_hash` (hash of all raw hex RLP-encoded transactions stored in the `top` field of the ValidatorConditionsV1 DTO)
    - When called, discards all current bids and blocks from builders with different `conditions_hash`(due to updated conditions)
    - Set a configurable window during which proposers can call the endpoint (ex. default 6 seconds)
    - When L1 slot finishes, previous conditions, blocks and bids are cleared
- Conditions API: GET `/relay/v1/builder/conditions` handler
    - Returns an array of upcoming proposers and their currently set condition transactions (inclusion tx or execution state tx).
    - Condition transactions returned by this endpoint are to be mocked
    - Each entry includes a slot, validator and the validator expressed conditions (hex RLP-encoded transactions)

**Builder**

- Continuously query Conditions API’ `/relay/v1/builder/conditions` endpoint
    - Configurable every `X` seconds
    - Dynamically creating and sending a block to the Relay using only the RLP encoded preconfirmation transactions fetched from the query

**Devnet setup**

- Using simple devnet setup configure the geth L1 nodes imitating the Ethereum network
- Deploy all Taiko related L1 smart contracts on that network
- Setup Taiko modified nodes potentially using predefined docker compose setups from the Taiko github repo
- The devnet should be accessible by end users and they can follow the flow completely

**Preconfirmation testing script**

- Write a script that sends a configurable number of signed preconfirmation transactions to GMEV-Boost every `Y` seconds.