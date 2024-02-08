# Types

## Undefined Types

Some primitive types are left undefined in the specification to allow implementation flexibility. They are:

- [Address](https://github.com/Consensys/qbft-formal-spec-and-verification/blob/clarify_specification_behaviour/dafny/spec/L1/types.dfy#L15C3) (e.g. the address of an Ethereum validator)
- [BlockBody](https://github.com/Consensys/qbft-formal-spec-and-verification/blob/clarify_specification_behaviour/dafny/spec/L1/types.dfy#L17C1-L17C1)
- [Transaction](https://github.com/Consensys/qbft-formal-spec-and-verification/blob/clarify_specification_behaviour/dafny/spec/L1/types.dfy#L19)
- [Hash](https://github.com/Consensys/qbft-formal-spec-and-verification/blob/clarify_specification_behaviour/dafny/spec/L1/types.dfy#L21)
- [Signature](https://github.com/Consensys/qbft-formal-spec-and-verification/blob/clarify_specification_behaviour/dafny/spec/L1/types.dfy#L23)

## Blockchain types



- [`Blockchain`](https://github.com/Consensys/qbft-formal-spec-and-verification/blob/clarify_specification_behaviour/dafny/spec/L1/types.dfy#L34): sequence of blocks.
- [`BlockHeader`](https://github.com/Consensys/qbft-formal-spec-and-verification/blob/clarify_specification_behaviour/dafny/spec/L1/types.dfy#L36): header of a block, containing basic information such as:
    - proposer's address
    - round number
    - height
    - timestamp
    - commit seals (list of signatures)
- [`Block`](https://github.com/Consensys/qbft-formal-spec-and-verification/blob/clarify_specification_behaviour/dafny/spec/L1/types.dfy#L44): a complete block with header, body and a sequence of transactions.
- [`RawBlockchain`](https://github.com/Consensys/qbft-formal-spec-and-verification/blob/clarify_specification_behaviour/dafny/spec/L1/types.dfy#L50): Represents a sequence of raw blocks.
- [`RawBlockHeader`](https://github.com/Consensys/qbft-formal-spec-and-verification/blob/clarify_specification_behaviour/dafny/spec/L1/types.dfy#L52): similar to block header but without round number and commit seals.
- [`RawBlock`](https://github.com/Consensys/qbft-formal-spec-and-verification/blob/clarify_specification_behaviour/dafny/spec/L1/types.dfy#L58): similar to a block but with a raw header.

# Message Types

It's defined messages for the QBFT protocol. It defines the following:

- [`UnsignedProposal`](https://github.com/Consensys/qbft-formal-spec-and-verification/blob/clarify_specification_behaviour/dafny/spec/L1/types.dfy#L77) containing:
    - height
    - round
    - digest (hash)
- [`UnsignedPrepare`](https://github.com/Consensys/qbft-formal-spec-and-verification/blob/clarify_specification_behaviour/dafny/spec/L1/types.dfy#L83): same as UnsignedProposal.
- [`UnsignedCommit`](https://github.com/Consensys/qbft-formal-spec-and-verification/blob/clarify_specification_behaviour/dafny/spec/L1/types.dfy#L89): same as UnsignedProposal but with an extra "commit seal" signature field.
- [`UnsignedRoundChange`](https://github.com/Consensys/qbft-formal-spec-and-verification/blob/clarify_specification_behaviour/dafny/spec/L1/types.dfy#L96) containing:
    - height
    - round
    - prepared value (optional hash)
    - prepared round (optional)

For every unsigned message, there is its signed form: [`SignedProposal`](https://github.com/Consensys/qbft-formal-spec-and-verification/blob/clarify_specification_behaviour/dafny/spec/L1/types.dfy#L103), [`SignedPrepare`](https://github.com/Consensys/qbft-formal-spec-and-verification/blob/clarify_specification_behaviour/dafny/spec/L1/types.dfy#L108), [`SignedCommit`](https://github.com/Consensys/qbft-formal-spec-and-verification/blob/clarify_specification_behaviour/dafny/spec/L1/types.dfy#L113), [`SignedRoundChange`](https://github.com/Consensys/qbft-formal-spec-and-verification/blob/clarify_specification_behaviour/dafny/spec/L1/types.dfy#L118).

The previous types are used to form [`QBFTMessage`](https://github.com/Consensys/qbft-formal-spec-and-verification/blob/clarify_specification_behaviour/dafny/spec/L1/types.dfy#L123) which can be:
- [`Proposal`](https://github.com/Consensys/qbft-formal-spec-and-verification/blob/clarify_specification_behaviour/dafny/spec/L1/types.dfy#L124): with
    - SignedProposal
    - Block (proposed)
    - Proposal justification (list of signed round changes)
    - Round change justification (list of Prepare)
- [`Prepare`](https://github.com/Consensys/qbft-formal-spec-and-verification/blob/clarify_specification_behaviour/dafny/spec/L1/types.dfy#L130): with only a SignedPrepare
- [`Commit`](https://github.com/Consensys/qbft-formal-spec-and-verification/blob/clarify_specification_behaviour/dafny/spec/L1/types.dfy#L133): with only a SignedCommit
- [`RoundChange`](https://github.com/Consensys/qbft-formal-spec-and-verification/blob/clarify_specification_behaviour/dafny/spec/L1/types.dfy#L136): with
    - SignedRoundChange
    - proposed block for the next round (optional)
    - Round change justification (list of Prepare)
- [`NewBlock`](https://github.com/Consensys/qbft-formal-spec-and-verification/blob/clarify_specification_behaviour/dafny/spec/L1/types.dfy#L141): just a block

There is also a more generic QBFT message with a recipient address [`QbftMessageWithRecipient`](https://github.com/Consensys/qbft-formal-spec-and-verification/blob/clarify_specification_behaviour/dafny/spec/L1/types.dfy#L145) that allows a node to specify the intended receiver of the message (not to confuse with fee recipient).


## State Types

State types representing the state of a QBFT node.

- [`Configuration`](https://github.com/Consensys/qbft-formal-spec-and-verification/blob/clarify_specification_behaviour/dafny/spec/L1/types.dfy#L161): Represents the configuration parameters for a QBFT network, including nodes' addresses, genesis block, and block time.
- [`NodeState`](https://github.com/Consensys/qbft-formal-spec-and-verification/blob/clarify_specification_behaviour/dafny/spec/L1/types.dfy#L170): main QBFT node state including:
    - blockchain
    - current round
    - local time
    - node's ID
    - configuration
    - received messages (list of QBFT messages)
    - proposal accepted for the current round (optional QBFT message)
    - last prepared block and round
    - start time of the last round

## Qbft Node Behaviour

It defines types that represent the lifecycle of a QBFT instance.

- [`QbftSpecificationStep`](https://github.com/Consensys/qbft-formal-spec-and-verification/blob/clarify_specification_behaviour/dafny/spec/L1/types.dfy#L212): represents a step which is defined by:
    - a list of received QBFT messages
    - a list of QBFT messages produced
    - a new Blockchain state
- [`QbftNodeBehaviour`](https://github.com/Consensys/qbft-formal-spec-and-verification/blob/clarify_specification_behaviour/dafny/spec/L1/types.dfy#L225): contains a Blockchain initial state and a list of _QbftSpecificationSteps_.