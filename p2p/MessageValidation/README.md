# Message Validation

This folder contains the specification for the message validation module implementation.

The purpose of message validation is to filter any received message by checking if it's valid or not. This saves processing time, allows better performance, and secures the node against malicious messages and certain types of DoS attacks.

Below you can find links to navigate through the content of this folder.

## Contents
- [Message validation design](./MessageValidationDesign.md)
    - [Scoring mechanism](./Scoring.md)

    - [Set of message validation rules](./Rules.md)
    
    - [Message structures validity properties for deriving the rules](./NetworkMessagesProperties.md)
    
- [The initial design](./InitialDesign.md)

- [Unit tests list](./unit_tests.md)

- [Metrics for the message validation module](./metrics.md)

- [Extra: Estimation of the expected number of received messages](./ReceivedMessagesEstimation.md)
