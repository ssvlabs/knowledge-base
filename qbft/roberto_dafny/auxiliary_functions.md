# Auxiliary functions

## Table of Contents
- [Auxiliary functions](#auxiliary-functions)
  - [Table of Contents](#table-of-contents)
  - [Cryptographic functions](#cryptographic-functions)
  - [Undefined functions](#undefined-functions)
  - [General functions](#general-functions)
  - [Proposal and Round-Change functions](#proposal-and-round-change-functions)
  - [Prepare validation](#prepare-validation)
  - [Commit validation](#commit-validation)
  - [Extra validation](#extra-validation)


## Cryptographic functions

This section defines cryptographic auxiliary functions such as:
- to produce a hash (digest)
- to sign QBFT messages
- to recover the signer from a signed message

## Undefined functions

Blockchain-specific functions such as:
- validate a raw block
- get the validators responsible for consensus
- get a new block

are left undefined for flexibility.

## General functions

Base QBFT auxiliary functions such as:
- calculating a quorum
- calculating a round timeout
- identifying if a node is a proposer for a QBFT round
- multicasting

## Proposal and Round-Change functions

Here, important `QBFT-related functions` are defined, such as functions and predicates used for Proposal and Round-Change message validation that H. Moniz defined in the [IBFT paper](https://arxiv.org/pdf/2002.03613.pdf). The most important are:

- [`validRoundChange(RoundChange, height, round, validators)`](https://github.com/Consensys/qbft-formal-spec-and-verification/blob/clarify_specification_behaviour/dafny/spec/L1/node_auxiliary_functions.dfy#L509): This predicate checks if the Round-Change is valid regarding an expected height and round, its signer and the correctness of the _prepared__ round_ and _prepared value_.

    **Note:** the list of Prepare messages contained in the Round-Change messages is not verified in this function. This is not wrong since Prepare messages are verified in the _isProposalJustification_ predicate (when the Round-Changed with the highest prepared value is selected). In the case that its prepared tuple is not supportable when searching for a quorum of Round-Changes such that the highest prepared tuple has support, this invalid Round-Change message won't be selected.

- [`isHighestPrepared(Round-Change, set<Round-Change>)`](https://github.com/Consensys/qbft-formal-spec-and-verification/blob/clarify_specification_behaviour/dafny/spec/L1/node_auxiliary_functions.dfy#L537): This predicate checks if the Round-Change message contains the highest prepared tuple of the list.

- [`isProposalJustification(set<Round-Change>,set<Prepare>, set<Block>, height, round, block, BlockValidationFunction, round leader, validators)`](https://github.com/Consensys/qbft-formal-spec-and-verification/blob/clarify_specification_behaviour/dafny/spec/L1/node_auxiliary_functions.dfy#L558): This predicate implements the _JustifyPrePrepare_ of H.Moniz paper. Thus, it does the following:
    - if the round is 0, just validate the block and check the block header fields
    - else, check that there's a quorum of Round-Change messages in the input and that each one is valid
    - then checks if every Round-Change has an empty _prepared_ tuple, just validate the block and check the block header
    -  else, check
        - if the block is in the set of available block
        - exists a Round-Change in the set that is the HighestPrepared (with the digest of the input block) and each Prepare message supports it (using the _validSignedPrepareForHeightRoundAndDigest_ function) and is valid

<br />

- [`IsValidProposal(QbftMessgae, NodeState)`](https://github.com/Consensys/qbft-formal-spec-and-verification/blob/clarify_specification_behaviour/dafny/spec/L1/node_auxiliary_functions.dfy#L615): This predicate performs the Proposal message validation. It checks if the signer is the expected leader, calls the _isProposalJustification_ predicate, checks if the proposed block digest is correct, and that the state variable _proposalAcceptedForCurrentRound_ was not set (if the proposal is for the current round).

- [`isReceivedProposalJustification(set<Round-Changes>, set<Prepares>, newRound, block, current NodeState)`](https://github.com/Consensys/qbft-formal-spec-and-verification/blob/clarify_specification_behaviour/dafny/spec/L1/node_auxiliary_functions.dfy#L654): This predicate implements the logic of the _JustifyRoundChange_ predicate of H.Moniz paper. It verifies if:
    - **the input Round-Changes are contained in the state's messages received** (this is important )
    - same for Prepare messages
    - _isProposalJustification_ is valid for the received block
    - if _proposalAcceptedForCurrentRound_ is set in the current state, then the "newRound" should be bigger than the current round. Else, should have the same value


<br />

- [`hasReceivedProposalJustificationForLeadingRound(NodeState)`](https://github.com/Consensys/qbft-formal-spec-and-verification/blob/clarify_specification_behaviour/dafny/spec/L1/node_auxiliary_functions.dfy#L692): This predicate implements the Round-Change message validation described in the H.Moniz paper. It checks if the node is the current leader and if there exists a set of Round-Changes, a set of Prepares, a new round and a Block that satisfies _isReceivedProposalJustification_.


## Prepare validation

- [`validSignedPrepareForHeightRoundAndDigest(Prepare, height, round, digest, validators)`](https://github.com/Consensys/qbft-formal-spec-and-verification/blob/clarify_specification_behaviour/dafny/spec/L1/node_auxiliary_functions.dfy#L769): It simply verifies correctness of message fields. Check if the signer is in validators and if it has an expected height, round and digest.

- [`validPreparesForHeightRoundAndDigest(seq<QbftMessgaes>, height, round, digest, validators)->set<QbftMessage>`](https://github.com/Consensys/qbft-formal-spec-and-verification/blob/clarify_specification_behaviour/dafny/spec/L1/node_auxiliary_functions.dfy#L790): This function returns a set of Prepare messages that have a certain height, round and digest.

## Commit validation

The Commit validation predicate and function
- [`validateCommit(Commit, height, round, block, validators)`](https://github.com/Consensys/qbft-formal-spec-and-verification/blob/clarify_specification_behaviour/dafny/spec/L1/node_auxiliary_functions.dfy#L822)
- [`validCommitsForHeightRoundAndDigest(seq<QbftMessgaes>, height, round, block, validators)->set<QbftMessage>`](https://github.com/Consensys/qbft-formal-spec-and-verification/blob/clarify_specification_behaviour/dafny/spec/L1/node_auxiliary_functions.dfy#L839)

are very similar to the Prepare validation.


## Extra validation

It also includes predicates for validating a new proposed block and the QBFT NodeState (e.g. assert that the prepared round and value should be both set or neither set, Prepare messages in the _messagesReceived_ container to support the prepared tuple, etc).