# Commit

![IBFT_commit](images/IBFT_commit.png)

*Moniz, H. The Istanbul BFT Consensus Algorithm. Algorithm 2. 2020*


**Pseudo-code:** when a quorum of *Commit* messages carrying the same value is received, the node:
- stops the timer,
- decides the consensus with the value received and the quorum as proof.


## Validation

The paper doesn't include any extra validation for the commit rule. But, as it was with the *Prepare* message, the implementation's *Commit* uses a hash of the value. So, the same hash validation as in the [prepare rule](PREPARE.md) is performed.

## Body (UponCommit)

The differences in the rule's implementation are:
- Upon receiving a quorum of *Commit* messages, an aggregated BLS signature is created. This is used as the return value to the object using the QBFT instance (QBFT controller) and serves as proof of termination.
- In the paper specification, the process can decide and terminate after receiving a quorum, $2f+1$, of *Commit* messages. In the implementation, even after the process has been decided, it shall still process new *Commit* messages in order to aggregate their signatures in the threshold signature. That way, it can update information regarding the decided set of nodes.
- The timer isn't explicitly stopped. However, in the main SSV repository, when the *Instance* object becomes decided, the timer [doesn't trigger the timeout rule](https://github.com/bloxapp/ssv/blob/main/protocol/v2/qbft/controller/timer.go). This [is to be added](https://github.com/bloxapp/ssv-spec/issues/294) to the spec repository.