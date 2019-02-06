# Data Structures

This document serves as an entry point for understanding all of the data structures in filecoin.

## CID

For most objects referenced by Filecoin, a Content Identifier (CID for short) is used. This is effectively a hash value, prefixed with its hash function (multihash) prepended with a few extra labels to inform applications about how to deserialize the given data. To learn more, take a look at the [CID Spec](https://github.com/ipld/cid). 

CIDs are serialized by applying binary multibase encoding, then encoding that as a CBOR byte array with a tag of 42.

## Block

A block represents an individual point in time that the network may achieve consensus on. It contains (via merkle links) the full state of the system, references to the previous state, and some notion of a 'weight' for deciding which block is the 'best'.

```go
// Block is a block in the blockchain.
type Block struct {
	// Miner is the address of the miner actor that mined this block.
	Miner Address

	// Ticket is the winning ticket that was submitted with this block.
	Ticket Signature

	// Parents is the set of parents this block was based on. Typically one,
	// but can be several in the case where there were multiple winning ticket-
	// holders for a round.
	Parents []Cid

	// ParentWeightNum is the numerator of the aggregate chain weight of the parent set.
	ParentWeightNum BigInteger

	// ParentWeightDenom is the denominator of the aggregate chain weight of the parent set
	ParentWeightDenom BigInteger 

	// Height is the chain height of this block.
	Height Uint64
    
    // StateRoot is a cid pointer to the state tree after application of the
	// transactions state transitions.
	StateRoot Cid

	// Messages is the set of messages included in this block
	// TODO: should be a merkletree-ish thing
	Messages []SignedMessage

	// MessageReceipts is a set of receipts matching to the sending of the `Messages`.
    // TODO: should be the same type of merkletree-list thing that the messages are
	MessageReceipts []MessageReceipt
}
```

## Message

```go
type Message struct {
	To   Address
	From Address
	
	// When receiving a message from a user account the nonce in
	// the message must match the expected nonce in the from actor.
	// This prevents replay attacks.
	Nonce Uint64

	Value BigInteger

	Method string
	
	Params []byte
}
```

### Parameter Encoding

Parameters are each individually ABI encoded (see [abi encoding](actrors.md#abi-types-and-encodings)) and then encoded as a CBOR array. (TODO: thinking about this, it might make more sense to just have `Params` be an array of things)

### Signing

A signed message is a wrapper type over the base message.

```go
type SignedMessage struct {
    Message Message
    Signature Signature
}
```

The signature is a serialized signature over the serialized base message. For more details on how the signature itself is done, see the [signatures spec](signatures.md).

## MessageReceipt

```go
type MessageReceipt struct {
    ExitCode uint8
    
    Return [][]byte
    
    GasUsed BigInteger
}
```

## Actor

```go
type Actor struct {
    // Code is a pointer to the code object for this actor
	Code    Cid
    
    // Head is a pointer to the root of this actors state
	Head    Cid
    
    // Nonce is a counter of the number of messages this actor has sent
	Nonce   Uint64
    
    // Balance is this actors current balance of filecoin
	Balance BigInteger
}
```

## State Tree

The state trie keeps track of all state in Filecoin. It is effectively a map of addresses to `actors` in the system. It is implemented using a HAMT.

## HAMT

TODO: link to spec for our CHAMP HAMT



# Filecoin Compact Serialization

Datastructures in Filecoin are encoded as compactly as is reasonable. At a high level, each object is converted into an ordered array of its fields (ordered by their appearance in the struct declaration), then CBOR marshaled, and prepended with an object type tag.

| object | tag  |
|---|---|
| block v1 | 43  |
| message v1 | 44 |
| signedMessage v1 | 45 |
| actor v1 | 46 |

For example, a message would be encoded as:

```cbor
tag<44>[msg.To, msg.From, msg.Nonce, msg.Value, msg.Method, msg.Params]
```

Each individual type should be encoded as specified:

| type | encoding |
| --- | ---- |
| Uint64 | CBOR major type 0 |
|  BigInteger | [CBOR bignum](https://tools.ietf.org/html/rfc7049#section-2.4.2) |
| Address | CBOR major type 2 |
| Uint8 | CBOR Major type 0 |