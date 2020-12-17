# Proposal: Descriptive specification of the User API for Two Party Payment Channels

* Author(s): Manoranjith
* Status: accept
* Related issue:
  [perun-node#45](https://github.com/hyperledger-labs/perun-node/issues/45),
  [perun-node#108](https://github.com/hyperledger-labs/perun-node/issues/108),
  [perun-node#100](https://github.com/hyperledger-labs/perun-node/issues/100),
  [perun-node#107](https://github.com/hyperledger-labs/perun-node/issues/107).


<!-- Use the above format for issues on GitHub and full links for issues on
other platforms. -->

## Summary

<!-- Provide a tl;dr summary -->

This specification describes the different APIs that should be provided by the
node for the user to open, use and close two party payment channels.

## Motivation

To have a common specification of the API interface, which can be used to
derive protocol specific formal specifications like Protocol Buffers (for gRPC)
or JSON Schema for JSON RPC.

## Details

The node shall provide three sets of APIs:

1. Node APIs - For accessing the node related functionality. Following APIs
   should be provided:

    1. [Get config](#1-get-config)
    2. [Open Session](#2-open-session)
    3. [Time](#3-time)
    4. [Help](#4-help)

2. Session APIs - For accessing the session related functionality. Each request
   should include a Session ID. Following APIs should be provided:

    1. [Add PeerID](#1-add-peerID)
    2. [Get PeerID](#2-get-peerID)
    3. [Open Payment Channel](#3-open-payment-channel)
    4. [Get Payment Channels Info](#4-get-payment-channels-info)
    5. [Subscribe To Payment Channel Proposals](#5-subscribe-to-payment-channel-proposals)
    6. [Unsubscribe From Payment Channel Proposals](#6-unsubscribe-from-payment-channel-proposals)
    7. [Respond To Payment Channel Proposal](#7-respond-to-payment-channel-proposal)
    8. [Close Session](#8-close-session)

3. Channel APIs - For accessing the channel related functionality. Each request
   should include a Session ID & Channel ID. Following APIs should be provided:

    1. [Send Payment Channel Update](#1-send-payment-channel-update)
    2. [Subscribe To Payment Channel Updates](#2-subscribe-to-payment-channel-updates)
    3. [Unsubscribe From Payment Channel Updates](#3-unsubscribe-from-payment-channel-updates)
    4. [Respond To Payment Channel Update](#4-respond-to-payment-channel-update)
    5. [Get Payment Channel Info](#5-get-payment-channel-info)
    6. [Close Payment Channel](#6-close-payment-channel)

### Data formats

The following 3 data formats will be used in the APIs.

#### 1. Peer

* `Alias`: [String] Alias for the peer. This will be used to reference a
  peer in all API calls.
* `Off-Chain Address`: [String] Permanent address (cryptographic) of the
  peer in the off-chain network.
* `Comm Address`: [String] Address (network) of the peer for off-chain
  communication.
* `Comm Type`: [String] Type of communication protocol supported by the
  peer for off-chain communication.

#### 2. Balance Info

* `Currency`: [String] Currency used for specifying the balance.
* `Aliases`: [List of String] Alias of each channel participants, as in the
  user's ID provider. `self` is a special alias for the user of the
  node. Each entry in the list should be unique.
* `Balance`: [List of String] Amount held by each channel participant, in
  the same order as that in the `Aliases`. Each entry will be a nonnegative
  value and the length of balance list should be same as that of the Aliases
  list.

* When using the `Balance Info` as a parameter in API calls, an error will
  be returned if:
    * the peer corresponding to any of the Aliases cannot be found in user's
      ID provider.
    * alias list does not have an entry `self` corresponding to the user.
    * alias list has duplicate entries.
    * any of the amounts in the balance is negative or,
    * lengths of Aliases and Balance list are different.

Note: Two lists (one each for Aliases and Balance) are used instead of a map.
Because in the case of map, when more than one currencies are used in the same
channel (in near future), the keys will be duplicated leading to larger data
size after serialization. While in case of list, the `Balance` list and
`Currency` can be combined to a map of Balance for each currency.

#### 3. Payment Channel Info

* `Channel ID`: [String] Unique ID of the channel.
* `BalanceInfo`: [[`Balance Info`](#2-balance-info)]
* `Version`: [String] Current version number of the channel. This will be
  zero when a channel is opened and will be incremented during each update.
  When registering the state on-chain, if different participants submit
  states with different versions, channel will be settled according to the
  state with the highest version number.

The following errors have specific meaning. Any other error should be returned
as `Internal Error` with additional details in the error information field.

1. `Unknown Session ID`: No session corresponding to the specified ID.
2. `Unknown Proposal ID`: No proposal corresponding to the specified ID.
3. `Unknown Channel ID`: No payment channel corresponding to the specified ID.
4. `Unknown Alias`: Peer corresponding to the specified ID not found in
   ID provider.
5. `Unknown Version`: No pending payment request with the specified version of
   state.
6. `Invalid Amount`: Invalid amount string.
7. `Invalid Balance`: Unknown currency or invalid amount string.
8. `Invalid Configuration`: Invalid configuration detected.
9. `Invalid Off-Chain Address`: Invalid off-chain address string.
10. `No Active Subscription`: No active subscription was found.
11. `Subscription Already Exists`: A subscription for this context already
    exists.
12. `Peer Alias Not Available`: Alias already used by another peer in the
    ID provider.
13. `Peer ID Exists`: Peer ID already available in the ID provider.
14. `Peer Not Responding`: Peer did not respond within expected timeout.
15. `Peer Rejected`: The request was rejected by the peer. Reason for rejection
    should be included in the error information.
16. `Response Timeout Expired`: Response to the notification was sent after the
    timeout has expired.
18. `Unclosed Payment Channels Exist`: Session cannot be closed (without force
    option) as there are unclosed channels.

### Node

#### 1. Get Config

Returns the configuration parameters of the node.

*Parameters* none

*Return*

* `Chain Address`: [String] Address of the default blockchain node used by the
  perun node.
* `Adjudicator Address`: [String] Address of the default Adjudicator contract
  used by the perun node.
* `Asset Address`: [String] Address of the default Asset Holder contract used
  by the perun node.
* `Comm Types`: [List of String] Communication protocols supported by the
  perun-node for off-chain communication.
* `ID provider Types`: [List of String] ID Provider backends supported by the
  perun-node.

#### 2. Open Session

Open a new session for the given user with the specified configuration file.
If the database has unclosed channels from the previous instance of the session,
these channels will be restored and their last known info will be returned.

*Parameters*

* `Config File`: [String] Path to the config file for the session. This should
   be present on a file system path accessible by the node.

*Return*

* `Session ID`: [String] Unique ID of the session.
* `Restored Channels Info`: [List of [Payment Channel Info](3-payment-channel-info)]

*Errors*

* `Invalid Configuration`

#### 3. Time

Returns the time as per perun node's clock in unix format. This time should be
used to check the expiry of a notification.

*Parameters* none

*Return*

* `Time`: [int64] Time in unix format.

#### 4. Help

Returns the help message.

*Parameters* none

*Return*

* `APIs`: [List of String] List of available APIs.

### Session

#### 1. Add Peer ID

Add a peer ID to the ID provider in the specified session.

*Parameters*

* `Session ID`: [String] Unique ID of the session.
* `Peer ID`: [[`Peer ID`](#1-peer ID)] Peer ID to be added to the ID provider.

*Return*

* `Success`: [Boolean]

*Errors*

* `Unknown Session ID`
* `Peer Exists`
* `Peer Alias Not Available`
* `Invalid Off-Chain Address`

#### 2. Get Peer ID

Get the peer ID corresponding to the given alias from the ID provider in
the specified session.

*Parameters*

* `Session ID`: [String] Unique ID of the session.
* `Alias`: [String] Alias of the peer whose details should be retrieved.

*Return*

* `Peer ID`: [[`Peer ID`](#1-peer ID)] Peer ID retrieved from ID provider.

*Errors*

* `Unknown Session ID`
* `Unknown Alias`

#### 3. Open Payment Channel

Open a payment channel with the participants and their balances as specified in
the `Opening Balance`. `Challenge duration` is the time available for the node
to refute in case of disputes when a state is registered on the blockchain.

*Parameters*

* `Session ID`: [String] Unique ID of the session.
* `Opening Balance`: [[`Balance Info`](#2-balance-info)]
* `Challenge Duration in Seconds`: [uint64]

*Return*

* `Opened Payment Channel Info`: [Payment Channel Info](3-payment-channel-info)

*Errors*

* `Unknown Session ID`
* `Unknown Alias`
* `Invalid Balance`
* `Peer Not Responding`
* `Peer Rejected`

#### 4. Get Payment Channels Info

Get the list of all payment channels that are open for off-chain transaction in
the specified session.

*Parameters*

* `Session ID`: [String] Unique ID of the session.

*Return*

* `Open Payment Channels Info`: [List of [Payment Channel Info](3-payment-channel-info)]

*Errors*

* `Unknown Session ID`

#### 5. Subscribe To Payment Channel Proposals

Subscribe to notifications on new incoming payment channel proposals in the
specified session.

Only one subscription can be made at a time. Making a new subscription without
canceling the previous one will return an error.

The incoming channel proposal received when there was no subscription will
have been cached by the node. Once a new subscription is made, node will send
these cached requests (if any), as individual notifications. It will then
continue to send a notification for each new incoming channel proposal.

Response to the notifications can be sent using the
[`Respond To Payment Channel Proposal`](#7-respond-to-payment-channel-proposal)
API before the notification expires.

If the proposal was received from a `Peer ID` that is not found in the ID
provider of the session, the proposal will be automatically rejected by the
node. User will still receive a notification of this proposal with the `Alias`
of the peer set to the hex representation of its off-chain address in the
`Opening Balance`. These notifications should not be responded to. If the user
still responds to it, an `Unknown Proposal ID` error will be returned.

*Parameters*

* `Session ID`: [String] Unique ID of the session.

*Return*

* `Success`: [bool]

*Errors*

* `Unknown Session ID`
* `Subscription Already Exists`

*Notification*

Each notification sent to the user should contain the following data:

* `Proposal ID`: [String] Unique ID of this channel proposal.
* `Opening Balance`: [[`Balance Info`](#2-balance-info)]
* `Challenge Duration in Seconds`: [uint64]
* `Expiry`: [int64] Time (in unix format) before which response should be sent.

#### 6. Unsubscribe From Payment Channel Proposals

Unsubscribe from notifications on new incoming payment channel proposals in the
specified session.

*Parameters*

* `Session ID`: [String] Unique ID of the session.

*Return*

* `Success`: [bool]

*Errors*

* `Unknown Session ID`
* `No Active Subscription`

#### 7. Respond To Payment Channel Proposal

Respond to an incoming payment channel proposal for which a notification was
received. Response should be sent before the notification expires. Use the
`Time` API to fetch current time of the perun node to check notification
expiry.

*Parameters*

* `Session ID`: [String] Unique ID of the session.
* `Proposal ID`: [String] Unique ID of this proposal as received in the
  notification.
* `Accept`: [Bool] If True, the proposal will be accepted, else rejected.

*Return*

* `Opened Payment Channel Info`: [[`Payment Channel Info`](#3-payment-channel-info)]

*Errors*

* `Unknown Session ID`
* `Unknown Channel ID`
* `Unknown Proposal ID`
* `Peer Not Responding`
* `Response Timeout Expired`

#### 8. Close Session

Close the specified session. All session data will be persisted to disk.

`Force` parameter determines what happens when there are open channels in the
session.
  * If `False` the API returns an error when there are open channels. This
    should be used by default.
  * If `True`, the session is forcibly closed and the API returns list of open
    channels that were persisted. When a session is opened with the same data
    files, these channels can be restored in open state.
    However, use this with caution, as closing a session with open channels
    creates a possibility for channel participants in any of the those open
    open channels to register an older, invalid state on the blockchain and
    finalize it.

*Parameters*

* `Session ID`: [String] Unique ID of the session.
* `Force`: [bool] Forcibly close the session.

*Return*

* `Open Channels Info`: [List of [Payment Channel Info](#3-payment-channel-info)]
   Open channels in the session, that were persisted to the disk. Relevant only
   when `Force` is `true`. This is empty when `Force` is `false` and the call is
   successful.

*Errors*

* `Unknown Session ID`
* `Unclosed Payment Channels Exist`

### Payment Channel

#### 1. Send Payment Channel Update

Send a payment channel update to the specified peer on the channel. Use
`self` in the `payee` field to request payments.

*Parameters*

* `Session ID`: [String] Unique ID of the session.
* `Channel ID`: [String] Unique ID of the channel.
* `Payee`: [String] Alias of the peer to which amount should be sent.
* `Amount`: [String] Amount to send.
* `IsFinal`: [bool] If 'True', this update will be marked as final and the
  channel will be closed after this update.

*Return*

* `Updated Channel Info`: [[`Payment Channel Info`](#3-payment-channel-info)]

*Errors*

* `Unknown Session ID`
* `Unknown Channel ID`
* `Unknown Alias`
* `Invalid Amount`
* `Peer Not Responding`

#### 2. Subscribe To Payment Channel Updates

Subscribe to notifications on new incoming payment channel updates for the
specified channel in the specified session.

Only one subscription can be made at a time. Making a new subscription without
canceling the previous one will return an error.

The incoming payment channel update received when there was no subscription
will have been cached by the node. Once a new subscription is made, node will
send these cached request (if any), as individual notifications. It will then
continue to send a notification for each new incoming payment channel update.

Response to the notifications can be sent using the
[`Respond To Payment Channel Update`](#4-respond-to-payment-channel-update) API
before the notification expires.

*Parameters*

* `Session ID`: [String] Unique ID of the session.
* `Channel ID`: [String] Unique ID of the channel.

*Return*

* `Success`: [bool]

*Errors*

* `Unknown Session ID`
* `Unknown Channel ID`
* `Subscription Already Exists`


*Notification*

Each notification sent to the user should contain the following data:

* `UpdateID`: [String] Unique ID that represents this channel update.
* `Proposed Payment Channel Info`: [[`Payment Channel Info`](#3-payment-channel-info)]
* `Status`: [String] Can be one of the three values: `Open`, `Final`, `Closed`.
    * `Open`: This is a normal channel update, if accepted off-chain state of
      the channel will be progressed to the proposed state.
    * `Final`: Indicates if this is a final update. Channel will be closed if a
      final update is accepted.
    * `Closed`: Indicates that the channel has been closed. This will be final
      update notification for the channel and the subscription should be
      cancelled once such an update is received. No response is expected in
      this case. If the user still responds, `Unknown Update ID` error will be
      returned.
* `Expiry`: [int64] Time (in unix format) before which response should be
  sent.

#### 3. Unsubscribe From Payment Channel Updates

Unsubscribe from notifications on new incoming payment channel updates for the
specified channel in the specified session.

*Parameters*

* `Session ID`: [String] Unique ID of the session.
* `Channel ID`: [String] Unique ID of the channel.

*Return*

* `Success`: [bool]

*Errors*

* `Unknown Session ID`
* `Unknown Channel ID`
* `No Active Subscription`

#### 4. Respond To Payment Channel Update

Respond to an incoming payment channel update for which a notification was
received. Response should be sent before the notification expires. Use the
`Time` API to fetch current time of the perun node to check notification
expiry.

*Parameters*

* `Session ID`: [String]
* `Channel ID`: [String]
* `Update ID`: [String] Update ID received in the notification.
* `Accept`: [Bool] If True, the update will be accepted, else rejected.

*Return*

* `Updated Channel Info`: [[`Payment Channel Info`](#3-payment-channel-info)]

*Errors*

* `Unknown Session ID`
* `Unknown Channel ID`
* `Unknown Version`
* `Peer Not Responding`
* `Response Timeout Expired`

#### 5. Get Payment Channel Info

Get the current info for the specified payment channel.

*Parameters*

* `Session ID`: [String]
* `Channel ID`: [String]

*Return*

* `Payment Channel Info`: [[`Payment Channel Info`](#3-payment-channel-info)]

*Errors*

* `Unknown Session ID`
* `Unknown Channel ID`

#### 6. Close Payment Channel

Finalize the current balance of the payment channel on the blockchain and
withdraw the amount corresponding to this user from the channel.

*Parameters*

* `Session ID`: [String]
* `Channel ID`: [String]

*Return*

* `Closed Channel Info`: [[`Payment Channel Info`](#3-payment-channel-info)]

*Errors*

* `Unknown Session ID`
* `Unknown Channel ID`

## Rationale

<!-- Provide a discussion of alternative approaches and trade offs; advantages
and disadvantages of the specified approach.  -->

1. An alternate approach to defining a dedicated API for payment channel would
   be to define an API for generalized state channels that can also be used for
   payment channel. But having a dedicated API improves the usability of the
   API for payment use cases. While it still allows another end point in the
   node to provide an API for generalized state channels.

## Impact

<!-- Choose the level of impact this proposal will have: -->

<!-- Minor (Does not impact any existing features) --> <!-- Major (Breaks one
or more existing features) --> <!-- New Feature (Introduces a functionality)
--> Architecture (Requires a modification of the architecture)

## Implementation

<!-- Provide a description of the implementation aspects. -->

Define protocol specific format definitions such as Protocol Buffers for gRPC
and implement a protocol agnostic abstract API layer according to this
specification.
