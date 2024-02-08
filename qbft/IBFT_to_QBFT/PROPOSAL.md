# Proposal / Pre-prepare

**Important note (!)**: The paper's "pre-prepare" message was renamed as the *proposal* message.

![IBFT_proposal](images/IBFT_preprepare.png)

*Moniz, H. The Istanbul BFT Consensus Algorithm. Algorithm 2. 2020*

**Pseudo-code:** when a proposal message is received, the node:
- checks if the message is valid ($\beta$ predicate),
- checks if the round leader is the sender of the message,
- verifies if it's a [justifiable proposal](MSG_JUST_FUNCTIONS.md) (it's the first proposal or a quorum of *Round-Change* and *Prepare* messages justifies the new proposal),
- if conditions hold, then start the timer for the current round,
- replies to the proposal broadcasting a *Prepare* message with the received value.


## Validation

In the implementation:
- **leader confirmation** is achieved by verifying that the author of the BLS signature is the actual leader for the indicated round.
- the **$\beta$ predicate** is verified as discussed [here](PROPERTIES.md).
- ***JustifyPrePrepare*** is performed by the *isProposalJustification* function.


## Body (UponProposal)

In the function's body, there are four differences to the paper specification.

1. The proposal message carries not only the (height, round, value) fields but also a list of Round-Change and a list of Prepare messages. Thus, in the *JustifyPrePrepare* function, instead of looking for a quorum received, the proposal message already provides such a quorum that justifies itself.
2. Besides the timer, the round attribute is also updated. The paper's algorithm performs this round update in the $f+1$ *round-change* rule. Here, such an update is checked in both rules. This is useful in the following scenario:
    - a node $j$ was temporarily isolated from the network and never received any Round-Change or Prepare messages.
    - since it never received $f+1$ Round-Change messages, it also never updated its *Round* attribute.
    - node $j$ receives a correct Proposal message from node $i$.
In such a scenario, the proposal message would justify itself and the *Round* attribute of node $j$ would be updated.
3. Instead of broadcasting the Prepare message with the value proposed, the hash of the value is used. This reduces communication complexity (number of bits exchanged).
4. At last, a variable *ProposalAcceptedForCurrentRound* is updated with the proposal value received. This allows the verification of incoming *Prepare* messages.