## Host can accidentally permanently enable rage quit 
Rage quit cannot be disabled permanently after initialization. But it can be enabled permanently after initialization. This means that the host might accidentally enable rage quit permanently, which, of course, cannot be undone.
Just like `DISABLE_RAGEQUIT_PERMANENTLY` is a strictly initial property, consider making `ENABLE_RAGEQUIT_PERMANENTLY` possible only during initialization as well.

## No reflection in code of the implied connection between `rageQuitTimestamp` and `executionDelay`
The main use case for rage quitting is during the execution delay of a ready proposal. As such there is a connection between `rageQuitTimestamp` and `executionDelay`. However, there is no reflection of this in the code.
If `executionDelay == 0` there is probably no use for rage quitting. But since the contract now has this functionality, consider enforcing a minimum `executionDelay`, such that rage quitting is always possible, except if explicitly denied within the new rage quit functionality itself, i.e. by `rageQuitTimestamp` being in the past or set to `DISABLE_RAGEQUIT_PERMANENTLY`.
It is also possible that `rageQuitTimestamp` falls within an execution delay, effectively shortening the window for rage quitting during the ready proposal. One might expect to be able to rage quit during the entire execution delay, if at all possible.
Consider also the possibility of allowing for rage quitting during any execution delay period, even when rage quitting is otherwise disabled.

## Grammar
[`would have otherwise not passed. -> would otherwise not have passed.`](https://github.com/code-423n4/2023-05-party/blob/f6f80dde81d86e397ba4f3dedb561e23d58ec884/contracts/party/PartyGovernance.sol#L595)