# Principles

Bridge `Solana-Everscale` must have the ability to transfer `Everscale` tokens from `Solana` to `Everscale`. The idea here is 
to use token root owned by `Token proxy` program in `Solana` blockchain to burn tokens.

## Algorithm

1. `Solana` `Token proxy` program receives withdraw request from user via Web3.
2. It gets info from `Token root` account.
3. It uses `SPL token` program to burn tokens received from user.
4. `SPL token` program decreases users balance and burns tokens.
5. Relays monitor `Solana` `Token proxy` program transactions to receive notification about new transfers via `Solana` Node RPC.
6. User deploys event of new transfer to `Ever event config` via Web3.
7. `Ever event config` deploys new `Ever event` with payload containing transfer.
8. Relays confirm `Ever event`.
9. `Ever event` sends correctness callback to `Ever event config`.
10. `Ever event config` unlocks `Everscale` tokens via `Token proxy` and sends them to users address in `Everscale` blockchain.

## Questions

1. Where to store `Token root` account address?
It will be passed on program input by user, and it must be validated in `Token proxy` program.
