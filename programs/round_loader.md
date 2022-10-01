# Design

`Round loader` program is supposed to be the first one to deploy on `Solana` blockchain. The purpose of it is to save
relays public keys for each round in different `Solana` accounts and prevent attackers to modify them or add fake relays. 
This is done by two mechanics: 
* Usage of PDAs, derived from `Round loader` program id, round number.
* Usage of proposals algorithm - only relays majority approves new round. 

## Settings

In order to allow loading new round relays by current round relays program must use settings PDA, storing last
round number.
To allow only one initialization it can also store corresponding flag.

## Deploy

In order to start working `Round loader` must be initialized with one authorized public key, this could allow starting voting 
algorithm. During initialization phase round zero is created with this public key as a relay. This affords to create
proposal for current relay round and vote for it. Program authorization logic will allow only one authorized public 
key to do so. Next rounds will use standard algorithm of majority voting.

1. Deployer of `Round loader` program calls init method, providing authorized public key (it can be the same as deployers).
2. `Round loader` program calculates `Settings` PDA address, checks that it is not created.
3. `Round loader` program creates `Settings` PDA, storing current number az zero and setting initialization flag to true.
4. `Round loader` program creates zero round relays PDA, passing authorized public key as relay. TTL of this round can 
be set via optional parameter, but it must be enough to make proposal for new round relays (for example, it can be one day).
5. User with authorized public key calls create new proposal method in `Round loader` program.
6. `Round loader` program calculates `Settings` PDA address, fetches it and gets last round (zero for the first time). 
7. `Round loader` program gets last round relays. 
8. `Round loader` program checks searches callers key in relays list.
9. `Round loader` program creates proposal PDA, using seed formed by new round number, callers key, `Round loader` program id.
Proposal contains author key.
10. User with authorized public key calls load relays batch to the proposal method of `Round loader` program, providing round number.
11. `Round loader` program calculates proposal PDA address and fetches it.
12. `Round loader` program checks that callers public key is equal to author of proposal public key.
13. `Round loader` program adds relays batch to proposal.
14. Steps 10-13 are repeated until all relays from current round are loaded.
15. User with authorized public key calls finished loading relays batch to the proposal method of `Round loader` program, providing round number.
16. `Round loader` program calculates PDA address and fetches it.
17. `Round loader` program checks that callers public key is equal to author of proposal public key.
18. `Round loader` program sets flag is_loaded to true.
19. User with authorized public key calls vote for proposal method of `Round loader` program, providing round number.
20. `Round loader` program calculates `Settings` PDA address, fetches it and gets last round (zero for the first time).
21. `Round loader` program gets last round relays.
22. `Round loader` program searches callers key in relays list.
23. If caller is in relays list, `Round loader` program calculates proposal PDA address and fetches it.
24. `Round loader` program checks, that callers public address is not in the voters list (prevents one user to vote multiple times).
25. `Round loader` program checks that proposal is loaded.
26. `Round loader` program changes voters list, adding users public key.
27. `Round loader` program checks that number of votes is bigger than needed (for the first time one vote must be enough) and
is_accepted flag is false.
28. `Round loader` program creates new round PDA, using seed formed by new round number, `Round loader` program id and copying
relays from proposal.
29. `Round loader` sets `current round number` in `Settings` PDA.

## Common algorithm

Common algorithm is equal to steps 5-29 of deploy. Any relay from current round can create proposal, load new relays and vote.

## Methods

There are only few methods in the program:

* Vote for proposal
* Initialize
* Update Settings
* Create relay round
* Create proposal
* Write relays to proposal
* Finalize proposal
* Execute proposal
* Execute proposal by admin

## Accounts

### New round relays proposal account

New round relays proposal is an account containing following:
* Is initialized flag
* Account kind - `Proposal`
* Is executed flag
* Author - proposal creator
* Relays round number - only relays from this round can approve this event 
* Required votes to be processed
* Proposal Data for approve (PDA) :
    * Settings address - corresponding settings program address
    * Event timestamp - timestamp from `Everscale` blockchain transaction
    * Event transaction logical time - transaction logical time from `Everscale` blockchain transaction
    * Event configuration address - `Everscale` event configuration address, that created event
* Event:
    * New relays addresses (public keys)
    * Round end - timestamp
    * Round number - new round number
* Meta - empty
* Signers - array of signs (confirm or reject)

### New round relays account

New round relays is an account containing following:
* Is initialized flag
* Account Kind: `RelayRound`
* Round number
* Round end
* Relay addresses (public keys)

### Settings account

Settings account contains:
* Is initialized flag
* Account Kind: `Settings`
* Current round number
* Round submitter - Public key of admin, that can create new round manually or approve proposal
* Min required votes - The standard formula for required votes is `2 / 3 * relays round len + 1`, If it is lower than 
this parameter, it is used as required votes for event instead
* Round TTL - In order to have smooth round changes, this parameter is always added to relays round end timestamp. It allows
users, that created events in `Everscale` on border of round changes, to be able to complete transfer.

## Upgrade

Deployer of `Round loader` program can upgrade code via BPF loader at any time, using his keys pair. It would replace
old program with the new one, but address will remain the same. So deployer keys must be stored very caution. Maybe
the best here is to use multi-signature account.
