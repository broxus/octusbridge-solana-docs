# Docs

Architecture is presented in draw.io format in current repo. Main idea is to create `Token Proxy` and `Bridge` programs and `Vault` and `Token Root` accounts.  

# Questions and Reminders
1. Check how to use and implement upgradeable accounts, programs. How to make them not upgradeable?
2. Remember to reserve more size for accounts and programs for future uses.
3. Add access to `Token Proxy` to mint / burn Everscale Tokens.
4. Make `Bridge` program, save its public key (address) to `Token Proxy` to check that only this `Bridge` program will be used.
5. Store each round relays in dependent account for `Bridge` program.
6. In case of transfer from Everscale, `Token Proxy` must store payload id, generate address and check that it was not created before. This will save us from double spending.
7. It is needed to decide what architecture to use - one `Token Proxy` for all tokens or one `Token Proxy` for each token.
8. Check that all programs and accounts are below Max size limits.
9. Validate all accounts on input for both `Token Proxy` and `Bridge` programs.
