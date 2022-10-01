# Design

Relay software must be upgraded to support `Solana`. The main idea is to make fewer changes. 

## RPC

Relay will connect to `Solana` node via RPC. The best option is to use official `Solana` client, because it is written in 
Rust and relay software is also written in Rust. The only thing to check is how stable is RPC node. Maybe it is better 
to use own or private node. 

## Start

`Solana` and `Eversacle` use key pairs in the same format - [ed25519](https://solana-labs.github.io/solana-web3.js/classes/Keypair.html).
This affords to use the same key pair for both sides. One must remember that, to work with `Solana`, relay needs an account.
So the only manual action here is an account creation. To do so relay needs `Sols`. According to estimate calculations relay needs 
1 `Sol` to start working. It will receive sols from proposal account on vote to cover blockchain fees.

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

User must create new transfer event in `Everscale`, after such proposal is created in `Solana`.

## Relays confirmation

In order to create universal mechanics for the future (for example, to add NFT transfer), relays confirm instruction must
be designed in most common way. Relays should not know specific details of the event occurred, but must have opportunity to check
the correctness of provided data. To do so on `Solana` side the `Event` structure is proposed.

### Event structure

`Event` structure:
* Is initialized flag
* Account kind - Settings, Deposit, Proposal, RelayRound
* Is executed flag
* Author - proposal creator
* Relays round number
* Required votes to be processed
* Proposal Data for approve (PDA) :
  * Settings address - corresponding settings program address
  * Event timestamp - timestamp from `Everscale` blockchain transaction
  * Event transaction logical time - transaction logical time from `Everscale` blockchain transaction
  * Event configuration address - `Everscale` event configuration address, that created event
* Event - bytes array
* Meta - bytes array
* Signers - array of signs (confirm or reject)

Event contains the same data as in `Everscale` and relay can calculate hash of this payload to compare events. Also PDA
and relays round number are used to check event and proposal equality.

Meta is specific `Solana` or `Everscale` information.

To use this kind of structure as `Withdrawal` account, the account data must be modified.

### Transfer from `Everscale` to `Solana` account

Common data is equal to Request structure and 2 additional structures: Withdrawal Event and Withdrawal Meta.

#### Common data

* Is initialized flag
* Account kind - Settings, Deposit, Proposal, RelayRound
* Is executed flag
* Author - proposal creator
* Relays round number
* Required votes to be processed
* Proposal Data for approve (PDA)

#### Withdrawal Event

To deserialize payload as bytes array the first field must contain bytes length.

* Event length - Bytes count of followed data
* Sender address in `Everscale`
* Receiver address in `Solana`
* Amount

#### Withdrawal Meta

To deserialize meta as bytes array the first field must contain bytes length.

* Meta length - Bytes count of followed data
* Status
* Bounty for transfer
* Epoch

