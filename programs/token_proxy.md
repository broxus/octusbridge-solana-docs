# Design

`Token proxy` program is the main program in bridge from `Solana` side. It owns root token and vault accounts and can
make transfers, mints, burns tokens.

## Deploy

`Root token` account and `Vault` account must be made as PDA via `SPL token` program.

Another important notice here is that `Token proxy` program must be deployed after `Round loader` program, because `Token proxy`
uses it to derive relays round account address.

## Start

In order to start working `Token proxy` must be initialized. On initialization - `Settings` account is created. 

###  Settings account

It must store following:
* Is initialized flag
* Account Kind: `Settings`
* Emergency mode - The mode that denies all deposits and withdrawals.
* Guardian - Public key of guardian, that can switch off / on emergency mode
* Withdrawal manager - Public key, that can allow approving big withdrawals.
* Manager - Public key, that can change limits of tokens. 

## Tokens

Bridge supports multi-token scheme, this means that new supported tokens can be added by users deposits and withdrawals. 
On first appearance of new token transfer `Token Settings` account will be created.
Here is two options: to use vault and to use token root (`Mint` in `Solana` vocabulary).
It depends on what kind of token this proxy needs to maintain. In case of `Everscale` token, it works with `Token root` account,
so its address must be stored in `Token Settings` account. In case of `Solana` token, it uses `Vault` account.

### Token Settings account

It must store following:
* Is initialized flag
* Account Kind: `Settings`
* Token name
* Token symbol
* Token Kind: `Ever` or `Solana`:
  - `Ever`: 
    * Mint address
    * Token root address in `Everscale`
    * Ever decimals
  - `Solana`:
    * Mint address
    * Vault address
* Deposit limit - Max vault amount limit for `Solana` tokens deposit. In order not to lose a lot of funds in case of non-consistent work
  the max deposit limit can be set.
* Withdrawals limit - Max auto withdrawals amount.
* Withdrawals daily limit - Max auto withdrawals amount for day (24h).
* Withdrawals daily amount - Amount of all withdrawals in that day (epoch).
* Withdrawal epoch - Number of day from UNIX Time 0.
* Emergency mode - The mode that denies all deposits and withdrawals for this token. 
* Fee information - Information about calculated fees
  * multiplier - special field for fee amount calculation 
  * divisor - special field for fee amount calculation
  * supply - total amount of collected fees

`Token settings` account address can be derived from `Everscale` root address in case of `Ever` token kind and from mint address
in `Solana` case.

By default, all limits are set to zero, so users can transfer tokens without any limits.

### Vault account

`Vault` account contains is created via `InitializeAccount` instruction from `SPL token` program and contains:
* Amount

### Solana token root account

`Token root` is also named `Mint` account in `Solana`. It is created via `InitializeMint` instruction from `SPL token` program.

### Initialization

1. Deployer of `Token proxy` program calls init method, providing settings data.
2. `Token proxy` program calculates `Settings` PDA address, checks that it is not created.
3. `Token proxy` program creates `Settings` PDA, storing data provided.

### Token Initialization

1. `Token proxy` program calls `InitializeMint` or `InitializeAccount` instructions from `SPL token` program, providing token settings data.
2. `Token proxy` program calculates `Token Settings` PDA address, checks that it is not created.
3. `Token proxy` program creates `Token Settings` PDA, storing data provided.

## Common work

Common work is divided into two big steps - deposit and withdrawal. In both cases new PDA, storing all related info, is created. 
This allows relays to be always up-to-date with what is happening on `Solana` side.

### Deposit

Deposit is the tokens transfer from `Solana` to `Everscale`. 

#### `Everscale` tokens deposit

1. User calls deposit method of `Token proxy` program.
2. `Token proxy` program calculates `Settings` PDA address, fetches it and gets `Token root` account address.
3. `Token proxy` program fetches `Token root` account.
4. `Token proxy` program checks that emergency mode is off.
5. It uses `SPL token` program address from settings to call burn users tokens.
6. `Token proxy` program creates `Deposit` PDA.

#### `Solana` tokens deposit

1. User calls deposit method of `Token proxy` program.
2. `Token proxy` program calculates `Settings` PDA address.
3. If it is the new token, the `Token Initialization` procedure by `Token proxy` program occurs.
4. `Token proxy` program fetches `Vault` account.
5. `Token proxy` program checks that emergency mode is off.
6. `Token proxy` program checks that Vault amount plus deposit amount is not bigger than deposit limit.
7. It uses `SPL token` program address from settings to call transfer users tokens from users account to `Vault` account.
8. `Token proxy` program creates `Deposit` PDA.

#### Deposit account

`Deposit` account contains:
* Is initialized flag
* Account Kind: `Deposit`
* Event:
  * Event length in bytes
  * Token address in `Everscale`
  * Amount of transferred tokens
  * Receiver address in `Everscale`
  * Value of sols applied for credit transfer
  * Expected Everscale native tokens to sols value be exchanged for by credit processor 
* Meta:
  * Meta length in bytes 
  * Account seed - unique value

`Deposit` address is derived from `Seed` address.

### Withdrawal

Withdrawal is the tokens transfer from `Everscale` to `Solana`. 

#### `Everscale` tokens withdrawal

It can be divided into 4 parts. 
At first user creates `Withdrawal` account. If it is the first time this token transferring, `Token Settings` PDA will be created.
Second, relays confirm, that it is a real withdrawal.
Third, relays can execute token transfer to user, if this option is set in `Everscale` token configuration.
Fourth, relays can execute payload, applied to withdrawal, if this option is set in `Everscale` token configuration and 
payload presents in withdrawal event.

##### User withdrawal request creation

1. User calls withdraw tokens from `Token` account in `Solana` `Token Proxy` program, transferring payload.
2. `Token proxy` program calculates `Settings` PDA address, fetches it and gets `Token root` account address.
3. `Token proxy` creates `Withdraw` account.

##### Relays confirmation

1. Relays send confirm withdrawal to `Token proxy`, containing the same payload from `Everscale`.
2. `Token proxy` program calculates `Token Settings` PDA address, fetches it and gets `Token root` account address, `Relay loader` program id,
`SPL token` program id.
3. `Token proxy` program calculates `Relays round` account, using seed, derived from `Round loader` program id and 
relays round number from payload.
4. `Token Proxy` gets round relays info (public keys, addresses), checks that callers address is relay and ttl of the round is not expired.
5. If all is ok, `Token Proxy` program saves relays approval to withdrawal account and checks if there are enough confirms.

##### Withdraw

1. If it is the new token, the `Token Initialization` procedure by `Token proxy` program occurs.
2. if there are enough confirms, `Token proxy` program checks that withdrawal amount is lower than limits from settings.
3. If amount is lower:
   1. `Token proxy` program uses `SPL token` program address from settings and calls mint tokens on `SPL token` program.
   2. `Token proxy` program changes withdrawal amount in current period.
   3. `Token proxy` program sets withdrawal status to `Processed`.
4. If amount is bigger, `Token proxy` program sets withdrawal status to `Waiting for approve`.

##### Approve over limit withdrawals algorithm

1. Admin calls approve pending withdrawal in `Solana` `Token Proxy` program.
2. `Token proxy` program calculates `Settings` PDA address and `Token Settings` PDA address, fetches it and gets `Token root` 
account address, `Withdrawal manager` account, `SPL token` program id.
3. `Token proxy` program checks that emergency mode is off.
4. `Token proxy` checks `Withdrawal manager` account with the received once.
5. `Token proxy` fetches withdrawal account.
6. `Token proxy` checks relays confirmations and status.
7. If all is ok, `Token proxy` program uses `SPL token` program address from settings and calls mint tokens on `SPL token` program
8. `Token proxy` sets withdrawal status to `Processed`.

#### `Solana` tokens withdrawal

It can be divided into 2 main parts. At first user created `Withdrawal` account.
Second, relays confirm, that it is a real withdrawal.
Third, relays can execute token transfer to user, if this option is set in `Everscale` token configuration.
Fourth, relays can execute payload, applied to withdrawal, if this option is set in `Everscale` token configuration and
payload presents in withdrawal event.
In addition, it must serve 5 more situations with vault.

##### User withdrawal request creation

1. User calls withdraw tokens from `Vault` account in `Solana` `Token Proxy` program, transferring payload.
2. `Token proxy` program calculates `Settings` PDA address, fetches it and gets `Vault` account address.
3. `Token proxy` creates `Withdraw` account.

##### Relays confirmation

1. Relays send confirm withdrawal to `Token proxy`, containing the same payload from `Everscale`.
2. `Token proxy` program calculates `Token Settings` PDA address, fetches it and gets `Vault` account address, `Round loader` program id,
   `SPL token` program id.
3. `Token proxy` program calculates `Relays round` account, using seed, derived from `Round loader` program id and
   relays round number from payload.
4. `Token Proxy` gets round relays info (public keys, addresses), checks that callers address is relay and the round is not expired.
5. If all is ok, `Token Proxy` program saves relays approval to withdrawal account.

##### WithdrawSol or WithdrawEver

1. If it is the new token, the `Token Initialization` procedure by `Token proxy` program occurs.
2. if there are enough confirms, `Token proxy` program checks that withdrawal amount is lower than limits from settings.
3. If amount is bigger, `Token proxy` program sets withdrawal status to `Waiting for approve`.
4. `Token proxy` fetches vault account.
5. If amount is lower, `Token proxy` program checks that `Vault` account contains more than `Withdrawal` amount.
6. If amount is bigger, `Token proxy` program sets withdrawal status to `Pending`.
7. If amount is lower, `Token proxy` program uses `SPL token` program address from settings and calls transfer tokens on `SPL token` program
8. `Token proxy` program changes withdrawal amount in current period.
9. `Token proxy` sets withdrawal status to `Processed`.

##### Approve over limit withdrawals algorithm

1. Admin calls approve pending withdrawal in `Solana` `Token Proxy` program.
2. `Token proxy` program calculates `Settings` PDA address and `Token Settings` PDA address, fetches it and gets `Vault` account address, `Withdrawal manager` account,
   `SPL token` program id.
3. `Token proxy` program checks that emergency mode is off.
4. `Token proxy` checks admin account with the received once.
5. `Token proxy` fetches withdrawal account.
6. `Token proxy` checks relays confirmations and status.
7. `Token proxy` fetches vault account.
8. If all is ok,`Token proxy` program checks that `Vault` account contains more than `Withdrawal` amount.
9. If amount is bigger, `Token proxy` program sets withdrawal status to `Pending`.
10. If amount is lower, `Token proxy` program uses `SPL token` program address from settings and calls transfer tokens on `SPL token` program.
11. `Token proxy` sets withdrawal status to `Processed`.

##### Force pending withdrawal algorithm

1. User calls force pending withdrawal in `Solana` `Token Proxy` program.
2. `Token proxy` program calculates `Settings` PDA address and `Token Settings` PDA address, fetches it and gets `Vault` account address,
   `SPL token` program id.
3. `Token proxy` fetches withdrawal account.
4. `Token proxy` checks relays confirmations, pending status.
5. `Token proxy` fetches vault account.
6. `Token proxy` checks vault balance.
7. If balance is enough, `Token proxy` program calls transfer tokens on `SPL token` program.
8. `SPL token` program decreases `Vault` account tokens and increases users tokens balance.
9. `Token proxy` program sets withdrawal status to `Processed`.

##### Cancel pending withdrawal algorithm

1. User calls cancel pending withdrawal in `Solana` `Token Proxy` program.
2. `Token proxy` program calculates `Settings` PDA address and `Token Settings` PDA address, fetches it.
3. `Token proxy` fetches withdrawal account.
4. `Token proxy` checks relays confirmations, pending status.
5. `Token proxy` creates `Deposit` [PDA](#deposit-account), containing mirrored data from withdrawal account, as like the user transferred
   funds in opposite direction.
6. `Token proxy` sets withdrawal status to `Cancelled`.

##### Change bounty for pending withdrawal algorithm

User can set bounty for proceeding his pending withdrawal, in order to other users have motivation to fill it and receive
bounty in `Everscale`.

1. User calls add or change bounty for pending withdrawal in `Solana` `Token Proxy` program, transferring new bounty size.
2. `Token proxy` program calculates `Settings` PDA address and `Token Settings` PDA address, fetches it.
3. `Token proxy` fetches withdrawal account.
4. `Token proxy` checks relays confirmations, pending status.
5. `Token proxy` checks that new bounty size is lower than amount in withdrawal.
6. `Token proxy` changes bounty size in withdrawal account.

##### Filling pending withdrawal algorithm

1. User calls fill pending withdrawal in `Solana` `Token Proxy` program, transferring payload.
2. `Token proxy` program calculates `Settings` PDA address and `Token Settings` PDA address, fetches it and gets `Vault` account address,
   `SPL token` program id.
3. `Token proxy` program fetches `Vault` account.
4. `Token proxy` fetches withdrawal account.
5. `Token proxy` checks relays confirmations, pending status for withdrawal.
6. `Token proxy` calculates deposit amount by user. It is withdrawal amount minus bounty.
7. `Token proxy` program calls transfer tokens (deposit amount) from user account to withdrawal receiver account on `SPL token` program.
8. `Token proxy` creates `Deposit` [PDA](#deposit-account), containing mirrored data from withdrawal account, as like the user transferred
   all funds in opposite direction.
9. `Token proxy` sets withdrawal status to `Processed`.

##### Execute Payload for Everscale tokens

1. User or relay calls execute payload for everscale tokens in `Solana` `Token Proxy` program.
2. `Token proxy` program calculates `Withdrawal` PDA address and `Token Settings` PDA address.
3. `Token proxy` fetches withdrawal account.
4. `Token proxy` checks relays confirmations, pending status for withdrawal.
5. `Token proxy` gets payload as instructions to be executed.
6. `Token proxy` program calls them, transferring proxy account signature.
7. `Token proxy` sets withdrawal status to `Processed`.

##### Execute Payload for Solana tokens

1. User or relay calls execute payload for solana tokens in `Solana` `Token Proxy` program.
2. `Token proxy` program calculates `Withdrawal` PDA address and `Token Settings` PDA address.
3. `Token proxy` fetches withdrawal account.
4. `Token proxy` checks relays confirmations, pending status for withdrawal.
5. `Token proxy` gets payload as instructions to be executed.
6. `Token proxy` program calls them, transferring proxy account signature.
7. `Token proxy` sets withdrawal status to `Processed`.

##### Withdraw from Proxy account

1. User calls execute payload for solana tokens in `Solana` `Token Proxy` program.
2. `Token proxy` program calculates `Proxy` PDA address deriving from users address and mint token address.
3. `Token proxy` program calls `SPL token` program to transfer tokens to users token account.

#### Admin methods

##### Change Guardian

1. Admin calls change guardian in `Solana` `Token Proxy` program.
2. `Token proxy` program calculates `Settings` PDA address, fetches it and gets `Guardian` account.
3. `Token proxy` checks that `Token Proxy` program data owner is the signer of the transaction.
4. If all is ok,`Token proxy` program changes `Guardian` account in `Settings` account.

##### Change Withdrawal manager

1. `Token proxy` program owner calls change `Withdrawal manager` in `Solana` `Token Proxy` program.
2. `Token proxy` program calculates `Settings` PDA address.
3. `Token proxy` checks that `Token Proxy` program data owner is the signer of the transaction.
4. If all is ok,`Token proxy` program changes `Withdrawal manager` account in the `Settings` account.

##### Change manager

1. `Token proxy` program owner calls change `manager` in `Solana` `Token Proxy` program.
2. `Token proxy` program calculates `Settings` PDA address.
3. `Token proxy` checks that `Token Proxy` program data owner is the signer of the transaction.
4. If all is ok,`Token proxy` program changes `manager` account in the `Settings` account.

##### Change Deposit limit

1. Admin calls change Deposit limit in `Solana` `Token Proxy` program.
2. `Token proxy` program calculates `Token Settings` PDA address, fetches it.
3. `Token proxy` checks that `Token Proxy` program data owner is the signer of the transaction.
4. If all is ok,`Token proxy` program changes deposit limit account in `Token Settings` account.

##### Change Withdrawal limit

1. Admin calls change Withdrawal limit in `Solana` `Token Proxy` program.
2. `Token proxy` program calculates `Token Settings` PDA address, fetches it.
3. `Token proxy` checks that `Token Proxy` program data owner is the signer of the transaction.
4. If all is ok,`Token proxy` program changes withdrawal limit account in `Token Settings` account.

##### Enable Emergency

1. Admin calls enable emergency in `Solana` `Token Proxy` program.
2. `Token proxy` program calculates `Settings` PDA address, fetches it and gets `Guardian` account.
3. `Token proxy` checks that `Token Proxy` program data owner or `Guardian` account is the signer of the transaction.
4. If all is ok,`Token proxy` program enables `Emergency` account in `Settings` account.

##### Disable Emergency

1. Admin calls disable emergency in `Solana` `Token Proxy` program.
2. `Token proxy` program calculates `Settings` PDA address, fetches it and gets `Guardian` account.
3. `Token proxy` checks that `Token Proxy` program data owner or `Guardian` account is the signer of the transaction.
4. If all is ok,`Token proxy` program disables `Emergency` account in `Settings` account.

##### Enable Token Emergency

1. Admin calls enable token emergency in `Solana` `Token Proxy` program.
2. `Token proxy` program calculates `Settings` PDA address and `Token Settings` PDA address, fetches it and gets `Guardian` account.
3. `Token proxy` checks that `Token Proxy` program data owner or `Guardian` account is the signer of the transaction.
4. If all is ok,`Token proxy` program enables `Emergency` account in `Token Settings` account.

##### Disable Token Emergency

1. Admin calls disable token emergency in `Solana` `Token Proxy` program.
2. `Token proxy` program calculates `Settings` PDA address and `Token Settings` PDA address, fetches it and gets `Guardian` account.
3. `Token proxy` checks that `Token Proxy` program data owner or `Guardian` account is the signer of the transaction.
4. If all is ok,`Token proxy` program disables `Emergency` account in `Token Settings` account.

##### Update Fee

1. Manager calls update fee of token in `Solana` `Token Proxy` program.
2. `Token proxy` program calculates `Settings` PDA address and `Token Settings` PDA address, fetches it and gets `Manager` account.
3. `Token proxy` checks that `Token Proxy` program data owner or `Manager` account is the signer of the transaction.
4. If all is ok,`Token proxy` program updates fees settings in `Token Settings` account.

##### Withdraw Ever Fee

1. Manager calls withdraw ever fee token in `Solana` `Token Proxy` program.
2. `Token proxy` program calculates `Settings` PDA address and `Token Settings` PDA address, fetches it and gets `Manager` account.
3. `Token proxy` checks that `Token Proxy` program data owner or `Manager` account is the signer of the transaction.
4. If all is ok,`Token proxy` program sends fees supply amount to managers' address and updates settings in `Token Settings` account.

##### Withdraw Solana Fee

1. Manager calls withdraw solana fee token in `Solana` `Token Proxy` program.
2. `Token proxy` program calculates `Settings` PDA address and `Token Settings` PDA address, fetches it and gets `Manager` account.
3. `Token proxy` checks that `Token Proxy` program data owner or `Manager` account is the signer of the transaction.
4. If all is ok,`Token proxy` program sends fees supply amount to managers' address and updates settings in `Token Settings` account.


#### Withdrawal account

`Withdrawal` account contains:
* Is initialized flag
* Account Kind: `Proposal`
* Is executed flag
* Author - proposal creator
* Relays round number - only relays from this round can approve this event
* Required votes to be processed
* Proposal Data for approve (PDA) :
    * Event timestamp - timestamp from `Everscale` blockchain transaction
    * Event transaction logical time - transaction logical time from `Everscale` blockchain transaction
    * Event configuration address - `Everscale` event configuration address, that created event
  
* Event:
    * Event length - Bytes count of followed data
    * Token root address in `Everscale`
    * Token name
    * Token symbol
    * Token decimals
    * Amount
    * Receiver address in `Solana`
    * Payload
* Meta:
    * Meta length - Bytes count of followed data
    * Bounty for withdrawal
    * Status: new, expired, processed, cancelled, pending, waiting for approve
      * `New` is needed to save relays confirms.
      * `Processed` - all funds were successfully transferred to user.
      * `Cancelled` - user asked to cancel withdrawal, his funds were minted in `Everscale` to his address back.
      * `Pending` - there is not enough funds on vault to process the withdrawal.
      * `Waiting for approve` - withdrawal amount is bigger than the limit
    * Epoch - timestamp to calculate daily limits correspondence
* Signers - array of relays votes


## Instructions

There are few instructions in the program:

* Vote For Withdrawal Request 
* Withdraw Ever Token
* Withdraw Solana Token
* Initialize 
* Deposit Ever Token
* Deposit Solana Token
* Withdrawal Ever Request
* Withdrawal Solana Request
* Approve Ever Token Withdrawal Request
* Approve Solana Token Withdrawal Request
* Cancel pending withdrawal
* Fill pending withdrawal
* Change bounty of pending withdrawal
* Update Fee
* Withdraw Ever Fee
* Withdraw Solana Fee
* Execute Payload for Ever tokens
* Execute Payload for Solana tokens
* Withdraw from Proxy account

## Upgrade

Deployer of `Token proxy` program can upgrade code via BPF loader at any time, using his keys pair. It would replace
old program with the new one, but address will remain the same. So deployer keys must be stored very caution. Maybe
the best here is to use multi-signature account.
