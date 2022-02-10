# Proposal #1

## Motivation

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
2. `Round loader` program calculates `Settings` PDA address, checks that it is not created.
3. `Round loader` program creates `Settings` PDA, storing data provided.


## Common work

Common work is divided into two big steps - deposit and withdraw. In both cases new PDA, storing all related info, is created. 
This allows relays to be always up to time with what is happening on `Solana` side.

### Deposit

This is transfer from `Solana` to `Everscale`. 

#### `Everscale` tokens deposit

1. User calls deposit method of `Token proxy` program.
2. `Token proxy` program calculates `Settings` PDA address, fetches it and gets `Token root` account address.
3. `Token proxy` program fetches info from `Token root` account.
4. It uses `SPL token` program address from settings to call burn users tokens.
5. `Token proxy` program creates `Deposit` PDA.

#### `Solana` tokens deposit

1. User calls deposit method of `Token proxy` program.
2. `Token proxy` program calculates `Settings` PDA address, fetches it and gets `Vault` account address.
3. `Token proxy` program fetches info from `Vault` account.
4. It uses `SPL token` program address from settings to call transfer users tokens from users account to `Vault` account.
5.`Token proxy` program creates `Deposit` PDA.

#### Deposit account

`Deposit` account contains:
* Payload Id
* Sender address in `Solana`
* Receiver address in `Everscale`
* Amount
* `Token root` account address
* Decimals count

### Withdraw

This is transfer from `Everscale` to `Solana`. 

#### `Everscale` tokens withdraw

It can be divided into 2 parts. At first user created `Withdraw` account, then relays confirm that it is real withdrawal.

##### User withdraw creation

1. User calls withdraw tokens from vault in `Solana` `Token Proxy` program, transferring payload.
2. `Token proxy` creates unique withdraw account with payload from event.
3. Relays get callback from `Token proxy` program about new withdrawal.

##### Relays confirmation

1. Relays send confirm withdrawal to `Token proxy`, containing payload from `Everscale`.
2. `Token Proxy` gets requested round relays info (public keys, addresses), checks that callers address is relay and round is not expired.
3. If all is ok, `Token Proxy` program saves relays approval to withdrawal account and checks if there are enough confirms.
4. if there are enough confirms, `Token proxy` program calls mint tokens on `SPL token` program.
5. `SPL token` program mints tokens and increases users balance.


#### `Solana` tokens withdraw

It can be divided into 2 main parts. At first user created `Withdraw` account, then relays confirm that it is real withdrawal.
But in addition it must serve 5 more situations with vault.

##### User withdraw creation

1. User calls withdraw tokens from vault in `Solana` `Token Proxy` program, transferring payload.
2. `Token proxy` creates unique withdraw account with payload from event.
3. Relays get callback from `Token proxy` program about new withdrawal.

##### Relays confirmation

1. Relays send confirm withdrawal to `Token proxy`, containing payload from `Everscale`.
2. `Token Proxy` gets requested round relays info (public keys, addresses), checks that callers address is relay and round is not expired.
3. If all is ok, `Token Proxy` program saves relays approval to withdrawal account and checks if there are enough confirms.
4. if there are enough confirms, `Token proxy` program calls transfer tokens on `SPL token` program.
5. `SPL token` program transfer tokens from `Vault` account and increases users balance.

#### Withdraw account

`Withdraw` account contains:
* Payload Id
* Sender address in `Everscale`
* Receiver address in `Solana`
* Amount
* `Token root` account address in `Everscale`
* Decimals count

## Methods

There are ten methods in the program:

* Initialize
...
