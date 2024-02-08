# Message Justification Functions


## HighestPrepared

![IBFT_highest_prepared](images/IBFT_HighestPrepared.png)

*Moniz, H. The Istanbul BFT Consensus Algorithm. Algorithm 4. 2020*


**Pseudo-code:** the function receives a quorum of *Round-Change* messages and returns a tuple (*prepared_round*,*prepared_value*) that corresponds to the highest prepared round and its value.

In the implementation, the only difference is the return value. Instead of returning the tuple (*prepared_round*,*prepared_value*), the function returns the signed message that contains such a tuple.

## JustifyRoundChange

![IBFT_JustifyRC](images/IBFT_JustifyRC.png)

*Moniz, H. The Istanbul BFT Consensus Algorithm. Algorithm 4. 2020*


**Pseudo-code:** the predicate receives a quorum of *Round-Change* messages and returns true if every message has an empty prepared value and round or if it has received a quorum of *Prepare* messages with the same value and round as the highest prepared of the quorum of *Round-Change*s.

The same logic is applied in the implementation. The only difference is that the process doesn't look for received *Prepare* messages. Instead, the [*Round-Change* message structure](ROUND_CHANGE.md#structure) carries a list of *Prepare* messages that justifies its *prepared_round* and *prepared_value* attributes.


## JustifyPrePrepare

![IBFT_JustifyPrePrepare](images/IBFT_JustifyPrePrepare.png)

*Moniz, H. The Istanbul BFT Consensus Algorithm. Algorithm 4. 2020*


**Pseudo-code:** the predicate receives a *Prepare* message and returns true if:
- round is 1 or
- received a quorum of *Round-Change* messages such that
    - they all have an empty prepared round and value or
    - there is a quorum of *Prepare* messages with the same round and value as the highest prepared of the quorum of *Round-Change*s.

The implementation follows the same logic as the paper's specification. However, there are two observations:
- The $\beta$ predicate verification is included in the function implementation. This is done since such verification is implicit in the validation of *Pre-Prepare* messages.
- The quorum of *Round-Change* and *Prepare* messages are contained in the *Proposal* message itself. Look [here](PROPOSAL.md) for reference. Therefore, the node doesn't have to look for any message that it has received.
