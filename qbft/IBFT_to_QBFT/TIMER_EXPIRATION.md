# Timer expiration

![IBFT_timer](images/IBFT_timer.png)

*Moniz, H. The Istanbul BFT Consensus Algorithm. Algorithm 3. 2020*


**Pseudo-code:** when the timer expires, the node:
- increment its round,
- starts a new timer for the new round,
- broadcasts a *Round-Change* message indicating to others that it has jumped to a new round.

## Body (UponRoundTimeout)

The only difference in the implementation is that the variable *ProposalAcceptedForCurrentRound* is set to *nil* since the node progressed to a new round.