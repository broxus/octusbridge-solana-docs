# Principles

Bridge `Solana-Everscale` must have the ability to transfer `Solana` tokens from `Solana` to `Everscale`. The idea here is 
to use vault in `Solana` blockchain to store received from users tokens and lock them until opposite direction transfer will occur.

## Algorithm

1. `Solana` `Token proxy` program receives withdraw request from user via Web3.
2. It uses `SPL token` program to transfer tokens from user to vault.
3. `SPL token` program decreases users balance.
4. `SPL token` program increases vault balance.
5. Relays monitor `Solana` `Token proxy` program transactions to receive notification about new transfers via `Solana` Node RPC.
6. User deploys event of new transfer to `Ever event config` via Web3.
7. `Ever event config` deploys new `Ever event` with payload containing transfer.
8. Relays confirm `Ever event`.
9. `Ever event` sends correctness callback to `Ever event config`.
10. `Ever event config` mints `Solana` tokens via `Token proxy` and sends them to users address in `Everscale` blockchain.

## Reminders

1. Set owner of `Vault` account to `Token proxy` program.

## Schema

![Solana Ever Solana tokens](../png/solana_ever_solana_tokens.png "Solana Ever Solana tokens")