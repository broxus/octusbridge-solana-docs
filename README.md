# Architecture

Architecture is presented in draw.io format in current repo with docs for each step.

Main idea is to create `Token Proxy` and `Bridge` programs and `Vault` and `Token Root` accounts in `Solana` 
and to use the same principles in `Everscale` as for `Ethereum`, `BSC`, ...  

# MVP

MVP includes transfer tokens from (to) `Solana` to (from) `Everscale` and contains 5 steps.

1. Load new round relays from `Everscale` to `Solana`.
2. Transfer `Everscale` tokens from `Everscale` to `Solana`.
3. Transfer `Solana` tokens from `Everscale` to `Solana`.
4. Transfer `Everscale` tokens from `Solana` to `Everscale`.
5. Transfer `Solana` tokens from `Solana` to `Everscale`.

# II Stage

II Stage includes transfer tokens from `Solana` to `Everscale` with the use of credits processor in `Everscale`.

# III Stage

III Stage includes transfer locked in `Solana` tokens to different protocols in `Solana` to earn on them.

# Questions and Reminders

1. Check how to use and implement upgradeable accounts, programs. How to make them not upgradeable?
2. Remember to reserve more size for accounts and programs for future uses.
3. Add access to `Token Proxy` to mint / burn `Everscale` Tokens.
4. Make `Bridge` program, save its public key (address) to `Token Proxy` to check that only this `Bridge` program will be used.
5. Store each round relays in dependent account for `Bridge` program.
6. In case of transfer from `Everscale`, `Token Proxy` must store payload id, generate address and check that it was not created before. This will save us from double spending.
7. It is needed to decide what architecture to use - one `Token Proxy` for all tokens or one `Token Proxy` for each token.
8. Check that all programs and accounts are below Max size limits.
9. Validate all accounts on input for both `Token Proxy` and `Bridge` programs.
