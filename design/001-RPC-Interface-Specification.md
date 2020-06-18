<!-- This is a template for proposing design changes to the dst-go project. -->

# Proposal: Descriptive specification of the User API for Two Party Payment Channels

* Author(s): Manoranjith
* Status: propose
* Related issue:
  [perun-node#45](https://github.com/hyperledger-labs/perun-node/issues/45)

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

    1. Get config
    2. Open Session
    3. Time
    4. Help

2. Session APIs - For accessing the session related functionality. Each request
   should include a Session ID. Following APIs should be provided:

    1. Add Contact
    2. Get Contact
    3. Open Payment Channel
    4. Get Payment Channels
    5. Subscribe To Payment Channel Proposals
    6. Unsubscribe To Payment Channel Proposals
    7. Respond To Payment Channel Proposal
    7. Subscribe To Payment Channel Closes
    8. Unsubscribe To Payment Channel Closes
    9. Close Session

3. Channel APIS - For accessing the channel related functionality. Each request
   should include a Session ID and Channel ID.

    1. Send Payment Channel Update
    2. Subscribe To Payment Channel Updates
    3. Unsubscribe To Payment Channel Updates
    4. Respond To Payment Channel Update
    5. Get Balance
    6. Close Payment Channel

### Data formats

The following 3 data formats will be used in the APIs.

1. `Peer`

    * `Alias`: [String] Alias for the peer. This will be used to reference a
      peer in all API calls.
    * `Off-Chain Address`: [String] Permanent address (cryptographic) of the
      peer in the off-chain network.
    * `Comm Address`: [String] Address (network) of the peer for off-chain
      communication.
    * `Comm Type`: [String] Type of communication protocol supported by the
      peer for off-chain communication.

2. `Balance Info`

    * `Currency`: [String] Currency used for specifying the amount.
    * `Balance`: [Map of `Alias` to `Balance`]
      * `Alias`: [String] `Self` is a special alias for the user of the node.
      * `Balances`: [String] Amount held by the `Peer` corresponding to the Alias.

3. `Payment Channel`

    * `Channel ID`: [String] Unique ID of the channel.
    * `BalanceInfo`: [Balance Info]
    * `Version`: [String] Current Version of the channel.

The following errors have specific meaning. Any other error should be returned
as `Internal Error` with additional details in the error information field.

1. `Unknown Session ID`: No session corresponding to the specified ID.
2. `Unknown Proposal ID`: No proposal corresponding to the specified ID.
3. `Unknown Channel ID`: No payment channel corresponding to the specified ID.
4. `Unknown Alias`: Peer corresponding to the specified ID not found in
   contacts provider.
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
    contacts.
13. `Peer Exists`: Peer already available in the contacts provider.
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
* `Comm Types`: [List of String] Communication protocols supported by the node
  for off-chain communication.
* `Contact Types`: [List of String] Contacts Provider backends supported by the
  node.

#### 2. Open Session

Open a new session for the given user with the specified configuration file.

*Parameters*

* `Config File`: [String] Path to the config file for the session. This should
   be present on a file system path accessible by the node.

*Return*

* `Session ID`: [String] Unique ID of the session.

*Errors*

* `Invalid Configuration`

#### 3. Time

Returns the time as per perun node's clock in unix format. This time should be used
to check if the timeout in a notification has expired or not.

*Parameters* none

*Return*

* `Time`: [int64] Time in unix format.

#### 3. Help

Returns the help message.

*Parameters* none

*Return*

* `APIs`: [List of String] List of available APIs.

### Session

#### 1. Add Contact

Add a peer to the contacts in the specified session.

*Parameters*

* `Session ID`: [String] Unique ID of the session.
* `Peer`: [Peer] Peer to be added to the contacts.

*Return*

* `Success`: [Boolean]

*Errors*

* `Unknown Session ID`
* `Peer Exists`
* `Peer Alias Not Available`
* `Invalid Off-Chain Address`

#### 2. Get Contact

Get the peer corresponding to the given alias from the contacts provider in
the specified session.

*Parameters*

* `Session ID`: [String] Unique ID of the session.
* `Alias`: [String] Alias of the peer whose details should be retrieved.

*Return*

* `Peer`: [Peer] Peer retrieved from contacts.

*Errors*

* `Unknown Session ID`
* `Unknown Alias`

#### 3. Open Payment Channel

Open a payment channel with the peer (corresponding to the `Alias`) with the
specified opening balance and challenge duration.

*Parameters*

* `Session ID`: [String] Unique ID of the session.
* `PeerAlias`: [String] Alias of peer with whom channel should be opened.
* `Opening Balance`: [Balance Info]
* `Challenge Duration in Seconds`: [uint64] Challenge Duration for the channel in seconds.

*Return*

* `Channel`: [Payment Channel]

*Errors*

* `Unknown Session ID`
* `Unknown Alias`
* `Invalid Balance`
* `Peer Not Responding`
* `Peer Rejected`

#### 4. Get Payment Channels

Get the list of all payment channels that are open for off-chain transactions
in the specified session.

*Parameters*

* `Session ID`: [String] Unique ID of the session.

*Return*

* `Open channels`: [List of Payment Channel]

*Errors*

* `Unknown Session ID`

#### 5. Subscribe To Payment Channel Proposals

Subscribe to notifications on new incoming payment channel proposals in the
specified session.

Only one subscription can be made at a time. Making a repeated subscription
without canceling the previous one will return an error.

The incoming channel proposal received when there was no subscription will
have been cached by the node. Once a new subscription is made, node will send
these cached requests (if any), as individual notifications. It will then
continue to send a notification for each new incoming channel proposal.

Response to the notifications can be sent using the `Respond To Channel
Proposal` API before the `timeout` expires.

If the proposal was received from a `Peer` that is not found in the contacts
provider of the session, the proposal will be automatically rejected by the
node. User will still receive a notification of this proposal with `Proposing
Peer` field set to empty string and these notifications should not be
responded to. If the user still responds to it, an `Unknown Proposal ID`
error will be returned.

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
* `Opening Balance`: [Balance Info]
* `Challenge Duration in Seconds`: [uint64] Challenge Duration for the channel in seconds.
* `Timeout`: [int64] Time (in unix format) before which response should be sent.

#### 6. Unsubscribe To Payment Channel Proposals

Unsubscribe to notifications on new incoming payment channel proposals in the
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
received. Response should be sent before the timeout in the notification
expires. Use the `Time` API to fetch current time of the perun node.

*Parameters*

* `Session ID`: [String] Unique ID of the session.
* `Proposal ID`: [String] Unique ID of this proposal as received in the notification.
* `Accept`: [Bool] If True, the proposal will be accepted, else rejected.

*Return*

* `Success`: [Boolean]
* `Channel`: [Payment Channel]

*Errors*

* `Unknown Session ID`
* `Unknown Channel ID`
* `Unknown Proposal ID`
* `Peer Not Responding`
* `Response Timeout Expired`

#### 7. Subscribe To Payment Channel Closes

Subscribe to notifications when channels in the specified session are closed by
the peer. User need not respond to these notifications.

Only one subscription can be made at a time. Making a repeated subscription
request without canceling the previous one will return an error.

The channel close events occurred when there was no subscription will have been
cached by the node. Once a new subscription is made, node will send these
cached events (if any), as individual notifications. It will then continue to
send a notification for each channel closed by a peer.

*Parameters*

* `Session ID`: [String] Unique ID of the session.

*Return*

* `Success`: [bool]

*Errors*

* `Unknown Session ID`
* `Subscription Already Exists`

*Notification*

Each notification sent to the user should contain the following data:

* `Closing State`: [Payment Channel]
* `Error`: [String] Error (if any) in closing the channel.

#### 8. Unsubscribe To Payment Channel Closes

Unsubscribe to notifications when channels in the specified session are closed
by the peer.

*Parameters*

* `Session ID`: [String] Unique ID of the session.

*Return*

* `Success`: [bool]

*Errors*

* `Unknown Session ID`
* `No Active Subscription`


#### 9. Close Session

Close the specified session. All session data are persisted to disk.

`Force` parameter determines what happens when there are unclosed channels in
the session (in open, dispute or closing in progress state).
  * If it is set to `False` the API returns an error when there are such
    channels. This should be used by default.
  * If `True`, the session is forcibly closed and the API returns list of
   unclosed channels. Use this with caution.

*Parameters*

* `Session ID`: [String] Unique ID of the session.
* `Force`: [bool] Forcibly close the session.

*Return*

* `Unclosed Channels`: [List of Payment Channel] This is relevant only when
  using `Force` = True.

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

*Return*

* `Success`: [bool]

*Errors*

* `Unknown Session ID`
* `Unknown Channel ID`
* `Unknown Alias`
* `Invalid Amount`
* `Peer Not Responding`

#### 2. Subscribe To Payment Channel Updates

Subscribe to notifications on new incoming payment channel updates for the
specified channel in the specified session.

Only one subscription can be made at a time. Making a repeated subscription
without canceling the previous one will return an error.

The incoming payment channel update received when there was no subscription
will have been cached by the node. Once a new subscription is made, node will
send these cached request (if any), as individual notifications. It will then
continue to send a notification for each new incoming payment channel update.

Response to the notifications can be sent using the `Respond To Payment
Channel Update` API.

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
* `Proposed Balance`: [Balance Info] Proposed balance for the update.
* `Version`: [String] Version of the channel state for the proposed payment.
* `Final`: [bool] Indicates if this is a final update. Channel will be closed
  once a final update is accepted.
* `Timeout`: [int64] Time (in unix format) before which response should be
  sent.

#### 3. Unsubscribe To Payment Channel Updates

Unsubscribe to notifications on new incoming payment channel updates for the
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

Respond to an incoming payment channel update. Response should be sent before
the timeout in the notification expires. Use the `Time` API to fetch current
time of the perun node.

*Parameters*

* `Session ID`: [String]
* `Channel ID`: [String]
* `Update ID`: [String] Update ID received in the notification.
* `Accept`: [Bool] If True, the update will be accepted, else rejected.

*Return*

* `Success`: [Boolean]

*Errors*

* `Unknown Session ID`
* `Unknown Channel ID`
* `Unknown Version`
* `Peer Not Responding`
* `Response Timeout Expired`

#### 5. Get Balance

Get balance info for the specified payment channel.

*Parameters*

* `Session ID`: [String]
* `Channel ID`: [String]

*Return*

* `Current Balance`: [Balance Info]
* `Current Version`: [String]

*Errors*

* `Unknown Session ID`
* `Unknown Channel ID`

#### 6. Close Payment Channel

Close the specified payment channel.

*Parameters*

* `Session ID`: [String]
* `Channel ID`: [String]

*Return*

* `Closing Balance`: [Balance Info]
* `Version Balance`: [Balance Info]

*Errors*

* `Unknown Session ID`
* `Unknown Channel ID`

## Rationale

<!-- Provide a discussion of alternative approaches and trade offs; advantages
and disadvantages of the specified approach.  -->

1. An alternate approach to defining a dedicated API for payment channel would
   be to define an API for generalized state channels that can also be used for
   payment channel. But having a dedicated API improves the usability of the API
   for payment use cases. While it still allows another end point in the node to
   provide an API for generalized state channels.

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
