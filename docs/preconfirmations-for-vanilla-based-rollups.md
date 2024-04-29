# Preconfirmations for Vanilla Based Rollups

[![hackmd-github-sync-badge](https://hackmd.io/hzvWsIVhS7qHCI5Nwqwelg/badge)](https://hackmd.io/hzvWsIVhS7qHCI5Nwqwelg)


# TLDR

One major component of UX where centralised sequencers excel is the ability to quickly pre-confirm the inclusion and/or execution of a transaction to its sender. It is a naturally important requirement for vanilla based sequencing to strive to reach and surpass the expected UX.

This document outlines the design and mechanics to support preconfirmations in Vanilla Based Rollups. It introduces two new transaction types in order to support the two preconfirmation types:

- Inclusion Preconfirmation - commitment to the inclusion of a transaction in a sequence
- Execution Preconfirmation - commitment to the inclusion and the resulting end state after the transaction. The constraints are specified as limited lists of the desired resulting storage slots, account balances and code hashes

In order to facilitate predictable fees, both transaction types will have their own predictable pricing - as a specific preconfirmation premium percentage on top of the EIP base fee.

In order to achieve UX on par with centralised sequencing, the two transaction types include a `deadline` field to enable the wallets to retry preconfirmation requests without involving the user. 

In order to incentivise the correct behaviour by sequencers, the design outlines signature-based commitment to preconfirmations - a signature of the transaction hash and the L1 block the committed transaction should appear.

# Goals

The main goals of the preconfirmations are in line with the overall vanilla based sequencing goals. Preconfirmations are an important pillar of the UX of a rollup and are becoming the standard expected by the users. The design aims to keep and improve this standard through:

- Support of quick preconfirmations of inclusion and execution
- Predictable pricing of preconfirmations
- Allowing wallets and nodes to perform most of the complex tasks under the hood
- Offer cryptography-based rather than a social commitment to preconfirmations

# Types of Preconfirmations
![Preconfirmation Flow - Simple](https://hackmd.io/_uploads/S11bFh5y0.png)

As suggested in the [Vanilla Based Sequencing](https://github.com/LimeChain/based-preconfirmations-research/blob/main/docs/vanilla-based-sequencing.md) design, two types of preconfirmations are expected of the system.

The first type of preconfirmation is transaction **inclusion** preconfirmation. This preconfirmation guarantees the inclusion of a transaction in the subsequent rollup slot. These are useful for scenarios like simple transfers.

The second type of preconfirmation is the stronger **execution state** preconfirmation. It allows specifying the desired values of parts of the state of the rollup after execution of the transaction. These are useful for more complex use cases like DEX trades and/or arbitrage.

# Preconfirmation Transactions

In order to support the two types of preconfirmations the design suggests introducing two new [EIP2718](https://eips.ethereum.org/EIPS/eip-2718) transaction types specific to vanilla based sequencing. 

The two transaction types serve as preconfirmation requests and require the sequencer to respond with a preconfirmation commitment or rejection. These two transactions will enable the sequencers and wallets alike to differentiate the request and its expected properties. Having two separate transactions offers several distinct advantages.

Firstly, two transaction types will enable the two types of preconfirmations to have separate tailor-made fields for their use case. Secondly, two transaction types will enable the transaction pricing mechanism to work separately and find the correct pricing for each. Lastly, having the transaction types separate allows the rollups to choose to support one or both of them.

## Preconfirmation Deadline

Both transaction types must include a field - `deadline`. Deadline indicates the latest possible L1 slot that this preconfirmation can be honoured.

The goal of the `deadline` field is to further optimize the UX and enable wallets and/or nodes to hide complexity away from the user. There are two scenarios where the `deadline` field can optimize UX.
![Preconf Flow - deadline.drawio](https://hackmd.io/_uploads/Hy_NYhcJ0.png)

Firstly, a preconfirmation request arriving late in the rollup slot of the sequencer might be rejected by the sequencer due to insufficient time to simulate and commit to the transaction. An inferior UX (especially considering hardware wallets) would be to re-request the user to sign the transaction. Having a `deadline` field allows the wallet to reuse the same signed transaction and send it to the next sequencer for preconfirmation.

Secondly, a reneged preconfirmation has an inferior UX. Having a `deadline` field enables a wallet that has detected preconfirmation reneging to re-submit the preconfirmation transaction to the next sequencer.

In both cases, the `deadline` field enables the wallet software to hide away the complexity of resubmission, without requiring further action by the user. On the sequencer side, the `deadline` introduces a simple check if the `deadline` has passed before committing to it.

## Preconfirmation Pricing

Both transaction types will have separate transaction pricing. The mechanism of pricing will be similar to EIP1559. This type of pricing enables easy price discovery by wallets and keeps the UX on par with centralised sequencing.

Preconfirmed transactions are fundamentally a higher-priced service compared to type 2 or legacy TX. Firstly, they require additional processing to handle. Secondly, they have an opportunity cost to the sequencer due to block building restrictions. 

In order to account for this, each new transaction type will be priced as a percentage `premium` on top of the type 2 base fee. This way the base fees for all transactions will be constantly discovered and dynamically adjusted by total usage of the protocol in order to provide stable, predictable transaction fees.  

The `premium` parameter for each transaction must be embedded in the protocol and be a percentage increase on top of the base fee. 

Let `inclusion_preconfirmation_fee_premium` and `execution_preconfirmation_fee_premium` be the targeted preconfirmation transactions premiums.

The respective `base_fee_per_gas` that a transaction must pay for each of the two transactions can be calculated as `*_preconfirmation_base_fee_per_gas = base_fee_per_gas * (1 + *_preconfirmation_fee_premium)`.

Similarly to EIP1559, if the gas used by the block exceeds the block target the base fee for all types of transaction goes up, and vice versa.

## Inclusion Preconfirmation

The inclusion preconfirmation transaction follows the same structure as ethereum EIP1559 type 2 transaction with the following additional field:

- `deadline` -  `int` indicating the latest L1 block that this preconfirmation request is valid to be committed and included in.

### Composing the Inclusion Preconfirmation

As the only differentiating field of the inclusion preconfirmation compared to type 2 transactions is the `deadline` field, the main changes needed in order to support inclusion preconfirmation transactions are within the dapps and wallets.

On the dapps side, the new transaction type needs to be supported, suggesting a deadline field. On the wallet side, the process of showing and signing the transaction must include the new field.

## Execution State Preconfirmation
![Execution Preconf.drawio](https://hackmd.io/_uploads/BJPLY3qyR.png)

The execution preconfirmation transaction follows the same structure as ethereum EIP1559 type 2 transaction.

The following fields are added on top of the type 2 fields:

- `deadline` - `uint256` indicating the latest L1 block that this preconfirmation request is valid to be committed and included in.
- `storage_list` - List of objects mapping an `address` to a list of tuples `(bytes32, bytes32)` of storage slot `bytes32` and its expected value.
- `code_list` - List of tuples `(address, bytes32)` of account address and its expected codehash
- `balance_list` - List of tuples `(address, uint)` of account address and its expected balance

### Lists Size Limiting

If left unbounded, all three constraining lists can greatly expand the transaction size. A greatly expanded transaction size is undesirable as it can lead to increased bandwidth and processing requirements for the nodes.

In order to combat this, the number of items in each list must be limited by the protocol. The specific limit is subject to a decision by the rollup protocols themselves.

### Lists Examples

`storage_list` example:

```jsx
[
    [
        "0xde0b295669a9fd93d5f28d9ec85e40f4cb697bae", // Account address
        [
            [// Tuple (slot, value) 
	            "0x0000000000000000000000000000000000000000000000000000000000000003", 
	            "0x000000000000000000000000000000000000000000000000000000000000000a"
            ],
            [
	            "0x00000000000000000000000000000000000000000000000000000000000000e7",
	            "0x000000000000000000000000000000000000000000000000000000000000000b"
            ]
        ]
    ],
    [
        "0xbb9bc244d798123fde783fcc1c72d3bb8c189413",
        [
            [
	            "0x0000000000000000000000000000000000000000000000000000000000000002",
	            "0x000000000000000000000000000000000000000000000000000000000000000c"
            ],
            [
	            "0x00000000000000000000000000000000000000000000000000000000000001e9",
	            "0x000000000000000000000000000000000000000000000000000000000000000d"
            ]
        ]
    ]
]
```

`code_list` example:

```jsx
[
	[
		"0xde0b295669a9fd93d5f28d9ec85e40f4cb697bae", // Account Address
		"0x0000000000000000000000000000000000000000000000000000000000000be9" //Code Hash
	],
	[
		"0xde0b295669a9fd93d5f28d9ec85e40f4cb697bae",
		"0x0000000000000000000000000000000000000000000000000000000000000cd8"
	]
]
```

`balance_list` example:

```jsx
[
	[
		"0xde0b295669a9fd93d5f28d9ec85e40f4cb697bae",
		"0x0000000000000000000000000000000000000000000000000000000000000abc"
	],
	[
		"0xde0b295669a9fd93d5f28d9ec85e40f4cb697bae",
		"0x0000000000000000000000000000000000000000000000000000000000000def"
	]
]
```

### Composing the Execution Preconfirmation

The process of composing the execution preconfirmation transaction is ultimately a responsibility of the user facing layers - the dapp and the wallet developers. Two types of user personas can be defined as the archetypical users of the execution preconfirmations - Regular dapp users (unsophisticated dapp users wanting to constraint the end result - e.g. outcome of a trade) and sophisticated users (e.g. arbitrage bots).

With regular dapp users ultimately the intent of the user is only well understood within the context of the dapp. Within its context, it is only possible to correctly constrain the desired execution outcome via the dapp application itself.

With sophisticated users, the intent is complex and is acceptable to assume that they can take care of the correct constraining of their desired execution outcome.

# Preconfirmation Flow - Submission & Commitment
![Preconfirmations Flow-Combined.drawio](https://hackmd.io/_uploads/SJMFK251R.png)

## Preconfirmation Submission

The preconfirmation transactions act as a request for preconfirmation from the user to the L2 sequencer. This request needs to be responded to by committing or rejecting the request.

Preconfirmations will continue reusing the standard way of submitting a transaction towards an RPC is to use the `eth_sendRawTransaction` method. As part of the specification of the rpc call it can either return a transaction hash or zero hash. 

A zero hash response must be treated as an indication of preconfirmation request rejection. A non-zero hash response must be treated as an indication of preconfirmation request acceptance.

## Preconfirmation Commitment

In order to commit to preconfirmation, a user receiving a non-zero response should obtain a preconfirmation commitment. This can be done through a newly introduced RPC method `eth_getCommitment` . 

The response of this method must be `signed_commitment` - bytes representing a signed commitment. 

`signed_commitment = sign(tx_hash, l1_blocknumber)`

The constituents of the commitment are:

- `tx_hash` - the transaction hash of the preconfirmation transaction
- `l1_blocknumber` - the L1 block number that should contain the sequencing transaction that includes the preconfirmed transaction.

One way that a sequencer can renege on their promise is to omit the transaction they have promised to include. This is a punishable offence and requires a renege detection mechanism.

While the specific mechanism of renege detection and proving (e.g. ZKP, optimistic challenge, attestations) is ultimately the design decision of the rollup, the `signed_commitment` provides a strong foundation for it.

# Additional considerations

## Lack of Timely Service Incentive

A rational block builder has a natural incentive to withhold commitments until the end of their monopoly slot in order to maximise their profit. This will effectively render preconfirmations slow and will hurt the UX of the rollup.

While this kind of behaviour is hard to detect and enforce punishment for, this document suggests an **optional** “Robustness incentive” approach by the rollups. Such an approach can see additional revenue (e.g. as part of the rollup revenue) be offered to the sequencers that are consistently offering high quality service.

## Synchronous Processing of Execution Preconfirmations

In order for a sequencer to commit to an execution preconfirmation, they need to ensure that the state prior to the execution of the transaction allows for the specified resulting conditions to occur. During the rollup slot, the pre-state conditions are subject to change by other transactions, and more importantly by other execution preconfirmations. This means that in order to avoid committing to conflicting preconfirmations, the sequencer needs to process the execution preconfirmation transactions synchronously rather than asynchronously.

While this synchronous processing can introduce a bottleneck, recent developments of the execution client EVM implementations offer increased processing speeds to the point that synchronous processing of these transactions won't affect the overall performance of the sequencer and the system.

Processing execution preconfirmations will not introduce a bottleneck at least initially. Sequencers are technically able to process execution preconfirmations at a rate of  [~200MGas (>1k TPS)](https://www.paradigm.xyz/2024/03/reth-beta#how-much-gas-and-transactions-per-second-tps-can-reth-support)  (excluding L1 sequencing and proving). That's ~40x more than the current demand based on [L2Beat data](https://l2beat.com/scaling/activity).

# Preconfirmation Requirements vs Rollup Design Decisions

## Rollup Requirements

- Protocol support of the new types of transactions
- Support for `eth_getCommitment` RPC call

## Rollup Design Decisions

- Preconfirmation transaction base fee premiums.
- Execution Preconfirmation lists limits
- Renege Detection and proving mechanism