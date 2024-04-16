# Block Building Pipelines for Vanilla Based Rollups

# TLDR

- **GMEV-Boost** is a drop-in replacement software to MEV-Boost. Its main goal is to enable the connection between the L2 pipelines and the L1 pipeline.
- The design separates the pipelines to a single **primary** and multiple **secondary** pipelines. The primary pipeline deals with the L1 block building. The secondary pipelines deal with the production of the L2 sequencing transactions.
- The primary pipeline is extended with a Conditions API - an extension to the external block building API - implemented within the relays and used by proposers (through GMEV-Boost) and builders. It allows the proposer to specify validity conditions over their block that the relay need to enforce. Validity conditions are expressed as complete transactions.
- GMEV-Boost requires access to the validator keystore (or other signing method - f.e. Web3Signer) in order to enable the relays of primary pipeline to authenticate validator when validity conditions are submitted.
- The secondary pipelines are used to produce the L2 sequencing transactions. GMEV-Boost exposes an authenticated Pipelines API where the individual secondary pipelines submit their individual conditions. GMEV-Boost combines the individual conditions and submits them to the primary pipeline.

# Introduction

The goal of this document is to outline how the external block building pipeline for L1 and the various L2 block building pipelines can converge within the context of Vanilla Based Sequencing. The goal is to showcase the addition and modifications required to existing software in order for this pipelines convergence to be implemented.

## Choosing the Orchestrator

The goal of based sequencing is to enable the L1 proposers to be L2 sequencers. This means that the L2 sequencing transactions need to be submitted into the existing L1 pipeline. The actor that receives this submission needs to orchestrate the inclusion of these transactions into the L1 block. In this section we refer to this actor the “orchestrator”.

There are two well-suited actors to be chosen as the orchestrator. On one side there are the actors that actually build the blocks - the L1 builders, and on the other are the actual proposers.

The L1 builders have the natural advantage of already being sophisticated and the L2 sequencing transactions can become a part of their block building pipeline with little-to-no modification - as if the L2 sequencer is another searcher.

Choosing the L1 builders as orchestrator, however introduces and/or exacerbates several risk factors.

Firstly, in practice the group of L1 builders is small in size and requires huge technological and business sophistication to enter. This makes the creation and operation of a new based rollup dependant on a small group of players. Unfortunately, this risk has been seen previously materialise within some of the major blockchain network development stacks - a new network needs to spend considerable resources to persuade major validators to join in and support their upcoming network. Similar scenario is a distinct possibility with based rollups and can hurt the rollups sovereignty - a desired characteristic for any rollup.

Secondly, the geopolitical risks of the small group of builders is transferred to the rollup. Any materialised geopolitical risk (i.e. state induced censorship) is transferred to the rollup itself. This could be particularly problematic for privacy-oriented rollups and their sequencers, who are subject to liveness fault risks to no fault of their own. An example of this would be the state of the major builders sanctioning such rollup transactions.

The second approach that this paper explores is assigning the L1 proposer as the orchestrator. This approach requires more modifications of the existing software and interactions, but is ultimately based on the wide, diverse, decentralised group of Ethereum L1 proposers.

The L1 proposer as the orchestrator enables permissionless creation and operation of a vanilla based rollup. The rollup team needs the rollup to appeal to a diverse set of rational actors - the l1 proposers. Furthermore they can even spawn L1 proposers themselves and be their bootstrap sequencers - an approach practically impossible with builders. This protects the rollups sovereignty.

Furthermore, this approach prevents additional expansion of control and consolidation of power to the L1 builders. An approach where the L1 proposer requires the inclusion of the L2 sequencing transactions, and the L1 builders are forced to oblige or lose their opportunity to  build the current block, gives the control of sequencing to the proposers. Such a system utilises the security of the PBS - as long as there is 1 block builder willing to meet the requirements, the system will operate as expected, and even if no builder wants to build this block, the standard fallback to local block building can be used. Importantly

# Systems Architecture - High Level Overview

![BasedPipelinesArchitecture](https://hackmd.io/_uploads/rksmPRsl0.png)

The architecture suggests a new software GMEV-Boost -  a drop-in replacement software to [MEV-Boost](https://github.com/flashbots/mev-boost). Its main goal is to enable convergence of L2 pipelines into the L1 pipeline and enable L1 proposers to specify validity conditions towards the external building pipeline.

The architecture introduces the concept of primary pipeline and secondary pipelines. 

There is a single primary pipeline - the L1 pipeline. The goal of the primary pipeline is to build and broadcast L1 blocks. It is currently in production and is implemented via MEV-Boost. GMEV-Boost will be fully compatible with MEV-Boost and stakers wanting to only service the L1 pipeline can use the GMEV-Boost as drop-in replacement for MEV-Boost.

The system design supports multiple secondary pipelines. The goal of the secondary pipelines is to produce L2 sequencing transactions. The produced sequencing transactions are then communicated as conditions to the GMEV-Boost software of the proposer. GMEV-Boost then handles their combination and propagation to the primary pipeline for inclusion in the L1 block.

In order to facilitate these communications two new APIs are introduced - Conditions API and Pipelines API. 

The Conditions API is an extension to the [Ethereum Builder API](https://ethereum.github.io/builder-specs). Its goal is to enable the proposer, through GMEV-Boost to communicate to the external builders the validity conditions they have for the externally built blocks. In practice this means that the Conditions API must be implemented within the relays in order to enable builders and proposers to communicate with each other these requirements. 

The communication between the proposer and the relay needs to be an authenticated one. This is in order for the relay to be able to authenticate validity conditions submissions coming from the proposers and mitigate impersonation attacks. 

This authentication is in the form of cryptographic signatures by the proposer. This necessitates the GMEV-Boost software to have access to the validator keys of the proposer. As the GMEV-Boost is a software ran by the stakers themselves, access to the keys can be done via the same mechanism the validator software uses - i.e. access to a keystore file or Web3Signer. **Importantly, GMEV-Boost is a software ran and controlled by the stakers themselves and no third party is getting an access to their validator keys.**

The Pipelines API serves the purpose to enable the communication between the L2 rollup nodes and the GMEV-Boost. It enables the secondary pipelines to both discover the connected (to this GMEV-Boost instance) validators and communicate the individual conditions each of the pipeline has.

In order for the GMEV-Boost software to trust requests from the secondary pipeline, the communication between the two components need to be authenticated. In a similar manner as the authentication between Consensus and Execution Layer clients, the rollup nodes need to authenticate with the GMEV-Boost via JWT tokens.

Once individual conditions are submitted, it is the responsibility of the GMEV-Boost software to combine and aggregate the secondary pipelines individual preferences into combined conditions for the primary pipeline.

## Primary Pipeline

![Primary Pipeline](https://hackmd.io/_uploads/ryxSwAogR.png)

The goal of the primary pipeline is to produce an L1 block. It is a pipeline already in production through the MEV-Boost software and the external block building APIs. 

The existing pipeline is extended to allow the specification of validity conditions from proposers to block builders - through the relays. Through this API the proposer can express their requirements of inclusion of the L2 sequencing transactions within the L1 block in order to ensure their duties are fulfilled.

Specification of the Conditions API can be found in a later section.

### Interaction Flow

1. The flow is triggered by GMEV-Boost when both of the following conditions are present:
    1. The validator is chosen to be a proposer AND
    2. There are conditions indicated by the secondary pipelines that need enforcing.
2. GMEV-Boost combines the validity conditions of the secondary pipelines and signs them
3. GMEV-Boost submits them via the `/eth/v1/builder/conditions/{slot}/{parent_hash}/{pubkey}` endpoint. The relays authenticates the new validity conditions, records them and discards all existing bids.
4. The builders continuously poll the relays for updates in the validity conditions via the `/relay/v1/builder/conditions` endpoint. Whenever there is a change of the `conditions_hash` of the slot that the builder is building for, the builders need to construct a new bid that complies with the indicated conditions.

The rest of the external block-building pipeline interactions remains the same.

Important consideration for proposers is the timing of the submission of validity conditions. The proposer needs to account for the time needed by the block building pipeline to produce a block after the validity conditions have changed. In practice this means that the proposer must place a validity condition submission deadline or risk missing a slot.

## Secondary Pipelines

The goal of а secondary pipeline is to produce and submit the L2 sequencing transactions to the proposer for inclusion within L1 block. Each pipeline is represented by the rollup node and is authenticated via JWT authentication between the rollup node and the GMEV-Boost software as part of the Pipelines API.

Specification of the Pipelines API can be found in a later section.

### Interaction Flow

1. In order to discover which validators are connected with this staker the secondary pipelines use the `/gmev/v1/validators` endpoint of the Pipelines API. This enables the rollup nodes to know when they are selected as a sequencer (primary or fallback seelection)
2. During the operation of the sequencer they gather a list of transactions (f.e. preconfirmations) and generate a sequencing transaction that needs to be included in the L1.
3. The sequencer submits the transaction to the GMEV-Boost via the `/gmev/v1/conditions` endpoint of the Pipelines API

The subsequent lifecycle of the sequencing transaction is in the scope of the GMEV-Boost software.

## GMEV-Boost

The goal of the GMEV-Boost software is to be an orchestrator and convergence point between the various secondary pipelines and the primary one. There are several important responsibilities of the GMEV-Boost software.

Firstly, it implements the Pipeline API. On one hand this provides a way for the secondary pipelines to discover the validators registered with this staker. On the other it is the method for secondary pipelines to submit their individual conditions.

Secondly, it is responsible for accepting or rejecting individual conditions based on the current validator status. For example if there is no validator chosen to be a proposer in the current (or subsequent) slots, the secondary pipelines are not supposed to be submitting individual conditions.

Thridly, it has the duty to facilitate the conditions combination, signing and submission processes. It aggregates the various individual conditions and prepares the data for signing and submission, facilitates the production of the authentication signature and lastly it calls the relays with the newly combined validity conditions.

Lastly, it is GMEV-Boost responsibility to cover for synchronisation edge cases. Such case might be a late submission of validity conditions where the GMEV-Boost might need to fallback to local block building as no bids can come on time. 

The GMEV-Boost component is an ideal point for further extension of the based rollups ecosystem with desired features like Universal Synchronous Composability.

# Conditions API - Builder API Extension

Introduces 1 new endpoint in the builder API implemented by the relays and 1 new endpoint in the relay api.

## Data Models

### ValidatorConditionsV1

```jsx
class ValidatorConditionsV1(Container):
    top: List[Transaction, MAX_TOP_TRANSACTIONS],
    rest: List[Transaction, MAX_REST_TRANSACTIONS]
```

**Note**

L1 transactions that are sequencing L2 transactions do not require `top` of the block positioning, however the equivalent pipeline and APIs can be re-used for external L2 block building pipelines. This can enable the sequencer specifying execution preconfirmations as top-of-the-block conditions for the L2 block builders.

### SignedValidatorConditionsV1

```jsx
class SignedValidatorConditionsV1(Container):
    message: ValidatorConditionsV1,
    conditions_hash: Hash32,
    signature: BLSSignature
```

## Endpoints

### Submit Conditions

Endpoint: POST `/eth/v1/builder/conditions/{slot}/{parent_hash}/{pubkey}` 

Description: Sets the validity conditions for this slot by its proposer. Requires signed message.

Request: `SignedValidatorConditionsV1`

Notes:

- Conditions can be submitted as JSON or SSZ, and optionally GZIP encoded. To be clear, there are four options: JSON, JSON+GZIP, SSZ, SSZ+GZIP. If JSON, the content type should be **`application/json`**. If SSZ, the content type should be **`application/octet-stream`**.
- To enable GZIP compression for the request body, the HTTP content encoding should be **`gzip`**. Compression is optional.
- The `conditions_hash` is a keccak hash over the SSZ encoded `message`. It is used for quick identification by the caller if the validity conditions have changed.
- Proposer is authenticated via signature over the `conditions_hash` **.**
- The relay will simulate block proposals and discard ones not complying with the conditions.
- Updating conditions discards all current bids.

Example Request:

```jsx
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

### Get Conditions

Endpoint: GET `/relay/v1/builder/conditions`

Description: Get a list of indicated conditions for validators scheduled to propose in the current and next epoch. Builders are expected to be polling these regularly in order to satisfy requirements.

Notes: 

- Used by builders to know what an upcomming proposal must contain in order to be accepted.
- Returns an array of upcomming proposers and their currently set conditions.
- Each entry includes a slot, validator and the validator expressed conditions

Example Response:

```jsx
[
  {
    "slot": "1",
    "validator_index": "1",
    "entry": {
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
    }
  }
]
```

# Pipelines API

GMEV-Boost offers two new endpoints for use by the secondary pipelines. Both endpoints require authentication via JWT token.

## Data Models

### ValidatorDataV1

```jsx
class ValidatorDataV1(Container):
    validator_index: uint256,
    pubkey: utring
```

## Endpoints

### Get Validators

Endpoint: `/gmev/v1/validators`

Description: Get the validators registered with this GMEV-Boost instance

Response: `List[ValidatorDataV1]`

Notes:

- Used by secondary pipelines to get the registered validators in this GMEV-Boost instance.
- Returns an array of validators.

Example Response:

```jsx
[
  {
    "validator_index": "42",
    "pubkey": "0x93247f2209abcacf57b75a51dafae777f9dd38bc7053d1af526f220a7489a6d3a2753e5f3e8b1cfe39b56f43611df74a"
  }
]
```

### Submit Condition

Endpoint: `/gmev/v1/conditions`

Description: Submits conditions for the current slot

Request: `ValidatorConditionsV1`

Notes:

- Conditions can be submitted as JSON or SSZ, and optionally GZIP encoded. To be clear, there are four options: JSON, JSON+GZIP, SSZ, SSZ+GZIP. If JSON, the content type should be **`application/json`**. If SSZ, the content type should be **`application/octet-stream`**.
- To enable GZIP compression for the request body, the HTTP content encoding should be **`gzip`**. Compression is optional.
- Secondary Pipeline is authenticated via JWT
- Subsequent conditions submission for the same slot will discard the previous

Example Request:

```jsx
"message": {
	"top": [
            "0x02f878831469668303f51d843b9ac9f9843b9aca0082520894c93269b73096998db66be0441e836d873535cb9c8894a19041886f000080c001a031cc29234036afbf9a1fb9476b463367cb1f957ac0b919b69bbc798436e604aaa018c4e9c3914eb27aadd0b91e10b18655739fcf8c1fc398763a9f1beecb8dddddd"
	],
	"rest": [
            "0x02f878831469668303f51d843b9ac9f9843b9aca0082520894c93269b73096998db66be0441e836d873535cb9c8894a19041886f000080c001a031cc29234036afbf9a1fb9476b463367cb1f957ac0b919b69bbc798436e604aaa018c4e9c3914eb27aadd0b91e10b18655739fcf8c1fc398763a9f1beecb8eeeee",
	]
}
```

# Relay API Modifications

The Relay API will have minimal modifications of its current behaviour. The only optional change would be in the bid submission endpoint `/relay/v1/builder/blocks` where an **optional** field `conditions_hash` can be added to the request. 

This field will enable the builders to specify against which `conditions_hash` their bid is and will cover for race condition edge cases where conditions have changed in-flight.

```jsx
"message": {
    ...
    "value": "1",
    // Optional
    "conditions_hash": "0xcf8e0d4e9587369b2301d0790347320302cc0943d5a1884560367e8208d920f2" 
  },
  ...
}
```

# Builder API Modifications

The Relay API will have minimal modifications of its current behaviour. The only optional change would be in the validator registration endpoint `/eth/v1/builder/validators` where an **optional** field `conditions_version` can be added to the request.

An omission or value of `0` indicates no support for conditions submission, whereas non-zero value indicates the version of the Pipelines API.

Example Request:

```jsx
 {
    "message": {
        ...,
        // Optional
        "conditions_version": "1",
        "pubkey": "0x93247f2209abcacf57b75a51dafae777f9dd38bc7053d1af526f220a7489a6d3a2753e5f3e8b1cfe39b56f43611df74a"
    },
    "signature": "..."
  }
```

# Rollup Node Modifications Needed

The main modification for the rollup nodes is the requirement to use the Pipelines API.

## **Get Validator**

Endpoint: `/gmev/v1/validators` 

Queried on startup in order to acquire the list of registered validators within the GMEV-Boost instance. This is required in order for the Rollup node to determine when they are selected to be a sequencer and start the list building process.

## **Submit Condition**

Endpoint: `/gmev/v1/conditions`

Used for submission of sequencing transactions when one of the registered validators within GMEV-Boost is selected to be sequencer via the **primary selection mechanism** the rollup node, in addition to publicly broadcasting the sequencing transactions, should also submit them to the GMEV-Boost via the Submit Conditions endpoint.

# Further Research Avenues

- Secondary Pipelines Proposer-Builder Separation