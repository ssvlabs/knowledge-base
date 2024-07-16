# Estimation of amount of messages received

To play with the below formulas, follow this [google sheet link](https://docs.google.com/spreadsheets/d/1TpXnVFzF4eGiQarXuPBrOIU4tJXhzOHav9PkGve3_Qc/edit?usp=sharing).

## Probability of having a duty per slot

The number of validators in the Ethereum network was considered to be 683334 (value at 23th July of 2023).

### Attestation

Each validator must do one attestation per epoch. Thus:

- $P(Attestation\,per\,epoch) = 1$
- $P(Attestation\,per\,slot) = 1/32$

### Attestation Aggregator

There are 16 aggregators per attestation committee. Committee size ranges from 128 to 2048. Thus:

- $P(Aggregator | Attester) \in [\frac{C(2047,15)}{C(2048,16)},\frac{C(127,15)}{C(128,16)}] = [\frac{16}{2048},\frac{16}{128}] = [0.0078,0.125]$


### Proposer

Supposing that the function that selects the proposer is random, each validator has equal probability. Thus:

- $P(Proposer) = \frac{1}{683334} = 1.46 \times 10^{-6}$

### Sync Committee

For sync committees, 512 validators are selected for 256 epochs. The probability to be selected is:

- $P(Sync Committee) \approx C(683333,511) / C(683334,512) = \frac{512}{683334} = 0.00075$

### Sync Committee Aggregator

The 512 validators are divided across 4 independent subnets (each with 128). Each has 16 aggregators. Thus:

- $P(Sync Commitee Aggregator | Sync Commitee) \approx \frac{C(127,15)}{C(128,16)} = \frac{16}{128} = 0.125$

The duties are non-exclusive meaning that a validator may have been requested to perform the 5 duties in a single slot.

## Number of messages exchanged for duty

Suppose there are N operators running a validator.

Each duty may have 3 steps: pre-consensus, consensus, post-consensus.
From one operator perspective, the interval of the possible number of messages received is:
- Pre-consensus: $[\lfloor\frac{(N+f)}{2}\rfloor+1, N]$
- Post-consensus: $[\lfloor\frac{(N+f)}{2}\rfloor+1, N]$
- Consensus: $\infty$ in theory (actually, the maximum number of rounds is 12, limitating the total number of messages)

Note: $\lfloor\frac{(N+f)}{2}\rfloor+1$ is the quorum formula, where $f = \lfloor\frac{(N-1)}{3}\rfloor$.

Let's consider the consensus to have a $0.95$ probability of success. Considering each as a `Bernoulli trial`, the geometric distribution, of success in round $k$, becomes
$$P(X=k) = (1-0.95)^{k-1}*(0.95)$$

with `expected value` for a successful round given by $E(X) = 1/p = 1/0.95$.

In a **`successful round`**, the interval of possibile messages received is (proposal + prepare (+ f round-change at maximum) + commit):
$$SR(N,F) = [1 + \lfloor\frac{(N+f)}{2}\rfloor+1 + \lfloor\frac{(N+f)}{2}\rfloor + 1, 1 + N + f + N] = [3 + 2\times\lfloor\frac{N+f}{2}\rfloor, 2N + f + 1]$$

In an **`unsuccessful round`**, the number of messages are ( only round-change for minimum or proposall + prepare + non quorum of commit + round-change for maximum)
$$FR(N,F) = [\lfloor\frac{(N+f)}{2}\rfloor+1, 1 + N + \lfloor\frac{(N+f)}{2}\rfloor + N]$$



The expected number of messages, considering the highest message estimation, would be
$$E(consensus\,messages\,per\,round) = 0.95 * (2N + f +1) + 0.05 * (1 + 2N + \lfloor\frac{(N+f)}{2}\rfloor)$$

So the `expected number of messages for consensus` is
$$E(consensus\,messages) = \frac{1}{0.95}\times E(consensus\,messages\,per\,round)$$

Also, once consensus is reached, **`decided messages`** may be received. Each node can send $N-Quorum+1$ (with lengths $Quorum$, $Quorum+1$, ..., $N$)  decided messages with different committee sizes. There are a few things to consider:
- The libp2p library won't send duplicated messages. So if peers 1 and 2 send a decided message with commits from operators 1, 2, and 3, only one message will be received.
- A decided message is sorted on its signers. So a decided message with commits from operators 1, 2, and 3 is the same as with 1, 3, and 2.

Thus, the minimum number of decided messages is 1, if everyone sends the same message (e.g. with f crashes), and the maximum number is the minimum between:
- sum of the combination of $Quorum$ out of $N$ with then $Quorum+1$ out of $N$, etc. So
$$\sum_{k=2f+1}^{N}  {N \choose k}$$
- $N \times (N - Quorum+1)$

Thus, we have:
- Decided: $[1, min(N \times (N - Quorum+1) , \sum_{k=Quorum}^{N}  {N \choose k})]$

This number can increase pretty fast as $N$ increases. It's enough to note that  $\sum_{k=0}^{N}  {N \choose k} = 2^N$ and the maximum number of decided messages is a fraction of it. A malicious peer could, thus, exploit this type of message. However, we can restrict a node to send up to $N-Quorum+1$ decided messages. The maximum value would, then, be $N\times(f+1)$ which has a complexity of $O(N^2)$.

Every duty must do consensus (plus decided messages) and post-consensus, while some duties don't require pre-consensus. The interval for post and pre-consensus messages is similar:
$$[\lfloor\frac{(N+f)}{2}\rfloor+1, N]$$
Thus, we have:
- Without pre-consensus: $E(messages) = E(consensus\,messages) + Decided + \text{Post-consensus}$
- With pre-consensus: $E(messages) = \text{Pre-consensus} + E(consensus\,messages) + Decided + \text{Post-consensus}$

Regarding pre-consensus for each duty, we have:

| Duty | Pre-consensus |
| --- | --- |
| Attestation | :x: |
| Attestation Aggregation | :heavy_check_mark: |
| Proposer | :heavy_check_mark: |
| Sync Committee | :x: |
| Sync Committee Aggregator | :heavy_check_mark: |

## Number of expected messages per slot

The final number of expected messages per slot becomes
$$E(messages\,per\,slot) = P(Attestation\,per\,slot) * E(messages|Attestation) +\\ P(Aggregator) * E(messages|Aggreator) +\\ P(Proposer) * E(messages|Proposer) +\\ P(SyncCommittee) * E(messages|SyncCommittee) +\\ P(SyncCommitteeAggregator) * E(messages|SyncCommitteeAggregator)$$

To get the highest estimation, we set the aggregator attestation to its higher probability (lowest committee size) and set each consensus step to have its maximum number of messages. Then, we have (for one validator with 4 operators):
$$E(messages\,per\,slot) = 0.832$$

If we set each consensus step to its minimum number of messages, we would have:
$$E(messages\,per\,slot) = 0.368$$


## Expected messages in a subnet

Suppose an operator belongs to a subnet. In this subnet, there are $V$ validators. Each validator has 4 operators assigned.

Note: Here, it doesn't matter if all validators are assigned to the same 4 operators or not. The total number of messages is determined by the number of validators and how many operators are assigned.

Suppose we keep active only one validator (and 4 operators). The expected number of messages per slot is $E(messages\,per\,slot)$.

If we activate one more validator, then it becomes $E(messages\,per\,slot)\times2$, and so on.

## Expected messages for several subnets

Expanding the view, supposing an operator may belong to numerous subnets. The number of expected messages becomes $E(messages\,per\,slot) \times V$ where V is the total number of validators in all subnets (supposing each validator has 4 operators assigned).

For example, if there were $10000$ validators, we would have with our highest estimation:
$$10000 \times 0.8321 = 8321 \text{ messages per slot} = 693 \text{ messages per second}$$

And with the lowest estimation:
$$10000 \times 0.3684 = 3684 \text{ messages per slot} = 307 \text{ messages per second}$$

Receiving on average 500.

## Duty weight on expected number of messages


| Duty | Weight |
| ---- | ------ |
| Attestation | 0.85 |
| Aggregate Attestation | 0.125 |
| Proposer | 0.00005 |
| Sync Committee| 0.02 |
| Sync Committee Aggregatio| 0.003 |

## Number of expected messages by number of operators

The table below shows how the expected number of messages grows as the number of operators by validator grows. 
| $f$ | $N = (3f+1)$ | $E(messages)$ for $V=10k$ per second (lowest estimation) | $E(messages)$ for $V=10k$ per second (highest estimation)
| ---- | ----- | ---- | ---- |
1 | 4 | 307 | 693 |
2 | 7 | 474 | 1557 |
3 | 10 | 642 | 2510 |
4 | 13 | 810 | 3644 |

## Number of expected messages by QBFT success probability

The number of expected messages also varies with how likely a QBFT round will be successful.

| Probability of success | $E(messages)$ for $V=10k$ per second (lowest estimation) | $E(messages)$ for $V=10k$ per second (highest estimation) | Mean
| --- | --- | --- | --- |
0.95 | 307 | 693 | 500
0.9 | 318 | 716 | 517
0.85 | 330 | 741 | 536
0.8 | 344 | 770 | 557