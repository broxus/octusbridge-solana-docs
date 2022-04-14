# Design

Relay software is to be upgraded to support `Solana`. The main idea is to make fewer changes. 

## RPC

Relay will connect to `Solana` node via RPC. The best option is to use official `Solana` client, because it is written in 
Rust and relay software is also written in Rust. The only thing to check is how stable is node RPC. Maybe it is better 
to run node in own environment. 

## Start

`Solana` and `Eversacle` use key pairs in the same format - [ed25519](https://solana-labs.github.io/solana-web3.js/classes/Keypair.html).
This affords to use the same key pair for both sides. One must remember that to work with `Solana` relay needs an account.
So the only manual action here is an account creation. To do so relay needs `Sols`. According to estimates relay needs to have
10 `Sols` per year.

## Processing events

Relay software must monitor all transactions that occur on `Everscale` part.
By design, `Everscale` bridge part doesn't make difference between, for example, transfer events and new round events.
To make relays more common, it is needed to these events look the same for approval. So in `Solana` accounts must be unified.

### Loading new relays rounds

One must create new round proposal in `Solana`, after such event is received from `Everscale`. This can be done 
in manual mode or in auto-mode - by special written script. After receiving new relays round event in `Everscale`, 
this script will create corresponding proposal.

### Tokens transfer from `Everscale` to `Solana`

User must create new transfer proposal in `Solana`, after such event is created in `Everscale`. 

### Tokens transfer from `Solana` to `Everscale`

User must create new transfer proposal in `Everscale`, after such event is created in `Solana`.

## Relays confirmation

In order to create universal mechanics for the future (for example, to add NFT transfer), relays confirm instruction must
be designed in most common way. Relays should not know specific details of the event occurred, but must have opportunity to check
the correctness of provided data. To do so on `Solana` side the `Event` structure is proposed.

### Event structure

`Event` structure:
* Payload Id
* Relays round number
* Minimum Number of confirmation to be processed
* Confirmed Relays
* Payload - bytes array
* Meta - bytes array

Payload contains the same data as in `Everscale` and `Payload Id` can contain calculated hash of this payload.
Meta is specific `Solana` or `Everscale` information.

To use this kind of structure as `Withdrawal` account, the account data must be modified.

### Transfer from `Everscale` to `Solana` account

Common data is equal to Request structure and 2 additional structures: Withdrawal Payload and Withdrawal Meta.

#### Common data

* Payload Id
* Relays round number
* Minimum Number of confirmation to be processed
* Confirmed Relays

#### Transfer Payload

To deserialize payload as bytes array the first field must contain bytes length.

* Payload length - Bytes count of followed data
* Sender address in `Everscale`
* Receiver address in `Solana`
* Amount
* Decimals count in `Solana`
* Timestamp - Withdrawal timestamp from `Everscale`

#### Transfer Meta

To deserialize meta as bytes array the first field must contain bytes length.

* Meta length - Bytes count of followed data
* Kind of transfer
* Author
* Status
* Bounty for transfer
* Name - currency ticker

