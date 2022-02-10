# Motivation

Each step design is presented in draw.io format in current repo with corresponding docs.

In the beginning the main idea was to create `Token Proxy` and `Round loader` ([Doc](../programs/round_loader.md)) programs and `Vault` and `Token Root` accounts in `Solana` 
and to use the same principles in `Everscale` as for `Ethereum`, `BSC`, ... 
But limits of one transaction size from client in `Solana` showed that the same architecture couldn't be used.
So it was a little modified to meet `Solana` standards. It affected new round relays loading and transferring
tokens from `Everscale` to `Solana`.

## MVP

MVP includes transfer tokens from (to) `Solana` to (from) `Everscale` and contains 5 big steps.

1. Load new round relays from `Everscale` to `Solana`. [Doc](../MVP/relays_round_loading.md)
2. Transfer `Everscale` tokens from `Everscale` to `Solana`. [Doc](../MVP/from_ever_to_solana_with_ever_tokens.md)
3. Transfer `Solana` tokens from `Everscale` to `Solana`. [Doc](../MVP/from_ever_to_solana_with_solana_tokens.md)
   1. Transfer tokens from vault, when there is enough funds.
   2. Force pending withdraw. The case when it was not enough on withdraw occurred, but someone has fulfilled the vault.
   3. Cancel pending withdraw.
   4. Add/Change bounty to pending withdraw.
   5. Fill pending withdraw, that has bounty reward for it.
   6. Approve withdrawals over limit
4. Transfer `Everscale` tokens from `Solana` to `Everscale`. [Doc](../MVP/from_solana_to_ever_with_ever_tokens.md)
5. Transfer `Solana` tokens from `Solana` to `Everscale`. [Doc](../MVP/from_solana_to_ever_with_solana_tokens.md)

## II Stage

II Stage includes transfer tokens from `Solana` to `Everscale` with the use of credits processor in `Everscale`. [Doc](../Stage 2/from_solana_to_ever_with_solana_tokens_2_stage.md)

## III Stage

III Stage includes transfer locked in `Solana` tokens to different protocols in `Solana` to earn on them. [Doc](../Stage 3/transfer_liquidity_in_solana_to_protocol_3_stage.md)

# Questions

1. Check how to use and implement upgradeable accounts, programs. How to make them not upgradeable?
`Solana` affords to upgrade programs via special instruction in SysVar program. Example is presented [here](https://medium.com/coinmonks/solana-internals-part-2-how-is-a-solana-deployed-and-upgraded-d0ae52601b99).
2. Add access to `Token Proxy` to mint / burn `Everscale` Tokens
This is done via [CPI](https://docs.solana.com/developing/programming-model/calling-between-programs).
3. It is needed to decide what architecture to use - one `Token Proxy` for all tokens or one `Token Proxy` for each token?
Main idea here - if attacker could have access to token proxy, it could make less damage, if one `Token Proxy` 
is used for each token.
4. What are programs and accounts size limits?
Max limit in `Solana` is 10Mb. It is enough to all accounts and programs, used for bridging.
5. What is client transaction size limit?
Max limit in `Solana` is 1232 bytes according to [docs](https://docs.solana.com/ru/proposals/transactions-v2). 
It is very small to transfer all relays and its signatures for transfer from `Everscale` like it was done in `Ethereum` bridge.
6. How to reuse relays keys without loading new one, specific for `Solana`?
`Solana` [supports](https://solana-labs.github.io/solana-web3.js/classes/Keypair.html) Ed25519 Keypair, so do `Everscale`. 
We can just take the same keys.
7. How to defend from big loss in case of emergency situation?
We can use limits and manual approve of withdrawals from `Everscale` to `Solana` that are bigger than that limits.
8. Who can approve over limits withdrawals?
It can be multisig in `Everscale` or `Solana`. In first case approve event will be transferred through bridge.

# Reminders for developers

1. Remember to reserve more size for accounts and programs for future uses.
2. Make `Round loader` program, save its program id to `Token Proxy` to check that only this `Round loader` program will be used.
3. Store each round relays in `Round loader` program dependent accounts. So all rounds relays will be stored and one can check relay public key from any round.
4. In case of transfer from `Everscale`, `Token Proxy` must store payload id, generate address and check that it was not created before. This will save us from double spending.
5. Validate all accounts on input for both `Token Proxy` and `Round loader` programs. As by design all accounts in `Solana` is 
transferred by user in client transaction, all programs must check all of them in order to not get fooled.
