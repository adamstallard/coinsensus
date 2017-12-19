# Coinsensus

Coinsensus is a token that's regularly minted and distributed according to near-consensus of a voting group with a growing or fluctuating membership.

The address of the token contract can hold other tokens given to it as fees or donations. Holders of the main contract token may receive these as dividends.

## Summary

The token is distributed according to votes.  During each time period, white-listed voters send their vote to the smart contract.  The transfer is a small amount of ether--the actual vote is an address that holds a proposal.  The proposal address is another smart contract that's capable of sending the following to the original contract.
1. Addresses to be added to the voting white-list
1. Addresses to be removed from the voting white-list
1. Addresses to receive new tokens

Members decide near-unanimously who their membership is and who receives tokens each round.  Token-holders can also be paid semi-regularly with the ether (or other tokens) that accumulate in the original smart contract that's taken from voting fees or other fees or from donor deposits.

Near-consensus is required among those that vote each round, but there's no requirement of quorum.

## Storage
### `voters`
The contract maintains a list of voter addresses. Voters vote each round by submitting the address of a [proposal smart contract](#proposals).

### `balances`
The contract stores balances of tokens. Holding tokens is not limited to [voters](#voters).

### `recentVotes`
The contract stores votes for the current round--one or zero votes per [voter](#voters).

### `acceptedTokens`
The contract stores a list of tokens it will accept.  It will have to be familiar with the logic for sending these tokens (for instance ERC20).

### `minimumTokenPayout`
For each accepted token, there's a minimum amount that can be paid to an address as a [dividend](#dividends). This is to avoid generating excessive transactions for small balances.

### `mostVotesPerRound`
The highest number of votes received so far in a round. Used for computing [`dividendWhenAdded`](#dividendWhenAdded).

### Other Variables
[Other variables](#variables) affecting the operation of the contract are updated to match the variables set by a winning [proposal](#proposals) when the [proposal is run](#runproposal).

## Functions

### `Vote`
The caller must already be on the list of [voter addresses](#voters). This sets or updates the caller's [vote](#recentvotes) for the current round.

### `RunProposal`
This may be called once after a round is closed by the [proposal caller](#proposalcaller) set in the proposal that was selected with [near-consensus](#near-consensus). If there was no such proposal, calling this function has no effect.

## Proposals
Proposals are created as smart contracts. They hold data that serve as instructions to update the main contract. Proposals are the subject of [votes](#vote), and to be enacted, participating voters must choose a proposal with [near consensus](#near-consensus).

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
An array of addresses to add to [voters](#voters).

#### `removeVoters`
An array of addesses to remove from [voters](#voters).

#### `dividendWhenAdded`
All tokens (including ether) held by the contract address will be paid out proportionally to token holders if the number of votes in a round exceeds the previous highest number of votes by this value. The value can be negative. A value of `INT256_MIN` will cause the dividend to be paid no matter what, while `INT256_MAX` will prevent payment of the dividend.

#### `acceptToken`
The main contract will start [accepting this kind of token.](#acceptedtokens).  Each accepted token is a pair of <``tokenAddress`` , [``minimumTokenPayout``](#minimumtokenpayout)>. This variable is also used to update the [minimumTokenPayout](#minimumtokenpayout) for an already accepted token.

#### `rejectToken`
The main contract will stop [accepting this kind of token.](#acceptedtokens)

### Vote Fee Variables
### :warning:
The following three variables can easily block the main contract from operating (for example, by setting the [voteFee](#voteFee) very high or setting the [voteFeeToken](#voteFeeToken) to a token that's hard to obtain). As such, the same variable values must appear in successive proposals for a [predetermined number of rounds](#vote_fee_rounds) before they take effect. There's also [a constant to disable these variables completely](#enable_vote_fees).

#### `voteFee`
A fee required for each vote. This must be paid in addition to the transaction fee required by the network. If [`voteFeeGasPriceMultiple`](#votefeegaspricemultiple) is non-zero, it's used instead.

#### `voteFeeGasPriceMultiple`
The fee required for each vote as a multiple of the gas price set in the call to [`vote`](#vote). If this variable is non-zero, it's used instead of [`voteFee`](#votefee).

#### `voteFeeToken`
A token contract address (`0x0` for ether) used to pay the [vote fee](#votefee). This must be on the [list of accepted tokens](#acceptedtokens) for the main contract.

## Dividends

If the number of voters participating in a round meets the [`dividendWhenAdded`](#dividendwhenadded) constraint, holders of the token will be paid a dividend from a predetermined [fraction of the contract's holdings of other types of tokens](#dividend_fraction). Each holder is paid proportionally to the number of main contract tokens they hold. Those whose payment would fall below the [minimum token payout](#minimum-token-payout) for a particular token will be excluded from the payout for that token.

## Contributions

To contribute to the success of the mission of a particular instance of the token, participants may contribute [any accepted tokens](#accepted-tokens)

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
The ratio of votes that need to agree for a proposal to be ratified. It's possible that during a round, no proposal will be ratified.

### `ROUND_LENGTH_HOURS`
#### suggested value: `25`
How long a voting round lasts.

### `VOTE_FEE_ROUNDS`
#### suggested value: `4`
The number of successive matching proposals needed to [change vote fees](#votefeevariables).

### `ENABLE_VOTE_FEES`
#### suggested value: `true`
Whether to use [vote fees](#votefeevariables). These can be helpful in bootstrapping value, but must be used carefully.

### `DIVIDEND_FRACTION`
The fraction of the total holdings to be paid out when the [`dividendWhenAdded`](#dividendwhenadded) of a proposal is met. A value of `1` means that 100% of the holdings will be paid. A value of `.05` means that 5% of the holdings will be paid.
