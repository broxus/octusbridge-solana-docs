# Design

`Token proxy` program is the main program in bridge from `Solana` side. It owns root token and vault accounts and can
make transfers, mints, burns tokens.

## Deploy

There is an issue with `Root token` account. It can not be made as PDA via `SPL token` program. So it must be deployed first,
then, after `Token proxy` program is deployed, the ownership of `Root token` account must be transferred to `Token proxy` program.
It is manual operation and one shouldn't forget about it!

Another important notice here is that `Token proxy` program must be deployed after `Round loader` program. This is needed
to save `Round loader` program id in `Token proxy` program `Settings` PDA.

## Start

In order to start working `Token proxy` must be initialized. Here is two options: to use vault and to use token root. It
depends on what kind of token this proxy needs to maintain. In case of `Everscale` token, it works with `Token root` account,
so its address must be stored in `Settings` account. In case of `Solana` token, it creates `Vault` account as PDA using 
token name as seed phrase along with program id.

### Settings account

It must store following:
* Token name
* Token Type: `Ever` or `Solana`
* Token address - In case of `Ever` it is `Token root` address, in case of `Solana` - `Vault` account address.
* Admin account - Account address that would allow approving big withdrawals
* `Round loader` program id
* `SPL token` program id

### Vault account

`Vault` account contains:
* Amount
* `Token root` account address
* Decimals count

### Initialization

1. Deployer of `Token proxy` program calls init method, providing settings data.
2. `Token proxy` program calculates `Settings` PDA address, checks that it is not created.
3. `Token proxy` program creates `Vault` PDA, if needed.
4. `Token proxy` program creates `Settings` PDA, storing data provided.

## Common work

Common work is divided into two big steps - deposit and withdraw. In both cases new PDA, storing all related info, is created. 
This allows relays to be always up-to-date with what is happening on `Solana` side.

### Deposit

This is the transfer from `Solana` to `Everscale`. 

#### `Everscale` tokens deposit

1. User calls deposit method of `Token proxy` program.
2. `Token proxy` program calculates `Settings` PDA address, fetches it and gets `Token root` account address.
3. `Token proxy` program fetches `Token root` account.
4. It uses `SPL token` program address from settings to call burn users tokens.
5. `Token proxy` program creates `Deposit` PDA.

#### `Solana` tokens deposit

1. User calls deposit method of `Token proxy` program.
2. `Token proxy` program calculates `Settings` PDA address, fetches it and gets `Vault` account address.
3. `Token proxy` program fetches `Vault` account.
4. It uses `SPL token` program address from settings to call transfer users tokens from users account to `Vault` account.
5. `Token proxy` program creates `Deposit` PDA.

#### Deposit account

`Deposit` account contains:
* Payload Id
* Sender address in `Solana`
* Receiver address in `Everscale`
* Amount
* `Token root` account address
* Decimals count in `Solana`

### Withdraw

This is the transfer from `Everscale` to `Solana`. 

#### `Everscale` tokens withdraw

It can be divided into 2 parts. At first user created `Withdraw` account, then relays confirm that it is a real withdrawal.

##### User withdraw creation

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
6. if there are enough confirms, `Token proxy` program checks that withdrawal amount is lower than limit from settings.
7. If amount is lower, `Token proxy` program uses `SPL token` program address from settings and calls mint tokens on `SPL token` program
and sets withdrawal state to `Processed`.
8. If amount is bigger, `Token proxy` program sets withdrawal state to `Waiting for approve`.

##### Approve over limit withdrawals algorithm

1. Admin calls approve pending withdraw in `Solana` `Token Proxy` program.
2. `Token proxy` program calculates `Settings` PDA address, fetches it and gets `Token root` account address, `Admin` account,
`SPL token` program id.
3. `Token proxy` checks admin account with the received once.
4. `Token proxy` fetches withdrawal account.
5. `Token proxy` checks relays confirmations and state.
6. If all is ok, `Token proxy` program uses `SPL token` program address from settings and calls mint tokens on `SPL token` program
and sets withdrawal state to `Processed`.

#### `Solana` tokens withdraw

It can be divided into 2 main parts. At first user created `Withdraw` account, then relays confirm that it is real withdrawal.
But in addition it must serve 5 more situations with vault.

##### User withdraw creation

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
5. If all is ok, `Token Proxy` program saves relays approval to withdrawal account and checks if there are enough confirms.
6. if there are enough confirms, `Token proxy` program checks that withdrawal amount is lower than limit from settings.
7. If amount is bigger, `Token proxy` program sets withdrawal state to `Waiting for approve`.
8. `Token proxy` fetches vault account.
9. If amount is lower, `Token proxy` program checks that `Vault` account contains more than `Withdrawal` amount.
10. If amount is bigger, `Token proxy` program sets withdrawal state to `Pending`.
11. If amount is lower, `Token proxy` program uses `SPL token` program address from settings and calls transfer tokens on `SPL token` program
    and sets withdrawal state to `Processed`.

##### Approve over limit withdrawals algorithm

1. Admin calls approve pending withdraw in `Solana` `Token Proxy` program.
2. `Token proxy` program calculates `Settings` PDA address, fetches it and gets `Vault` account address, `Admin` account,
   `SPL token` program id.
3. `Token proxy` checks admin account with the received once.
4. `Token proxy` fetches withdrawal account.
5. `Token proxy` checks relays confirmations and state.
6. `Token proxy` fetches vault account.
7. If all is ok,`Token proxy` program checks that `Vault` account contains more than `Withdrawal` amount.
8. If amount is bigger, `Token proxy` program sets withdrawal state to `Pending`.
9. If amount is lower, `Token proxy` program uses `SPL token` program address from settings and calls transfer tokens on `SPL token` program
   and sets withdrawal state to `Processed`.

##### Force pending withdrawal algorithm

1. User calls force pending withdraw in `Solana` `Token Proxy` program.
2. `Token proxy` program calculates `Settings` PDA address, fetches it and gets `Vault` account address,
   `SPL token` program id.
3. `Token proxy` fetches withdrawal account.
4. `Token proxy` checks relays confirmations, pending state.
5. `Token proxy` fetches vault account.
6. `Token proxy` checks vault balance.
7. If balance is enough, `Token proxy` program calls transfer tokens on `SPL token` program.
8. `SPL token` program decreases `Vault` account tokens and increases users tokens balance.
9. `Token proxy` program sets withdrawal state to `Processed`.

##### Cancel pending withdrawal algorithm

1. User calls cancel pending withdraw in `Solana` `Token Proxy` program.
2. `Token proxy` program calculates `Settings` PDA address, fetches it.
3. `Token proxy` fetches withdrawal account.
4. `Token proxy` checks relays confirmations, pending state.
5. `Token proxy` creates `Deposit` [PDA](#deposit-account), containing mirrored data from withdrawal account, as like the user transferred
   funds in opposite direction.
6. `Token proxy` sets withdrawal state to `Cancelled`.

##### Change bounty for pending withdrawal algorithm

User can set bounty for proceeding his pending withdrawal, in order to other users have motivation to fill it and receive
bounty in `Everscale`.

1. User calls add or change bounty for pending withdraw in `Solana` `Token Proxy` program, transferring new bounty size.
2. `Token proxy` program calculates `Settings` PDA address, fetches it.
3. `Token proxy` fetches withdrawal account.
4. `Token proxy` checks relays confirmations, pending state.
5. `Token proxy` checks that new bounty size is lower than amount in withdrawal.
6. `Token proxy` changes bounty size in withdrawal account.

##### Filling pending withdrawal algorithm

1. User calls fill pending withdraw in `Solana` `Token Proxy` program, transferring payload.
2. `Token proxy` program calculates `Settings` PDA address, fetches it and gets `Vault` account address,
   `SPL token` program id.
3. `Token proxy` program fetches `Vault` account.
4. `Token proxy` fetches withdrawal account.
5. `Token proxy` checks relays confirmations, pending state for withdrawal.
6. `Token proxy` calculates deposit amount by user. It is withdrawal amount minus bounty.
7. `Token proxy` program calls transfer tokens (deposit amount) from user account to withdrawal receiver account on `SPL token` program.
8. `Token proxy` creates `Deposit` [PDA](#deposit-account), containing mirrored data from withdrawal account, as like the user transferred
   all funds in opposite direction.
9. `Token proxy` sets withdrawal state to `Processed`.

#### Withdraw account

`Withdraw` account contains:
* Payload Id
* Relays round number
* Sender address in `Everscale`
* Receiver address in `Solana`
* Amount
* `Token root` account address in `Everscale`
* Decimals count in `Everscale`
* Decimals count in `Solana`
* Confirmed Relays
* Bounty for withdrawal
* Minimum Number of confirmation to be processed
* State: new, expired, processed, cancelled, pending, waiting for approve
    * `New` is needed to save relays confirms.
    * `Expired` - current round ttl is expired and withdrawal can not be processed.
    * `Processed` - all funds were successfully transferred to user.
    * `Cancelled` - user asked to cancel withdrawal, his funds were minted in `Everscale` to his address back.
    * `Pending` - there is not enough funds on vault to process the withdrawal.
    * `Waiting for approve` - withdraw amount is bigger than the limit

## Methods

There are nine methods in the program:

* Initialize
* Deposit
* Withdraw
* Confirm withdraw
* Approve pending withdraw
* Force pending withdraw
* Cancel pending withdraw
* Change bounty for pending withdraw
* Fill pending withdraw

## Upgrade

Deployer of `Token proxy` program can upgrade code via BPF loader at any time, using his keys pair. It would replace
old program with the new one, but address will remain the same. So deployer keys must be stored very caution. Maybe
the best here is to use multi-signature account.
