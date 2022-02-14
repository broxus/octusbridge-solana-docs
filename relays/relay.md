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

## Monitoring

Relay software must monitor all transactions that occur on both `Round loader` and `Token proxy` programs.

### Round loader monitoring

In order to vote for new relays round relay software monitors `Round loader` program transactions. The proposal creation transaction
contains data, that is needed to check with the data from `Everscale` event.

#### Round proposal

Each relay can create new round proposal in `Solana`, after such event is received from `Everscale`. This can be done 
in manual mode with relays keys usage or in auto-mode - relay software can have special flag. If this flag is true, 
after receiving new relays round event in `Everscale`, relay software will create corresponding proposal.

### Token proxy monitoring

In order to confirm withdrawal and deposit events relay software monitors `Token proxy` program transactions. 

#### Deposit events

On each deposit event in `Solana` new `Deposit` account is created. Relay software can receive info at any time, knowing
payload id, because address of each account is derived from `Token proxy` program id and payload id.

#### Withdrawal events

On each withdrawal event in `Solana` new `Withdrawal` account is created. Relay software can receive info at any time, knowing
payload id, because address of each account is derived from `Token proxy` program id and payload id.


