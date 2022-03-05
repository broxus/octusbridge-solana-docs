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

### Approve withdrawal proposal

Contains following:
* Author - author of proposal public key, can be any custodian
* Payload id - withdrawal payload id
* Voters - all voted for custodians
* Is executed

### Create Multisig

1. Deployer of `Admin multisig` program calls create multisig method, providing custodians and name.
2. `Admin multisig` program calculates `Multisig` PDA address, checks that it is not created.
3. `Admin multisig` program creates `Multisig` PDA, storing data provided.

## Common work

Common work is following: create proposal, vote for proposal, approve proposal. It creates new PDA, storing all related info. 

### Create Proposal

1. Custodian calls create approve withdrawal proposal method of `Admin multisig` program.
2. `Admin multisig` program calculates `Multisig` PDA address, fetches it and gets custodians.
3. `Admin multisig` program checks that author is in custodian list.
4. `Admin multisig` program creates `Proposal` PDA.

#### Vote for proposal

1. Custodian calls vote for `Proposal` account in `Admin multisig` program.
2. `Admin multisig` program calculates `Proposal` PDA address, fetches it.
3. `Admin multisig` program checks that custodian votes for the first time and saves vote in the list.

#### Approve proposal

1. Custodian calls approve proposal in `Admin multisig` program.
2. `Admin multisig` program calculates `Proposal` PDA address, fetches it.
3. `Admin multisig` program checks that votes is enough (2 / 3 + 1) and the proposal was not executed before.
4. `Admin multisig` program calls `Token proxy` program `Approve` instruction via CPI (Cross-program invocation).
5. `Admin multisig` program sets that proposal is executed.

## Upgrade

Deployer of `Admin multisig` program can upgrade code via BPF loader at any time, using his keys pair. It would replace
old program with the new one, but address will remain the same. So deployer keys must be stored very caution. Maybe
the best here is to use multi-signature account.
