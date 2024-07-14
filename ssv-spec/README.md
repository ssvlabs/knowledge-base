# SSV Module

This document contains the specification of the [`SSV`](https://github.com/bloxapp/ssv-spec/) module.

## Table of Contents

- [SSV Module](#ssv-module)
  - [Table of Contents](#table-of-contents)
  - [Beacon Node](#beacon-node)
    - [Domain Calls](#domain-calls)
    - [Proposer Calls](#proposer-calls)
    - [Attester Calls](#attester-calls)
    - [Aggregator Calls](#aggregator-calls)
    - [Sync Committee Calls](#sync-committee-calls)
    - [Sync Committee Contribution Calls](#sync-committee-contribution-calls)
  - [Runner](#runner)
    - [Runner Interface](#runner-interface)
    - [Base Runner](#base-runner)
      - [State](#state)
        - [Partial Signature Container](#partial-signature-container)
    - [Proposer Runner](#proposer-runner)
    - [Committee Runner](#committee-runner)
    - [Aggregator Runner](#aggregator-runner)
    - [Sync Committee Aggregator Runner](#sync-committee-aggregator-runner)
    - [Validator Registration Runner](#validator-registration-runner)
  - [Validator](#validator)
    - [Duty runners](#duty-runners)
  - [Additional documents](#additional-documents)



## Beacon Node

The `Beacon node interface` is a composition of all interfaces below along with a method to get a Beacon Network object.

It comprises the interface, as an API style, between the operator implementation and the Beacon Network.

Its purpose is to provide a set of functions that the operator will use to interact with the Beacon Nodes, both for requesting data objects, and states, and for submitting signed objects to the network.

```mermaid
---
title: Beacon Node Interface
---
flowchart LR
    Validator([Validator])
    BeaconNode{Beacon Node Interface}
    BeaconNetwork[(Beacon Network)]

    Validator-->|Request Data|BeaconNode
    BeaconNode-->|Returns Data|Validator
    Validator-->|Submits Data|BeaconNode
    BeaconNode---BeaconNetwork
```

Each interface below will provide methods for getting and submitting data related to their duty type.

### Domain Calls

`DomainCalls` is an interface with a single method *`DomainData`* used to return a signature domain.

For each different duty, Ethereum adds a specific [salt](https://en.wikipedia.org/wiki/Salt_(cryptography)), the `DomainType`, to the hash of the object to be signed. Therefore, for different duty types, equal objects would have different hashes and, therefore, signatures. This is a common technique to prevent the Rainbow attack to hashing functions.

Similarly, SSV also defines a `DomainType`, defined by the network and its fork version, to be included in the signing process.

### Proposer Calls

`ProposerCalls` is an interface used for the *Proposer duty*. It defines functions for getting Beacon blocks, blinded Beacon blocks (blocks with only a transaction root instead of a full transactions list) and for submitting these blocks to the node.

### Attester Calls

`AttesterCalls` is an interface for the *Attestation duty*. It defines functions for getting the attestation data and for submitting it to the node.

### Aggregator Calls

`AggregatorCalls` is an interface for the *Aggregator duty*. It has functions to get an [AggregateAndProof object](https://github.com/ethereum/consensus-specs/blob/dev/specs/phase0/validator.md#aggregateandproof), that contains:
- Aggregator index (validator index)
- aggregate attestation
- BLSSignature selection proof,

and to submit a signed aggregator message.

### Sync Committee Calls

`SyncCommitteeCalls` is an interface for the *Sync Committee's duty*. It has a method to get beacon block roots and another method to submit a signed sync committee message.

### Sync Committee Contribution Calls

`SyncCommitteeCalls` is an interface for the *Sync committee aggregator duty*. It has:
- a predicate to check if it's an aggregator,
- a function to get the [subnet ID](https://github.com/ethereum/consensus-specs/blob/dev/specs/altair/validator.md#sync-committees) for a certain [subcommittee index](https://github.com/ethereum/consensus-specs/blob/dev/specs/altair/validator.md#subcommittee-index),
- a function to get the [Contributions object](https://github.com/ethereum/consensus-specs/blob/dev/specs/altair/validator.md#contributionandproof) and another one to submit the signed object.

## Runner
The runner is an entity responsible for performing validator duties for an operator. For that,
- it will collaborate with other operators to create signatures and
- it will count on the [Beacon node](#beacon-node) to get and send data to the blockchain.

For each duty, there is a specific runner since each duty requires a different sequence of steps, or subprotocol, to be followed by the operators. In general, it comprises three phases:
- Pre-consensus: a threshold signature is created on some structure for later getting specific data from the Beacon node.
- Consensus: decides on the data to send back to the Beacon node.
- Post-consensus: generates a threshold signature on the decided data and sends s signed object to the Beacon node.



### Runner Interface

The `Runner interface` establishes every method that a runner of any type should have. It includes functions to process pre-consensus, consensus and post-consensus methods and to execute duties.

### Base Runner

The `BaseRunner` structure represents a common ground for all different types of runners. It's comprised by:
- a [state](#state),
- *share* that contains information about the validators the node runs, and their committees.
- *committee member* that contains infomration about the operator itself seperated from the validators it runs.
- a QBFT controller to manage consensus instances,
- the Beacon network, its Beacon role and the highest decided slot number.

#### State

`RunnerState` stores the state of the runner during its execution. The state is composed of:
- a duty type,
- the duty-decided output,
- the consensus instance that decided the output and
- [partial signature containers](#partial-signature-container) for the pre-consensus and post-consensus steps.


##### Partial Signature Container

The `PartialSigContainer` structure stores partial signatures and performs validator signature reconstruction after a quorum is reached.

The reconstruction uses [Lagrange Interpolation](https://en.wikipedia.org/wiki/Lagrange_polynomial) to reconstruct a message signed with the shared secret.

This is crucial for DVT technology since it provides a way to construct a validator signature without any party ever having possession of the private key. This is accomplished by [Adi Shamir Secret Sharing](https://web.mit.edu/6.857/OldStuff/Fall03/ref/Shamir-HowToShareASecret.pdf) and the mathematical associative property of signing in [BLS signatures](https://www.iacr.org/archive/asiacrypt2001/22480516.pdf).

```mermaid
---
title: Signature reconstruction 
---
flowchart LR
    op1([Operator 1])
    op2([Operator 2])
    op3([Operator 3])
    op4([Operator 4])
    container[(PartialSigContainer)]
    signature{{Validator Signature}}
    op1-->|partial signature|container 
    op2-->|partial signature|container 
    op3-->|partial signature|container 
    op4-->|partial signature|container 
    container-->|Lagrange Interpolation|signature 
```

### Proposer Runner

To `propose a block`, the validator must:
- Fetch the head of the chain using the fork choice rule.
- Construct a [Beacon block](https://github.com/ethereum/consensus-specs/blob/dev/specs/phase0/beacon-chain.md#beacon-blocks).
- Sign the beacon block and broadcast it.

Block creation is abstracted by the [Beacon Node](#beacon-node) interface and the operator can just fetch the block from it.

However, there is a block field, *`randao_reveal`*, which is a signature of the validator over the epoch number. Thus, for the Beacon node to return the block, it needs to receive the signature first.

Thus the overall steps are:
1. Produce the *randao_reveal* by broadcasting and collecting partial signatures over the epoch number.
2. Send *rando_reveal* and get the Beacon block from the Beacon node interface.
3. Run consensus on the duty and block data.
3. Produce the signed Beacon block, by broadcasting and collecting partial signatures, and send it to the Beacon node.

```mermaid
---
title: Block proposer steps
---
%%{ init: { 'flowchart': { 'curve': 'linear' } } }%%
flowchart LR
        randao_cell(Construct Randao Signature)
        beaconnode(Get Beacon Block)
        Consensus(Consensus)

        signedBlock(Construct Block Signature)
        send(Send Signed Block to Beacon Node)


        randao_cell-->beaconnode
        beaconnode-->Consensus
        Consensus-->signedBlock
        signedBlock-->send

```


### Committee Runner

This runner creates several beacon objects from one consensus.
For the `attestation duty` and `sync committee duty`, the validator should construct an [AttestationData](https://github.com/ethereum/consensus-specs/blob/dev/specs/phase0/beacon-chain.md#attestationdata), which is composed by:
- a slot,
- committee index,
- a beacon block root (for the LMD GHOST vote)
- a source and target checkpoint (for the FFG vote)

From the *AttestationData* it can create an [Attestation](https://github.com/ethereum/consensus-specs/blob/dev/specs/phase0/beacon-chain.md#attestation) and [SyncCommitteeMessage](https://github.com/ethereum/consensus-specs/blob/dev/specs/altair/validator.md#synccommitteemessage).

The operator can rely on the [Beacon node](#beacon-node) to get the *AttestationData* object.

The overall steps are:
1. Get an *AttestationData* from the Beacon node.
2. Reach a consensus on the duty and the attestation data.
3. Construct the validator signature for the Attestation and Sync Committee objects and send it back to the Beacon node.

```mermaid
---
title:  steps
---
%%{ init: { 'flowchart': { 'curve': 'linear' } } }%%
flowchart LR
        attestationData(Get AttestationData)
        Consensus(Consensus)
        attestation(Construct Attestations Signatures)
        SyncCommittee(Construct Sync Committees Signatures)
        send(Send Attestations to Beacon Node)
        sendSync(Send Sync Committees to Beacon Node)
        

        attestationData-->Consensus
        Consensus-->attestation
        Consensus-->SyncCommittee
        attestation-->send
        SyncCommittee-->sendSync

```

### Aggregator Runner

For the `aggregator duty`, the validator must collect [AttestationData](https://github.com/ethereum/consensus-specs/blob/dev/specs/phase0/beacon-chain.md#attestationdata) from other attesters that are similar to its own created. Then, it should create an [AggregateAndProof](https://github.com/ethereum/consensus-specs/blob/dev/specs/phase0/validator.md#aggregateandproof) object, sign it and broadcast a [SignedAggregateAndProof](https://github.com/ethereum/consensus-specs/blob/dev/specs/phase0/validator.md#signedaggregateandproof) to the network.

The AggregateAndProof object can be obtained by the [Beacon node](#beacon-node). However, one of its fields is the validator's signature over the slot value (*`selection_proof`*). Thus, this field must be created and delivered to the Beacon node in order to get the AggregateAndProof object.

Thus, the overall steps are:
1. Construct the validator signature over the slot value (*selection_proof*).
2. Send it to the Beacon node and request the AggregateAndProof object.
3. Do consensus over the duty and the data.
4. Construct the validator signature on the consensus output.
5. Construct the SignedAggregateAndProof object and send it to the Beacon node.


```mermaid
---
title: Aggregator steps
---
%%{ init: { 'flowchart': { 'curve': 'linear' } } }%%
flowchart LR
        selectionProof(Construct selection_proof)
        aggregateAndProof(Get AggregateAndProof)
        consensus(Consensus)
        signedAggregateAndProof(Construct SignedAggregateAndProof)
        send(Send to Beacon node)

        selectionProof-->aggregateAndProof
        aggregateAndProof-->consensus
        consensus-->signedAggregateAndProof
        signedAggregateAndProof-->send
```


### Sync Committee Aggregator Runner

Similarly to an attestation aggregator, a validator with a `Sync Committee Aggregator` duty should collect [SyncCommitteeMessages](#sync-committee-runner), aggregate them into a [SyncCommitteeContribution](https://github.com/ethereum/consensus-specs/blob/dev/specs/altair/validator.md#synccommitteecontribution) and send it to the network.

To know if a validator is an aggregator for the current sync committee slot, it must compute its signature over a [SyncAggregatorSelectionData](https://github.com/ethereum/consensus-specs/blob/dev/specs/altair/validator.md#syncaggregatorselectiondata) (*`selection proof`*) and use it as input of a function *[is_sync_committee_aggregator](https://github.com/ethereum/consensus-specs/blob/dev/specs/altair/validator.md)*.

Using the SyncCommitteContribution data, it constructs a [ContributionAndProof](https://github.com/ethereum/consensus-specs/blob/dev/specs/altair/validator.md#contributionandproof) message, signs it to create a [SignedContributionAndProof](https://github.com/ethereum/consensus-specs/blob/dev/specs/altair/validator.md#signedcontributionandproof) and sends it to the network.

The [Beacon node](#beacon-node) provides the *is_sync_committee_aggregator* function and the ContributionAndProof object.

Thus, the operator's steps for this duty are:
1. Compute the validator signature over SyncAggregatorSelectionData (*selection proof*).
2. Ask the Beacon node if it's an aggregator. If not, stop.
3. If so, get the ContributionAndProof object with the Beacon node.
4. Compute the validator signature to create a SignedContributionAndProof and send it to the Beacon node.

```mermaid
---
title: Sync committee aggregator steps
---
%%{ init: { 'flowchart': { 'curve': 'linear' } } }%%
flowchart LR
        selectionProof(Selection Proof)
        checkAggregator(Check if it's aggregator)
        contribution(Get Contribution)
        contributionSignature(Construct validator signature)
        send(Send to Beacon node)

        selectionProof-->checkAggregator
        checkAggregator-->contribution
        contribution-->contributionSignature
        contributionSignature-->send
```

> **_NOTE:_** a validator may be part of multiple sync committees and, therefore, it should check if it's an aggregator for every committee that it's part of.

### Validator Registration Runner

**_NOTE:_** This is not an actual validator duty. However, here, it's considered a duty for MEV-Boost purpose, to set the validator's *fee_recipient* and *gas_limit* preferences.


To `register a validator`, it needs to create the operator needs to:
1. Get the ValidatorRegistration object.
2. Compute the validator signature over the object.
3. Submit the validator registration to the [Beacon node](#beacon-node).


```mermaid
---
title: Validator registration steps
---
%%{ init: { 'flowchart': { 'curve': 'linear' } } }%%
flowchart LR
        registrationObject(Get ValidatorRegistration)
        signature(Compute signature)
        send(Submit to Beacon node)

        registrationObject-->signature
        signature-->send
```

## Validator

The `Validator` entity represents a share of a validator that participates in the Ethereum Beacon chain consensus. Note that it's not the validator itself, but actually a virtual representation of a validator that the real operator entity will use.

The operator will use this structure to execute validator-related duties, managed by [DutyRunners](#duty-runners).

### Duty runners

`DutyRunners` is a map: Beacon roles (as proposer, attester, etc) &rarr; [Runner](#runner). Each duty type has its unique runner.


## Additional documents
- [Class Diagram](docs/CLASS_DIAGRAM.md)
