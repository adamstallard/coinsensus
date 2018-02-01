# Coinsensus

## Purpose

Each instance of coinsensus is a fund that's managed by a voting group.  The instance has its own token which entitles its holders to receive dividends from a vault of contributed tokens of other types.  The instance token is meant to circulate and have value in anticipation of future dividends.

## Use Cases

The token can be used to 

1.  Fund organizations or projects
2.  Reward processes that happen "off-chain", such as research, validation, or mining.

The voting group can consist of people, entities, or programs.  The group structure is intended to provide checks and avoid single points of failure.  The voting group should be able to agree with near-consensus.

The vault can represent a combination of fees, payments, and contributions.

In the particular case of BrightID, the token is used to reward nodes for running anti-sybil software; the voting group consists of the nodes themselves.  The funding comes from partners that are interested in using BrightID to limit duplicate and fake accounts.

## Operation

The voting group votes on their own membership, instance token recipients, and what triggers a dividend payment.

During each time period, white-listed voters send their vote to the smart contract.  The transfer is a small amount of ether--the actual vote is for an address that holds a proposal.  The proposal address is another smart contract that contains:
1. Addresses to be added to the voting white-list
1. Addresses to be removed from the voting white-list
1. Addresses to receive new tokens

and [a few other variables](#variables).

There are also [constants](#constants) that are set at the time the instance is created to enforce bounds for the variables in proposals.

[Near-consensus](#near_consensus) is required among those that vote each round, but there's no requirement of quorum.

## Storage
### `voters`
The contract maintains a list of voter addresses. Voters vote each round by submitting the address of a [proposal contract](#proposals).

### `balance`
The contract stores balances of tokens. Holding tokens is not limited to [voters](#voters).

### `recentVotes`
The contract stores votes for the current round--one or zero votes per [voter](#voters).

### `acceptedTokens`
The contract stores a list of tokens it will accept.  It has to be familiar with the logic for sending these tokens (for instance ERC20, ERC223).

### `mostVotesPerRound`
The highest number of votes received so far in a round. Used for computing [`dividendWhenAdded`](#dividendwhenadded).

### `dividendRatio`
For each type of accepted token, `dividendRatio` represents the number of tokens previously made available as dividends to the total supply of the instance token.

For example, if 10 XYZ tokens are made available to be claimed as dividends, and the total supply of the instance token is 1000, then the dividend ratio for XYZ tokens would be .01.  If the next dividend event makes 20 more XYZ tokens available and the total supply of the instance token at that time were 4000, then the dividend ratio would increase by .005, i.e. it would increase from .01 to .015.

`DividendRatio`s don't decrease when dividends are claimed--they represent all the dividends that were ever made available.

### `totalSupply`
Total supply of the instance token.

### `owed`
When an account's balance changes or [dividends are claimed](#claim), the values of `owed` for each type of token are incremented for that account by the current [`dividendRatio`](#dividendratio) minus the account's [`lastRatio`](#lastratio) value for each token multiplied by the account balance. I.e. `(DR - LR) * b`.  Sent tokens are included the sender's balance (not the receivers), and newly minted tokens aren't included in the balance in this calculation.

### `lastRatio`
After an account's [`owed`](#owed) values have been updated due to a balance change, the account's [`lastRatio`](#lastratio) value for each type of token is set to the current [`dividendRatio`](#dividendratio) for that token.

### Other Variables
[Other variables](#variables) affecting the operation of the contract are updated to match the variables set by a winning [proposal](#proposals) when the [proposal is run](#runproposal).

## Functions

### `Vote`
The caller must already be on the list of [voter addresses](#voters). This sets or updates the caller's [vote](#recentvotes) for the current round.

### `RunProposal`
This may be called once after a round is closed by the [proposal caller](#proposalcaller) set in the proposal that was selected with [near-consensus](#near-consensus). If there was no such proposal, calling this function has no effect.

### `Send`
Send instance tokens to another account.  [`balance`](#balance), [`lastRatio`](#lastratio), and [`owed`](#owed) are updated for both accounts.

### `Claim`
Claim all dividends of the specified token type.

## Proposals
Proposals are created as contracts. They hold data that serve as instructions to update the main contract. Proposals are the subject of [votes](#vote), and to be enacted, participating voters must choose a proposal with [near consensus](#near-consensus). The exception to this is [adding new voters](#addvoters), which can be done with a [simple majority](#add_voters_majority)

Proposals are executed with the [`RunProposal`](#runproposal) function of the main contract. Because this can be expensive, it's expected that a proposal will include the [calling address](#proposalcaller) in the [recipients](#recipients) with a fair ratio of new coins.

### Variables
The following variables can be included in a proposal and if the [proposal is run](#runproposal), they will be used as parameters in the current run or to change the corresponding storage on the main contract.

#### `newTokenRatio`
This is the number of new tokens to mint as a ratio of the amount of existing tokens, rounded up to the nearest whole number. A value of `.01` means that if there are 100 tokens in existence, 1 token will be minted next round. With 101 tokens in existence, and a value of `.01`, 2 new tokens would be minted.

The value for `newTokenRatio` is constrained by [`MAX_NEW_TOKEN_RATIO`](#max_new_token_ratio).

#### `recipients`
An map of addresses to ratios. Each address will receive that ratio of the coins minted this round. The ratios must add up to one.

#### `proposalCaller`
The address that's authorized to call [RunProposal](#runproposal) on behalf of this proposal.

#### `addVoters`
An array of addresses to add to [voters](#voters).  This part of a proposal can be executed by itself if [Add_Voters_Marjority](#add_voters_majority) is reached, while [Near_Consensus](#near_consensus) isn't.

#### `removeVoters`
An array of addesses to remove from [voters](#voters).

#### `dividendWhenAdded`
More dividends are added if the number of votes in a round exceeds the previous highest number of votes by this value. The value can be negative. A value of `INT256_MIN` will always cause more dividends to be added, while `INT256_MAX` will always prevent more dividends from being added.

Dividends are added for each token held by the contract, by increasing the [`dividendRatio`](#dividendratio) values for each token type.

#### `acceptToken`
The main contract will start [accepting this kind of token.](#acceptedtokens).

#### `rejectToken`
The main contract will stop [accepting this kind of token.](#acceptedtokens)

## Constants
Each instance of the token contract is initialized with the following constants:

### `MAX_VOTERS_ADD_RATIO`
#### suggested value: `.03`

A value of `.01` means that if there are 100 current voters, a maximum of 1 can be added in the next round.  `currentVoters * MAX_VOTERS_ADD_RATIO` is rounded up; with 101 current voters and a value of `.01`, 2 voters could be added.

### `MAX_VOTERS_REMOVE_RATIO`
#### suggested value: `.01`

Operates similarly to [MAX_VOTERS_ADD_RATIO](#max-voters-add-ratio), but for removing voters.

### `MAX_NEW_TOKEN_RATIO`
#### suggested value: `.03`
This limits the [`newTokenRatio`](#newtokenratio) variable set by proposal contracts.

### `NEAR_CONSENSUS`
#### suggested value: `.9`
The ratio of votes that need to agree for a proposal to be enacted. It's possible that during a round, no proposal will be enacted.

### `ADD_VOTERS_MAJORITY`
#### suggested value: `.51`
The ratio of votes a proposal needs for the [addVoters](#addvoters) portion of a proposal to be enacted. Having this value distinct from [NEAR_CONSENSUS](#NEAR_CONSENSUS) allows voters to unblock proposals by adding more voters.

### `ROUND_LENGTH_HOURS`
#### suggested value: `25`
How long a voting round lasts.

### `DIVIDEND_FRACTION`
The fraction of the total holdings of tokens to be [converted to dividends](#dividendratio) when the [`dividendWhenAdded`](#dividendwhenadded) constraint of a proposal is met. A value of `1` means that 100% of the holdings will be paid. A value of `.05` means that 5% of the holdings will be paid.
