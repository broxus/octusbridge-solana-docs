# Principles

In order to check the signatures of relays for each transfer from `Everscale` to `Solana`, the actual relays in current 
round must be stored in `Bridge` [PDA](https://pencilflip.medium.com/learning-solana-3-what-is-a-program-derived-address-732b06def7c1) accounts. 

This will cost a very little of money, paid for the [rent](https://docs.solana.com/developing/programming-model/accounts#rent). For each round new non-upgradeable PDA account will be created.

## Algorithm

1. `Everscale` `Staking` sends new round relays to `Ever event config`.
2. `Ever event config` deploys new `Ever event` with payload containing new round relays.
3. Relays confirm this event.
4. Relays transfer this event to `Solana` `Bridge` program.
5. `Bridge` program gets old round relays info (public keys, addresses) from PDA and checks correctness of signatures received in event info.
6. `Bridge` stores in PDA new round relays info.

## Questions

1. Where to store current round account address?
It will be passed on program input by user, and it must be validated in `Bridge` program.
2. Who transfers new round relays to `Solana`?
If it is relays, they must have lamports to pay gas fee. 
If it is some kind of admin, it must have access to do it. Maybe he mustn't as this event is signed by old round relays and this check is enough.

## Schema

![Relays round loading](../png/Relays_round_loading.png "Relays round loading")