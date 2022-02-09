# Motivation

Each step design is presented in draw.io format in current repo with corresponding docs.

In the beginning the main idea was to create `Token Proxy` and `Bridge` programs and `Vault` and `Token Root` accounts in `Solana` 
and to use the same principles in `Everscale` as for `Ethereum`, `BSC`, ... 
But limits of one transaction size from client in `Solana` showed that the same architecture couldn't be used.
So it was a little modified to meet `Solana` standards. It affected new round relays loading and transferring
tokens from `Everscale` to `Solana`.

## MVP

MVP includes transfer tokens from (to) `Solana` to (from) `Everscale` and contains 5 big steps.

1. Load new round relays from `Everscale` to `Solana`.
2. Transfer `Everscale` tokens from `Everscale` to `Solana`.
3. Transfer `Solana` tokens from `Everscale` to `Solana`.
   1. Transfer tokens from vault, when there is enough funds.
   2. Force pending withdraw. The case when it was not enough on withdraw occurred, but someone has fulfilled the vault.
   3. Cancel pending withdraw.
   4. Add/Change bounty to pending withdraw.
   5. Fill pending withdraw, that has bounty reward for it.
4. Transfer `Everscale` tokens from `Solana` to `Everscale`.
5. Transfer `Solana` tokens from `Solana` to `Everscale`.

## II Stage

II Stage includes transfer tokens from `Solana` to `Everscale` with the use of credits processor in `Everscale`.

## III Stage

III Stage includes transfer locked in `Solana` tokens to different protocols in `Solana` to earn on them.

# Questions

1. Check how to use and implement upgradeable accounts, programs. How to make them not upgradeable?
`Solana` affords to upgrade programs via special instruction in SysVar program. Example is presented [here](https://medium.com/coinmonks/solana-internals-part-2-how-is-a-solana-deployed-and-upgraded-d0ae52601b99).
2. Add access to `Token Proxy` to mint / burn `Everscale` Tokens
This is done via [CPI](https://docs.solana.com/developing/programming-model/calling-between-programs).
3. It is needed to decide what architecture to use - one `Token Proxy` for all tokens or one `Token Proxy` for each token?
The main idea here is that if attacker could have access to token proxy it could make less damage, if one `Token Proxy` 
is used for each token.
4. What are programs and accounts size limits?
The max limit in `Solana` is 10Mb. It is enough to all accounts and programs, used for bridging.
5. What is client transaction size limit?
The max limit in `Solana` is 1232 bytes according to [docs](https://docs.solana.com/ru/proposals/transactions-v2). 
It is very small to transfer all relays and its signatures for transfer from `Everscale` like it was done in `Ethereum` bridge.

# Reminders for developers

1. Remember to reserve more size for accounts and programs for future uses.
2. Make `Bridge` program, save its public key (address) to `Token Proxy` to check that only this `Bridge` program will be used.
3. Store each round relays in `Bridge` program dependent account. So `Bridge` will have all rounds relays and can check transfer in any round.
4. In case of transfer from `Everscale`, `Token Proxy` must store payload id, generate address and check that it was not created before. This will save us from double spending.
5. Validate all accounts on input for both `Token Proxy` and `Bridge` programs. As by design all accounts in `Solana` is 
transferred by user in client transaction, all programs must check all of them in order to not get fooled.
