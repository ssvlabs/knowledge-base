# Message validation metrics


## Counters by duty type, round

- Consensus messages
  - Proposal messages
  - Prepare messages
  - Commit messages
  - Decided messages
  - Round-Change messages
- Partial Signature messages
- Total Received Messages
- Valid Messages
- Ignored Messages
- Rejected Messages

## Counters of error types by duty type, round

- Total Violations
- Count per violation type

### QBFT
- Invalid Old Consensus Round Message
- Invalid Future Consensus Round Message


### Partial Signature
- Late Message to Slot
- Early Message to Slot

## Queue
- Message queue size/capacity
- Dropped queue message count
- Time in queue  
- Queue fill rate
- Queue consumption rate

## Message analysis by duty type, round
- Average Message Size
- Minimum Message Size
- Maximum Message Size
- Message Processing Time by Stage
- Message arrival rate
- Message processing rate
- Margin of message-per-slot/round violations by committee size