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
1. As a part of go-perun client itself (ideal for capable hardware).
2. As an independent component on the same or different computer (ideal for
   constrained hardware).

## Details

<!-- Provide a detailed description of the proposal. -->

### Functionalities of the watcher

1. User should be able to initialize the watcher without associating it to any
   particular channel. After initialization, it must have access to a wallet
   containing keys for an on-chain account with sufficient funds (to pay
   transaction fees when registering off-chain states).

2. Main component should be able to register/deregister channel IDs to
   start/stop watching for on-chain events.

3. After a channel is registered,

   1. Main component should be able to periodically send the newer off-chain
      states.

   2. Main component should be notified by the watcher, when a state is
      registered or progressed on the blockchain.

4. If the state registered on the blockchain and is not the latest off-chain
   state known to the watcher, then it must register the latest state on the
   blockchain.

### Interfaces of the watcher

In order to realize the desired functionalities, watcher should provide the
following interfaces:


```go
type Watcher struct{}

// For initalizing the watcher.
func NewWatcher(rs RegisterSubscriber) *Watcher {...}

// For interacting with the blockchain.
type RegisterSubscriber interface {
    Subscriber
    Registerer
}

// For registering off-chain state for a given ledger channel,
// along with the off-chain states of its sub-channels and virtual channels.
type Registerer interface {
    Register(State, SubStates)
}

// For receiving registered and progressed events for a given channel ID.
type Subscriber interface {
    Subscribe(ChannelID)
}

// For registering/de-registering a channel ID with/from the watcher.
func (w *Watcher) Register(ChannelID, OffChainStatesSub, OnChainEventsPub) {...}
func (w *Watcher) Deregister(ChannelID) {...}

// For receiving off-chain states from the main component.
type OffChainStatesSub interface {
	Next() OffChainState
}

// For sending on-chain events to the main component.
type OnChainEventsPub inteface {
    // OnChainEvent can be a registered event or a progressed event.
    Publish(OnChainEvent)
}

// Interfaces for the main component to interact with the watcher.

// For receiving on-chain events from the watcher.
type OnChainEventsSub interface {
	Next() OnChainEvent
}

// For sending off-chain states to the watcher.
type OffChainStatesPub inteface {
	Publish(OffChainState)
}

// For shutting down the watcher
func (w *Watcher) Shutdown()
```

The pub-sub instances for off-chain states and on-chain events are initialized
by the main component and passed on to the watcher. Because,

1. Initial implementation can be a one channel to one watcher pub-sub.

2. Later, if one channel wants to use multiple watcher instances (because one
 watcher could not provide expected level of availability), then the one
 channel to multiple watcher pub-sub can also be implemented.

On shutdown call, the watcher must,
1. Stop watching for all the channel IDs registered with it.
2. Complete any on-going refutation process.
3. Close the `OffChainSub` and `OnChainEventsPub` on each of the channels
  registered with it. This ensures that the channels are notified of the
  watcher shutdown.
4. Finally, it should stop.

### Interaction between the watcher and the main component

Interaction between the watcher and main component based using the above
interfaces is shown in the diagram below.

![Interaction between the main component and watcher](004/watcher.svg)

Figure 1: Interaction between the main component and watcher.

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

The implementation hints are described assuming the language to be `go`.
However, these could also be extended to other languages as well.

1. The `Register` and `Subscribe` methods are already part of `Adjudicator`
   interface. Hence, the adjudicator implementations can be used as
   `RegisterSubsriber`.

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
