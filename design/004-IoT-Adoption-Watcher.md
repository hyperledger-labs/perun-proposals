<!-- This is a template for proposing design changes to the perun project. -->

# Proposal: Watching service for IoT adoption

* Author(s): Manoranjith
* Status: accepted
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

### Interaction between the watcher and the main component

This section presents an overview of the interaction between the watcher and
main component.

![Interaction between the main component and watcher](004/watcher.svg)

Figure 1: Interaction between the main component and watcher.

### Interfaces of the watcher

This section presents a description of the interfaces of the watcher. These
interfaces are designed based on the interaction between the watcher and the
main component presented in the previous section.

#### For initializing and shutting down the watcher

```go
// Watcher represents the watching component.
type Watcher struct {...}

// NewWatcher instantiates the watcher.
//
// The RegistererSubscriber instance will be used by the watcher for
// interacting with the blockchain.
func NewWatcher(rs RegistererSubscriber) (*Watcher, error) {...}

// BindToGrpcServer binds the API of Watcher to a gRPC server and makes it
// accessible via gRPC protocol.
func BindToGrpcServer(w *Watcher) (error) {...}

// Shutdown completes any on-going refutation processes,
// closes all the subscriptions and terminates the watcher gracefully.
func (w *Watcher) Shutdown() error {...}
```

The shutdown call is used to terminate the watcher instance and notify all the
main component of the same. On shutdown call, the watcher must,
1. Stop watching for all the channel IDs it has been watching.
2. Complete any on-going refutation process.
3. Close the `OffChainSub` and `OnChainEventsPub` on each of the channels. This
   ensures that the channels are notified of the watcher shutdown.
4. Finally, it should stop.


RegistererSubscriber interface used by the watcher for interacting with the
blockchain.

```go
// RegistererSubscriber is composed of Registerer and Subscriber interfaces.
// These are used by the watcher to interact with the blockchain.
type RegistererSubscriber interface {
    Registerer
    Subscriber
}

// Registerer provides an interface to register the off-chain state of a
// channel along with the states of all its sub-channels and virtual channels
// on the blockchain. Channel ID is contained in the State.
// 
// It must encapsulate a connection to the blockchain and an access to an
// on-chain account that will be used for sending register transactions.
type Registerer interface {
    Register(State, SubStates) error
}

// Subscriber provides an interface for subscribing to all events on the
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

#### For connecting to the watcher and start/stop watching for channels

Watcher must be initialized before connecting to it. When using a

1. Local watcher, the watcher instance initialized in the previous step can be
   directly used.

2. Remote watcher, a connection must be established with the remote watcher
   instance before using it.

```go

// ConnectToWatcherGrpc establishes a connection with a remote watcher via
// gRPC protocol.
//
// It returns a watcher instance that implements client stubs for interacting
// with the remote watcher.
func ConnectToWatcherGrpc(cfg Config) (Watcher, error) {...}

// StartWatching starts watching for adjudicator events for given a channel.
//
// Watcher will receive the newer off-chain states from the main component via
// the StatesSub. It will notify the main component of the adjudicator events
// via the AdjudicatorEventsPub.
func (w *Watcher) StartWatching(ChannelID, StatesSub, AdjudicatorEventsPub) error {...}

// To stop watching for adjudicator events for given a channel.
func (w *Watcher) StopWatching(ChannelID) error {...}
```

#### For sending off-chain states from the main component to watcher

[`State`](https://github.com/hyperledger-labs/go-perun/blob/44cbda6af209edee2413102a91dec2289b0a569e/channel/state.go#L31)
is already implemented in go-perun.

```go
// StatesPubSub is used for sending newer off-chain states from the main
// component to the watcher.
type StatesPubSub interface {
   StatesPub
   StatesSub
}

// StatesPub is used by the main component to send newer off-chain states to
// the watcher.
type StatesPub interface {
    // Publish publishes the given state to all the subscribers.
    // Returns an error if the subscription was closed.
    Publish(State) error
    
    // Close closes the publisher instance and all the subscriptions associated
    // with it. Any call to Publish should immediately return error.
    Close() error
}

// StatesSub is used by the watcher to receive newer off-chain states that
// are sent by the main component.
type StatesSub interface {
    // Next returns the most recently published state in the subscription.
    // Returns nil if the subscription is closed or any other error occurs.
    Next() State
    
    // Err returns the error status of the subscription. After Next returns nil,
    // Err should be checked for an error.
    Err() error
    
    // Close closes the subscription. After subscription closes, any call to
    // Next should immediately return nil and any call to Err should return
    // an error.
    Close() error
}
```

#### For notifying the main component of registered and progressed events

[`AdjudicatorEvent`](https://github.com/hyperledger-labs/go-perun/blob/44cbda6af209edee2413102a91dec2289b0a569e/channel/adjudicator.go#L122) and [`AdjudicatorEventsSub`](https://github.com/hyperledger-labs/go-perun/blob/44cbda6af209edee2413102a91dec2289b0a569e/channel/adjudicator.go#L105)
are already defined and implemented in go-perun. These are used by the watcher
to receive different types of Adjudicator events (including registered and
progressed events) from the blockchain. Same interface can be used by the
watcher to notify the main component.

```go
// AdjudicatorEvent could be a registered event or a progressed event.
// This type is already defined and implemented in go-perun.
type AdjudicatorEvent interface {...}

// AdjudicatorEventsPubSub is used by the watcher for notifying the main
// component of registered and progressed events for the channel.
type AdjudicatorEventsPubSub interface {
   AdjudicatorEventsPub
   AdjudicatorEventsSub
}

// AdjudicatorEventsPub is used by the watcher to send notifications of
// adjudicator events to the main component.
type AdjudicatorEventsPub interface {
    // Publish publishes the given AdjudicatorEvent to all the subscribers.
    // Returns an error if the subscription was closed.
	Publish(AdjudicatorEvent) error
    
    // Close closes the publisher instance and all the subscriptions associated
    // with it. Any call to Publish should immediately return error.
    Close() error
}

// AdjudicatorEventsSub is used by the main component to receive notifications of 
// on-chain events sent by the watcher.
//
// This is already defined in go-perun/channel package. Same can be used.
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

#### Note on pub-sub interfaces between the main component and the watcher

The main component initializes and passes the subscriber instance for off-chain
states and the publisher instance for on-chain events to the watcher. Because,

1. Initial implementation can be a one channel to one watcher pub-sub.

2. Later, if one channel wants to use multiple watcher instances (because one
   watcher could not provide expected level of availability), it can be
   achieved by extending the pub-sub implementation while retaining the same
   interfaces. For this, the main component could
   
   1. Initialize a one to many pub-sub for off-chain states and pass a new
      instances of subscriber to each watcher instance. Main component can send
      newer off-chain states on the single publisher instance.
      
   2. Initialize a many to one pub-sub for on-chain events and pass a new
      instance of publisher to each watcher instance. Main component can
      receive all the on-chain events on the single subscriber instance.

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

2. In case sub-channel; the latest off-chain state of the parent ledger
   channel, of the sub-channel and, of all the other sub-channels, virtual
   channels funded by the parent ledger channel must be registered.

2. In case virtual channel; the latest off-chain state of the relevant parent
   ledger channel, of the sub-channel and, of all the other sub-channels,
   virtual channels funded by the relevant parent ledger channel must be
   registered.

   Relevant parent ledger channel is the ledger channel between this user and
   the common intermediary.

### Sample Usage

The main component can either initialize a local watcher or connect to a remote
watcher.

#### Using the watcher from the main component

```go
// For initializing a local watcher
w, err := Watcher(rs RegistererSubscriber)

// For connecting to the remote watcher
w, err := ConnectToWatcherGrpc(cfg Config)

// After establishing a channel (say "ch1"),
// 1. Setup the subscription,
// 2. Instruct the watcher to start watching for events on that channel and,
// 3. Publish the initial state.
statesPub, statesSub := newStatesPubSub()
adjudicatorEventsPub, adjudicatorEventsSub := newAdjudicatorEventsPubSub()

err := w.StartWatching(ch1.ID(), statesSub, adjudicatorEventsPub)

statesPub.Publish(ch.StateWithSignatures())

// Start a handler for adjudicator events.
go func() {
    adjEvent := adjudicatorEventsSub.Next()
    if adjEvent == nil {
        err := adjudicatorEventsSub.Err()
        // handleError and probably return.
    } else {
        // handleEvent
    }
}

// When new off-chain states are created, publish them.
statesPub.Publish(ch.StateWithSignatures())

// Instruct the watcher to stop watching for events.
err := w.StopWatching(ch1.ID())
```

#### Gracefully shutting down the watcher

When the watching service is to be terminated gracefully, the `Shutdown` method
is used. Any on-going refutations are completed and all the open subscriptions
are closed before terminating.

This is especially useful when using watcher as remote service.

```go
err := w.Shutdown()
// Log the error.
```

## Rationale

<!-- Provide a discussion of alternative approaches and trade offs; advantages
and disadvantages of the specified approach.  -->

The proposed design

- Ensures watching component can be run either locally or as a remote service
  and be connected to the blockchain in either case.

- Ensures a channel instance can register with one or more watching components.
  This is achieved by keeping the control of initializing all the pub-sub
  components with the channel instance.

## Impact

<!-- Choose the level of impact this proposal will have: -->

<!-- Minor (Does not impact any existing features) -->
<!-- Major (Breaks one or more existing features) -->
<!-- New Feature (Introduces a functionality) -->
Architecture (Requires a modification of the architecture)

## Implementation

<!-- Provide a description of the implementation aspects. -->

The implementation hints are described for adding implementing the watcher
component to work with the current implementation: ``go-perun``. However, over
the remote interface, the watcher should work with any new perun client
implementations that will be added in the future.

1. The `Register` and `Subscribe` methods are already part of `Adjudicator`
   interface. the adjudicator implementations can be used as
   `RegisterSubscriber`.

2. The `State` method on a channel returns the off-chain state without the
   signatures. So, add a new method to retrieve the latest off-chain state along
   with signatures.

3. To implement a watching component that works locally,

    1. The watcher component can be initialized to run as a go-routine.
    2. The pub-sub interfaces can be implemented using `go-channels`.

4. To implement a watching component that runs as a remote service,

    1. The watching component can be initialized to run as an independent
       program.
    2. The main program can connect with the watching component via remote
       interface.
    3. The pub-sub can be implemented using protocols such as `grpc`, `MQTT`,
       `webhooks` or web sockets`.
