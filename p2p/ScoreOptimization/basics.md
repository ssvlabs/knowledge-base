# Libp2p scoring system

In libp2p:
- Each peer maintains a score for other peers.
- The score is computed locally and **not shared**.
- The score, a **real value**, is computed as a **weighted mix** of parameters. It's possible to add application-specific scoring.
- The score is **global for topics**. I.e., it's computed across all topics with a weighted mix.
- The score is retained for a certain period even if a peer disconnects.

There are many score thresholds (see the [documentation](https://github.com/libp2p/specs/blob/50db89f3a71a87b096b0994a43a2dce0d251aeec/pubsub/gossipsub/gossipsub-v1.1.md#score-thresholds) for more information). The value configuration for each is defined in the [ssv-spec p2p documentation](https://github.com/bloxapp/ssv-spec/blob/312afd757009a26101a64ed009991638653e080f/p2p/SCORING.md#peer-score-thresholds) and they are:

| Name                          | Default Value (libp2p) | Default Value (SSV) | Default Value (ETH2) |
|-------------------------------|------------------------|---------------------|----------------------|
| `0`                           | -                      | -                   | -                    |
| `GossipThreshold`             | `-4000`                | `-4000`             | `-4000`              |
| `PublishThreshold`            | `-8000`                | `-8000`             | `-8000`              |
| `GraylistThreshold`           | `-16000`               | `-16000`            | `-16000`             |
| `AcceptPXThreshold`           | `100`                  | `100`               | `100`                |
| `OpportunisticGraftThreshold` | `5`                    | `5`                 | `5`                  |

The score serves many purposes such as:
- When pruning the mesh, the peer keeps the $D_{score}$ best scoring peers.
- When selecting peers to graft, the ones with negative scores are ignored.
- Periodically (1 minute), a node checks if the median score of its mesh is below the `OpportunisticGraftThreshold`. If so, the node grafts at least two peers with a score above the mesh's median. This mechanism is used as a defense against Eclipse attacks.


The score function is a weighted sum between 4 topics and 3 global parameters.
$$Score(p) = TopicCap(\sum_i t_i\times( w_1(t_i)\times P_1(t_i) + w_2(t_i)\times P_2(t_i) + w_3(t_i)\times P_3(t_i) + w_3b(t_i)\times P_3b(t_i) + w_4(t_i)\times P_4(t_i) )) + w_5\times P_5 + w_6\times P_6 + w_7\times P_7$$

where the variables are:
- `TopicCap`: function to allow the application to define an optional cap for the total topic contribution.
- `tᵢ`: topic weight for topic $i$.
- `P₁`: **Time in Mesh** for a topic (the time a peer has been in a mesh divided by an application-defined period). Used to boost peers already in the mesh so that they are not prematurely pruned. The value has an upper bound also defined by the application.
- `P₂`: **First Message Deliveries** for a topic. Number of messages first delivered by the peer in the topic. This value has an upper bound too.
- `P₃`: **Mesh Message Delivery Rate** for a topic. This is 0 if the rate at which the peer is sending messages is more than a certain threshold of message delivery rate. If it's below this threshold, then the value will be the square of the deficit. The weight of this parameter, $w_3$, is negative.
- `P₃b`: **Mesh Message Delivery Failures** for a topic. This is a sticky parameter. It's increased when the peer is pruned with a negative score. The increment is the rate deficit of delivered messages at the time of the prune. The purpose is to keep a history of prunes so that a peer cannot quickly get re-grafted into the mesh. Its weight, $w_{3b}$, is also negative.
- `P₄`: **Invalid Messages** for a topic. This is the number of invalid messages (according to application-specific rules) delivered on the topic. The weight, $w_4$, is also negative. It doesn't have an upper bound.
- `P₅`: **Application-Specific** score. The score is assigned to the peer by the application. Weight is positive and the parameter has a real value. Thus, the application can either punish the peer with a negative score or reward it with a positive score.
- `P₆`: **IP Colocation Factor**: Threshold parameter for the number of peers using the same IP. If the number of peers with the same IP is above the threshold, the parameter will be the square of the excess. Else, it's 0. It also uses a negative weight.
- `P₇`: **Behavioural Penalty**: Parameter for penalties applied for misbehavior according to router-specific rules or events. The value is the square of a decaying counter. The weight, $w_7$, is negative. The score is incremented if a peer attempts to re-graft before the prune backoff or if didn't follow up an IWANT request (3 seconds tolerance as default).

Each parameter has a counter associated with it, denoted by $P_j(t_i)$ for $P_j$. In the GossipSub code, $P_j$ is defined at time $t_i$ as $P_j = w_j \times P_j(t_i)$.

The parameters $P_2, P_3, P_3b, P_4$ and $P_7$ decay periodically. The application can set the `decay factor` and `decay interval`. Moreover, after the value is below a `DecayToZero` threshold, it's assigned to 0.


To update the `P4` counter, an extended validation verifies the message and decides whether to increase the $P_4$ penalty parameter or not. At a minimum, the decision interface should encompass:
- `Accept`: valid message, to be delivered and forwarded to the network.
- `Reject`: invalid message to be rejected, triggering the $P_4$ penalty.
- `Ignore`: not forwarded or delivered and doesn't trigger $P_4$.


## Parameters Review (extracted from [gossipsub spec table](https://github.com/libp2p/specs/blob/50db89f3a71a87b096b0994a43a2dce0d251aeec/pubsub/gossipsub/gossipsub-v1.1.md#overview-of-new-parameters))

| Parameter                         | Type     | Description                                                              | Constraints                                  |
|-----------------------------------|----------|--------------------------------------------------------------------------|----------------------------------------------|
|                                   |          | **Thresholds**                                                           |
| `GossipThreshold`                 | Float    | No gossip emitted to peers below threshold; incoming gossip is ignored.  | Must be < 0                                  |
| `PublishThreshold`                | Float    | No self-published messages are sent to peers below threshold.            | Must be <= `GossipThreshold`                 |
| `GraylistThreshold`               | Float    | All RPC messages by peers below the threshold are ignored.               | Must be < `PublishThreshold`                 |
| `AcceptPXThreshold`               | Float    | PX information by peers below this threshold is ignored.                 | Must be >= 0                                 |
| `OpportunisticGraftThreshold`     | Float    | Opportunistically graft if mesh median score is below this value.        | Must be >= 0                                 |
| `DecayInterval`                   | Duration | Interval at which parameter decay is calculated.                         |                                              |
| `DecayToZero`                     | Float    | Limit below which we consider a decayed param to be "zero".              | Should be close to 0.                        |
| `RetainScore`                     | Duration | Time to remember peer scores after a peer disconnects.                   |                                              |
| `TopicWeight`                     | Weight   | How much does behavior in this topic contribute to the overall score?    |                                              |
| **`P₁`**                          |          | **Time in Mesh**                                                         |                                              |
| `TimeInMeshWeight`                | Weight   | Weight of `P₁`.                                                          | Should be a small positive value.            |
| `TimeInMeshQuantum`               | Duration | Time a peer must be in mesh to accrue one "point" for `P₁`.              |                                              |
| `TimeInMeshCap`                   | Float    | Maximum value for `P₁`.                                                  | Should be a small positive value.            |
| **``P₂``**                        |          | **First Message Deliveries**                                             |                                              |
| `FirstMessageDeliveriesWeight`    | Weight   | Weight of `P₂`.                                                          | Should be positive, to reward fast peers.    |
| `FirstMessageDeliveriesDecay`     | Float    | Decay factor for `P₂`.                                                   |                                              |
| `FirstMessageDeliveriesCap`       | Float    | Maximum value for `P₂`.                                                  |                                              |
| **`P₃`**                          |          | **Mesh Message Delivery Rate**                                           |                                              |
| `MeshMessageDeliveriesWeight`     | Weight   | Weight of `P₃`.                                                          | Should be negative.                          |
| `MeshMessageDeliveriesDecay`      | Float    | Decay factor for `P₃`.                                                   |                                              |
| `MeshMessageDeliveriesThreshold`  | Float    | Value for `P₃` below which we start penalizing peers.                    | Depends on the topic's expected rate.        |
| `MeshMessageDeliveriesCap`        | Float    | Maximum value for `P₃`.                                                  | Must be >= `MeshMessageDeliveriesThreshold`. |
| `MeshMessageDeliveriesActivation` | Duration | Time a peer must be in the mesh before we start applying the `P₃` score. |                                              |
| `MeshMessageDeliveryWindow`       | Duration | Time after first delivery that is considered "near-first".               | Should be small, e.g. 1-5 ms.                |
| **`P₃b`**                         |          | **Mesh Message Delivery Failures**                                       |                                              |
| `MeshFailurePenaltyWeight`        | Weight   | Weight of `P₃b`.                                                         | Should be negative.                          |
| `MeshFailurePenaltyDecay`         | Float    | Decay factor for `P₃b`.                                                  |                                              |
| **`P₄`**                          |          | **Invalid Messages**                                                     |                                              |
| `InvalidMessageDeliveriesWeight`  | Weight   | Weight of`P₄`.                                                           | Should be negative.                          |
| `InvalidMessageDeliveriesDecay`   | Float    | Decay factor for `P₄`.                                                   |                                              |
| **`P₅`**                          |          | **Application-specific**                                                 |                                              |
| `AppSpecificWeight`               | Weight   | Weight of `P₅`, the application-specific score.                          | Positive but the score may be negative.      |
| **`P₆`**                          |          | **IP colocation**                                                        |                                              |
| `IPColocationFactorWeight`        | Weight   | Weight of `P₆`, the IP colocation score.                                 | Must be negative.                            |
| `IPColocationFactorThreshold`     | Integer  | Number of IPs a peer may have before being penalized.                    | Must be at least 1.                          |
| **`P₇`**                          |          | **Behaviour penalty**                                                    |                                              |
| `BehaviourPenaltyWeight`          | Weight   | Weight of `P₇`, the behaviour penalty.                                   | Must be negative.                            |
| `BehaviourPenaltyDecay`           | Float    | Decay factor for `P₇`.                                                   | Must be between 0 and 1.                     |


Reference:
- [libp2p's pubsub's gossipsub v1.1 documentation](https://github.com/libp2p/specs/blob/50db89f3a71a87b096b0994a43a2dce0d251aeec/pubsub/gossipsub/gossipsub-v1.1.md#score-thresholds).

