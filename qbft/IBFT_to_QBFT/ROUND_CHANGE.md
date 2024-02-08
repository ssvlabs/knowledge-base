# Round change

![IBFT_round_change_f1](images/IBFT_RC_f1.png)

*Moniz, H. The Istanbul BFT Consensus Algorithm. Algorithm 3. 2020*


**Pseudo-code:** when a weak support ($f+1$) of *Round-Change* messages is received, the node:
- jumps to the minimum round value received, updating its round attribute,
- starts a new timer for the new round,
- broadcasts a *Round-Change* message indicating to others that it has jumped to a new round.

![IBFT_round_change_Q](images/IBFT_RC_Q.png)

*Moniz, H. The Istanbul BFT Consensus Algorithm. Algorithm 3. 2020*


**Pseudo-code:** when a quorum ($2f+1$) of *Round-Change* messages for a certain round is received, the node:
- verifies if it's the new leader,
- checks if the *Round-Change* quorum is [justifiable](MSG_JUST_FUNCTIONS.md) (have empty proposed round and value or a quorum of *Prepare*s support the highest prepared round and value of the *Round-Change*s),
- if conditions hold, it has to decide which value to broadcast in its proposal. The value should be the highest prepared value if it exists. Otherwise, it should be its own input value.
- Then, it broadcasts a *Proposal* message, starting the new round.


## Structure

In the implementation, the *Round-Change* message has a different structure. Besides the paper fields, it also carries a list of *Prepare* messages that justifies the fields (*prepared_ round*,*prepared_value*). That way, even if the recipient has not received any *Prepare* messages, it can verify the correctness of the *Round-Change* message.

Therefore, when validating a *Round-Change* message, an extra step is performed in order to verify the authenticity of the *Prepare* messages appended.

## Round-Change $F+1$ rule

The only difference in the implementation is that the variable *ProposalAcceptedForCurrentRound* is set to *nil*, as it's done in the [timeout rule](TIMER_EXPIRATION.md).

## Round-Change Quorum rule

This rule's implementation does exactly what is specified in the paper.