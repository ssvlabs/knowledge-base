# Rules

> [!WARNING]  
> Rule specific score is deprecated. A common score is used and defined in the [scoring](./Scoring.md) document.

Each rule has a classification tag.

**`Ethereum Alignment`:** Every rule should respect the Ethereum consensus specification. In other words, no rule should reject a message following a normal case of any Beacon duty.

**`SSV Alignment`:** Every rule should respect the SSV specification.


Rules are divided into the following types: Syntax, Semantics, QBFT Logic and Duty Logic.

## Types of prevented messages

Rules may prevent
- Malicious (try to mess with state transition).
- Invalid (e.g. bad sig).
- Seemingly honest but won't generate protocol state transition (e.g. delayed message after duty finished, duplicated message).

On the other hand, a valid message is a message that can be generated in a run (sequence of states) aligned with the Ethereum and SSV Protocol. We also restrict duplicated messages, not only by identical messages but also by accepting only one pre-consensus (if the duty allows it), one post-consensus message for each duty, and one of each QBFT message per duty and QBFT round.


## Properties

A set of rules should be:
- `Consistent`: any honest non-delayed messages should pass through every rule.
- `Complete`: if a message is approved by the whole set of rules, then it's also honest. In other words, no malicious message can pass through the set of rules.

The `quality` of a set of rules is defined as the ability to minimize the number of malicious messages tolerated without compromising the `Ethereum and the SSV specification`.


The goal is to define a consistent and complete set of rules with maximum quality.

## Code

Next, we present a high-level code for the message validation module. It contains functions for syntax, semantics, QBFT and duty logic rules. Some of its auxiliary functions are called but not described here. These functions are left for implementation.

### Error structure

First of all, we extend the `error` type to encompass the validation result. We present some basic error instances but implementation may extend it to encompass more detailed errors.

```go
type Error struct {
	text   string
	reject bool
}

func (e Error) Error() string {
	var sb strings.Builder
	sb.WriteString(e.text)

	return sb.String()
}

func (e Error) Reject() bool {
	return e.reject
}

func (e Error) Text() string {
	return e.text
}

var (
	ErrPubSubMessageHasNoData = Error{text: "pub-sub message has no data", reject: true}
	ErrPubSubDataTooBig       = Error{text: "pub-sub message data too big"}
	ErrUndecodableData        = Error{text: "undecodable data", reject: true}
	ErrMalformedPubSubMessage = Error{text: "pub-sub message is malformed", reject: true}
	ErrNilSignedSSVMessage    = Error{text: "decoded SignedSSVMessage is nil", reject: true}
	ErrNilSSVMessage          = Error{text: "SSVMessage is nil", reject: true}
	ErrSignatureVerification  = Error{text: "signature verification", reject: true}
	ErrSSVMessageDataTooBig   = Error{text: "SSVMessage.Data is too big"}
	ErrEmptyData 			  = Error{text: "SSVMessage.Data is empty", reject: true}

	ErrIncorrectTopic                          = Error{text: "incorrect topic"}
	ErrWrongDomain                             = Error{text: "wrong domain"}
	ErrNonExistentCommitteeID                  = Error{text: "non existent cluster ID"}
	ErrValidatorLiquidated                     = Error{text: "validator is liquidated"}
	ErrUnknownValidator                        = Error{text: "validator does not exist"}
	ErrValidatorNotAttesting                   = Error{text: "validator is not attesting"}
	ErrInvalidRole                             = Error{text: "invalid role", reject: true}
	ErrNoSigners                               = Error{text: "no signers", reject: true}
	ErrZeroSigner                              = Error{text: "zero signer ID", reject: true}
	ErrDuplicatedSigner                        = Error{text: "signer is duplicated", reject: true}
	ErrSignersNotSorted                        = Error{text: "signers are not sorted", reject: true}
	ErrSignersAndSignaturesWithDifferentLength = Error{text: "signers and signatures with different length", reject: true}
	ErrNoSignatures                            = Error{text: "no signatures", reject: true}
	ErrWrongRSASignatureSize                   = Error{text: "wrong RSA signature size", reject: true}
	ErrWrongBLSSignatureSize                   = Error{text: "wrong BLS signature size", reject: true}
	ErrSignerNotInCommittee                    = Error{text: "signer is not in committee", reject: true}
	ErrUnknownSSVMessageType                   = Error{text: "unknown SSV message type", reject: true}
	ErrEventMessage                            = Error{text: "event messages are not broadcast", reject: true}
	ErrDKGMessage                              = Error{text: "DKG messages are not supported", reject: true}

	ErrUnknownQBFTMessageType              = Error{text: "unknown QBFT message type", reject: true}
	ErrRoundTooHigh                        = Error{text: "round is too high for this role", reject: true}
	ErrDecidedWithSameSigners              = Error{text: "decided with the same signers as sent before"}
	ErrPrepareOrCommitWithFullData         = Error{text: "prepare or commit with full data", reject: true}
	ErrFullDataNotInConsensusMessage       = Error{text: "full data in message different than consensus", reject: true}
	ErrMismatchedIdentifier                = Error{text: "message ID mismatched", reject: true}
	ErrUnexpectedConsensusMessage          = Error{text: "unexpected consensus message for this role", reject: true}
	ErrSlotNotInTime					   = Error{text: "current time is not between duty's start time and +34(committee and aggregator) or +3(else) slots"}
	ErrEarlySlotMessage					   = Error{text: "message was sent before slot starts"}
	ErrLateSlotMessage					   = Error{text: "current time is above duty's start +34(committee and aggregator) or +3(else) slots"}

	ErrSignerNotLeader                     = Error{text: "signer is not leader", reject: true}
	ErrInvalidHash                         = Error{text: "root doesn't match full data hash", reject: true}
	ErrNonDecidedWithMultipleSigners       = Error{text: "non-decided with multiple signers", reject: true}
	ErrDecidedNotEnoughSigners             = Error{text: "decided signers size is less than quorum size", reject: true}
	ErrRoundAlreadyAdvanced                = Error{text: "signer has already advanced to a later round"}
	ErrEstimatedRoundNotInAllowedSpread	   = Error{text: "message is early or late for the given round with an allowed spread of 1 round."}
	ErrDuplicatedProposalWithDifferentData = Error{text: "duplicated proposal with different data", reject: true}
	ErrDuplicatedMessage				   = Error{text: "message is duplicated", reject: true}
	ErrZeroRound                           = Error{text: "round is zero", reject: true}
	ErrSlotAlreadyAdvanced          	   = Error{text: "signer already advanced to later slot"}
	ErrUnexpectedRoundChangeJustifications = Error{text: "message has a round-change justification but it's not a proposal or round-change", reject: true}
	ErrUnexpectedPrepareJustifications 	   = Error{text: "message has a prepare justification but it's not a proposal", reject: true}


	ErrPartialSigOneSigner              		= Error{text: "partial signature message with len(signers) != 1", reject: true}
	ErrTooManyPartialSignatureMessages  		= Error{text: "too many signatures for cluster in partial signature message"}
	ErrTripleValidatorIndexInPartialSignatures  = Error{text: "validator index appear 3 times in partial signature message", reject: true}
	ErrNoPartialSignatureMessages       		= Error{text: "no partial signature messages", reject: true}
	ErrInconsistentSigners              		= Error{text: "inconsistent signers", reject: true}
	ErrValidatorIndexMismatch           		= Error{text: "validator index mismatch"}
	ErrInvalidPartialSignatureType      		= Error{text: "invalid partial signature type", reject: true}
	ErrPartialSignatureTypeRoleMismatch 		= Error{text: "partial signature type and role don't match", reject: true}
	ErrInvalidPartialSignatureTypeCount 		= Error{text: "sent more partial signature messages of a certain type than allowed", reject: true}

	ErrTooManyDutiesPerEpoch = Error{text: "too many duties per epoch"}
	ErrNoDuty                = Error{text: "no duty for this epoch"}
)
```

### The main structure, function and constant values

The main structure is the `MessageValidation` structure which has a `ValidatePubsubMessage` function to serve as a handle for the GossipSub extended validator.

```go

const (
	MaxMsgSize                               = 4945164
	maxConsensusMsgSize                      = 722412
	maxPartialSignatureMsgSize               = 144020
	maxSSVMessageDataSize                    = max(maxConsensusMsgSize, maxPartialSignatureMsgSize)
	PartialSignatureSize                     = 48
	MessageSignatureSize                     = 256
	SyncCommitteeSize                        = 512
	MaxSignaturesInSyncCommitteeContribution = 13
)

type MessageValidation struct {
	selfPID peer.ID
	signatureVerifier types.SignatureVerifier
}

func (mv *MessageValidation) Validate(_ context.Context, _ peer.ID, pmsg *pubsub.Message) pubsub.ValidationResult {

	var peerState *PeerState
	var err error

	peerState = GetMessageValidationStateForPeer(pmsg.ReceivedFrom)

	err = mv.ValidateMessage(pmsg)

	// Verify signature if validation chain is successful
	if err == nil {
		err = mv.VerifyMessageSignature(pmsg)
	}

	// Update the signer's state
	if err == nil {
		mv.UpdateState(peerState, pmsg)
	}

	// Check error
	if err != nil {

		var valErr Error
		if errors.As(err, &valErr) {
			// Update state
			peerState.OnError(valErr)

			if valErr.Reject() {
				// Reject
				return pubsub.ValidationReject
			} else {
				// Ignore
				return pubsub.ValidationIgnore
			}
		} else {
			panic(err)
		}
	} else {
		return pubsub.ValidationAccept
	}
}

func (mv *MessageValidation) VerifyMessageSignature(pmsg *pubsub.Message) error {
	// Already verified
	signedSSVMessage := &types.SignedSSVMessage{}
	_ = signedSSVMessage.Decode(pmsg.Data)

	committeeMembers := make([]*types.CommitteeMember, 0)
	for _, signer := range signedSSVMessage.GetOperatorIDs() {
		signerPublicKey := GetPublicKeyFromOperator(signer)
		committeeMembers = append(committeeMembers, &types.CommitteeMember{
			OperatorID:        signer,
			SSVOperatorPubKey: signerPublicKey,
		})
	}

	err := mv.signatureVerifier.Verify(signedSSVMessage, committeeMembers)
	if err != nil {
		return ErrSignatureVerification
	}

	return nil
}
```


### Syntax Rules

The `ValidateMessage` function is the main function that initializes the validation chain with the syntax checks.


|           Verification           |                   Error                    | Classification |                                Explanation                                |
| -------------------------------- | ------------------------------------------ | -------------- | ------------------------------------------------------------------------- |
| Empty pubsub.Message.Data        | ErrPubSubMessageHasNoData                  | Reject         | pubsub.Message.Data must not be empty.                                    |
| Big pubsub.Message.Data          | ErrPubSubDataTooBig                        | Ignore         | pubsub.Message.Data must be below a size limit                            |
| Can't decode pubsub.Message.Data | ErrMalformedPubSubMessage                  | Reject         | pubsub.Message.Data must be decodable to SignedSSVMessage.                |
| Nil SignedSSVMessage             | ErrNilSignedSSVMessage                     | Reject         | SignedSSVMessage can't be nil.                                            |
| No signers                       | ErrNoSigners                               | Reject         | Len(SignedSSVMessage.OperatorIDs) must be >= 0.                           |
| No Signatures                    | ErrNoSignatures                            | Reject         | Len(SignedSSVMessage.Signatures) must be >= 0.                            |
| Wrong signature size             | ErrWrongRSASignatureSize                   | Reject         | $\forall i$ Len(SignedSSVMessage.Signatures[i]) must be of expected size. |
| Sorted signers                   | ErrSignersNotSorted                        | Reject         | SignedSSVMessage.OperatorIDs must be sorted.                              |
| Non-zero signers                 | ErrZeroSigner                              | Reject         | $\forall i$ SignedSSVMessage.OperatorIDs[i] must not be 0.                |
| Unique signers                   | ErrDuplicatedSigner                        | Reject         | SignedSSVMessage.OperatorIDs must have unique values.                     |
| Len(signers) = Len(signatures)   | ErrSignersAndSignaturesWithDifferentLength | Reject         | The number of signers and signatures must match                           |
| Nil SSVMessage                   | ErrNilSSVMessage                           | Reject         | SignedSSVMessage.SSVMessage can't be nil.                                 |
| Empty SSVMessage.Data            | ErrEmptyData                               | Reject         | SSVMessage.Data can't be empty.                                           |
| Big SSVMessage.Data              | ErrSSVMessageDataTooBig                    | Ignore         | SSVMessage.Data must be below a size limit.                               |
| Can't decode SSVMessage.Data     | ErrUndecodableData                         | Reject         | SSVMessage.Data must be decodable.                                        |
| Can't decode QBFT Justification  | ErrUndecodableData                         | Reject         | A qbft.Message must have decodable justifications.                        |


```go

func (mv *MessageValidation) ValidateMessage(pmsg *pubsub.Message) error {
	// Validation Chain: Syntax -> Semantics -> QBFT Semantics | Partial Signature Semantics -> QBFT Logic -> Duty Rules
	return mv.ValidateSyntax(pmsg)
}

func (mv *MessageValidation) ValidateSyntax(pmsg *pubsub.Message) error {

	// Syntax validation

	/*
		Messages structures and checks (->)

			// PubSub Message
			type Message struct {
				*pb.Message
				ID
				ReceivedFrom
				ValidatorData
				Local
			}
			// pb.Message
				type Message struct {
				From
				Data -> Not empty, Size limit, Decodable to SignedSSVMessage
				Seqno
				Topic
				Signature
				Key
				XXX_NoUnkeyedLiteral
				XXX_unrecognized
				XXX_sizecache
			}
			type SignedSSVMessage struct { -> Can't be nil
				Signatures  [][]byte -> Len > 0, Signature size
				OperatorIDs []OperatorID -> Len > 0, Unique signers, Sorted, Signer not 0, Len = Len(signatures),
				SSVMessage  *SSVMessage -> Can't be nil
				FullData    []byte
			}
			type SSVMessage struct {
				MsgType MsgType
				MsgID   MessageID
				Data 	[]byte -> Non empty, Size limit, Decodable to qbft.Message or types.PartialSignatureMessages
			}
			type Message struct {
				MsgType    				 MessageType
				Height     				 Height
				Round      				 Round
				Identifier 				 []byte
				Root                     [32]byte
				DataRound                Round
				RoundChangeJustification [][]byte -> Decodable to []SignedSSVMessage
				PrepareJustification     [][]byte -> Decodable to []SignedSSVMessage
			}
	*/

	// Rule: Pubsub.Message.Message.Data must not be empty and respect size upper limit
	if err := ValidatePubSubMessage(pmsg); err != nil {
		return err
	}

	// Rule: Pubsub.Message.Message.Data decoding
	signedSSVMessage := &types.SignedSSVMessage{}
	if err := signedSSVMessage.Decode(pmsg.Data); err != nil {
		return ErrMalformedPubSubMessage
	}

	// Rule: SignedSSVMessage cannot be nil
	if signedSSVMessage == nil {
		return ErrNilSignedSSVMessage
	}

	// Rule: Must have at least one signer
	if len(signedSSVMessage.OperatorIDs) == 0 {
		return ErrNoSigners
	}

	// Rule: Must have at least one signature
	if len(signedSSVMessage.Signatures) == 0 {
		return ErrNoSignatures
	}

	// Rule: Signature size
	for _, sig := range signedSSVMessage.Signatures {
		if len(sig) != MessageSignatureSize {
			return ErrWrongRSASignatureSize
		}
	}

	// Rule: Signers must be sorted
	if !slices.IsSorted(signedSSVMessage.OperatorIDs) {
		return ErrSignersNotSorted
	}

	// Rule: Signer can't be zero
	for _, signer := range signedSSVMessage.OperatorIDs {
		if signer == 0 {
			return ErrZeroSigner
		}
	}

	// Rule: Signers must be unique
	var prevSigner spectypes.OperatorID
	for _, signer := range signedSSVMessage.OperatorIDs {
		if signer == prevSigner {
			return ErrDuplicatedSigner
		}
		prevSigner = signer
	}

	// Rule: Len(Signers) must be equal to Len(Signatures)
	if len(signedSSVMessage.OperatorIDs) != len(signedSSVMessage.Signatures) {
		return ErrSignersAndSignaturesWithDifferentLength
	}

	// Rule: SSVMessage cannot be nil
	if signedSSVMessage.SSVMessage == nil {
		return ErrNilSSVMessage
	}

	// Rule: SSVMessage.Data must not be empty
	if len(signedSSVMessage.SSVMessage.Data) == 0 {
		return ErrEmptyData
	}

	// SSVMessage.Data must respect the size limit
	if len(signedSSVMessage.SSVMessage.Data) > maxSSVMessageDataSize {
		return ErrSSVMessageDataTooBig
	}

	switch signedSSVMessage.SSVMessage.MsgType {
	case types.SSVConsensusMsgType:
		// Rule: SSVMessage.Data decoding
		var qbftMessage qbft.Message
		if err := qbftMessage.Decode(signedSSVMessage.SSVMessage.Data); err != nil {
			return ErrUndecodableData
		}

		// Rule: Message.RoundChangeJustification or Message.PrepareJustification decoding
		if _, err := qbftMessage.GetPrepareJustifications(); err != nil {
			return ErrUndecodableData
		}
		if _, err := qbftMessage.GetRoundChangeJustifications(); err != nil {
			return ErrUndecodableData
		}

	case types.SSVPartialSignatureMsgType:
		// Rule: SSVMessage.Data decoding
		var partialSignatureMessages types.PartialSignatureMessages
		if err := partialSignatureMessages.Decode(signedSSVMessage.SSVMessage.Data); err != nil {
			return ErrUndecodableData
		}
	}

	return mv.ValidateSemantics(pmsg.ReceivedFrom, signedSSVMessage, pmsg.GetTopic())
}

func (mv *MessageValidation) ValidatePubSubMessage(pmsg *pubsub.Message) error {
	// Rule: Pubsub.Message.Message.Data must not be empty
	if len(pmsg.Data) == 0 {
		return ErrPubSubMessageHasNoData
	}

	// Rule: Pubsub.Message.Message.Data size upper limit
	if float64(len(pmsg.Data)) > MaxMsgSize {
		return ErrPubSubDataTooBig
	}

	return nil
}

```

### Semantics General Rules


|     Verification     |           Error           | Classification |                              Explanation                              |
| -------------------- | ------------------------- | -------------- | --------------------------------------------------------------------- |
| Signers in committee | ErrSignerNotInCommittee   | Reject         | Signers must belong to validator's or CommitteeID's committee.        |
| Different Domain     | ErrWrongDomain            | Ignore         | MsgID.Domain is different than self domain.                           |
| Invalid Role         | ErrInvalidRole            | Reject         | MsgID.Role is not known.                                              |
| Validator exists     | ErrUnknownValidator       | Ignore         | If MsgID.SenderID is a validator, it must exist.                      |
| Active Validator ID  | ErrValidatorNotAttesting  | Ignore         | If MsgID.SenderID is a validator, it must be active active validator. |
| Validator Liquidated | ErrValidatorLiquidated    | Ignore         | If MsgID.SenderID is a validator, it must not be liquidated.          |
| CommitteeID exists   | ErrNonExistentCommitteeID | Ignore         | If MsgID.SenderID is a committee, it must exist.                      |
| Wrong topic          | ErrIncorrectTopic         | Ignore         | The message should be sent in the correct topic                       |
| Event Message        | ErrEventMessage           | Reject         | MsgType can't be of event message.                                    |
| DKG Message          | ErrDKGMessage             | Reject         | MsgType can't be of DKG message.                                      |
| Unknown MsgType      | ErrUnknownSSVMessageType  | Reject         | MsgType is not known.                                                 |

```go

func (mv *MessageValidation) ValidateSemantics(peerID peer.ID, signedSSVMessage *types.SignedSSVMessage, topic string) error {

	/*
		Messages structures and checks (->)

			type SignedSSVMessage struct {
				Signatures  [][]byte
				OperatorIDs []OperatorID -> Signers are in committee
				SSVMessage  *SSVMessage
				FullData    []byte
			}
			type SSVMessage struct {
				MsgType MsgType -> Can't be event, Can't be DKG, Must be valid (known)
				MsgID   MessageID -> Equal domain, Valid role (known), Validator exists, Active Validator, Validator Liquidated, CommitteeID exists
				Data 	[]byte
			}
	*/

	// Rule: Signers must belong to validator committee or CommitteeID
	for _, signer := range signedSSVMessage.GetOperatorIDs() {
		if !mv.SignerBelongsToCommittee(signer, signedSSVMessage.SSVMessage.MsgID) {
			return ErrSignerNotInCommittee
		}
	}

	// Rule: If domain is different then self domain
	domain := signedSSVMessage.SSVMessage.MsgID.GetDomain()
	if !mv.ValidDomain(domain) {
		return ErrWrongDomain
	}

	// Rule: If role is invalid
	role := signedSSVMessage.SSVMessage.MsgID.GetRoleType()
	if !mv.ValidRole(role) {
		return ErrInvalidRole
	}

	senderID := signedSSVMessage.SSVMessage.MsgID.GetSenderID()
	if role != types.RoleCommittee {
		validatorPK := senderID

		// Rule: Validator does not exist
		if !mv.ExistingValidator(validatorPK) {
			return ErrUnknownValidator
		}

		// Rule: If validator is not active
		if !mv.ActiveValidator(validatorPK) {
			return ErrValidatorNotAttesting
		}

		// Rule: If validator is liquidated
		if mv.ValidatorLiquidated(validatorPK) {
			return ErrValidatorLiquidated
		}
	} else {
		// Rule: Cluster does not exist
		if !mv.ExistingCommitteeID(senderID) {
			return ErrNonExistentCommitteeID
		}
	}

	// Rule: Check if message was sent in the correct topic
	if !mv.CorrectTopic(signedSSVMessage.SSVMessage.MsgID, topic) {
		return ErrIncorrectTopic
	}

	if signedSSVMessage.SSVMessage.MsgType == types.SSVEventMsgType {
		// Rule: Event message
		return ErrEventMessage
	} else if signedSSVMessage.SSVMessage.MsgType == types.DKGMsgType {
		// Rule: DKG message
		return ErrDKGMessage
	} else if signedSSVMessage.SSVMessage.MsgType == types.SSVConsensusMsgType {
		return mv.ValidateConsensusMessageSemantics(peerID, signedSSVMessage)
	} else if signedSSVMessage.SSVMessage.MsgType == types.SSVPartialSignatureMsgType {
		return mv.ValidatePartialSignatureMessageSemantics(peerID, signedSSVMessage)
	} else {
		// Unknown message type
		return ErrUnknownSSVMessageType
	}
}
```

### Consensus

#### Semantics


|           Verification            |              Error               | Classification |                         Explanation                         |
| --------------------------------- | -------------------------------- | -------------- | ----------------------------------------------------------- |
| Non-decided with multiple signers | ErrNonDecidedWithMultipleSigners | Reject         | Non-decided message must have one signer.                   |
| Decided with enough signers       | ErrDecidedNotEnoughSigners       | Reject         | Decided message must have at least a quorum of signers      |
| Prepare or commit with full data  | ErrPrepareOrCommitWithFullData   | Reject         | Prepare or Commit messages must not have FullData.          |
| Invalid full data hash            | ErrInvalidHash                   | Reject         | If there's a FullData field, Message.Root must be the hash. |
| Unknown MsgType                   | ErrUnknownQBFTMessageType        | Reject         | Message.MsgType must be known.                              |
| Zero round                        | ErrZeroRound                     | Reject         | Message.Round must not be zero.                             |
| Mismatched identifier             | ErrMismatchedIdentifier          | Reject         | Message.Identifier must match SSVMessage.Identifier         |


```go

func (mv *MessageValidation) ValidateConsensusMessageSemantics(peerID peer.ID, signedSSVMessage *types.SignedSSVMessage) error {

	/*
		Messages structures and checks (->)

			type SignedSSVMessage struct {
				Signatures  [][]byte
				OperatorIDs []OperatorID -> Valid length for decided
				SSVMessage  *SSVMessage
				FullData    []byte -> Must be empty for prepare and commit (with 1 signer)
			}
			type SSVMessage struct {
				MsgType MsgType
				MsgID   MessageID
				Data 	[]byte
			}
			type Message struct {
				MsgType    				 MessageType -> Valid (known), Must be of type commit for decided message
				Height     				 Height
				Round      				 Round -> Valid (>0)
				Identifier 				 []byte -> Match SSVMessage.MsgID
				Root                     [32]byte -> Must be hash of FullData
				DataRound                Round
				RoundChangeJustification [][]byte
				PrepareJustification     [][]byte
			}
	*/

	// Already verified
	var qbftMessage qbft.Message
	_ = qbftMessage.Decode(signedSSVMessage.SSVMessage.Data)

	signers := signedSSVMessage.GetOperatorIDs()

	if len(signers) > 1 {
		// Rule: Decided msg with different type than Commit
		if qbftMessage.MsgType != qbft.CommitMsgType {
			return ErrNonDecidedWithMultipleSigners
		}

		// Rule: Number of signers must be >= quorum size
		if !mv.ValidSignersLengthForCommitMessage(signers) {
			return ErrDecidedNotEnoughSigners
		}
	}

	if len(signedSSVMessage.FullData) > 0 {
		// Rule: Prepare or commit messages must not have full data
		if (qbftMessage.MsgType == qbft.PrepareMsgType) ||
			(qbftMessage.MsgType == qbft.CommitMsgType && len(signers) == 1) {
			return ErrPrepareOrCommitWithFullData
		}

		// Rule: Full data hash must match root
		if !mv.ValidFullDataRoot(signedSSVMessage.FullData, qbftMessage.Root) {
			return ErrInvalidHash
		}
	}

	// Rule: Consensus message type must be valid
	if !mv.ValidConsensusMessageType(qbftMessage.MsgType) {
		return ErrUnknownQBFTMessageType
	}

	// Rule: Round must not be zero
	if qbftMessage.Round == qbft.NoRound {
		return ErrZeroRound
	}

	// Rule: consensus message must have the same identifier as the ssv message's identifier
	if !mv.MatchedIdentifiers(qbftMessage.Identifier, signedSSVMessage.SSVMessage.MsgID[:]) {
		return ErrMismatchedIdentifier
	}

	return mv.ValidateQBFTLogic(peerID, signedSSVMessage)
}
```

#### QBFT Logic


|              Verification               |                 Error                  | Classification |                              Explanation                               |
| --------------------------------------- | -------------------------------------- | -------------- | ---------------------------------------------------------------------- |
| Not Leader                              | ErrSignerNotLeader                     | Reject         | Signer is not leader for round.                                        |
| Different decided with same signers     | ErrDecidedWithSameSigners              | Ignore         | Signer already sent different decided for duty with the same signers.  |
| Double Proposal with different FullData | ErrDuplicatedProposalWithDifferentData | Reject         | Signer already sent a different proposal for round.                    |
| Double Proposal                         | ErrDuplicatedMessage                   | Reject         | Signer already sent a proposal for round.                              |
| Double Prepare                          | ErrDuplicatedMessage                   | Reject         | Signer already sent a Prepare for round.                               |
| Double Commit                           | ErrDuplicatedMessage                   | Reject         | Signer already sent a Commit for round.                                |
| Double Round-Change                     | ErrDuplicatedMessage                   | Reject         | Signer already sent a Round-Change for round.                          |
| Round in round-spread                   | ErrEstimatedRoundNotInAllowedSpread             | Ignore         | Message must be in time for round with an allowed spread of 1 round.   |
| Already advanced round                  | ErrRoundAlreadyAdvanced                | Ignore         | Signer is already in a future round.                                   |
| Unexpected Round-Change justification   | ErrUnexpectedRoundChangeJustifications | Reject         | Round-Change justification can only be for a Proposal or Round-Change. |
| Unexpected prepare justification        | ErrUnexpectedPrepareJustifications     | Reject         | Preapre justification can only be for Proposal.                        |

```go

func (mv *MessageValidation) ValidateQBFTLogic(peerID peer.ID, signedSSVMessage *types.SignedSSVMessage) error {

	/*
		Messages structures and checks (->)

			type SignedSSVMessage struct {
				Signatures  [][]byte
				OperatorIDs []OperatorID -> Must be leader if message is Proposal, Decided msg can't have the same signers as previously sent before for the same duty
				SSVMessage  *SSVMessage
				FullData    []byte
			}
			type SSVMessage struct {
				MsgType MsgType
				MsgID   MessageID
				Data 	[]byte
			}
			type Message struct {
				MsgType    				 MessageType -> Can't send two proposals with different data, Message count rules
				Height     				 Height
				Round      				 Round -> Must belong to round spread, Already advanced to later round
				Identifier 				 []byte
				Root                     [32]byte
				DataRound                Round
				RoundChangeJustification [][]byte -> Can only exist for Proposal or Round-Change messages
				PrepareJustification     [][]byte -> Can only exist for Proposal messages
			}
	*/

	// Already verified
	var qbftMessage qbft.Message
	_ = qbftMessage.Decode(signedSSVMessage.SSVMessage.Data)

	peerState, err := mv.GetPeerState(peerID, signedSSVMessage.SSVMessage.MsgID, qbftMessage.Height)
	if err != nil {
		return err
	}

	signers := signedSSVMessage.GetOperatorIDs()

	if qbftMessage.MsgType == qbft.ProposalMsgType {
		// Rule: Signer must be the leader
		if !mv.IsLeader(signers[0], qbftMessage.Height, qbftMessage.Round) {
			return ErrSignerNotLeader
		}
	}

	if len(signers) > 1 {
		// Rule: Decided msg can't have the same signers as previously sent before for the same duty
		if mv.HasSentSameDecidedSigners(peerID, signedSSVMessage.OperatorIDs, signedSSVMessage.SSVMessage.MsgID, qbftMessage.Height) {
			return ErrDecidedWithSameSigners
		}
	}

	if len(signers) == 1 {

		if consensusMessage.Round == peerState.Round {
		    // Rule: Peer must not send two proposals with different data
			if mv.PeerHasSentProposalWithDifferentData(peerID, qbftMessage.MsgType, signedSSVMessage.SSVMessage.MsgID, qbftMessage.Height, qbftMessage.Round, signedSSVMessage.FullData) {
				return ErrDuplicatedProposalWithDifferentData
			}

		    // Rule: Peer must send only 1 proposal, 1 prepare, 1 commit and 1 round-change per round
		    if !mv.ValidConsensusMessageCount(peerID, signedSSVMessage.SSVMessage.MsgID, qbftMessage.Height, qbftMessage.MsgType, qbftMessage.Round) {
				return ErrDuplicatedMessage
		    }
		}

		// Rule: Round must not be smaller then current peer's round -1 or +1. Only for non-decided messages
		if !mv.RoundBelongToAllowedSpread(peerID, qbftMessage.Round, signedSSVMessage.SSVMessage.MsgID) {
			return ErrEstimatedRoundNotInAllowedSpread
		}

		// Rule: Ignore if peer already advanced to a later round. Only for non-decided messages
		if mv.PeerAlreadyAdvancedRound(peerID, signedSSVMessage.SSVMessage.MsgID, qbftMessage.Height, qbftMessage.Round) {
			return ErrRoundAlreadyAdvanced
		}
	}

    // Rule: Can only exist for Proposal or Round-Change messages
    if !mv.ValidRoundChangeJustificationForMessageType(qbftMessage) {
		return ErrUnexpectedRoundChangeJustifications
    }

    // Rule: Can only exist for Proposal messages
    if !mv.ValidPrepareJustificationForMessageType(qbftMessage) {
		return ErrUnexpectedPrepareJustifications
    }

	return mv.ValidateQBFTMessageByDutyLogic(peerID, signedSSVMessage)
}
```

#### Duty Logic

|       Verification        |             Error             | Classification |                                                                                             Explanation                                                                                              |
| ------------------------- | ----------------------------- | -------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Already advanced slot     | ErrSlotAlreadyAdvanced        | Ignore         | (Non-committee roles) Signer already advanced to later slot.                                                                                                                                         |
| Invalid role for consensus  | ErrUnexpectedConsensusMessage | Reject         | SSVMessage.MsgID.Role must not be ValidatorRegistration or VoluntaryExit.                                                                                                                            |
| No beacon duty            | ErrNoDuty                     | Ignore         | If Proposal or Sync committee contribution duty, check if duty exists with beacon node.                                                                                                              |
| Slot not in time for role | ErrEarlySlotMessage or ErrLateSlotMessage              | Ignore         | Current time must be between duty's starting time and<br> +34 (committee and aggregator) or +3 (else) slots.                                                                                         |
| Too many duties per epoch | ErrTooManyDutiesPerEpoch      | Ignore         | If role is either aggregator, voluntary exit and validator registration,<br> it's allowed 2 duties per epoch. Else if committee,<br> 2*V (if no validator is doing sync committee).<br> Else accept. |
| Valid round for role      | ErrRoundTooHigh               | Reject         | For committee and aggregation, round can go up to 12. Else, it can go up to 6.                                                                                                                       |


```go

func (mv *MessageValidation) ValidateQBFTMessageByDutyLogic(peerID peer.ID, signedSSVMessage *types.SignedSSVMessage) error {

	/*
		Messages structures and checks (->)

			type SignedSSVMessage struct {
				Signatures  [][]byte
				OperatorIDs []OperatorID
				SSVMessage  *SSVMessage
				FullData    []byte
			}
			type SSVMessage struct {
				MsgType MsgType
				MsgID   MessageID -> Role must have consensus, Must be assigned to duty if role is Proposal or Sync Committee Aggregator
				Data 	[]byte
			}
			type Message struct {
				MsgType    				 MessageType
				Height     				 Height -> Must belong to spread for role, Satisfies a maximum number of duties per epoch for role, Must not be "old"
				Round      				 Round -> Must be below cut-off for role
				Identifier 				 []byte
				Root                     [32]byte
				DataRound                Round
				RoundChangeJustification [][]byte
				PrepareJustification     [][]byte
			}
	*/

	// Already verified
	var qbftMessage qbft.Message
	_ = qbftMessage.Decode(signedSSVMessage.SSVMessage.Data)

	// Rule: Height must not be "old". I.e., signer must not have already advanced to a later slot.
	if signedSSVMessage.SSVMessage.MsgID.GetRoleType() != types.RoleCommittee { // Rule only for validator runners
		if !mv.MessageFromOldSlot(peerID, signedSSVMessage.SSVMessage.MsgID, qbftMessage.Height) {
			return ErrSlotAlreadyAdvanced
		}
	}

	// Rule: Duty role has consensus (true except for ValidatorRegistration and VoluntaryExit)
	if !mv.ValidRoleForConsensus(signedSSVMessage.SSVMessage.MsgID.GetRoleType()) {
		return ErrUnexpectedConsensusMessage
	}

    // Rule: For proposal and sync committee aggregation duties, we check if the validator is assigned to it
    if err := mv.ValidBeaconDuty(signedSSVMessage.SSVMessage.MsgID.GetSenderID(), signedSSVMessage.SSVMessage.MsgID.GetRoleType(), phase0.Slot(qbftMessage.Height)); err != nil {
		return err
    }

	// Rule: current slot(height) must be between duty's starting slot and:
	// - duty's starting slot + 34 (committee and aggregation)
	// - duty's starting slot + 3 (other types)
	if err != mv.ValidDutySlot(peerID, phase0.Slot(qbftMessage.Height), signedSSVMessage.SSVMessage.MsgID.GetRoleType()); err != nil {
		// Err should be ErrEarlySlotMessage or ErrLateSlotMessage
		return err
	}

	// Rule: valid number of duties per epoch:
	// - 2 for aggregation, voluntary exit and validator registration
	// - 2*V for Committee duty (where V is the number of validators in the cluster) (if no validator is doing sync committee in this epoch)
	// - else, accept
	if !mv.ValidNumberOfDutiesPerEpoch(peerID, signedSSVMessage.SSVMessage.MsgID, phase0.Slot(qbftMessage.Height)) {
		return ErrTooManyDutiesPerEpoch
	}

	// Rule: Round cut-offs for roles:
	// - 12 (committee and aggregation)
	// - 6 (other types)
	if !mv.ValidRoundForRole(qbftMessage.Round, signedSSVMessage.SSVMessage.MsgID.GetRoleType()) {
		return ErrRoundTooHigh
	}

	return nil
}

func (mv *MessageValidation) ValidBeaconDuty() error {

	// Rule: For a proposal duty message, we check if the validator is assigned to it
	if signedSSVMessage.SSVMessage.MsgID.GetRoleType() == types.RoleProposer {
		if !mv.HasProposerDuty(signedSSVMessage.SSVMessage.MsgID.GetSenderID(), phase0.Slot(qbftMessage.Height)) {
			return ErrNoDuty
		}
	}

	// Rule: For a sync committee aggregation duty message, we check if the validator is assigned to it
	if signedSSVMessage.SSVMessage.MsgID.GetRoleType() == types.RoleSyncCommitteeContribution {
		if !mv.HasSyncCommitteeDuty(signedSSVMessage.SSVMessage.MsgID.GetSenderID(), phase0.Slot(qbftMessage.Height)) {
			return ErrNoDuty
		}
	}

    return nil
}

```

### Partial Signatures

#### Semantics

|        Verification        |                Error                | Classification | Explanation |
| -------------------------- | ----------------------------------- | -------------- | ----------- |
| More than one signer       | ErrPartialSigOneSigner              | Reject         | Must have only 1 signer.            |
| Unexpected FullData         | ErrFullDataNotInConsensusMessage    | Reject         | Must not have FullData.            |
| Unknown type               | ErrInvalidPartialSignatureType      | Reject         | Type not known.            |
| Wrong type for role        | ErrPartialSignatureTypeRoleMismatch | Reject         | Type must match role:<br> PostConsensusPartialSig for Committee,<br> RandaoPartialSig or PostConsensusPartialSig for Proposer,<br> SelectionProofPartialSig or PostConsensusPartialSig for Aggregator,<br> SelectionProofPartialSig or PostConsensusPartialSig for Sync committee contribution,<br> ValidatorRegistrationPartialSig for Validator Registration,<br> VoluntaryExitPartialSig for Voluntary Exit |
| No PartialSignatureMessage                 | ErrNoPartialSignatureMessages | Reject | Message must have at least one PartialSignatureMessage. |
| Wrong BLS Signature Size                   | ErrWrongBLSSignatureSize      | Reject | $\forall i$ PartialSignatureMessages.Message[i].Signature must have the correct length. |
| Inconsistent signer                        | ErrInconsistentSigners        | Reject | $\forall i$ PartialSignatureMessages.Message[i].Signer must be the same as the<br> SignedSSVMessage.OperatorIDs[i]. |
| Validtor's index mismatch                  | ErrValidatorIndexMismatch     | Ignore | $\forall i$ PartialSignatureMessages.Message[i].ValidatorIndex must belong to SSVMessage.SenderID(). |

```go

func (mv *MessageValidation) ValidatePartialSignatureMessageSemantics(peerID peer.ID, signedSSVMessage *types.SignedSSVMessage) error {

	/*
		Messages structures and checks (->)

			type SignedSSVMessage struct {
				Signatures  [][]byte
				OperatorIDs []OperatorID -> Must have len = 1
				SSVMessage  *SSVMessage
				FullData    []byte -> Must be empty
			}
			type SSVMessage struct {
				MsgType MsgType
				MsgID   MessageID
				Data 	[]byte
			}
			type PartialSignatureMessages struct {
				Type     PartialSigMsgType -> Valid, Aligned to MsgID.Role
				Slot     phase0.Slot
				Messages []*PartialSignatureMessage -> Mut not be empty
			}

			type PartialSignatureMessage struct {
				PartialSignature Signature -> Correct size
				SigningRoot      [32]byte
				Signer         	 OperatorID -> Consistent with SignedSSVMessage.OperatorIDs[0]
				ValidatorIndex 	 phase0.ValidatorIndex -> Must belongs to MsgID.Committee or Validator
			}
	*/

	// Already verified
	var partialSignatureMessages types.PartialSignatureMessages
	_ = partialSignatureMessages.Decode(signedSSVMessage.SSVMessage.Data)

	// Rule: Partial Signature message must have 1 signer
	signers := signedSSVMessage.GetOperatorIDs()
	if len(signers) != 1 {
		return ErrPartialSigOneSigner
	}
	signer := signers[0]

	// Rule: Partial signature message must not have full data
	if len(signedSSVMessage.FullData) > 0 {
		return ErrFullDataNotInConsensusMessage
	}

	// Rule: Valid signature type
	if !mv.ValidPartialSignatureType(partialSignatureMessages.Type) {
		return ErrInvalidPartialSignatureType
	}

	// Rule: Partial signature type must match expected type:
	// - PostConsensusPartialSig, for Committee duty
	// - RandaoPartialSig or PostConsensusPartialSig for Proposer
	// - SelectionProofPartialSig or PostConsensusPartialSig for Aggregator
	// - SelectionProofPartialSig or PostConsensusPartialSig for Sync committee contribution
	// - ValidatorRegistrationPartialSig for Validator Registration
	// - VoluntaryExitPartialSig for Voluntary Exit
	if !mv.ExpectedPartialSignatureTypeForRole(partialSignatureMessages.Type, signedSSVMessage.SSVMessage.MsgID) {
		return ErrPartialSignatureTypeRoleMismatch
	}

	// Rule: Partial signature message must have at least one signature
	if len(partialSignatureMessages.Messages) == 0 {
		return ErrNoPartialSignatureMessages
	}

	for _, psigMsg := range partialSignatureMessages.Messages {
		// Rule: Partial signature must have expected length
		if len(psigMsg.PartialSignature) != PartialSignatureSize {
			return ErrWrongBLSSignatureSize
		}
		// Rule: Partial signature signer must be consistent
		if psigMsg.Signer != signer {
			return ErrInconsistentSigners
		}
		// Rule: Validator index must match with validatorPK or one of CommitteeID's validators
		if !mv.ValidatorIndexBelongsToCommittee(psigMsg.ValidatorIndex, signedSSVMessage.SSVMessage.MsgID) {
			return ErrValidatorIndexMismatch
		}
	}

	return mv.ValidatePartialSigMessagesByDutyLogic(peerID, signedSSVMessage)
}
```

#### Duty Logic

|         Verification         |          Error           | Classification |                                                                                             Explanation                                                                                              |
| ---------------------------- | ------------------------ | -------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Already advanced slot        | ErrSlotAlreadyAdvanced   | Ignore         | (Non-committee roles) Signer already advanced to later slot.                                                                                                                                         |
| No beacon duty               | ErrNoDuty                | Ignore         | If Proposal or Sync committee contribution duty, check if duty exists with beacon node.                                                                                                              |
| Invalid signature type count | ErrInvalidPartialSignatureTypeCount                | Reject         | It's allow only:<br> 1 PostConsensusPartialSig, for Committee duty,<br> 1 RandaoPartialSig and 1 PostConsensusPartialSig for Proposer,<br> 1 SelectionProofPartialSig and 1 PostConsensusPartialSig for Aggregator,<br> 1 SelectionProofPartialSig and 1 PostConsensusPartialSig for Sync committee contribution,<br> 1 ValidatorRegistrationPartialSig for Validator Registration,<br> 1 VoluntaryExitPartialSig for Voluntary Exit. |
| Slot not in time for role    | ErrEarlySlotMessage or ErrLateSlotMessage         | Ignore         | Current time must be between duty's starting time and<br> +34 (committee and aggregator) or +3 (else) slots.                                                                                         |
| Too many duties per epoch    | ErrTooManyDutiesPerEpoch | Ignore         | If role is either aggregator, voluntary exit and validator registration,<br> it's allowed 2 duties per epoch. Else if committee,<br> 2*V (if no validator is doing sync committee).<br> Else accept. |
| Too many partial signatures  | ErrTooManyPartialSignatureMessages                | Reject         | For the committee role, it's allowed $min(2*V, V + $ SYNC_COMMITTEE_SIZE $)$ <br> where $V$ is the number of committee's validatos.<br> For sync committee contribution, it's allowed 13.<br> Else, only 1.  |
| Triple validator index       | ErrTripleValidatorIndexInPartialSignatures                | Reject         | A validator index can not be associated to more than 2 signatures. |


```go

func (mv *MessageValidation) ValidatePartialSigMessagesByDutyLogic(peerID peer.ID, signedSSVMessage *types.SignedSSVMessage) error {

	/*
		Messages structures and checks (->)

			type SignedSSVMessage struct {
				Signatures  [][]byte
				OperatorIDs []OperatorID
				SSVMessage  *SSVMessage
				FullData    []byte
			}
			type SSVMessage struct {
				MsgType MsgType
				MsgID   MessageID -> Must be assigned to duty if role is Proposal or Sync Committee Aggregator
				Data 	[]byte
			}
			type PartialSignatureMessages struct {
				Type     PartialSigMsgType -> Message count rules
				Slot     phase0.Slot -> Must belong to allowed spread, Satisfies a maximum number of duties per epoch for role, Must not be "old"
				Messages []*PartialSignatureMessage -> Valid number of signatures (3 cases: committee duty, sync committee contribution, others)
			}

			type PartialSignatureMessage struct {
				PartialSignature Signature
				SigningRoot      [32]byte
				Signer         	 OperatorID
				ValidatorIndex 	 phase0.ValidatorIndex -> Can't appear more than 2 times for role Committee
			}
	*/

	// Already verified
	var partialSignatureMessages types.PartialSignatureMessages
	_ = partialSignatureMessages.Decode(signedSSVMessage.SSVMessage.Data)


	// Rule: Height must not be "old". I.e., signer must not have already advanced to a later slot.
	if signedSSVMessage.SSVMessage.MsgID.GetRoleType() != types.RoleCommittee { // Rule only for validator runners
		if !mv.MessageFromOldSlot(peerID, signedSSVMessage.SSVMessage.MsgID, partialSignatureMessages.Slot) {
			return ErrSlotAlreadyAdvanced
		}
	}


    // Rule: For proposal and sync committee aggregation duties, we check if the validator is assigned to it
    if err := mv.ValidBeaconDuty(signedSSVMessage.SSVMessage.MsgID.GetSenderID(), signedSSVMessage.SSVMessage.MsgID.GetRoleType(), partialSignatureMessages.Slot); err != nil {
		return err
    }

	// Rule: peer must send only:
	// - 1 PostConsensusPartialSig, for Committee duty
	// - 1 RandaoPartialSig and 1 PostConsensusPartialSig for Proposer
	// - 1 SelectionProofPartialSig and 1 PostConsensusPartialSig for Aggregator
	// - 1 SelectionProofPartialSig and 1 PostConsensusPartialSig for Sync committee contribution
	// - 1 ValidatorRegistrationPartialSig for Validator Registration
	// - 1 VoluntaryExitPartialSig for Voluntary Exit
	if err := mv.ValidPartialSigMessageCount(peerID, signedSSVMessage.SSVMessage.MsgID, &partialSignatureMessages); err != nil {
		return ErrInvalidPartialSignatureTypeCount
	}

	// Rule: current slot must be between duty's starting slot and:
	// - duty's starting slot + 34 (committee and aggregation)
	// - duty's starting slot + 3 (other duties)
	if err := mv.ValidDutySlot(peerID, partialSignatureMessages.Slot, signedSSVMessage.SSVMessage.MsgID.GetRoleType()); err != nil {
		// Err should be ErrEarlySlotMessage or ErrLateSlotMessage
		return err
	}

	// Rule: valid number of duties per epoch:
	// - 2 for aggregation, voluntary exit and validator registration
	// - 2*V for Committee duty (where V is the number of validators in the cluster) (if no validator is doing sync committee in this epoch)
	// - else, accept
	if !mv.ValidNumberOfDutiesPerEpoch(peerID, signedSSVMessage.SSVMessage.MsgID, partialSignatureMessages.Slot) {
		return ErrTooManyDutiesPerEpoch
	}

	if signedSSVMessage.SSVMessage.MsgID.GetRoleType() == types.RoleCommittee {

		// Rule: The number of signatures must be <= min(2*V, V + SYNC_COMMITTEE_SIZE) where V is the number of validators assigned to the cluster
		if !mv.ValidNumberOfSignaturesForCommitteeDuty(signedSSVMessage.SSVMessage.MsgID.GetSenderID(), &partialSignatureMessages) {
			return ErrTooManyPartialSignatureMessages
		}

		// Rule: a ValidatorIndex can't appear more than 2 times in the []*PartialSignatureMessage list
		if !mv.NoTripleValidatorOccurrence(&partialSignatureMessages) {
			return ErrTripleValidatorIndexInPartialSignatures
		}
	} else if signedSSVMessage.SSVMessage.MsgID.GetRoleType() == types.RoleSyncCommitteeContribution {
		// Rule: The number of signatures must be <= MaxSignaturesInSyncCommitteeContribution for the sync comittee contribution duty
		if len(partialSignatureMessages.Messages) > MaxSignaturesInSyncCommitteeContribution {
			return ErrTooManyPartialSignatureMessages
		}
	} else {
		// Rule: The number of signatures must be 1 for the other types of duties
		if len(partialSignatureMessages.Messages) > 1 {
			return ErrTooManyPartialSignatureMessages
		}
	}

	return nil
}
```


### Observations

- Proposal and round-change justifications were not included because they are too complex to implement at the message validation level. The cost of adding this complexity is not justified since the message count check already prevents any related attack.


### Rules suggestions for future
- Priority-based message handling: priority based on type and sender.
- Message aggregation: aggregate similar messages (e.g. wait for a quorum of prepares, commits, round-changes, partial-sig) before delivering to the app.