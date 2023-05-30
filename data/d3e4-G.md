## `rageQuitTimestamp` can be `uint256` instead of `uint40`
[`rageQuitTimestamp` is `uint40`](https://github.com/code-423n4/2023-05-party/blob/f6f80dde81d86e397ba4f3dedb561e23d58ec884/contracts/party/PartyGovernanceNFT.sol#L55) but doesn't seem to be in any struct where this would be more efficiently packed. It is then cheaper to type it as a full `uint256`.

## Change the order of conditions in `rageQuit()`
Currently the order for checking if rage quit is allowed is [`currentRageQuitTimestamp == DISABLE_RAGEQUIT_PERMANENTLY || currentRageQuitTimestamp < block.timestamp`](https://github.com/code-423n4/2023-05-party/blob/f6f80dde81d86e397ba4f3dedb561e23d58ec884/contracts/party/PartyGovernanceNFT.sol#L302-L303). Since `DISABLE_RAGEQUIT_PERMANENTLY` can only be set during initialization, it should be expected that `rageQuit()` is exceptionally called when `currentRageQuitTimestamp == DISABLE_RAGEQUIT_PERMANENTLY`. Therefore it is more likely to be true than `currentRageQuitTimestamp < block.timestamp`. Consider changing the order of these conditions such that it is more likely that only the first is evaluated.

