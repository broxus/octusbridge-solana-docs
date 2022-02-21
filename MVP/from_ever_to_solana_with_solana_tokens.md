# Proposal #1

## Motivation

The `Solana-Everscale` bridge must have the ability to transfer `Solana` tokens from `Everscale` to `Solana`. The idea here is
to use the vault owned by the `Token proxy` program in the `Solana` blockchain to unlock tokens.

## Algorithm

1. The `Everscale` `Token proxy` burns user tokens after Web3 request.
2. The `Token proxy` sends new event to the `Ever event config`.
3. The `Ever event config` deploys new `Ever event` with payload containing transfer.
4. Relays confirm the `Ever event`.
5. User calls withdraw tokens from vault in the `Solana` `Token Proxy` program transferring the payload with relays signs.
6. The `Token Proxy` program calls the `Round loader` program to check relays signs.
7. The `Round loader` program gets current round relays info (public keys, addresses) and checks signs.
8. If there are no errors wuth the `Round loader` program, then the `Token proxy` creates unique withdraw account with a `PayloadId` from the event.
9. If a withdrawal account was not created before, then the `Token proxy` program calls transfer tokens on the `SPL token` program.
10. The `SPL token` program decreases `Vault` account tokens and increases users tokens balance.

## Questions

1. Where to store the `Vault` account address?
It will be passed on program input by the user, and it must be validated in the `Token proxy` program.
2. Where to store the `Round loader` program address?
It will be passed on program input by the user, and it must be validated in the `Token proxy` program.
3. Where to store the `Current round relays` account address?
It will be passed on program input by the user, and it must be validated in the `Round loader` program.

## Reminders

* Set owner of the `Vault` account to the `Token proxy` program.

## Scheme

![Ever Solana Solana tokens](../png/ever_solana_solana_tokens.png "Ever Solana Solana tokens")

## Issues

The main issue of this scheme is that the user must pass all relays signs when creating a withdrawal on `Solana`. 
The relays signs do not fit the max size for client transactions, and the user has to engage in batching, signing every part with his keys.

# Proposal #2

## Motivation

As described in `Relay round loading` proposal #2, we do not need to check signatures in programs, because we always know 
who is calling when it’s on `Solana`. So this is the first modification. Another one is that we can avoid loading 
all relays approvals on the user side, because they can confirm withdrawals themselves on `Solana`.

## Withdrawal Algorithm

Standard withdrawal algorithm

1. The `Everscale` `Token proxy` burns user tokens after Web3 request.
2. The `Token proxy` sends new event to `Ever event config`.
3. The `Ever event config` deploys the new `Ever event` with a payload containing transfer.
4. Relays get info from the `Ever event`, containing the entire withdrawal payload.
5. User calls withdraw tokens from the vault in the `Solana` `Token Proxy` program, transferring the payload.
6. The `Token proxy` creates a unique withdrawal account with the payload from the event.
7. Relays get callback from `Token proxy` program about a new withdrawal.
8. Relays send confirm withdrawal to the `Token proxy`, containing the payload from `Everscale`.
9. The `Token Proxy` gets requested round relays info (public keys, addresses), checks that callers address is the relay and the round is not expired.
10. If all is ok, the `Token Proxy` program saves relays approval to the withdrawal account and checks if there are enough confirms.
11. The `Token proxy` checks that the withdrawal amount is below the limit. If that is not the case it changes the status to `Waiting for approve`.
12. The `Token proxy` checks the vault balance. 
13. If the balance is enough, the `Token proxy` program calls transfer tokens on the `SPL token` program.
14. The `SPL token` program decreases the amount of `Vault` account tokens and increases the user's tokens balance.
15. The `Token proxy` sets the status to `Processed`.

### Withdrawal account

Withdrawal accounts contain the following:
* Payload Id
* Relays round number
* Sender address in `Everscale`
* Receiver address in `Solana`
* Amount
* `Token root` account address in `Everscale`
* Decimals count in `Everscale`
* Decimals count in `Solana`
* Confirmed Relays
* Bounty for withdrawal
* Minimum Number of confirmation to be processed
* State: new, expired, processed, cancelled, pending, waiting for approve
  * `New` relays confirmations needed to be saved.
  * `Expired` - current round ttl has expired and the withdrawal can not be processed.
  * `Processed` - all funds were successfully transferred to user.
  * `Cancelled` - the user asked to cancel the withdrawal, his funds were minted back to his `Everscale` address.
  * `Pending` - there are not enough funds in the vault to process the withdrawal.
  * `Waiting for approve` - the withdrawal amount is bigger than the limit.

## Scheme

![Ever Solana Solana tokens 2](../png/ever_solana_solana_tokens_2.png "Ever Solana Solana tokens 2")

## Force pending withdrawal algorithm

A user can force his pending withdrawal when he sees that the vault balance is enough.

1. The User calls force pending withdraw in the `Solana` `Token Proxy` program, transferring the payload.
2. The `Token proxy` verifies the existence of the withdrawal account, relays confirmations and pending status.
3. The `Token proxy` checks the vault balance.
4. If the balance is enough, the `Token proxy` program calls transfer tokens on the `SPL token` program.
5. The `SPL token` program decreases the amount of `Vault` account tokens and increases the user’s token balance.

## Cancel pending withdrawal algorithm

A user can cancel his pending withdrawal if he wants to decline it.

1. The User calls cancel pending withdrawal in the `Solana` `Token Proxy` program, transferring the payload.
2. The `Token proxy` verifies the existence of the withdrawal account, relays confirmations, pending status.
3. The `Token proxy` sets the withdrawal status to cancelled.
4. The `Token proxy` creates `Deposit` PDA, containing mirrored data from the withdrawal account, as if the user were 
transferring funds in the opposite direction.
5. Relays monitor this transaction and begin the transferring procedure from `Solana` to `Everscale` (this is described in the corresponding chapter).

## Add / change bounty for pending withdrawal algorithm

A user can set a bounty for processing his pending withdrawal, thereby providing other users with motivation to fill it 
and receive a bounty in `Everscale`.

1. The User calls add or change bounty for pending withdrawal in the `Solana` `Token Proxy` program, transferring the payload and the bounty size.
2. The `Token proxy` verifies the existence of the withdrawal account, relays confirmations, pending status.
3. The `Token proxy` checks that the bounty size is lower than the amount of the withdrawal.
4. The `Token proxy` changes the bounty size in withdrawal account.

## Filling pending withdrawal algorithm

A user from the `Solana` side of the bridge can see that by filling a withdrawal from the `Everscale` side he can receive a bounty
in `Everscale`. So he creates a corresponding transfer of tokens from `Solana` to `Everscale`.

1. The user calls fill pending withdraw in the `Solana` `Token Proxy` program, transferring the payload.
2. The `Token proxy` verifies the existence of the withdrawal account, relays confirmations, pending status.
3. The `Token proxy` checks that users balance is bigger than the amount of the withdrawal minus the bounty.
4. The `Token proxy` program calls transfer tokens (withdrawal amount minus the bounty) on the `SPL token` program.
5. The `SPL token` program decreases the user’s account tokens and increases the receiver of the withdrawal’s token balance.
6. The `Token proxy` creates a `Deposit` PDA, containing mirrored data from the withdrawal account, as if the user 
transferred all funds in the opposite direction.
7. Relays monitor this transaction and begin the transferring procedure from `Solana` to `Everscale` (this is described in the corresponding chapter).

## Approve over limit withdrawals algorithm

All withdrawals that exceed limits will get stuck in the `Waiting for approve` status. When this happens, the admin key 
is used to approve this withdrawal.

1. The Admin calls approve pending withdraw in the `Solana` `Token Proxy` program.
2. The `Token proxy` verifies the existence of the withdrawal account, relays confirmations, status.
3. The `Token proxy` checks the admin key for correctness.
4. The `Token proxy` changes the status to `Pending` for withdrawal.
5. The `Token proxy` checks the vault balance.
6. If the balance is enough, the `Token proxy` program calls transfer tokens on the `SPL token` program.
7. The `SPL token` program decreases the amount of the `Vault` account tokens and increases the user’s token balance.
8. The `Token proxy` sets the status to `Processed`.

