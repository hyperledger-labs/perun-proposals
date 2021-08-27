<!-- This is a template for proposing design changes to the perun project. -->

# Proposal: Watching service for IoT adoption

* Author(s): Manoranjith
* Status: propose
* Related issue: [perun-proposals#008](https://github.com/hyperledger-labs/perun-proposals/pull/008)


<!-- Use the above format for issues on github and full links for issues on other platforms. -->

## Summary

<!-- Provide a tl;dr summary -->
The high level requirements for adopting the perun-framework for IoT use cases
was described in [perun-proposal#003](./003-IoT-Adoption.md). Detailed
description of the functionality and design for implementing the watching
component is presented in this proposal.


## Motivation

<!-- introduction to the problem being solved & its background.  -->

In the current implementation of the framework (go-perun v0.6.0), the watcher is
integrated into `client.Channel` type and an instance must be started for each
channel.

However, it is desirable to run the watcher as a separate component for IoT use
cases. Because, for the perun protocol to work, the watcher should be actively
watching the blockchain for any states being registered and; if the registered
state was not the latest off-chain state, the watcher must immediately register
the latest off-chain state.  Given the connectivity and power constraints of
IoT devices, it might not be possible to meet these requirements if watching
service is running on the IoT device.

The current design cannot be extended to implement a watcher that can run
as a remote service. Hence, a new design is proposed where, the watcher can be run:
1. As a part of go-perun client itself (for capable hardware).
2. As an independent component on the same or different computer (for
   constrained hardware).

## Details

<!-- Provide a detailed description of the proposal. -->

### Functionalities of the watcher

1. User should be able to initialize the watcher without associating it to any
   particular channel. After initialization, it must have access to a wallet
   containing keys for an on-chain account. The account must have sufficient
   funds to pay the transaction fees when registering off-chain states.

2. Main component should be able to request the watcher to start/stop watching
   for on-chain events for a given channel ID.

3. After a watcher starts watching for a given channel,

   1. Main component should be able to periodically send the newer off-chain
      states to the watcher.

   2. Main component should be notified by the watcher, when a state is
      registered or progressed on the blockchain.

4. If a state is registered on the blockchain and is not the latest off-chain
   state known to the watcher, then it must register the latest off-chain state
   on the blockchain.

5. User should be able to shutdown the watcher. In which case, the watcher must
   complete any ongoing dispute resolution or force execution process; close
   the pub-sub interfaces on all of the channels and then terminate itself.

### Interaction between the watcher and the main component

This section presents an overview of the interaction between the watcher and
main component.

![Interaction between the main component and watcher](004/watcher.svg)

Figure 1: Interaction between the main component and watcher.

### Interfaces of the watcher

This section presents a description of the interfaces of the watcher. These
interfaces are designed based on the interaction between the watcher and the
main component presented in the previous section.

#### The watcher interface


Any implementation of watcher must satisfy this interface. Usage of methods in
this interface is described in later sections.

```go
type Watcher interface {
    StartWatching(ChannelID) (StatesPub, AdjudicatorEventsSub, error)
    StopWatching(ChannelID) error
    Shutdown() error
}
```

#### Initializing and shutting down a watcher


```go
// NewWatcher instantiates the watcher.
func NewWatcher(cfg Config) (Watcher, error) {...}

// Shutdown gracefully terminates the watcher.
//
// For each of the channels, it ensures any ongoing dispute resolution or
// force execution process is completed and then closes the StatesSub and
// AdjudicatorEventsPub for that channel.
func (w *Watcher) Shutdown() error {...}
```
The configuration parameter in `NewWatcher` API is specific to the
implementation of watcher.

#### RegistererSubscriber interface
It is used by the watcher for interacting with the blockchain.

```go
// RegistererSubscriber is composed of Registerer and Subscriber interfaces.
// These are used by the watcher to interact with the blockchain.
type RegistererSubscriber interface {
    Registerer
    Subscriber
}

// Registerer is used to register the off-chain state of a channel, along
// with the states of all its sub-channels and virtual channels, on the
// blockchain. Channel ID is contained in the State.
//
// It encapsulates a connection to the blockchain and an access to an
// on-chain account that will be used for sending register transactions.
type Registerer interface {
    Register(State, SubStates) error
}

// Subscriber is used for subscribing to adjudicator events on the
// blockchain for a given channel ID.
//
// It encapsulates a connection to the blockchain.
type Subscriber interface {
    Subscribe(ChannelID) (AdjudicatorEventSub, error)
}
```

**Note**:
[`AdjudicatorEventSub`](https://github.com/hyperledger-labs/go-perun/blob/44cbda6af209edee2413102a91dec2289b0a569e/channel/adjudicator.go#L105)
is already implemented in go-perun.

#### To start/stop watching for channels

```go
// StartWatching starts watching for adjudicator events for the channel.
//
// Main component can send the newer off-chain states via the StatesPub.
// Main component can receive AdjudicatorEvents via the AdjudicatorEventsSub.
func (w *Watcher) StartWatching(ChannelID) (StatesPub, AdjudicatorEventsSub, error) {...}

// StopWatching stops watching for adjudicator events for the channel.
func (w *Watcher) StopWatching(ChannelID) error {...}
```

#### For sending off-chain states from the main component to watcher

In go-perun,
[`State`](https://github.com/hyperledger-labs/go-perun/blob/44cbda6af209edee2413102a91dec2289b0a569e/channel/state.go#L31)
type is already defined. However, this does not include the signatures of
participants on the state. Hence, a new type (say `StateWithSignatures`) should
be defined that include a `State` and signatures of all participants on the
state.

```go
// StatesPub is used by the main component to send newer off-chain states to
// the watcher.
type StatesPub interface {
    // Publish publishes the given state to all the subscribers.
    // Returns an error if the subscription was closed.
    Publish(StateWithSignatures) error

    // Close closes the publisher instance and all the subscriptions associated
    // with it. Any further call to Publish should immediately return error.
    Close() error
}
```

#### For notifying the main component of registered and progressed events

[`AdjudicatorEvent`](https://github.com/hyperledger-labs/go-perun/blob/44cbda6af209edee2413102a91dec2289b0a569e/channel/adjudicator.go#L122)
and
[`AdjudicatorEventsSub`](https://github.com/hyperledger-labs/go-perun/blob/44cbda6af209edee2413102a91dec2289b0a569e/channel/adjudicator.go#L105)
are already defined and implemented in go-perun for a different purpose (for
receiving Adjudicator events from the blockchain.

The same interface can be used to receive notification on Adjudicator events
from the watcher.

```go
// AdjudicatorEvent could be a registered event or a progressed event.
// This type is already defined and implemented in go-perun.
type AdjudicatorEvent interface {...}

// AdjudicatorEventsSub is used by the main component to receive notifications
// of adjudicator events from the watcher.
type AdjudicatorEventsSub interface {
    // Next returns the most recently published state in the subscription.
    // Returns nil if the subscription is closed or any other error occurs.
    Next() AdjudicatorEvent

    // Err returns the error status of the subscription. After Next returns nil,
    // Err should be checked for an error.
    Err() error

    // Close closes the subscription. After subscription closes, any call to
    // Next should immediately return nil and any call to Err should return
    // an error.
    Close() error
}
```

### How different types of channels should be handled by watcher

#### When main component registers a channel with the watcher

1. For ledger channel: only its channel ID and latest off-chain state at the
   time of registering is sufficient.

   After registering a ledger channel, it is the main component's responsibility
   to individually register all its sub-channels and virtual channels.

2. For sub-channel: sub-channel ID, parent ledger channel ID and, the latest
   off-chain states of both the sub-channel and parent ledger channel are
   required.

3. For virtual channel: virtual channel ID, relevant parent channel ID and, the
   latest off-chain states of both the virtual channel and the relevant parent
   ledger channel are required.

   Relevant parent ledger channel is the ledger channel between this user and
   the common intermediary.

#### When main component sends new off-chain states for the watcher

For all types of channels, watcher will have to store the newer state and
discard the older ones.

#### When registering states on the blockchain

1. For ledger channel: the latest off-chain states of the ledger channel and,
   of its sub-channels, virtual channels must be registered.

   In some scenarios (for common intermediary of a virtual channel), the latest
   off-chain state of the virtual channel might not be known. In this case,
   whatever state known to it can be registered. Other participants will refute
   with the latest state.

2. In case sub-channel; the latest off-chain state of the parent ledger
   channel, of the sub-channel and, of all the other sub-channels, virtual
   channels funded by the parent ledger channel must be registered.

2. In case virtual channel; the latest off-chain state of the relevant parent
   ledger channel, of the sub-channel and, of all the other sub-channels,
   virtual channels funded by the relevant parent ledger channel must be
   registered.

   Relevant parent ledger channel is the ledger channel between this user and
   the common intermediary.

## Rationale

<!-- Provide a discussion of alternative approaches and trade offs; advantages
and disadvantages of the specified approach.  -->

1. Watcher is associated with the client, because `StartWatching` (after
   opening a channel) and `StopWatching` (after closing a channel) are done in
   the context of a client.

2. `StatesPub` and `AdjudicatorEventsSub` are associated with a channel because
   generating newer off-chain states and reacting to adjudicator events are
   done in the context of a channel.

## Impact

<!-- Choose the level of impact this proposal will have: -->

<!-- Minor (Does not impact any existing features) -->
<!-- Major (Breaks one or more existing features) -->
<!-- New Feature (Introduces a functionality) -->
Architecture (Requires a modification of the architecture)

## Implementation

<!-- Provide a description of the implementation aspects. -->

### Implementing the watcher

1. The `Register` and `Subscribe` methods are already part of `Adjudicator`
   interface. Hence, the existing adjudicator implementations can be used as
   `RegisterSubscriber`.

2. The `State` method on a channel returns the off-chain state without the
   signatures. So, a new method (say `StateWithSignatures`) to retrieve the
   latest off-chain state along with signatures should be added.

3. To implement a watching component that works locally,

    1. The watcher component can be initialized to run as a go-routine.
    2. The pub-sub interfaces can be implemented using `go-channels`.

4. To implement a watching component that runs as a remote service,

    1. The watching component can be initialized to run as an independent
       program.
    2. The main program can connect with the watching component via remote
       interface.
    3. The pub-sub can be implemented using protocols such as `gRPC` or `MQTT`.

### Integration with go-perun

One possible way of integrating the watcher API into go-perun is suggested.
The suggestion tries to make minimal changes to how the watcher is currently
being used.

#### Setup

Similar to the other components, an instance of watcher must be initialized and
passed to the `client.New` API.

```go
// Either initialize a local watcher
w, err := local.NewWatcher(cfg local.Config)

// Or connect to the remote watcher
w, err := remote.NewWatcher(cfg remote.Config)

// Like the other components (funder, adjudicator, message bus) etc.,
// watcher can be passed as an argument to the `client.New` API.
//
// Integrating the watcher API into the client is described in a following
// section.
client, err := pclient.New(
    offChainAddress,
    msgBus,
    funder,
    adjudicator,
    offChainWallet,
    watcher)
```

#### Usage

1. After a channel is opened, user calls the `Watch` function on the channel.
   The existing implementation of the `Watch` function can be replaced with one
   shown below.

    ```go
    func (ch *Channel) Watch(h AdjudicatorEventHandler) {
       watcher := ch.client.watcher

       statesPub, eventsSub, err := watcher.StartWatching(ch.ID())
       // Handle err.

       // Register "StopWatching" to be called when the channel is closed.
       c.OnCloseAlways(func() { watcher.StopWatching(ch.ID()) }

       // Set the "statePub" handler for the channel.
       // On  each off-chain update, the updated state along with signatures
       // will be published on this handler.
       err = ch.setStatePub(statePub)
       // Handle err.

       for {
          e, err := eventSub.Next()
          if err != nil {
             return err
          }
          h(e) // Handle event.
       }
    }
    ```

2. Add a method `setStatesPub` and a field `statesPub` on `client.Channel`
   type.

    ``` go
    type Channel struct {
    ...
    statesPub watcher.StatesPub
    ...
    }

    func (ch *Channel) setStatesPub (p watcher.StatesPub) {
    ch.statesPub = p
    }
    ```

3. On each state update, publish the state along with the signatures by calling
   the `ch.statesPub.Publish` API. This should be done in two places: when
   [proposing](https://github.com/hyperledger-labs/go-perun/blob/1fc30ba43eaae379ed5c566bdcf79edafc598b88/client/update.go#L217)
   and when
   [accepting](https://github.com/hyperledger-labs/go-perun/blob/1fc30ba43eaae379ed5c566bdcf79edafc598b88/client/update.go#L370).
