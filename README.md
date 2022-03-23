# Why Solana?

`Solana` has more than 300 dApps, with more than 10b dollars locked. It's good to add tha ability to transfer funds
to or from `Solana`. More can be read [here](https://docs.solana.com/ru/introduction#why-solana).

# What is Solana?

Can be read [here](https://docs.solana.com/ru/introduction#what-is-solana).

# Bridge design

Each design step is presented in draw.io format in current repo with corresponding docs.

In the beginning the main idea was to create `Token Proxy` ([Doc](../programs/token_proxy.md)) and `Round loader` ([Doc](../programs/round_loader.md)) programs and `Vault` and `Token Root` accounts in `Solana` 
and to use the same principles in `Everscale` as used with `Ethereum`, `BSC`, ... 
But the limits of one transaction size from client in `Solana` showed that the same architecture couldn't be used.
So it was slightly modified to meet `Solana` standards. This has affected new round relays for loading and transferring
tokens from `Everscale` to `Solana`.

## MVP

The MVP provides the ability to transfer tokens from (to) `Solana` to (from) `Everscale` and contains 5 big steps.

1. Load new round relays from `Everscale` to `Solana`. ([Doc](../MVP/relays_round_loading.md))
2. Transfer `Everscale` tokens from `Everscale` to `Solana`. ([Doc](../MVP/from_ever_to_solana_with_ever_tokens.md))
3. Transfer `Solana` tokens from `Everscale` to `Solana`. ([Doc](../MVP/from_ever_to_solana_with_solana_tokens.md))
   1. Transfer tokens from the vault, when there are enough funds.
   2. Force pending withdrawal. This occurs when there are not enough funds in a vault account, and someone replenished it.
   3. Cancel pending withdrawal.
   4. Change bounty of pending withdrawal.
   5. Fill pending withdrawal.
   6. Approve big amount withdrawals.
4. Transfer `Everscale` tokens from `Solana` to `Everscale`. ([Doc](../MVP/from_solana_to_ever_with_ever_tokens.md))
5. Transfer `Solana` tokens from `Solana` to `Everscale`. ([Doc](../MVP/from_solana_to_ever_with_solana_tokens.md))

## Relays

Relays will support `Solana` as long as `Everscale` and `Ethereum` and its forks. ([Doc](../relays/relay.md))

## Stage II

Stage II allows users to transfer tokens from `Solana` to `Everscale` with the use of credits processor in `Everscale`. ([Doc](../Stage 2/from_solana_to_ever_with_solana_tokens_2_stage.md))

## Stage III

Stage III allows users to transfer locked-in `Solana` tokens to different protocols on `Solana` to earn on them. ([Doc](../Stage 3/transfer_liquidity_in_solana_to_protocol_3_stage.md))

# Questions

1. Check how to use and implement upgradeable accounts, programs. How to make them not upgradeable?
With `Solana`, you can upgrade programs via special instruction in the SysVar program. An example is presented [here](https://medium.com/coinmonks/solana-internals-part-2-how-is-a-solana-deployed-and-upgraded-d0ae52601b99).
2. How to add access to `Token Proxy` to mint / burn `Everscale` Tokens?
This is done via [CPI](https://docs.solana.com/developing/programming-model/calling-between-programs).
3. What architecture should be used - one `Token Proxy` for all tokens or one `Token Proxy` for each token?
The main idea here is that if an attacker could have access to the token proxy, there could could be less damage 
if one `Token Proxy` is used for each token.
4. What are program and account size limits?
The max limit for `Solana` is 10Mb. It is enough for all accounts and programs used for bridging.
5. What is the client transaction size limit?
The max limit for `Solana` is 1232 bytes according to [docs](https://docs.solana.com/ru/proposals/transactions-v2). 
It is very small to transfer all relays and their signatures for transfer from `Everscale` like was done for `Ethereum` bridge.
6. How can you reuse relays keys without loading a new one, specificly for `Solana`?
`Solana` [supports](https://solana-labs.github.io/solana-web3.js/classes/Keypair.html) the Ed25519 Keypair and so does `Everscale`.
Therefore, you can just use the same keys.
7. How can you defend yourself from big losses in emergency situations?
We can use limits and manually approve withdrawals from `Everscale` to `Solana` when the amount is bigger than the limits.
8. Who can approve withdrawals that are over the established limits?
Approvals for bigger amounts can be done via multi-signature on `Everscale` or `Solana`. In the first case, the approval event must be transferred through the Bridge.
9. Is there anything like `Ethereum` chain id in `Solana`?
No, for the same purpose `Solana` use [recent blockhash](https://docs.solana.com/ru/implemented-proposals/durable-tx-nonces) in transactions.
10. Can `Solana` program be updated and are there any limitations?
Yes, it can be done via special [program](https://docs.rs/solana-program/latest/solana_program/bpf_loader_upgradeable/index.html). Program
on deploy have limits of space being used for program data. It couldn't be increased in the future. Knowing that, one must 
set space size a little more than current program size, when first time deploying the program.

# Reminders for developers

1. Remember to reserve more size for accounts and programs for future uses.
2. Make the `Round loader` program and save its program id to the `Token Proxy` to make sure that only this `Round loader` program will be used.
3. Store all round relays in the `Round loader` program-dependent accounts. So all round relays will be stored, and you can check relay public keys from any round.
4. When transferring from `Everscale`, the `Token Proxy` must store the payload id, generate the address and check that it was not created before. This will save us from double spending.
5. Validate all accounts on input for both the `Token Proxy` and `Round loader` programs. By design, all accounts in `Solana` are 
transferred by user in client transaction, and all programs must check all of them in order to not get fooled.
