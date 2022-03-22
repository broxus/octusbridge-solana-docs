# Design

`Admin msig` program is the program, that affords to use multiple signers to approve over-limit withdrawals.

## Deploy

`Admin msig` program must be deployed before `Token proxy` program, because `Token proxy` uses admin account.

## Start

`Admin msig` program can create multiple multisig accounts for different `Token proxy` accounts.

### Multisig account

It must store following:
* Custodians - List of custodians public keys
* Name - multisig name, used for PDA creation
* Threshold - minimum number of confirmation  

### Transaction proposal

Contains following:
* Author - author of proposal public key, can be any custodian
* Transaction instruction - instruction to be executed after the approval
* Voters - all custodians, voted for 
* Is executed

### Create Multisig

1. Deployer of `Admin multisig` program calls create multisig method, providing custodians and name.
2. `Admin multisig` program calculates `Multisig` PDA address, checks that it is not created.
3. `Admin multisig` program creates `Multisig` PDA, storing data provided.

## Common work

Common work is following: create transaction, vote for transaction, approve transaction, execute transaction. 
It creates new PDA, storing all related info. 

### Create transaction

1. Custodian calls create approve withdrawal proposal method of `Admin multisig` program.
2. `Admin multisig` program calculates `Multisig` PDA address, fetches it and gets custodians.
3. `Admin multisig` program checks that author is in custodian list.
4. `Admin multisig` program creates `Transaction` PDA.

#### Vote for transaction

1. Custodian calls vote for `Transaction` account in `Admin multisig` program.
2. `Admin multisig` program calculates `Transaction` PDA address, fetches it.
3. `Admin multisig` program checks that custodian votes for the first time and saves vote in the list.

#### Approve transaction

1. Custodian calls approve proposal in `Admin multisig` program.
2. `Admin multisig` program calculates `Transaction` PDA address, fetches it.
3. `Admin multisig` program checks the proposal was not executed before and increases votes.

#### Execute transaction

1. Custodian calls execute proposal in `Admin multisig` program.
2. `Admin multisig` program calculates `Transaction` PDA address, fetches it.
3. `Admin multisig` program checks that votes count is equal or higher than threshold.
4. `Admin multisig` program calls instruction from transaction proposal.
5. `Admin multisig` program sets that proposal is executed.

## Upgrade

Deployer of `Admin multisig` program can upgrade code via BPF loader at any time, using his keys pair. It would replace
old program with the new one, but address will remain the same. So deployer keys must be stored very caution. Maybe
the best here is to use multi-signature account from this multisig program.

## Frontend

The research showed that currently there is no multisig frontend, that fits the requirements to call different instructions
and have ledger (hardware wallet) support. Using of [wallet adapter library](https://github.com/solana-labs/wallet-adapter)
one can create the frontend under requirements.
