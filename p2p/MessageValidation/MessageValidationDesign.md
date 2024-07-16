# Message Validation Design


### Problem

Current node implementation is susceptible to network attacks.
- Example: a malicious node can flood a subnet with messages, causing a DoS attack.
- The main target of such network attacks is the BLS verification task, which is CPU consuming (on average 5ms).

### Goal

Design a Message Validation mechanism to prevent DoS attacks. This mechanism will rely on
- Syntax (format) verification
- Semantic verification
- Knowledge of expected messages according to duty's logic
- Knowledge of expected messages according to QBFT's logic

## Attacker definition


> An attacker is any node acting differently from the defined behaviour by ssv.

- `Signer Attacker`: An attacker may have taken control of a node, possessing a share key.
- `Non-Signer Attacker`: It can be a malicious network scheduler, delaying, tampering, or creating invalid messages.

Also, attacks may be of the following types:
- `Direct`: tries to attack direct peers.
- `Covert`: tries to attack nodes it isn't peered to

## Classifier abstraction

The main component of the mechanism can be abstracted as a classifier. The possible classifications are:
- `Accept`: for messages that pass through all validations.
- `Reject`: for messages that either
    - go against the protocol specification (malicious)
    - are invalid
    - may be an attempt of a DoS attack.
    
    The sender should be penalized.

- `Ignore`: valid messages that shouldn't be processed but are seemingly honest (i.e. delayed messages).

> **Note**
> This classification is due to the `libp2p/go-libp2p-pubsub` [validation classification](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/gossipsub-v1.1.md#extended-validators).


## Scoring systems

In `libp2p`, when we reject a message, the peer that sent the message has its `score` decreased. We define in _libp2p_ a certain threshold value such that, after the _score_ is lower than this value, the peer is ignored.

Check out the [scoring](./Scoring.md) document for more information.


## Classifier structure

- The classifier shall be built on top of rules.
- If the message doesn't pass a certain rule, the rule triggers a classification tag and an error for this message.
- If a message is accepted by every rule, it's considered a valid message.

You can check a detailed list of the rules in the [Rules document](Rules.md).
