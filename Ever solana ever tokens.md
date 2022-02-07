# Principles

Bridge `Solana-Everscale` must have the ability to transfer `Everscale` tokens from `Everscale` to `Solana`. The idea here is
to use token root owned by `Token proxy` program in `Solana` blockchain to mint tokens.

## Algorithm

1. `Everscale` `Token proxy` locks user tokens after Web3 request.
2. `Token proxy` sends new event to `Ever event config`.
3. `Ever event config` deploys new `Ever event` with payload containing transfer.
4. Relays confirm `Ever event`.
5. User calls mint tokens in `Solana` `Token Proxy` program transferring payload with relays signs.
6. `Token Proxy` program calls `Bridge` program to check relays signs.
7. `Bridge` program gets current round relays info (public keys, addresses) and checks signs.
8. If no error got from `Bridge` program, then `Token proxy` gets info from `Token root` account.
9. `Token proxy` program calls mint on `SPL token` program.
10. `SPL token` program mints tokens and increases users balance.

## Questions

1. Where to store `Token root` account address?
It will be passed on program input by user, and it must be validated in `Token proxy` program.
2. Where to store `Bridge` program address?
It will be passed on program input by user, and it must be validated in `Token proxy` program.
3. Where to store `Current round relays` account address?
It will be passed on program input by user, and it must be validated in `Bridge` program.


## Schema

![Ever Solana Ever tokens](../png/ever_solana_ever_tokens.png "Ever Solana Ever tokens")
