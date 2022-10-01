# Proposal #1 (DEPRECATED!)

## Motivation

Bridge `Solana-Everscale` must have the ability to transfer `Solana` tokens from `Solana` to `Everscale`. The idea here is 
to use vault in `Solana` blockchain to store received from users tokens and lock them until opposite direction transfer will occur.

## Algorithm

1. `Solana` `Token proxy` program receives withdraw request from user via Web3.
2. It uses `SPL token` program to transfer tokens from user to vault.
3. `SPL token` program decreases users balance.
4. `SPL token` program increases vault balance.
5. Relays monitor `Solana` `Token proxy` program transactions to receive notification about new transfers via `Solana` Node RPC.
6. User deploys event of new transfer to `Everscale event configuration` via Web3.
7. `Everscale event configuration` deploys new `Everscale event` with payload containing transfer.
8. Relays confirm `Everscale event`.
9. `Everscale event` sends correctness callback to `Everscale event configuration`.
10. `Everscale event configuration` mints `Solana` tokens via `Token proxy` and sends them to users address in `Everscale` blockchain.

## Reminders

1. Set owner of `Vault` account to `Token proxy` program.

## Scheme

![Solana Ever Solana tokens](../../png/solana_ever_solana_tokens.png "Solana Ever Solana tokens")

## Proposal #2 (Current)

To save all deposits by user it is good to have PDA, created on each user request.
Between steps 4 and 5 `Token proxy` program should create such one.

### Deposit account

Deposit account description can be found in ([Round loader](../programs/token_proxy.md)) in `Deposit account` section
