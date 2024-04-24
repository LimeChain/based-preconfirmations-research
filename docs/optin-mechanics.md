# Sequencer Opt-In, Discovery and Communication

# TLDR

- The L1 proposers opt-in to be sequencers via two-step process - register and activate. Sequencers are eligible to be selected as sequencers after some time delay from their activation.
- Registration requires a signed message by the proposer validator BLS keypair in order to authenticate the request. Furthermore the registration requires a proof that the validator is still active.
- Registered sequencers advertise an RPC url to be connected to and an execution layer address to be representing them when sequencing.
- Sequencing transactions are accepted optimistically and rollups are to decide whether to enforce further sequencer eligibility validation for the sequencing or implement challenge period. If Ethereum L1 exposes a direct access from the execution layer to the L1 proposer BLS pubkey, the challenge period can be replaced with validation of the sequencer selection on submission.
- If a L1 proposer exits the L1 validator set, a forced deactivation can be triggered by any actor.

# Opt-in Mechanics

In order for a L1 proposer to become eligible to be selected as sequencer of the rollup, they must register themselves as L2 sequencer. The registration process includes:

- Register, advertise their RPC url, their validator BLS public key and their representing address.
- Activation of the registered sequencer upon meeting rollup defined activation criteria (i.e. stake)

Both these actions are performed onchain in  L1 through smart contracts.

## Sequencers Registry Contract

The sequencer registry contract is the contract keeping track of the current registered sequencers, their metadata and their status.

```solidity
interface ISequencerRegistry {
    struct ValidatorProof {
        uint64 currentEpoch;
        uint64 activationEpoch;
        uint64 exitEpoch;
        uint256 validatorIndex;
        bool slashed;
        uint256 proofSlot;
        bytes sszProof;
    }

    struct Sequencer {
        bytes pubkey;
        bytes metadata;
        address signer;
        uint256 activationBlock;
        uint256 deactivationBlock;
    }

    /**
     *     Registers the sequencer without activating them.
     *
     *     Authorised operation. Requires a signature over a digest by the validator via their BLS pubkey.
     *     Requires EIP-2537 in order to verify the signature.
     *
     *     Rollup contracts will use the signer address when enforcing the primary or secondary selection.
     *     The signature must authenticate the validator and must be over an authorisation hash.
     *     The authorisation hash must be verifiable by the contract and must include a nonce in order to guard against replay attacks.
     *     The nonce derivation is a decision of the implementer, but can be as simple an incremental counter.
     *     The implementation MUST check if the authHash is a keccak256(protocol_version,contract_address,chain_id,nonce).
     *
     *     @param signer - the secp256k1 wallet that will be represeting the sequencer via its signatures
     *     @param metadata - metadata of the sequencer - including but not limited to version and endpoint url
     *     @param authHash - the authorisation hash - keccak256(protocol_version,contract_address,chain_id,nonce). The authorisation signature was created through signing over these bytes.
     *     @param signature - the signature over the authHash performed by the validator key
     *     @param validatorProof - all the data needed to validate the existence of the validator in the state tree of the beacon chain
     */
    function register(
        address signer,
        bytes calldata metadata,
        bytes32 authHash,
        bytes calldata signature,
        ValidatorProof calldata validatorProof
    ) external;

    /**
     *     Changes the sequencer signer and/or metadata.
     *
     *     Authorised operation. Similar requirements apply as in `register`
     *
     *     @param signer - the new wallet that will be represeting the sequencer via its signatures
     *     @param metadata - the new metadata of the sequencer - including but not limited to version and endpoint url
     *     @param authHash - the authorisation hash - keccak256(protocol_version,contract_address,chain_id,nonce). The authorisation signature was created through signing over these bytes.
     *     @param signature - the signature over the authHash performed by the validator key
     */
    function changeRegistration(address signer, bytes calldata metadata, bytes32 authHash, bytes calldata signature)
        external;

    /**
     *     Activates the sequencer finalising the registration process
     *     Implementers must make sure that the sequencer meets the activation (i.e. stake) requirements before changing their status.
     *
     *     @param pubkey - the validator a BLS12-381 public key - 48 bytes
     *     @param validatorProof - all the data needed to validate the existence of the validator in the state tree of the beacon chain
     */
    function activate(bytes calldata pubkey, ValidatorProof calldata validatorProof) external;

    /**
     *     Deactivates the sequencer.
     *
     *     Authorised operation. Similar requirements apply as in `register`.
     *     Implementers of the staking process must make sure that the sequencer is no longer active before withdrawal disbursal
     *
     *     @param authHash - the authorisation hash - keccak256(protocol_version,contract_address,chain_id,nonce). The authorisation signature was created through signing over these bytes.
     *     @param signature - the signature over the authHash performed by the validator key
     */
    function deactivate(bytes32 authHash, bytes calldata signature) external;

    /**
     *     Forcefully deactivates a sequencer.
     *
     *     The caller must provide a proof that the validator is no longer active or has been slashed.
     *
     *     @param pubkey - the validator a BLS12-381 public key - 48 bytes
     *     @param validatorProof - all the data needed to validate the existence and state of the validator in the state tree of the beacon chain
     */
    function forceDeactivate(bytes calldata pubkey, ValidatorProof calldata validatorProof) external;

    /**
     *     Used to get the eligibility status of the sequencer identified by this pubkey
     *     @param pubkey - the validator a BLS12-381 public key - 48 bytes
     */
    function isEligible(bytes calldata pubkey) external view returns (bool);

    /**
     *     Returns the saved data for the sequencer identified by this pubkey
     *     @param pubkey - the validator a BLS12-381 public key - 48 bytes
     */
    function statusOf(bytes calldata pubkey) external view returns (Sequencer memory metadata);

    /**
     *     Used to get the activation status of the sequencer with this signer address
     *     @param signer - the associated signer address of a sequencer
     */
    function isEligibleSigner(address signer) external view returns (bool);

    /**
     *     Returns the data for a sequencer by its index
     */
    function sequencerByIndex(uint256 index)
        external
        view
        returns (address signer, bytes memory metadata, bytes memory pubkey);

    /**
     *     Number of Blocks after activation that the sequencer becomes eligible for sequencing
     */
    function activationTimeout() external view returns (uint8);

    /**
     *     Number of Blocks after deactivation that the sequencer becomes ineligible for sequencing
     */
    function deactivationPeriod() external view returns (uint8);

    /**
     *     Returns the total count of sequencers at this block number
     */
    function eligibleCountAt(uint256 blockNum) external view returns (uint256);

    /**
     *     Returns the protocol version used for authorising the digests.
     */
    function protocolVersion() external view returns (uint8);
}
```

### Authorisation

Some of the functions in the repository must only be triggered by the L1 validators themselves. A signature - `signature` - produced through their BLS keypair is used for authentication of the validator. The signature is over 32 bytes - `authHash`.

The implementations can choose what `authHash` is. A suggestion is `keccak256(protocol_version,contract_address,chain_id,nonce,function_identifier,params_hash)` where:

- `protocol_version`  - version of the authorisation protocol
- `contract_address` - the address of the contract intended to be authorising this message - the registry contract
- `chain_id` - the EIP155 chain id of the intended network - `mainnet`
- `nonce` - a number used once as anti replay attack mechanism.
- `function_identifier` - the function identifier of the intended function
- `params_hash` - keccak256 hash of the function parameters - f.e. `signer` and `metadata`

The implementations must recover the 48 bytes BLS12 pub key - `pubkey` - of the caller through the signature recovery process over the `signature` and `authHash`. **Note that BLS signature recovery is not possible until [EIP-2573](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-2537.md) is included (currently scheduled for Pectra).**

Next, the contract must check the existence of the validator in the current validator set. The implementations can achieve that via two-step process:

1. Obtaining the parent beacon chain hash tree root - `beaconRoot` through the contract at `BEACON_ROOTS_ADDRESS` introduced with EIP4788 in the Dencun upgrade for the slot specified through the `proofSlot` parameter.
2. Verifying that the `pubkey` belongs to an active validator through [SSZ Merkle inclusion multiproofs](https://github.com/ethereum/consensus-specs/blob/fa09d896484bbe240334fa21ffaa454bafe5842e/ssz/merkle-proofs.md#merkle-multiproofs) with the `beaconRoot` and the provided `validatorProof`, and performing the checks of [is_slashable_validator](https://github.com/ethereum/consensus-specs/blob/fa09d896484bbe240334fa21ffaa454bafe5842e/specs/phase0/beacon-chain.md#is_active_validator) function specified in the consensus spec.

### Registration

A L1 proposer can start the registration process by triggering the `register` method. This method requires authorisation and should comply with the authorisation protocol outlined above.

Upon successful check, the implementation must create an entry for the sequencer - `Sequencer`. It should associate its `signer` address and `metadata` bytes with the `pubkey`. The `signer` is the address that the new sequencer will use to operate. It will be the expected sender address when sequencing and committing to preconfirmations.

The `metadata` is information about the sequencer and how the external parties can connect to it. It should include but is not limited to version and endpoint url.

### Activation

A registered sequencer must activate before becoming eligible for selection. The separation between registration and activation allows the rollup protocol to embed specific activation rules. An example of such activation rules can be a minimum stake posted by the sequencer or activation delay.

The activation is performed via the `activate` method. Implementers must check that the activation rules are met. Similarly to registration, mandatory check is that the validator is active. Additional checks might include calling a staking contract to check if stake has been posted. 

If all checks are passed the implementations must set the sequencer `activationBlock` to the current block number and include it in the list of sequencers.

Note: While the sequencer is active it is not yet eligible for selection as activation timeout needs to pass. Its role is similar to the activation period enforced by the beacon chain.

### Changing Registration & Deactivation

In case a registered sequencer wants to change their registration - i.e. updating their metadata - they can call the authorised `changeRegistration` method. The implementations must associate the new `signer` and `metadata` with the `pubkey`.

In case a registered sequencer wants to stop being a sequencer they can call the authorised `deactivate` method. The implementations must change the sequencer `deactivationBlock` to the current block number so that interfacing contracts can enforce withdrawal timeouts.

Note: While the sequencer is deactivated it can still be selected by the selection algorithms until deactivation timeout period passes. Deactivated selected sequencers will miss their slots.

### Withdrawn Validators

Having the sequencers be an active L1 validators is an important property of vanilla based sequencing. Within primary selection it ensures the highest level of censorship resistance over sequencing transaction inclusion. 

One possible corner-case scenario sees an opted-in sequencer exit the Ethereum validator set or be slashed. In order to cover for this case a publicly available function `forceDeactivation` is provided. The caller can supply a proof that the validator is no longer active. The implementations must change the sequencer `deactivationBlock` to the current block number.

Note: While the sequencer is deactivated it can still be selected by the selection algorithms until deactivation timeout period passes. Deactivated selected sequencers will miss their slots.

### Activation and Deactivation Timeout

In order to ensure predictable lookahead sequencer queue, the randomness selection algorithm for fallback selection requires a stable total count of eligible validators during the lookahead period. This requires the enforcement of activation and deactivation timeouts. Choosing the number of blocks `X` timeout, effectively specifies the lookahead queue of proposers.

Implementations need to make sure that they respond to the `function eligibleCountAt(uint256 blockNum)` method considering both the `activationBlock` of newly activated sequencers and the activation timeout. 

Similarly, implementations of the fallback mechanism must consider sequencers eligible within timeout blocks after their `deactivationBlock` and the deactivation timeout.

## Making Sequencing Contracts “Vanilla Based”

All rollups have their own sequencing contracts. The following paragraphs list modifications to the rollup contracts that need to be done in order to support vanilla based sequencing.

### Deterministic Fallback Selection

The rollup contract must implement the fallback selection for sequencers - selecting a random sequencer from the opted-in proposers.

In order for this selection to be replicated offchain and provide sufficient lookahead to sequencers and followers alike, the selection must be based on a property known in advance. 

An example could be taking the `blockhash` of the `X`-th parent block of the current block as a seed to a randomness algorithm. Using `blockhash` is a good candidate as one of its constituents is the beacon chain randomness beacon and is hard to manipulate. The parameter `X` specifies the lookahead provided by the rollup to the following nodes.

In order to select the sequencer the `SequencerRegistry` contract provides two methods - `eligibleCountAt` and `sequencerByIndex`. Through, these two methods a random active sequencer from the currently eligible sequencers set is selected. Similarly to the [beacon chain proposer selection algorithm](https://github.com/ethereum/consensus-specs/blob/fa09d896484bbe240334fa21ffaa454bafe5842e/specs/phase0/beacon-chain.md#compute_proposer_index), if a deactivated sequencer is selected, the algorithm must retry until an active eligible one is selected.

### Optimistic Selection Algorithm

One of the major requirements of vanilla based rollups is to implement the primary and fallback sequencer selection algorithms. These are used by the sequencers to identify if and when they are selected to be sequencer in one of the two modes. 

An intuitive approach can see the L1 smart contracts requiring sequencing transactions from sequencers that are 1) not current L1 proposers in case of primary selection and 2) not the selected sequencer in case of fallback selection. Such an approach, however, is currently not implementable within the Ethereum L1 - there is lack of information within the execution layer about the current block proposer.

Firstly, the current proposer cannot be proven via SSZ multiproof due `BEACON_ROOTS_ADDRESS` being the bacon tree hash root of the parent block - thus the information about the current slot is not yet available.

Secondly, the address available via the `COINBASE` opcode cannot be deterministically linked with the proposer. Since the merge this opcode returns the fee recipient set by the block builder, [but this is not necessarily the proposer](https://github.com/ethereum/execution-apis/blob/main/src/engine/paris.md#payload-building). 

Both of these obstacles prohibits the rollup contracts to reject sequencing submissions in-flight. Implementation of such an approach requires an L1 hard fork to expose the current block proposer within the execution layer.

In order to tackle this, it is suggested to implement an optimistic selection - performing the selection algorithm off-chain while allowing for proving and/or punishing submissions by ineligible sequencers afterwards. There are several arguments for this choice.

First, an honest following node will always know off-chain who is the valid sequencer and whose sequence they should apply to their state.

Second, such an optimistic approach “optimises the happy path”. If the sequencers are acting honestly, only one sequencer will submit a sequencing transaction - the selected one. 

In order to account for malicious cases two workarounds can be chosen. 

First option is to introduce challenge period (something existing in the optimistic rollups already). If a sequencer acts maliciously and submits sequencing transaction when not selected as a sequencer their submission can be proven ineligible (through SSZ merkle multiproof) and the sequencer can be punished and the challenger is rewarded.

Second option is to introduce validation function in the subsequent blocks (or along-side the proving transaction for zk rollups). This transaction aims to provide eligibility proof (through SSZ merkle multiproof) for the sequencer - proving that they were the correctly selected sequencer.

Both options introduce drawbacks to the sequencing flow, but are implementable immediately. It is suggested that if/when Ethereum L1 provides a way for the execution layer to access the current block proposer, the implementations switch to a safer validation approach.

Regardless of the chosen optimistic selection option, the two selection types require their distinct ways of proving eligibility/ineligibility. Within the context of the primary selection, the proof must be a SSZ multiproof that the sequencer is/is not the proposer of the slot that the sequencing transaction came in. Within the context of the fallback selection, the selection verification can be achieved deterministically by the L1 smart contracts through performing the deterministic fallback selection algorithm outlined in the previous section.

### Rollup Contract Requirements

- MUST accept all sequences by signers of active sequencers. The sequences must indicate which rollup slot they are aimed for.
- MUST implement a deterministic fallback selection algorithm.
- MUST honour activation and deactivation timeouts when performing selection.
- MUST provide methods for submission and verification of a proof of eligible/ineligible sequence.
- MUST enforce punishment for ineligible submissions and reward for valid ineligible sequence proof submission.

## Staking and Interfacing with Staking Contracts

Many rollups have their own sequencing contracts. This stake is normally used as crypto-economical incentive for correct behaviour. 

### Interfacing with SequencingRegistry

While staking is not a requirement for a rollup to be “Vanilla Based” (bonds can be utilized as-well) the suggested `SequencerRegistry` interface is designed with staking in mind.

Firstly, the separation between registration (performed via `register`) and activation (performed via `activate`) provides a useful entry point for staking contracts. The rollup staking contract can only permit the triggering of activate if the staking requirements are met.

Secondly, the `deactivationBlock` field within the `Sequencer` data enable the staking contracts to enforce withdrawal conditions.

### Notes on Staking

In this section you can find several important notes and considerations when implementing a rollup staking mechanism. 

Firstly, vanilla based sequencing is not dependent on the asset and amount being staked. It is the design decision of the rollup which asset and how much of it should be staked in order to meet the activation criteria. Furthermore, the rollup designers can opt for a more complex staking mechanism enabling stake delegation and/or validator restaking (i.e. EigenLayer-style restaking and LST).

Secondly, it is important to consider the time to finality when releasing the stake of deactivated validators. As the finality is when most of the misbehaviour is detected (fraud proofs or proving invalidity) it is important to consider a grace period after deactivation before making the stake withdrawable.

# Sequencer Discovery & Communication

Important part of the flow of interaction with a vanilla based rollup is the discovery of the subsequent sequencers. In this section it is outlined of the mechanism to discover and communicate with the subsequent sequencers.

## Sequencer Discovery

In order for a wallet or a user to discover the current sequencer and the sequencers in the lookahead queue they need to get several pieces of information from the L1 beacon chain and the rollup `SequencerRegistry`. 

First, the wallet needs to determine what are the known L1 proposers for the next `X` slots - with `X` being a parameter set by the rollup team and should ideally be a single epoch (32 slots).

Second, the wallet needs to get the list of current eligible opted-in L1 validators and cross-check it against the next X L1 proposers. Any match can be considered a primary selected sequencer - the L1 proposer will be the L2 sequencer in this slot.

For slots where the L1 proposer has not opted in to be a sequencer, the wallet needs to run the deterministic fallback selection algorithm as specified in the “Deterministic Fallback Selection” section above. Keep in mind that these sequencers will be revealed `X` slots ahead of time. The randomly selected L2 sequencers will be the sequencers for the respective slots.

Last, the wallet must obtain the `metadata` information for the sequencer they want to communicate with.

## Sequencer Communication

As part of the metadata the sequencers must communicate an RPC endpoint. This endpoint must be the RPC endpoint for transaction submission by the sequencer and the requests it should service should include, but are not limited to:

- Sending raw transaction
- Getting preconfirmation commitments or rejections
- Any other rollup specific methods.

As slots change, the sequencers URLs are going to change too. This is a complexity that is to be handled by the wallets software.