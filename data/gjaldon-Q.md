## [L-01] Validate rageQuitTimestamp in `PartyGovernanceNFT._initialize()`
https://github.com/code-423n4/2023-05-party/blob/main/contracts/party/PartyGovernanceNFT.sol#L101

`rageQuit()` is a valuable feature for Party members since it is a protective measure for them to be able to do an emergency withdrawal of their assets. Given that, it would be a good and sane default for `rageQuitTimestamp` to be initialized to some value in the future so that Party members are assured they can rageQuit in the early stages of the Party.

## [L-02] getDistributionShareOf() does not need to check that totalVotingPower == 0 in rageQuit()
https://github.com/code-423n4/2023-05-party/blob/main/contracts/party/PartyGovernanceNFT.sol#L151-L154

`getDistributionShareOf()` is only used in [`PartyGovernanceNFT.rageQuit()`](https://github.com/code-423n4/2023-05-party/blob/main/contracts/party/PartyGovernanceNFT.sol#L316). There is no point in rage-quitting while `totalVotingPower` is 0 because a user will not be able to withdraw their assets and the later `burn()` call will fail anyway. So `getDistributionShareOf()` should instead remove the check for `totalVotingPower == 0` and fail with a divisionByZero error. In that way, the function fails earlier and wastes less gas and does not rely on `burn()` reverting when totalVotingPower is 0.

## [L-03] Host that owns Party NFTs can circumvent reentrancy guard
https://github.com/code-423n4/2023-05-party/blob/main/contracts/party/PartyGovernanceNFT.sol#L309-L310

Since a host can `setRageQuitTimestamp`, the reentrancy guard in the above line can be easily circumvented by the host by setting `rageQuitTimestamp` before reentering `rageQuit()`. I suggest to either replace that with a traditional reentrancyGuard modifier or remove it and apply the CEI pattern.


## [L-04] transferEth() should not copy returnData
https://github.com/code-423n4/2023-05-party/blob/main/contracts/utils/LibAddress.sol#L112-L14

We only care about transferring ETH in the `transferEth()` function and we do not do anything to the returnData. This also prevents any reverts caused by lacking gas due to large return data causing memory expansion which can be the case if the address receiving ether is a contract with a `fallback()` function that returns data and no `receive()` function. [This assembly block](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/contracts/libraries/SafeCall.sol#L52-L62) can be used in place of the `receiver.call{}()` code. It works the same but does not assign memory to any returnData.

## [L-05] Host shouldn't be able to disable emergencyExecute
https://github.com/code-423n4/2023-05-party/blob/main/contracts/party/PartyGovernance.sol#L816-L819

Emergency execute is a protective measure so that the PartyDAO can execute any arbitrary function as the Party contract and move assets. This is useful for when the Party contract is in a bad state and assets are stuck. This gives Party members a chance to recover their stuck assets. Given this, allowing any Host to `disableEmergencyExecute()` seems like too much power for the role and requires that the Host role highly trusted. Furthermore, disabling emergency execute is permanent and irreversible. May be worth considering giving activeMembers the ability to disable and enable emergencyExecute.

## [R-01] Use VETO_VALUE for clarity
https://github.com/code-423n4/2023-05-party/blob/main/contracts/party/PartyGovernance.sol#L1024-L1027

Replace `type(uint96).max` in the above code with `VETO_VALUE` for clarity and easier refactoring in the future.