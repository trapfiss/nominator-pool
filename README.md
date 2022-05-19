# Nominator pool smart contract

## Get-method `get_pool_data` 

Returns:

1. state - uint - current state of nominator pool. 0 - does not participate in validation, 1 - sent a `new_stake` request to participate in the validation round, 2 - received a successful confirmation about participation in the validation round.   
2. nominators_count - uint - current number of nominators in the pool.
3. stake_amount_sent - nanotons - with such a stake amount, the pool participates in the current round of validation.
4. validator_amount - nanotons - amount of coins owned by the validator.
5. validator_address - immutable - uint - validator wallet address. To get the address do `"-1:" + dec_to_hex(validator_address)`.
6. validator_reward_share - immutable - uint - what share of the reward from validation goes to the validator. `validator_reward = (reward * validator_reward_share) / 10000`.  For example set 4000 to get 40%.
7. max_nominators_count - immutable - uint - the maximum number of nominators in this pool.
8. min_validator_stake - immutable - nanotons - minimum stake for a validator in this pool.
9. min_nominator_stake - immutable - nanotons - minimum stake for a nominator in this pool.
10. nominators - Cell - raw dictionary with nominators.
11. withdraw_requests - Cell - raw dictionary with withdrawal requests from nominators.
12. stake_at - uint - ID of the validation round in which we are/are going to participate. Supposed start of next validation round (`utime_since`).  
13. saved_validator_set_hash - uint - technical information.
14. validator_set_changes_count - uint - technical information.
15. validator_set_change_time - uint - technical information.
16. stake_held_for - uint - technical information.
17. config_proposal_votings - Cell - raw dictionary with config proposals votings.

## Get-method `list_nominators`

Returns list of current pool's nominators.

Each entry contains:

1. address - uint - nominator wallet address. To get the address do `"0:" + dec_to_hex(address)`.
2. amount - nanotons - current active stake of the nominator.
3. pending_deposit_amount - nanotons - deposit amount that will be added to the nominator's active stake at the next round of validation.
4. withdraw_request - int - if `-1` then this nominator sent a request to withdraw all of his funds.

## Get-method `get_nominator_data`

It takes as an argument the address of the nominator and returns:

1. amount - nanotons - current active stake of the nominator.
2. pending_deposit_amount - nanotons - deposit amount that will be added to the nominator's active stake at the next round of validation.
3. withdraw_request - int - if `-1` then this nominator sent a request to withdraw all of his funds.

Throws an `86` error if there is no such nominator in the pool.

To get a nominator for example with an address `EQA0i8-CdGnF_DhUHHf92R1ONH6sIA9vLZ_WLcCIhfBBXwtG` you need to convert address to raw form `0:348bcf827469c5fc38541c77fdd91d4e347eac200f6f2d9fd62dc08885f0415f`, drop `0:` and invoke `get_nominator_data 0x348bcf827469c5fc38541c77fdd91d4e347eac200f6f2d9fd62dc08885f0415f`.

## Reward distribution

For each round of validation, the pool sends a stake to the elector smart contract.

After the completion of the validation round, the pool recover its funds from the elector.

Usually the amount received is greater than the amount sent, the difference is the validation reward.

The validator receives a share of the reward, according to the immutable pool parameter `validator_reward_share`.

```
validator_reward = (reward * validator_reward_share) / 10000;
nominators_reward = reward - validator_reward;
```

Nominators share the remaining reward according to the size of their stakes.

For example, if there are two nominators in the pool with stakes of 100k and 300k TON, then the first one will take 25% and the second 75% of the `nominators_reward`.

In case of a large validation fine, when the amount received is less than the amount sent, the loss is debited from the validator's funds. 

If the validator's funds are not enough, then the remaining loss will be deducted from the nominators in proportion to their stakes.

Note that the pool is designed in such a way that validator funds should always be enough to cover the maximum fine.

## Validator's responsibility

A pool can participate in validation only if the validator funds exceeds the immutable pool parameter `min_validator_stake`.  

Also, the validator's funds must exceed the maximum possible fine for bad validation. The recommended fine is calculated based on the network config.

Otherwise, the pool will not send requests to participate in the validation round.

## Nominator`s messages

To interact with the nominator pool, nominators send simple messages with a text comment (can be sent from any wallet application) to the pool smart contract.

**Messages must be sent in bounceable mode!**

In case of a typo or invalid message, the message will bounce back to the sender.

If you send a misspelled message or an invalid message in non-bounceable mode, you will lose coins.

## Nominator's deposit

In order for the nominator to make a deposit, he needs to send message to nominator-pool smart contract with Toncoins and text comment "d".

The nominator can only send message from a wallet located in the basechain (with raw address `0:...`).

The amount of Toncoins must be greater than or equal to `min_nominator_stake + 1 TON`.

1 TON upon deposit is deducted as a commission for deposit processing.

If the pool is not currently participating in validation (`state == 0`), then the deposit will be credited immediately.

If the pool is currently participating in the validation (`state != 0`), then the amount will be added to the `pending_deposit_amount` of the nominator, and will be credited after the completion of the current round of validation.

The nominator can subsequently send more Toncoins to increase his deposit.

Note that if the nominator-pool has already reached the number of nominators equal to the `max_nominators_count`, then deposits from new nominators will be rejected (they will bounce back to the sender).

## Nominator's withdrawal

In order for the nominator to make a withdrawal, he needs to send message to nominator-pool smart contract with text comment "w" and some Toncoins for network fee (1 TON is enough). Unspent TONs attached to message will be returned except in very rare cases.

If there are enough Toncoins on the balance of the nominator-pool, the withdrawal will be made immediately. All funds will be on the balance of the nominator-pool when it has completed participation in the validation round, but has not yet submitted a request for participation in a new round.

If there are not enough Toncoins on the nominator-pool balance, then a `withdraw_request` will be made for the nominator, and the Toncoins will be withdrawn after the end of the current validation round.

The nominator can only withdraw all of his funds at once. Partial withdrawal not supported.

## Emergency withdrawal

When operating normally, the validator must periodically send operational messages to the nominator pool, such as `process withdraw requests`, `update current validator set`, `new_stake`, `recover_stake`.

The validator software mytonctrl does this automatically.

In an emergency, for example if a validator goes missing and ceases to perform his duties, these operational messages can be sent by anyone and thus the nominators can withdraw their funds.

## Voting for network config proposals

In TON, network configuration changes occur by [voting of validators](https://ton.org/docs/#/smart-contracts/governance?id=proposalvoting-mechanism).

In the case of a nominator pool, it is make sense to all participants can vote, and the final result would be sent on behalf of the pool.

Thus, the nominator pool smart contract has a built-in functionality where the validator and nominators can indicate their vote for/against a specific proposal.

Based on such a vote, the validator sends the final vote to the network configuration smart contract through the validator software.

If the validator sent the final vote to the network configuration smart contract, and the vote does not match the opinion of the majority in the pool, in this case, the nominators can leave (and will leave) this pool and move to another one.

Since everything happens through transactions on-chain, such a mismatch will be stored on the blockchain and visible to everyone.

## Nominator`s vote

Each new network configuration change proposal is initially posted to the TON Foundation channels [@tonblockchain](https://t.me/tonblockchain) or [@tonstatus](https://t.me/tonstatus).

In this post, in addition to the description of the proposal, the proposal's hash will be indicated in HEX form, e.g `D855FFBCF813E50E10BEAB902D1177529CE79785CAE913EB96A72AE8EFBCBF47`.

In order for the nominator to vote for the proposal, he needs to send a message to the nominator pool smart contract with a text comment `y<HASH>`, e.g. `yD855FFBCF813E50E10BEAB902D1177529CE79785CAE913EB96A72AE8EFBCBF47`.

In order for the nominator to vote against the proposal, he needs to send a message to the nominator pool smart contract with a text comment `n<HASH>`, e.g. `nD855FFBCF813E50E10BEAB902D1177529CE79785CAE913EB96A72AE8EFBCBF47`.

Some amount of tokens must be attached to this message in order to pay the network fee (1 TON is enough). Unspent TONs attached to message will be returned.

Votes are stored in the pool contract for 30 days.

Only the validator and current nominators who have an active stake in the pool can vote.

## Get-method `list_votes`

Returns list of votes.

Each entry contains:

1. proposal_hash - uint - hash of proposal. Use `dec_to_hex(proposal_hash)` to convert hash to HEX form.
2. votes_create_time - uint - the unixtime this vote was created.

## Get-method `list_voters`

It takes as an argument the hash of the proposal and returns list of voters:

Each entry contains:

1. address - voter's address. To get the nominator address do `"0:" + dec_to_hex(address)`, if `address = validator_address` do `"-1:" + dec_to_hex(address)`. 
2. support - int - if `-1` then it is a "vote for", otherwise it is a "vote against".
3. vote_time - uint - unixtime when he voted.

Voting results are calculated off-chain.