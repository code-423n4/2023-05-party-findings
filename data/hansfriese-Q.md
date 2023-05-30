### [L-01] DOS: Users can't call `accept()` if others called `rageQuit()` within the same block.

- https://github.com/code-423n4/2023-05-party/blob/f6f80dde81d86e397ba4f3dedb561e23d58ec884/contracts/party/PartyGovernance.sol#L596

```solidity
    if (lastBurnTimestamp == block.timestamp) {
        revert CannotRageQuitAndAcceptError();
    }
```

While proposing a new proposal, it should use `totalVotingPower` at `block.timestamp - 1` but it uses the current total voting power.

```solidity
    function propose(
        Proposal memory proposal,
        uint256 latestSnapIndex
    ) external onlyActiveMember onlyDelegateCall returns (uint256 proposalId) {
        proposalId = ++lastProposalId;
        // Store the time the proposal was created and the proposal hash.
        (
            _proposalStateByProposalId[proposalId].values,
            _proposalStateByProposalId[proposalId].hash
        ) = (
            ProposalStateValues({
                proposedTime: uint40(block.timestamp),
                passedTime: 0,
                executedTime: 0,
                completedTime: 0,
                votes: 0,
                totalVotingPower: _governanceValues.totalVotingPower //@audit should be at block.timestamp - 1
            }),
            getProposalHash(proposal)
        );
        emit Proposed(proposalId, msg.sender, proposal);
        accept(proposalId, latestSnapIndex);
    }
```

It will be problematic if `burn()` and `propose()` are called within the same block so the proposal has the decreased total voting power and each user can vote using the voting power at `proposedTime - 1`.(So undcreased voting power)

To resolve that, `accept()` checks the `lastBurnTimestamp` condition so `burn()` and `accept()` can't be called within the same block. (Then `burn()` and `propose()` can't be called together as well because `propose()` calls `accept()` inside the function.)

But our original concern is to prevent `burn()` and `propose()` are called together and currently it checks the stricter requirement.

As a result, voters can't call `accept()` if others call `rageQuit()` within the same block even if it should work properly.

As a mitigation, `propose()` should check the `lastBurnTimestamp` condition, and `accept()` shouldn't check.


### [L-02] A reentrancy attack is possible in `rageQuit()` by hosts
- https://github.com/code-423n4/2023-05-party/blob/f6f80dde81d86e397ba4f3dedb561e23d58ec884/contracts/party/PartyGovernanceNFT.sol#L293

```solidity
    function rageQuit(
        uint256[] calldata tokenIds,
        IERC20[] calldata withdrawTokens,
        address receiver
    ) external {
        // Check if ragequit is allowed.
        uint40 currentRageQuitTimestamp = rageQuitTimestamp;
        if (currentRageQuitTimestamp != ENABLE_RAGEQUIT_PERMANENTLY) {
            if (
                currentRageQuitTimestamp == DISABLE_RAGEQUIT_PERMANENTLY ||
                currentRageQuitTimestamp < block.timestamp
            ) {
                revert CannotRageQuitError(currentRageQuitTimestamp);
            }
        }

        // Used as a reentrancy guard. Will be updated back after ragequit.
        delete rageQuitTimestamp; //@audit reentrancy by host
```

`rageQuit()` prevents the reentrancy attack by removing `rageQuitTimestamp` but it can be bypassed easily by hosts.

1. Let's assume a host has 2 party NFTs and the party has some ERC777 tokens that have `beforeTokenTransferHook`.
2. The host calls `rageQuit()` with the first NFT.
3. Within the `beforeTokenTransferHook`, he enables `rageQuitTimestamp` using `setRageQuit()` and calls `rageQuit()` with the second NFT again.
4. Then during the [token amount calculation](https://github.com/code-423n4/2023-05-party/blob/f6f80dde81d86e397ba4f3dedb561e23d58ec884/contracts/party/PartyGovernanceNFT.sol#L341), it will use the previous token balance before the first `rageQuit()` and he can charge more tokens.

I am not so sure the `host` role is absolutely `trusted` or `restricted` and I think it would be a medium risk for the `restricted host`.

As a mitigation, we can set `rageQuitTimestamp = DISABLE_RAGEQUIT_PERMANENTLY` [here](https://github.com/code-423n4/2023-05-party/blob/f6f80dde81d86e397ba4f3dedb561e23d58ec884/contracts/party/PartyGovernanceNFT.sol#L310) so `setRageQuit()` can't be called even by hosts.


### [L-03] Don't validate address(0)
- https://github.com/code-423n4/2023-05-party/blob/f6f80dde81d86e397ba4f3dedb561e23d58ec884/contracts/party/PartyGovernanceNFT.sol#L296

```solidity
    if (address(token) == ETH_ADDRESS) {
        // Transfer fair share of ETH to receiver.
        uint256 amount = (address(this).balance * shareOfVotingPower) / 1e18;
        if (amount != 0) {
            payable(receiver).transferEth(amount);
        }
    } else {
        // Transfer fair share of tokens to receiver.
        uint256 amount = (token.balanceOf(address(this)) * shareOfVotingPower) / 1e18;
        if (amount != 0) {
            token.compatTransfer(receiver, amount);
        }
    }
```

In `rageQuit()`, it should validate `address(0)` before transfering funds.

### [N-01] Use two-phase role transfers
- https://github.com/code-423n4/2023-05-party/blob/f6f80dde81d86e397ba4f3dedb561e23d58ec884/contracts/party/PartyGovernance.sol#L448

```solidity
    function abdicateHost(address newPartyHost) external onlyHost onlyDelegateCall {
        // 0 is a special case burn address.
        if (newPartyHost != address(0)) {
            // Cannot transfer host status to an existing host.
            if (isHost[newPartyHost]) {
                revert InvalidNewHostError();
            }
            isHost[newPartyHost] = true;
        }
        isHost[msg.sender] = false;
        emit HostStatusTransferred(msg.sender, newPartyHost);
    }
```

`Host` role might be lost if the new address is invalid. `Host` is an important role in this protocol and it would be good to use two-phase transfers.