# MerkleMine Specification

The MerkleMine algorithm is a distribution mechanic for allocating token at the genesis state of the network across a large number of participants.

The distribution algorithm consists of:

* A Genesis State
* A `generate()` function

## Genesis State

A genesis state `G` is consists of:

* `genesisRoot` - the Merkle root of the genesis state
* `balanceThreshold` - the minimum ETH balance for an account's address to be included in the genesis state
* `totalGenesisTokens` - the amount of token to be distributed for the duration of the generation period
* `totalGenesisRecipients` - the # of account addresses used in the construction of the genesis state
* `genesisBlock` - the Ethereum block number at which a state dump of accounts is used to construct the genesis state
* `callerAllocationStartBlock` - the block at which a 3rd party caller can generate and split a token allocation with a recipient
* `callerAllocationEndBlock` - the block at which a 3rd party caller can generate the full token allocation of the original intended recipient

### Algorithm for generating a token allocation

*This is done offline and is repeatable by anyone to validate the `genesisRoot` and construct their own proofs to be used for token allocation generation*

1. Observe the Ethereum account state database at `genesisBlock`.
2. For each account:
    1. If `getBalance(account) > balanceThreshold && len(getCode(account)) == 0` then add the account to `candidateAccounts` array. `// Only user controlled acounts with balance greater than balanceThreshold.`
3. Sort the `candidateAccounts` array in ascending order of uint value of the account hex address.
4. Use sorted `candidateAccounts` as ordered leaf nodes of Merkle tree construction. The hashing algorithm used for the Merkle tree construction is keccak256. Each leaf node of the Merkle tree is the keccak256 hash of the hex encoded byte representation of a particular address string. Let the root of the Merkle tree be `localRoot`.
5. Validate that `localRoot == genesisRoot`.
6. Validate that `len(candidateAccounts) == totalGenesisRecipients`.
7. Validate that `token.balanceOf(merkleMine) >= tokenAllocationAmount`.
8. Validate that the token allocation for the account has not been generated.
9. Invoke `generate()` with the Merkle proof of an account's inclusion in `genesisRoot`.

## generate() function

The `generate()` function has the following signature:

```
function generate(address _recipient, bytes _merkleProof)
```

A user who wishes to generate token passes in the address of the recipient (it may be themselves), and the Merkle proof that verifies the recipient is in the genesis state as specified by the `genesisRoot` genesis parameter.

**Validations**

The method should `throw` and revert state if the following validations do not pass:

* Tokens were not already generated for this `_recipient`
* The `_merkleProof` provided validates against the `genesisRoot`
* If the txn sender (the "caller") does not equal the `_recipient`, then make sure `block.number` is greater than or equal to `callerAllocationStartBlock`

**Generation**

Let `tokensPerAllocation` = `totalGenesisTokens` / `totalGenesisRecipients`. The starting point is that all recipients get an even distribution of token.

* If the caller is the `_recipient` then transfer `tokensPerAllocation` to the caller.
* If the caller is not the `_recipient`:
    * Calculate the caller portion using the `callerTokenAmountAtBlock` curve.
    * Calculate the recipient portion by subtracting `callerTokenAmountAtBlock` from `tokensPerAllocation`.
    * Send the caller portion to the caller and the recipient portion to the `_recipient`.
* Record state that this recipient's tokens have been generated, so that tokens can no longer be generated with this same `_recipient` argument and Merkle proof again.


### callerTokenAmountAtBlock curve

This curve indicates how much LPT is claimable by the txn sender (caller) versus how much gets distributed to the `_recipient`. This amount grows linearly with how many blocks have passed between `callerAllocationStartBlock` and `callerAllocationEndBlock`.

* If `block.number` > `callerAllocationEndBlock` then return the full `tokensPerAllocation`.
* Otherwise calculate how far proportionally you are into the period between `callerAllocationStartBlock` and `callerAllocationEndBlock`. Return the `tokensPerAllocation` multiplied by this proportional percentage.

For example if tokensPerAllocation is 10, and you are 50% into the period, return 5.

## End result

The end result of all of this is that there is a wide generation of token, potentially to all accounts encoded into the genesis state. But there are incentives for others to perform the generation on behalf of many of these accounts through the growing `callerTokenAmountAtBlock` value.

* Participation is open to all, whether you had enough ETH to be in the genesis set or not
* There is a period of time for recipients to learn about the process such that they can generate their full allocation
* The caller allocation period allows active participants to generate more tokens than passive recipients while still allowing for the possibility for passive participants
to end up with some amount of token (depending on when active participants generate allocations on behalf of recipients) that they can use to start
interacting with the protocol later