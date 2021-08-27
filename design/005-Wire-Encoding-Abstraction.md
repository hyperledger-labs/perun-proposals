# Proposal: Abstraction of Wire Messages Encoding

* Author(s): Sebastian Stammler (@sebastianst)
* Status: accepted
* Related issue(s): https://github.com/hyperledger-labs/go-perun/issues/167
* Affected repository: go-perun

## Summary

The goal is to be able to choose different de/-serialization methods for wire
messages when using go-perun's `wire` package for inter-node wire message
exchange. After the changes, it should be possible to choose protobuf
serialization.

## Motivation

Wire messages in go-perun currently use a home-baked and non-exchangeable
encoding and decoding format. When developing new Perun channel client
implementations, it would currently be necessary to mirror this custom
serialization format.

## Details

Wire messages in go-perun currently use a home-baked and non-exchangeable
encoding and decoding format, as specified by `Encode` and `Decode`
methods on wire messages (see, e.g., message definitions in `client/*msgs.go`),
methods `Envelope.(En|De)code` and functions `(En|De)code` in `wire/msg.go`.

It should be possible to choose a different wire serialization protocol, e.g., protobuf.

### Note on `pkg/io` serialization

Note that the `pkg/io` serialization is _not_ Go-specific. It is a packed,
little-endian binary encoding format and the serialization logic of actual Go
types is defined in package `pkg/io` and in particular in file
`pkg/io/serialize.go`. (However, I just realize that `*big.Int`s are actually
encoded in big-endian byte order...so we might as well switch to big-endianess
in `pkg/io/serialize.go` to be more consistent
([related issue](https://github.com/hyperledger-labs/go-perun/issues/167)).)

## Rationale

See #motivation.

## Impact

New Feature

## Implementation

Luckily, this can be achieved with a few changes to package `wire` and the
`wire.Msg` implementations itself.

The core idea is to create an envelope serialization backend, which can be set
at runtime (e.g., using `init()` of the respecive serialization backend
implementation). This backend's functions will then be used by
`Envelope.(En|De)code` of package `wire`. The `pkg/io` serialization would then
just be the default backend. Furthermore, the `Encode` methods of all `Msg`
implementations would also be moved to this backend.

Some of the proposed steps:
* Create _Wire Serialization Backend_
```go
package wire

// EnvelopeSerializer is a serialization backend for wire messages. It should be
// set by the implementation package's init() function.
type EnvelopeSerializer interface {
	Encode(w io.Writer, env *Envelope) error
	Decode(r io.Reader) (*Envelope, error)
}

// current serializer
var envelopeSerializer EnvelopeSerializer

func EncodeEnvelope(env *Envelope, w io.Writer) error {
	return envelopeSerializer.Encode(env, w)
}
// same for DecodeEnvelope

// Allows to set the serialization backend at most once in an init function.
// Panics if called twice.
func SetEnvelopeSerializer(EnvelopeSerializer)
```
* `Envelope.(En|De)code` just call the functions of this backend, or we remove
  those functions altogether and use the free functions `(En|De)codeEnvelope`. I
  prefer the latter option.

* Current implementations of `Envelope.(En|De)code`, which are `pkg/io`
  specific, will be moved to a new backend implementation package
  `wire/perun`.
  * Together with this move many other functions currently in package `wire`,
    like `(En|De)code`, `DecodeAddress` and possibly many others that are
    specific to `pkg/io` serialization.

* We remove `perun.Encoder` from the `Msg interface`, Since en/decoding is now
  injected via the backend.

* We also move all implementations of methods `(En|De)code` on all `wire.Msg`
  implementations, which currently are contained in package `client`, into
  the new `wire/perun` package.
  * This would also make the scope of the `client` package cleaner.
  * I'd propose to register the encoders in a mapping similar to the `decoders`
    in file `wire/msg.go`.

Note that it is sufficient to create an abstraction of the envelope
serialization, because the `wire/net.IoConn` (`wire/net/ioconn.go`) is the only
relevant place right now where `Envelope` en/decoding happens.

This can be implemented in a backwards-compatible fashion, where `pkg/io`
encoding would be the default and other envelope serializers could be injected
at runtime.

### Client Persistence

Note that the channel persistence implementation in
`channel/persistence/keyvalue` needs the `(En|De)code` methods of the types
`State` and `Params`. So we might either move them into package
`channel/persistence` or even a new package `channel/io`. Or we just leave them
as methods in package `channel`. This last option, however, would mean that the
`channel` package keeps its dependence on `pkg/io`, which we could get rid of as
it is conceptionally cleaner.
