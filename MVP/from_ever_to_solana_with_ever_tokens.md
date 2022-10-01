# Proposal #1 (DEPRECATED!)

## Motivation

Bridge `Solana-Everscale` must have the ability to transfer `Everscale` tokens from `Everscale` to `Solana`. The idea here is
to use token root owned by the `Token proxy` program on the `Solana` blockchain to mint tokens.

## Algorithm

1. `Everscale` `Token proxy` locks user tokens after Web3 request.
2. `Token proxy` sends new event to `Everscale event configuration`.
3. `Everscale event configuration` deploys new `Everscale event` with payload containing transfer.
4. Relays confirm `Everscale event`.
5. User calls mint tokens in `Solana` `Token Proxy` program transferring payload with relays signs.
6. `Token Proxy` program calls `Round loader` program to check relays signs.
7. `Round loader` program gets current round relays info (public keys, addresses) and checks signs.
8. If no error got from the `Round loader` program, then the `Token proxy` creates unique withdraw account with `PayloadId` from the event.
9. If a withdrawal account was created before, then the `Token proxy` program calls mint on the `SPL token` program.
10. The `SPL token` program mints tokens and increases users balance.

## Questions

1. Where to store the `Token root` account address?
It will be passed on program input by the user, and it must be validated in the `Token proxy` program.
2. Where to store the `Round loader` program address?
It will be passed on program input by the user, and it must be validated in the `Token proxy` program.
3. Where to store the `Current round relays` account address?
It will be passed on program input by user, and it must be validated in the `Round loader` program.
4. Is it needed to separate confirm withdraw by relays to two parts:
    1. Relay's confrim.
    2. Mint tokens to user.
   In that case relay will pay for the confirm and user can pay for the mint.


## Scheme

![Ever Solana Ever tokens](../png/ever_solana_ever_tokens.png "Ever Solana Ever tokens")

## Issues

The main issue of this scheme is the same as with the proposal for transferring `Solana` tokens from `Everscale` to `Solana`.

# Proposal #2 (Current)

## Motivation

The same as with the proposal #2 for transferring `Solana` tokens from `Everscale` to `Solana`.

## Mint Algorithm

Mint algorithm

1. `Everscale` `Token proxy` burns user tokens after Web3 request.
2. `Token proxy` sends new event to `Everscale event configuration`.
3. `Everscale event configuration` deploys new `Everscale event` with payload containing transfer.
4. Relays get info from the `Everscale event`, containing the entire withdrawal payload.
5. User calls withdraw tokens from token root in the `Solana` `Token Proxy` program, transferring payload.
6. `Token proxy` creates unique withdraw account with payload from event.
7. Relays get callback from the `Token proxy` program about new withdrawal.
8. Relays send confirm withdrawal to `Token proxy`, containing payload from `Everscale`.
9. `Token Proxy` gets round relays info (public keys, addresses), checks that callers address is relay and round is not expired.
10. If all is ok, the `Token Proxy` program saves relays approval to withdrawal account and checks if there are enough confirms. 
11. if there are enough confirms, the `Token proxy` program calls mint tokens on the `SPL token` program.
12. The `SPL token` program mints tokens and increases users balance.

In this case withdrawal account can have only three states: new, processed and waiting for approval.

## Scheme

![Ever Solana Ever tokens 2](../png/ever_solana_ever_tokens_2.png "Ever Solana Ever tokens 2")