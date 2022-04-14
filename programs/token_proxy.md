# Design

`Token proxy` program is the main program in bridge from `Solana` side. It owns root token and vault accounts and can
make transfers, mints, burns tokens.

## Deploy

`Root token` account and `Vault` account must be made as PDA via `SPL token` program.

Another important notice here is that `Token proxy` program must be deployed after `Round loader` program, because `Token proxy`
uses it to derive relays round account address.

## Start

In order to start working `Token proxy` must be initialized. Here is two options: to use vault and to use token root (`Mint` in `Solana` vocabulary). 
It depends on what kind of token this proxy needs to maintain. In case of `Everscale` token, it works with `Token root` account,
so its address must be stored in `Settings` account. In case of `Solana` token, it uses `Vault` account.

### Settings account

It must store following:
* Token name
* Token Type: `Ever` or `Solana`
* Token address - In case of `Ever` it is `Token root` account address, in case of `Solana` - `Vault` account address.
* Admin account - Account address that would allow approving big withdrawals.
* Withdrawals limit - Max auto withdrawals amount.
* Withdrawals period limit - Max auto withdrawals amount for current period.
* Withdrawals period - Withdrawal period used for limitation(day, week, month, etc.).
* Withdrawals current period amount - Amount of all withdrawals in current period.
* Withdrawals current period ttl - End of current period.
* Deposit limit - Max vault amount limit for `Solana` tokens deposit. In order not to lose a lot of funds in case of non-consistent work
the max deposit limit can be set.
* Decimals - Number of base 10 digits to the right of the decimal place.
* Emergency mode - The mode that denies all deposits and withdrawals.

### Vault account

`Vault` account contains is created via `InitializeAccount` instruction from `SPL token` program and contains:
* Amount

### Token root account

`Token root` is also named `Mint` account in `Solana`. It is created via `InitializeMint` instruction from `SPL token` program.

### Initialization

1. Deployer of `Token proxy` program calls init method, providing settings data.
2. `Token proxy` program calculates `Settings` PDA address, checks that it is not created.
3. `Token proxy` program creates `Settings` PDA, storing data provided.

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
2. `Token proxy` program calculates `Settings` PDA address, fetches it and gets `Vault` account address.
3. `Token proxy` program fetches `Vault` account.
4. `Token proxy` program checks that emergency mode is off.
5. `Token proxy` program checks that Vault amount plus deposit amount is not bigger than deposit limit.
6. It uses `SPL token` program address from settings to call transfer users tokens from users account to `Vault` account.
7. `Token proxy` program creates `Deposit` PDA.

#### Deposit account

`Deposit` account contains:
* Payload Id
* Sender address in `Solana`
* Receiver address in `Everscale`
* Amount
* `Token root` account address
* Decimals count in `Solana`
* Name - token ticker

### Withdrawal

Withdrawal is the tokens transfer from `Everscale` to `Solana`. 

#### `Everscale` tokens withdrawal

It can be divided into 2 parts. At first user created `Withdraw` account, then relays confirm that it is a real withdrawal.

##### User withdrawal request creation

1. User calls withdraw tokens from `Token root` account in `Solana` `Token Proxy` program, transferring payload.
2. `Token proxy` program calculates `Settings` PDA address, fetches it and gets `Token root` account address.
3. `Token proxy` creates `Withdraw` account.

##### Relays confirmation

1. Relays send confirm withdrawal to `Token proxy`, containing the same payload from `Everscale`.
2. `Token proxy` program calculates `Settings` PDA address, fetches it and gets `Token root` account address, `Relay loader` program id,
`SPL token` program id.
3. `Token proxy` program calculates `Relays round` account, using seed, derived from `Relay loader` program id and 
relays round number from payload.
4. `Token Proxy` gets round relays info (public keys, addresses), checks that callers address is relay and ttl of the round is not expired.
5. If all is ok, `Token Proxy` program saves relays approval to withdrawal account and checks if there are enough confirms.

##### Withdraw

1. if there are enough confirms, `Token proxy` program checks that withdrawal amount is lower than limits from settings.
2. If amount is lower:
   1. `Token proxy` program uses `SPL token` program address from settings and calls mint tokens on `SPL token` program.
   2. `Token proxy` program changes withdrawal amount in current period.
   3. `Token proxy` program sets withdrawal status to `Processed`.
3. If amount is bigger, `Token proxy` program sets withdrawal status to `Waiting for approve`.

##### Approve over limit withdrawals algorithm

1. Admin calls approve pending withdrawal in `Solana` `Token Proxy` program.
2. `Token proxy` program calculates `Settings` PDA address, fetches it and gets `Token root` account address, `Admin` account,
`SPL token` program id.
3. `Token proxy` program checks that emergency mode is off.
4. `Token proxy` checks admin account with the received once.
5. `Token proxy` fetches withdrawal account.
6. `Token proxy` checks relays confirmations and status.
7. If all is ok, `Token proxy` program uses `SPL token` program address from settings and calls mint tokens on `SPL token` program
8. `Token proxy` sets withdrawal status to `Processed`.

#### `Solana` tokens withdrawal

It can be divided into 2 main parts. At first user created `Withdraw` account, then relays confirm that it is real withdrawal.
But in addition it must serve 5 more situations with vault.

##### User withdrawal request creation

1. User calls withdraw tokens from `Vault` account in `Solana` `Token Proxy` program, transferring payload.
2. `Token proxy` program calculates `Settings` PDA address, fetches it and gets `Vault` account address.
3. `Token proxy` creates `Withdraw` account.

##### Relays confirmation

1. Relays send confirm withdrawal to `Token proxy`, containing the same payload from `Everscale`.
2. `Token proxy` program calculates `Settings` PDA address, fetches it and gets `Vault` account address, `Relay loader` program id,
   `SPL token` program id.
3. `Token proxy` program calculates `Relays round` account, using seed, derived from `Relay loader` program id and
   relays round number from payload.
4. `Token Proxy` gets round relays info (public keys, addresses), checks that callers address is relay and ttl of the round is not expired.
5. If all is ok, `Token Proxy` program saves relays approval to withdrawal account.

##### Withdraw

1. if there are enough confirms, `Token proxy` program checks that withdrawal amount is lower than limits from settings.
2. If amount is bigger, `Token proxy` program sets withdrawal status to `Waiting for approve`.
3. `Token proxy` fetches vault account.
4. If amount is lower, `Token proxy` program checks that `Vault` account contains more than `Withdrawal` amount.
5. If amount is bigger, `Token proxy` program sets withdrawal status to `Pending`.
6. If amount is lower, `Token proxy` program uses `SPL token` program address from settings and calls transfer tokens on `SPL token` program
7. `Token proxy` program changes withdrawal amount in current period.
8. `Token proxy` sets withdrawal status to `Processed`.

##### Approve over limit withdrawals algorithm

1. Admin calls approve pending withdrawal in `Solana` `Token Proxy` program.
2. `Token proxy` program calculates `Settings` PDA address, fetches it and gets `Vault` account address, `Admin` account,
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
2. `Token proxy` program calculates `Settings` PDA address, fetches it and gets `Vault` account address,
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
2. `Token proxy` program calculates `Settings` PDA address, fetches it.
3. `Token proxy` fetches withdrawal account.
4. `Token proxy` checks relays confirmations, pending status.
5. `Token proxy` creates `Deposit` [PDA](#deposit-account), containing mirrored data from withdrawal account, as like the user transferred
   funds in opposite direction.
6. `Token proxy` sets withdrawal status to `Cancelled`.

##### Change bounty for pending withdrawal algorithm

User can set bounty for proceeding his pending withdrawal, in order to other users have motivation to fill it and receive
bounty in `Everscale`.

1. User calls add or change bounty for pending withdrawal in `Solana` `Token Proxy` program, transferring new bounty size.
2. `Token proxy` program calculates `Settings` PDA address, fetches it.
3. `Token proxy` fetches withdrawal account.
4. `Token proxy` checks relays confirmations, pending status.
5. `Token proxy` checks that new bounty size is lower than amount in withdrawal.
6. `Token proxy` changes bounty size in withdrawal account.

##### Filling pending withdrawal algorithm

1. User calls fill pending withdrawal in `Solana` `Token Proxy` program, transferring payload.
2. `Token proxy` program calculates `Settings` PDA address, fetches it and gets `Vault` account address,
   `SPL token` program id.
3. `Token proxy` program fetches `Vault` account.
4. `Token proxy` fetches withdrawal account.
5. `Token proxy` checks relays confirmations, pending status for withdrawal.
6. `Token proxy` calculates deposit amount by user. It is withdrawal amount minus bounty.
7. `Token proxy` program calls transfer tokens (deposit amount) from user account to withdrawal receiver account on `SPL token` program.
8. `Token proxy` creates `Deposit` [PDA](#deposit-account), containing mirrored data from withdrawal account, as like the user transferred
   all funds in opposite direction.
9. `Token proxy` sets withdrawal status to `Processed`.

#### Admin methods

##### Change settings

1. Admin calls change settings in `Solana` `Token Proxy` program.
2. `Token proxy` program calculates `Settings` PDA address, fetches it and gets `Admin` account.
3. `Token proxy` checks admin account with the received once.
4. If all is ok,`Token proxy` program changes data (limits, emergency mode) in the `Settings` account.

##### Withdraw from `Vault` account

1. Admin calls withdraw from `Vault` account in `Solana` `Token Proxy` program.
2. `Token proxy` program calculates `Settings` PDA address, fetches it and gets `Vault` account address, `Admin` account,
   `SPL token` program id.
3. `Token proxy` checks admin account with the received once.
4. If all is ok, `Token proxy` program uses `SPL token` program address from settings and calls transfer tokens on `SPL token` program

#### Withdrawal account

`Withdrawal` account contains:
* Author
* Payload Id
* Relays round number
* Sender address in `Everscale`
* Receiver address in `Solana`
* Amount
* Decimals count in `Solana`
* Confirmed Relays
* Bounty for withdrawal
* Kind of withdrawal
* Minimum Number of confirmation to be processed
* Status: new, expired, processed, cancelled, pending, waiting for approve
    * `New` is needed to save relays confirms.
    * `Processed` - all funds were successfully transferred to user.
    * `Cancelled` - user asked to cancel withdrawal, his funds were minted in `Everscale` to his address back.
    * `Pending` - there is not enough funds on vault to process the withdrawal.
    * `Waiting for approve` - withdrawal amount is bigger than the limit

## Methods

There are few methods in the program for `Solana` tokens:

* Initialize 
* Deposit
* Withdrawal Request
* Confirm withdrawal
* Withdraw
* Force pending withdrawal
* Cancel pending withdrawal
* Change bounty for pending withdrawal
* Fill pending withdrawal

There are few methods in the program for `Ever` tokens:

* Initialize
* Deposit
* Withdrawal Request
* Confirm withdrawal
* Withdraw
* Force pending withdrawal

There are few methods in the program for admin:
* Approve pending withdrawal
* Change settings
* Withdraw from vault

## Upgrade

Deployer of `Token proxy` program can upgrade code via BPF loader at any time, using his keys pair. It would replace
old program with the new one, but address will remain the same. So deployer keys must be stored very caution. Maybe
the best here is to use multi-signature account.

## Multi-token design

To enable transferring all custom tokens created in `Everscale` the following three steps are provided:

* Global init instruction - Add super admin public key, that will be the default admin key for all of this kind of tokens.
* Specific Withdraw request instruction - it will save public key of Mint PDA, created after confirmation.
* Initialize `Everscale` token by confirmed withdrawal instruction - it will create Mint PDA by the name of token in withdrawal meta.

By default, such tokens will be created without any limits both for deposit and withdrawal. 

### Global init

Can be called only by program authority and will create `Global Settings` account

#### Global Settings

* Super admin - public key of the default admin for custom `Everscale` tokens 

### Specific withdraw request

It will take decimals and name not from `Settings` account, but by user input. If they will not be appropriate, relays will not
confirm such request.

### Initialize by confirmed withdrawal

Anyone can call this instruction in order to create `Mint` account and `Settings` account for this token.

1. User calls initialize by confirmed withdrawal in `Solana` `Token Proxy` program.
2. `Token proxy` program calculates `Global Settings` PDA address, fetches super admin public key.
2. `Token proxy` program calculates `Withdrawal` PDA address, checks that it is confirmed.
3. `Token proxy` program calls initialize `Mint` account via `SPL token` program with data, provided by withdrawal.
4. `Token proxy` program creates `Settings` PDA address, and fills it with data, provided by withdrawal.

After the initialization success, user can call `withdraw`, and he will receive funds from withdrawal request.
To be reminded! To receive funds, associated token account must be created before the mint process occurs. This must be done
manually on user side.
