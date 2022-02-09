# Proposal #1

Bridge `Solana-Everscale` must have the ability to transfer `Solana` tokens from `Everscale` to `Solana`. The idea here is
to use vault owned by `Token proxy` program in `Solana` blockchain to unlock tokens.

## Algorithm

1. `Everscale` `Token proxy` burns user tokens after Web3 request.
2. `Token proxy` sends new event to `Ever event config`.
3. `Ever event config` deploys new `Ever event` with payload containing transfer.
4. Relays confirm `Ever event`.
5. User calls withdraw tokens from vault in `Solana` `Token Proxy` program transferring payload with relays signs.
6. `Token Proxy` program calls `Bridge` program to check relays signs.
7. `Bridge` program gets current round relays info (public keys, addresses) and checks signs.
8. If no error got from `Bridge` program, then `Token proxy` creates unique withdraw account with `PayloadId` from event.
9. If no withdraw account was created before, then `Token proxy` program calls transfer tokens on `SPL token` program.
10. `SPL token` program decreases `Vault` account tokens and increases users tokens balance.

## Questions

1. Where to store `Vault` account address?
It will be passed on program input by user, and it must be validated in `Token proxy` program.
2. Where to store `Bridge` program address?
It will be passed on program input by user, and it must be validated in `Token proxy` program.
3. Where to store `Current round relays` account address?
It will be passed on program input by user, and it must be validated in `Bridge` program.

## Reminders

1. Set owner of `Vault` account to `Token proxy` program.

## Schema

![Ever Solana Solana tokens](../png/ever_solana_solana_tokens.png "Ever Solana Solana tokens")

## Issues

The main issue of this scheme is that user must pass all relays signs in transaction of creating withdrawal in `Solana`.
They do not fit the max size of client transaction, and user have to us batching, signing every part with his keys.

# Proposal #2

As it has been described in `Relay round loading` proposal #2 we do not need to check signatures in programs, because 
we always know who is calling its in `Solana`. So this is the first modification. Another on is that we can avoid loading
all relays approvals from user side, because they can themselves confirm withdrawal in `Solana`.

## Withdrawal Algorithm

1. `Everscale` `Token proxy` burns user tokens after Web3 request.
2. `Token proxy` sends new event to `Ever event config`.
3. `Ever event config` deploys new `Ever event` with payload containing transfer.
4. Relays get info from `Ever event`, containing all withdrawal payload.
5. User calls withdraw tokens from vault in `Solana` `Token Proxy` program, transferring payload.
6. `Token proxy` creates unique withdraw account with payload from event.
7. Relays get callback from `Token proxy` program about new withdrawal.
8. Relays send confirm withdrawal to `Token proxy`, containing payload from `Everscale`.
9. `Token Proxy` gets requested round relays info (public keys, addresses), checks that callers address is relay and round is not expired.
10. If all is ok, `Token Proxy` program saves relays approval to withdrawal account and checks if there are enough confirms.
11. `Token proxy` checks vault balance. 
12. If balance is enough, `Token proxy` program calls transfer tokens on `SPL token` program.
13. `SPL token` program decreases `Vault` account tokens and increases users tokens balance.

### Withdraw account

Withdraw account is containing following:
* Round number
* Receiver address in `Solana`
* Sender address in `Everscale`
* Amount
* Payload Id
* Bounty for withdrawal
* State: new, expired, processed, cancelled, pending
    * `New` is needed to save relays confirms.
    * `Expired` - current round ttl is expired and withdrawal can not be processed.
    * `Processed` - all funds were successfully transferred to user.
    * `Cancelled` - user asked to cancel withdrawal, his funds were minted in `Everscale` to his address back.
    * `Pending` - there is not enough funds on vault to process the withdrawal.

## Schema

![Ever Solana Solana tokens 2](../png/ever_solana_solana_tokens_2.png "Ever Solana Solana tokens 2")

## Force withdrawal Algorithm

User can force his withdrawal in pending state when he sees that vault balance is enough.


