# Fuse Network FIP11 Validator Jailing

This is a brief commentary on how to design and implement the FIP11 of Fuse Network for validator Jailing problem. 
https://github.com/fuseio/FIPs/blob/master/FIPS/fip-11.md

I spent sometime looking into the Fuse-Network contracts in the context of FIP11 (validator jailing). I have some recommendations on how to approach this feature.

This current FIP11 scope is written to include only Liveness slashing as a form of strikes and then followed by jail time. There are other bigger validator misbehaviors such as  consensus safety fault, double signing or submitting incorrect proposals, spamming, DOS, etc. In my opinion  consensus safety fault should trigger permanent jail time; where as Validator double signing is far more serious than liveness infraction.

Overall, it is better to design a Slashing module that governs the different penalties for different kinds of validator misbehaviors or infractions or slashing types. This can be designed through current Governance mechanism so that the community can participate on the quality and quantity of the parameters that go into Slashing module for validator infractions.

We should also consider a signal of routine validator maintenance as exception where such slashing is skipped.

## Design & Implementation Suggestions
It is better to bring in validator state and validator signing info abstractions to keep track of the validator activity on its own. The current smart contracts iteratively loop through the main storage arrays every time you need some info. When the network grows, this will be a huge bottleneck. Instead maintain two separate data strcutures codifying Validator current state and validator signing info for each block/cycle/validator.

In practice, there is a time delay between between an infraction occurring, and evidence of the infraction reaching the state machine. This is why you need a unbonding period.

### Validator State:
A validator can have one of three states:
Active - Validator is in the active validator set and participates in consensus. The validator is earning rewards and can be slashed for misbehavior.
Inactive - Validator is not in the active set, and therefore not signing blocks. The validator cannot be slashed and does not earn any reward. It is still possible to delegate Fuse tokens to an unbonded validator. Undelegating from an unbonded validator is immediate, meaning that the tokens are not subject to the unbonding period.
Jailed - Validator misbehaved and is in jail, i.e. outside of the validator set.

### ValidatorSigningInfo:
We can start with something like Cosmos SDK and iterate on what parameters best suite the Fuse Network and the community.

#### Parameterization of ValidatorSigningInfo:
SignedBlocksWindow - Window for being offline without being slashed, in blocks.
MinSignedPerWindow - Minimum proportion of blocks signed per window without being slashed.
DowntimeJailDuration - The suspension time (aka jail time) for a validator that is offline too long.
SlashFractionDoubleSign - Proportion of stake-backing that is burned for equivocation (aka double-signing).
SlashFractionDowntime - Proportion of stake that is slashed for being offline too long.

At the beginning of each block, the ValidatorSigningInfo for each validator is updated and whether they’ve crossed below the liveness threshold over a sliding window is checked. This sliding window is defined by SignedBlocksWindow, and the index in this window is determined by IndexOffset found in the validator’s ValidatorSigningInfo. For each block processed, the IndexOffset is incremented regardless of whether the validator signed. After the index is determined, the MissedBlocksBitArray and MissedBlocksCounter are updated accordingly.

Finally, to determine whether a validator crosses below the liveness threshold, the maximum number of blocks missed, maxMissed, which is SignedBlocksWindow - (MinSignedPerWindow * SignedBlocksWindow), and the minimum height at which liveness can be determined, minHeight, are fetched. If the current block is greater than minHeight and the validator’s MissedBlocksCounter is greater than maxMissed, they are slashed by SlashFractionDowntime, jailed for DowntimeJailDuration, and have the following values reset: MissedBlocksBitArray, MissedBlocksCounter, and IndexOffset.

The way I will design this will be using Handler Manager for Validator Signature Info. This handles validator signature, must be called once per validator per block.

In later versions, you can also consider proportional slashing: the larger a validator is, the more they should be slashed. One way to achieve this is using a measure of voting power a given validator has.

slash_amount = const_k * validator_voting_power
Here const_k is some predefined on-chain constant. It can be driven through current governance process by community involvement.

But, from a design perspective, this may encourage big validators to split (sybil attack) into multiple accounts to minimize the slashing penalty.

You can solve this by accounting for the voting percentage of all the other validators who get slashed in a specified time frame.

Now, if someone splits a validator of 20% into four validators of 5% each which all fault, then they fault in the same time frame, they will get slashed at the sum 20% amount.

But, this is linear. We typically don't want this for more serious infractions like double signature. You can better than the linear by fitting a logistic function or a S-curve. The logistic function based slashing will allow the slashing factor to be minimal for small values, and then grow very rapidly near some threshold point where the risk posed becomes notable.

There is another edge case here; it is called griefing, the act of intentionally getting oneself slashed in order to make another's slash worse, could be a concern here. In a logistic function based slashing, they both should get impacted equally leaving no extra economic incentives for the victim.

##### Positives
    Increases decentralization by disincentivizing delegating to large validators
    More severely punishes attacks than accidental faults
    More flexibility in slashing rates parameterization
##### Negatives
    More computationally expensive than current implementation. Will require more data about "recent slashing events" to be stored on chain.

Here is some comparisons of variety of slashing across different blockchain networks. Note that both Ethereum and Polka have a more complicated and dynamic slashing method because they designed their mechanisms to punish the validators if more than 33% of the network is inactive. The majority of the PoS protocols want two third of the total validators to be active and honest during each block addition. 

