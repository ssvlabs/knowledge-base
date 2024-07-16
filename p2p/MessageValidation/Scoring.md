# Scoring


## Table of Contents
- [Scoring](#scoring)
  - [Table of Contents](#table-of-contents)
  - [Introduction](#introduction)
  - [Scoring design and specification](#scoring-design-and-specification)


## Introduction

More than validating a message, the message validation goal is to be able, by identifying malicious messages, to punish the peer who sent the message, ultimately not listening to its messages for a certain period.

We can do this in two ways:
- until a certain threshold value is met, penalize the peer with `equal` weights for every malicious message.
- until a certain threshold value is met, penalize the peer with `different` weights depending on the violated rule.

For the sake of simplicity, currently, we will use an `equal weight system`. Though this can be changed in future releases.


To review the basics of the GossipSub scoring system, check out the [explanatory file](./../ScoreOptimization/basics.md)


## Scoring design and specification

Message validation scoring precisely takes place within the $P_4$ parameter and the decision interface.

As we saw, once we reject a message, $P_4$ increases by one. To control if a peer will be rejected faster or slower we can adjust the value of $w_4$ (`InvalidMessageDeliveriesWeight`).

Roughly, if we want for the peer to be rejected after $r$ malicious messages, we should set $w_4 = \frac{GraylistThreshold}{r}$.

Since $GraylistThreshold=-16000$, if we want to `tolerate 20 malicious messages`, then $InvalidMessageDeliveriesWeight = \frac{-1600}{20} = -800$.

Notice that before reaching the _GraylistThreshold_, we will reach first _PublishThreshold_ since it's -8000. Therefore, before avoid listening to the peer, we stop publishing message to it.

Regarding whether or not we should `reject` or `ignore` a message, it depends on the rules, which can be checked in the [rules table](./Rules.md#list-of-rules).


<details>
  <summary><b>Current <i>InvalidMessageDeliveriesWeight</i> value</b></summary>


---

Currently, _InvalidMessageDeliveriesWeight_ is defined as $-\frac{MaxPeerScore}{TopicWeight}$, where $TopicWeight = \frac{4}{subnetsCount}$ and $MaxPeerScore=(MaxInMeshScore + MaxFirstDeliveryScore)\times TotalTopicsWeight$. _MaxInMeshScore_ and _MaxFirstDeliveryScore_ are configurable and $TotalTopicsWeight=4.5$.

In current configuration, we have $MaxInMeshScore=10$, $MaxFirstDeliveryScore=40$ and thus
$$InvalidMessageDeliveriesWeight=-\frac{50 \times 4.5}{4/subnetsCount} \approx - 56.25\times subnetsCount$$

Reference: 
- [Parameter Configuration Spec](https://github.com/bloxapp/ssv-spec/blob/312afd757009a26101a64ed009991638653e080f/p2p/SCORING.md#subnet-topic-params)
- [Implemented code](https://github.com/bloxapp/ssv/blob/95694524ea134337a4c9a661dff6e8aae63403db/network/topics/params/topic_score.go#L170)

---

</details>


