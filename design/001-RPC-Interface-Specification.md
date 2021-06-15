# Proposal: Descriptive specification of the User API for Two Party Payment Channels

* Author(s): Manoranjith
* Status: accept
* Related issue:
  [perun-node#45](https://github.com/hyperledger-labs/perun-node/issues/45),
  [perun-node#108](https://github.com/hyperledger-labs/perun-node/issues/108),
  [perun-node#100](https://github.com/hyperledger-labs/perun-node/issues/100),
  [perun-node#107](https://github.com/hyperledger-labs/perun-node/issues/107).
  [perun-node#132](https://github.com/hyperledger-labs/perun-node/issues/132).


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
    4. [Register Currency](#4-register-currency)
    5. [Help](#5-help)

2. Session APIs - For accessing the session related functionality. Each request
   should include a Session ID. Following APIs should be provided:

    1. [Add Peer ID](#1-add-peer-id)
    2. [Get Peer ID](#2-get-peer-id)
    3. [Open Payment Channel](#3-open-payment-channel)
    4. [Get Payment Channels Info](#4-get-payment-channels-info)
    5. [Subscribe To Payment Channel Proposals](#5-subscribe-to-payment-channel-proposals)
    6. [Unsubscribe From Payment Channel Proposals](#6-unsubscribe-from-payment-channel-proposals)
    7. [Respond To Payment Channel Proposal](#7-respond-to-payment-channel-proposal)
    8. [Close Session](#8-close-session)
    9. [Deploy ERC20 Asset Holder](#9-deploy-erc20-asset-holder)

3. Channel APIs - For accessing the channel related functionality. Each request
   should include a Session ID & Channel ID. Following APIs should be provided:

    1. [Send Payment Channel Update](#1-send-payment-channel-update)
    2. [Subscribe To Payment Channel Updates](#2-subscribe-to-payment-channel-updates)
    3. [Unsubscribe From Payment Channel Updates](#3-unsubscribe-from-payment-channel-updates)
    4. [Respond To Payment Channel Update](#4-respond-to-payment-channel-update)
    5. [Get Payment Channel Info](#5-get-payment-channel-info)
    6. [Close Payment Channel](#6-close-payment-channel)

### Data formats

In addition to basic data types string, int64, uint64 and bool, the following 5
data formats will be used in the APIs.

Also, when data type is mentioned as [Duration string], it implies duration in
this format `2h3m4s`.

`String` type is used for integers instead of numbers, because these number of
digits vary widely. For examples, amount in `ETH` currency should have 0 to a
maximum of 9 digits after the decimal place. Using numbers might lead to errors
arising due to approximation (during calculations) or during serialization /
de-serialization.

#### 1. Peer ID

* `Alias`: [String] Alias for the peer. This will be used to reference a
   peer in all API calls.
* `Off-Chain Address`: [String] Permanent address (cryptographic) of the
  peer in the off-chain network.
* `Comm Address`: [String] Address (network) of the peer for off-chain
  communication.
* `Comm Type`: [String] Type of communication protocol supported by the
  peer for off-chain communication.

#### 2. Balance Info

* `Currencies`: [List of String] Currencies used for specifying the balance.
* `Aliases`: [List of String] Alias of each channel participants, as in the
  user's ID provider. `self` is a special alias for the user of the
  node. Each entry in the list should be unique.
* `Balances`: [List of List of String] Amount held by each channel participant.
  Each `List of string` correspond to balance in a specific currency, in the
  same order as in the `Currencies`. Within each list, the entries correspond to
  the amount of channel participant for that currency, in the same order as in
  the `Aliases`. Amount should be a non-negative value.

* When using the `Balance Info` as a parameter in API calls, an error will
  be returned if:
    * the peer corresponding to any of the Aliases cannot be found in user's
      ID provider.
    * Aliases does not have an entry `self` (it represents the user).
    * Aliases has duplicate entries.
    * Any of the amount in the balance is negative or,
    * Number of lists of string in `Balances` is not same as length of `Aliases`
      are different.
    * Number of entries in each list of string in `Balances` is not same as the
      length of `Currencies`.

Note: Lists are used instead of a map. Because in the case of map, when more
than one currencies are used in the same channel, the keys will be duplicated
leading to larger data size after serialization. While in case of list, the
`Balance` list and `Currency` can be combined to a map of Balance for each
currency.

#### 3. Payment Channel Info

* `Channel ID`: [String] Unique ID of the channel.
* `BalanceInfo`: [[`Balance Info`](#2-balance-info)]
* `Version`: [String] Current version number of the channel. This will be
  zero when a channel is opened and will be incremented during each update.
  When registering the state on-chain, if different participants submit
  states with different versions, channel will be settled according to the
  state with the highest version number.

#### 4. Contract Info

* `Name`: [String] Name of the contract. Will be one of the following values: `Adjudicator`, `Asset`.
* `Address`: [String] Address at which the contract is deployed.

#### 5. Payment

* `Currency`: [String] Currency for this payment.
* `Payee`   : [String] Person being paid. If it is the user itself, use `self`.
* `Amount`  : [String] Amount. The maximum number digits this number can have
              after the decimal point depends on the currency.

### Errors

The errors returned by the API contain the following information:

  1. Category: Indicates the cause of error and how it should be handled.
  2. Code: Identifies specific errors.
  3. Message: Describes the error.
  5. Additional Info: Data necessary for handling the error as key value pairs.

#### Error categories
Below table summarizes the error categories:

| Category | Cause| Handling |
|-------------|-----------------|--------------|
| Participant | Any channel participant not acting as per protocol | Negotiate with peer outside of the system and retry. |
| Client | The request from client. Could be error in arguments or the configuration provided by the client to access external systems. | Depending upon the context, retry using valid arguments or by providing correct configuration to access external systems or after fixing the external systems. |
| Protocol Fatal |  Unexpected failure in external system. Protocol was aborted, could result in loss of funds. | Requires manual inspection of the error |
| Internal | Unknown error in the node. |  Requires manual inspection of the error |

#### Error codes

The errors returned by the perun-node application will contain one of the below
error codes. The codes are three digits with

- first digit representing the category (1,2,3,4 for participant, client, protocol fatal and internal respectively)
- next two digits represent specific errors within that category.

For each error code, data necessary for handling the error is passed in the
additional info field as key value pairs. The keys are fixed for a given error
code.

##### 101

- Error           : Peer response timed out
- Category        : Participant
- Additional Info : PeerAlias [String], ResponseTimeout [Duration string]

##### 102

- Error           : Peer rejected
- Category        : Participant
- Additional Info : PeerAlias [String], Reason [String]

##### 103

- Error           : Peer not funded
- Category        : Participant
- Additional Info : PeerAlias [String], FundingTimeout [Duration string]

##### 104

- Error           : User response timedout
- Category        : Participant
- Additional Info : Expiry [Int64], ResponseReceivedAt [Int64] (unix time format)

##### 201

- Error           : Resource not found
- Category        : Client
- Additional Info : ResourceType [String], ResourceID [String]

Note: Resource type takes only specific values, will be mentioned in each API.

##### 202

- Error           : Resource already exists
- Category        : Client
- Additional Info : ResourceType [String], ResourceID [String]

Note: Resource type takes only specific values, will be mentioned in each API.

##### 203

- Error           : Invalid arguments
- Category        : Client
- Additional Info : Name [String], Value [String]

##### 204

- Error           : Failed pre-condition
- Category        : Client
- Additional Info : <additional-allowed>

Only in this case, the additional info field can contain keys that are not
defined here. These keys will be specified in the specific APIs that will use
additional keys for this data type.

##### 205

- Error           : Invalid configuration
- Category        : Client
- Additional Info : Name [String], Value [String]

##### 206

- Error           : Invalid contracts
- Category        : Client
- Additional Info : Contracts [List of [contract info](#4-contract-info)]

##### 301

- Error           : Protocol aborted as to tx timed out
- Category        : Protocol
- Additional Info : TxType [String], TxID [String], TxTimeout [Duration string]

##### 302

- Error           : Protocol aborted as blockchain node is not reachable
- Category        : Protocol
- Additional Info : BlockchainNodeURL [String]

##### 401

- Error           : Unknown internal error
- Category        : Internal
- Additional Info : Nil

Note: When errors are described for each API, if the value of a field in the
additional info is enclosed in `<>`, then the value will be filled in when
creating the error.

### API Description

Description of the API is given below for accessing the node, session and channel related functionalities.

### Node

#### 1. Get Config

Returns the configuration parameters of the node.

*Parameters* none

*Return*

* `Chain Address`: [String] Address of the blockchain node.
* `Adjudicator Address`: [String] Address of the Adjudicator contract.
* `Asset Address`: [String] Address of the Asset Holder contract.
* `Comm Types`: [List of String] Supported off-chain communication protocols.
* `ID provider Types`: [List of String] Supported ID Provider backends.

#### 2. Open Session

Initializes a new session with the configuration in the given file. If channels
were persisted during the previous instance of the session, they will be
restored and their last known info will be returned.

*Parameters*

* `Config File`: [String] Path to the config file for the session. It should
  be present on a file system path accessible by the node.

*Success response*

* `Session ID`: [String] Unique ID of the session.
* `Restored Channels Info`: [List of [Payment Channel Info](#3-payment-channel-info)]

*Error response*

If there is an error, it will be one of the following codes:
- [203](#203) Name:"configFile" when config file cannot be accessed.
- [205](#205) when any of the configuration is invalid.
- [206](#206) when the contracts at the addresses in config are invalid.
- [401](#401)

#### 3. Time

Returns the time as per perun node's clock. It should be used to check the
expiry of notifications.

*Parameters* none

*Return*

* `Time`: [Int64] Time in unix format.

#### 4. Register Currency

Register a new currency. The currency can be any ERC20 token. An asset holder
contract that was deployed specifically for this ERC20 token is required. If
such a contract does not exists, then use
[Deploy ERC20 Asset Holder](9-deploy-erc20-asset-holder) API instead, because an
on-chain account with sufficient balance is required for deploying contracts.

*Parameters*

* `Currency`: [String] Symbol of the ERC20 token, as specified in the token contract.
* `Asset holder`: [String] Address of the asset holder contract for this ERC20 token.

*Success response*

*Error response*

If there is an error, it will be one of the following codes:
- [202](#202) ResourceType: "currency"  when the currency is already registered
              with the same asset holder address.
- [203](#203) Name:"assetHolder" when the currency is already registered with a
              different asset older address.
- [206](#206) when the contracts at the given address is invalid.
- [401](#401)

#### 5. Help

Returns the list of user APIs served by the node.

*Parameters* none

*Return*

* `APIs`: [List of String] List of available APIs.

### Session

#### 1. Add Peer ID

Adds the peer ID to the ID provider instance of the session.

*Parameters*

* `Session ID`: [String] Unique ID of the session.
* `Peer ID`: [[`Peer ID`](#1-peer-id)] Peer ID to be added to the ID provider.

*Return*

* `Success`: [Boolean]

*Errors*

If there is an error, it will be one of the following codes:
- [201](#201) ResourceType: "session" when session ID is not known.
- [202](#202) ResourceType: "peerID" when peer ID is already registered.
- [203](#203) Name:"peerAlias" when peer alias is used for another peer.
- [203](#203) Name:"offChainAddress" when off-chain address is invalid.
- [401](#401)

#### 2. Get Peer ID

Gets the peer ID for the given alias from the ID provider instance of the
session.

*Parameters*

* `Session ID`: [String] Unique ID of the session.
* `Alias`: [String] Alias of the peer whose details should be retrieved.

*Return*

* `Peer ID`: [[`Peer ID`](#1-peer-id)] Peer ID retrieved from ID provider.

*Errors*

If there is an error, it will be one of the following codes:
- [201](#201) ResourceType: "session" when session ID is not known.
- [201](#201) ResourceType: "peerID" when peer alias is not known.

#### 3. Open Payment Channel

Proposes a payment channel to the participants with the specified opening
balance, funds it on the blockchain when the proposal is accepted and sets it up
for off-chain transactions when all the participants have funded the channel on
the blockchain.

`Challenge duration` is the time available for the node to refute in case of
disputes when a state is registered on the blockchain.

*Parameters*

* `Session ID`: [String] Unique ID of the session.
* `Opening Balance`: [[`Balance Info`](#2-balance-info)]
* `Challenge Duration in Seconds`: [Uint64]

*Return*

* `Opened Payment Channel Info`: [Payment Channel Info](#3-payment-channel-info)

*Errors*

If there is an error, it will be one of the following codes:
- [201](#201) ResourceType: "session" when session ID is not known.
- [201](#201) ResourceType: "peerID" when any of the peer aliases are not known.
- [201](#201) ResourceType: "currency" when the currency is not known.
- [203](#203) Name:"Amount" when any of the amounts is invalid.
- [101](#101) when peer request times out.
- [102](#102) when peer rejects the request.
- [103](#103) when peer did not fund the channel in time.
- [301](#301) TxType: "Fund" when funding tx times out.
- [302](#302) when connection to blockchain drops while funding.
- [401](#401)


#### 4. Get Payment Channels Info

Gets the list of all payment channels in the session with their latest agreed
state.

*Parameters*

* `Session ID`: [String] Unique ID of the session.

*Return*

* `Open Payment Channels Info`: [List of [Payment Channel Info](#3-payment-channel-info)]

*Errors*

If there is an error, it will be one of the following codes:
- [201](#201) ResourceType: "session" when session ID is not known.

#### 5. Subscribe To Payment Channel Proposals

Subscribes to notifications on new incoming payment channel proposals in the
session. Only one subscription can be made at a time. Making a new subscription
without canceling the previous one will return an error.

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
till responds to it, a [201 Resource Not Found](#201) error will be returned.

*Parameters*

* `Session ID`: [String] Unique ID of the session.

*Return*

* `Success`: [bool]

*Errors*

If there is an error, it will be one of the following codes:
- [201](#201) ResourceType: "session"  when session ID is not known.
- [202](#202) ResourceType: "proposalsSub" when a subscription already exists.

*Notification*

Each notification sent to the user should contain the following data:

* `Proposal ID`: [String] Unique ID of this channel proposal.
* `Opening Balance`: [[`Balance Info`](#2-balance-info)]
* `Challenge Duration in Seconds`: [Uint64]
* `Expiry`: [Int64] Time (in unix format) before which response should be sent.

#### 6. Unsubscribe From Payment Channel Proposals

Unsubscribes from notifications on new incoming payment channel proposals in the
specified session.

*Parameters*

* `Session ID`: [String] Unique ID of the session.

*Return*

* `Success`: [bool]

*Errors*

If there is an error, it will be one of the following codes:
- [201](#201) ResourceType: "session"  when session ID is not known.
- [201](#201) ResourceType: "proposalsSub" when a subscription does not exist.

#### 7. Respond To Payment Channel Proposal

Responds to the specified payment channel proposal for which a notification had
been received. Response should be sent before the notification expires. Use the
`Time` API to fetch current time of the perun node as as reference for checking
notification expiry.

*Parameters*

* `Session ID`: [String] Unique ID of the session.
* `Proposal ID`: [String] Unique ID of this proposal as received in the
  notification.
* `Accept`: [Bool] If True, the proposal will be accepted, else rejected.

*Return*

* `Opened Payment Channel Info`: [[`Payment Channel Info`](#3-payment-channel-info)]

*Errors*

If there is an error, it will be one of the following codes:
- [201](#201) ResourceType: "session"  when session ID is not known.
- [201](#201) ResourceType: "proposal"  when proposal ID is not known.
- [203](#203) when session is closed.
- [103](#103) when peer did not fund the channel in time.
- [104](#104) when user responded after time out expired.
- [301](#301) TxType: "Fund" when funding tx times out.
- [302](#302) when connection to blockchain drops while funding.
- [401](#401)

#### 8. Close Session

Closes the specified session. All session data will be persisted to disk.

`Force` parameter determines what happens when there are open channels in the
session.
  * If `False` the API returns an error when there are open channels. This
    should be used by default.
  * If `True`, the session is forcibly closed and the API returns list of open
    channels that were persisted. When a session is re-opened with the same
    config file, these channels can be restored in open state. However, use this
    with caution, as closing a session with open channels creates a possibility
    for channel participants in any of the those open open channels to register
    an older, invalid state on the blockchain and finalize it.

*Parameters*

* `Session ID`: [String] Unique ID of the session.
* `Force`: [bool] Forcibly close the session.

*Return*

* `Open Channels Info`: [List of [Payment Channel Info](#3-payment-channel-info)]
   Open channels in the session, that were persisted to the disk. Relevant only
   when `Force` is `true`. This is empty when `Force` is `false` and the call is
   successful.

*Errors*

If there is an error, it will be one of the following codes:
- [201](#201) ResourceType: "session" when session ID is not known.
- [204](#204) when session is closed with force=false and unclosed channels
              exists.  Additional Info will contain an extra field:
              OpenChannelsInfo: [List of [`Payment Channel Info`](#3-payment-channel-info)]

- [401](#401)

#### 9. Deploy ERC20 Asset Holder

Deploys an asset holder contract for an ERC20 token that is not already
registered as a currency. User's on-chain account will be used for funding this
deploy transaction. The token is then registered as new currency, with the
symbol taken from the `Symbol` API of the ERC20 token contract.

Returns error, if the ERC20 token is already registered as a currency with the
node. For this validation, symbol of the token obtained from the `Symbol` API
will be used.

*Parameters*

* `Session ID`: [String] Unique ID of the session.
* `ERC20 Token Address`: [String] Address of the ERC20 token contract.

*Return*

* `Asset holder Address`: [String] Address of the ERC20 token contract.
* `Currency`: [String] Symbol for this ERC20 token.

*Errors*

If there is an error, it will be one of the following codes:
- [201](#201) ResourceType: "session"
- [202](#202) ResourceType: "currency"
- [301](#301)
- [302](#302)
- [401](#401)


### Payment Channel

#### 1. Send Payment Channel Update

Sends a payment update on the channel that can send or request funds. Use
`self` in the `payee` field of [Payment](#5-payment) to pay the user itself and
`<alias-of-the-peer> to pay the peer.

*Parameters*

* `Session ID`: [String] Unique ID of the session.
* `Channel ID`: [String] Unique ID of the channel.
* `Payments`: [List of [Payment](#5-payment)]

*Return*

* `Updated Channel Info`: [[`Payment Channel Info`](#3-payment-channel-info)]

*Errors*

If there is an error, it will be one of the following codes:
- [201](#201) ResourceType: "session" when session ID is not known.
- [201](#201) ResourceType: "channel" when channel ID is not known.
- [201](#201) ResourceType: "peerID" when any of the peer alias are not known.
- [201](#201) ResourceType: "currency" when any of the currencies are not known.
- [203](#203) Name:"Amount" when any of the amounts is invalid.
- [101](#101) when peer request times out.
- [102](#102) when peer rejects the request.
- [401](#401)

#### 2. Subscribe To Payment Channel Updates

Subscribes to notifications on new incoming payment channel updates for the
specified channel in the specified session. Only one subscription can be made at
a time. Making a new subscription without canceling the previous one will return
an error.

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

If there is an error, it will be one of the following codes:
- [201](#201) ResourceType: "session" when session ID is not known.
- [201](#201) ResourceType: "channel" when channel ID is not known.
- [202](#202) ResourceType: "updatesSub" when a subscription already exists.

*Notification*

Each notification sent to the user should contain the following data:

* `UpdateID`: [String] Unique ID that represents this channel update.
* `Proposed Payment Channel Info`: [[`Payment Channel Info`](#3-payment-channel-info)]
* `Status`: [String] Can be one of the three values: `Open`, `Final`, `Closed`.
    * `Open`: This is a normal channel update, if accepted off-chain state of
      the channel will be progressed to the proposed state.
    * `Final`: This is a final update. Channel will be closed if a
      final update is accepted.
    * `Closed`: The channel has been closed. This will be final
      update notification for the channel and the subscription should be
      cancelled once such an update is received. No response is expected in this
      case. If the user still responds, [201 Resource Not Found](#201) error
      will be returned.
* `Expiry`: [Int64] Time (in unix format) before which response should be
  sent.

#### 3. Unsubscribe From Payment Channel Updates

Unsubscribes from notifications on new incoming payment channel updates for the
specified channel in the specified session.

*Parameters*

* `Session ID`: [String] Unique ID of the session.
* `Channel ID`: [String] Unique ID of the channel.

*Return*

* `Success`: [bool]

*Errors*

If there is an error, it will be one of the following codes:
- [201](#201) ResourceType: "session" when session ID is not known.
- [201](#201) ResourceType: "channel" when channel ID is not known.
- [201](#201) ResourceType: "updatesSub" when a subscription does not exist.

#### 4. Respond To Payment Channel Update

Responds to an incoming payment channel update for which a notification had been
received. Response should be sent before the notification expires. Use the
`Time` API to fetch current time of the perun node for checking notification
expiry.

If multiple payments (in different currencies) is received, the user can either
accept or reject all of them. It is not possible to accept only specific
payments and reject others.

*Parameters*

* `Session ID`: [String]
* `Channel ID`: [String]
* `Update ID`: [String] Update ID received in the notification.
* `Accept`: [Bool] If True, the update will be accepted, else rejected.

*Return*

* `Updated Channel Info`: [[`Payment Channel Info`](#3-payment-channel-info)]

*Errors*

If there is an error, it will be one of the following codes:
- [201](#201) ResourceType: "session" when session ID is not known.
- [201](#201) ResourceType: "channel" when channel ID is not known.
- [201](#201) ResourceType: "update" when update ID is not known.
- [104](#104) when user responded after time out expired.
- [401](#401)

#### 5. Get Payment Channel Info

Gets the last agreed state of the specified payment channel.

*Parameters*

* `Session ID`: [String]
* `Channel ID`: [String]

*Return*

* `Payment Channel Info`: [[`Payment Channel Info`](#3-payment-channel-info)]

*Errors*

If there is an error, it will be one of the following codes:
- [201](#201) ResourceType: "session" when session ID is not known.
- [201](#201) ResourceType: "channel" when channel ID is not known.

#### 6. Close Payment Channel

Closes the channel. First it tries to finalize the last agreed state of the
payment channel off-chain (by sending a finalizing update) and then settling it
on the blockchain. If the channel participants reject/not respond to the
finalizing update, the last agreed state will be finalized directly on the
blockchain. The call will return after this.

The node will then wait for the challenge duration to pass (if the channel was
directly settled on the blockchain) and the withdraw the balance as per the
settled state to the user's account. It then sends a channel update
notification with update types as `Closed`.

*Parameters*

* `Session ID`: [String]
* `Channel ID`: [String]

*Return*

* `Closed Channel Info`: [[`Payment Channel Info`](#3-payment-channel-info)]

*Errors*

If there are any errors in the closing update, it will be one of the following codes:
- [301](#301) TxType: "Conclude" or "ConcludeFinal" when on-chain finalizing tx times out.
- [301](#301) TxType: "Withdraw"  when withdrawing tx times out.
- [302](#302) when connection to blockchain drops while registering.
- [401](#401)

If there is an error returned by this API, it will be one of the following codes:
- [201](#201) ResourceType: "session" when session ID is not known.
- [201](#201) ResourceType: "channel" when channel ID is not known.
- [301](#301) TxType: "Register" when registering tx times out.
- [302](#302) when connection to blockchain drops while registering.
- [401](#401)

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
