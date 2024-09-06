# High Severity

### [H-1] Incorrect fee calculation in `TSwapPool::getINputAmountBasedOnOutput` causes protocol to take too many tokens from users, resulting in lost fees.

**Description:** The `getInputAmountBasedOnOutput` function is intended to calculate the amount of tokens a user should deposit given an amount of tokens of output tokens. However, the function currently miscalculates the resulting amount. When calculating the fee, it scales the amount by 10_000 instead of 1_000.

**Impact:** Protocol takes more fees than expected from users.

**Recommended Mitigation:**

```diff
function getInputAmountBasedOnOutput(
        uint256 outputAmount,
        uint256 inputReserves,
        uint256 outputReserves
    )
        public
        pure
        revertIfZero(outputAmount)
        revertIfZero(outputReserves)
        returns (uint256 inputAmount)
    {
-        return ((inputReserves * outputAmount) * 10000) / ((outputReserves - outputAmount) * 997);
+        return ((inputReserves * outputAmount) * 1000) / ((outputReserves - outputAmount) * 997);
    }
```

### [H-2] Lack of slippage protection in `TSwapPool::swapExactOutput` causes users to potentially receive way fewer tokens

**Description:** The `swapExactOutput` function does not include any sort of slippage protection. This function is similar to what is done `TSwapPool::swapExactInput`, where the function specifies a `minOutputAmount`, the `swapExactOutput` function should specif a `maxInputAmount`.

**Impact:** If market conditions change before the transaction processes, the user could get a much worse swap.

**Proof of Concept:**

1. The price of WETH is 1000 USD
2. User inputs a `swapExactOutput` looking for 1 WETH
3. inputToken = USDC
4. outputToken = WETH
5. outputAmount = 1
6. deadline = whatever
7. The function does not offer a maxInput amount.
8. As the transaction is pending in the mempool, the market changes. And the price moves HUGE -> 1 WETH is now 10,000 USDC instead of the expected 1,000 USDC

**Recommended Mitigation:**

```diff

 function swapExactOutput(
        IERC20 inputToken,
        IERC20 outputToken,
        uint256 outputAmount,
        uint64 deadline
+       uint256 maxInputAmount
    )
        public
        revertIfZero(outputAmount)
        revertIfDeadlinePassed(deadline)
        returns (uint256 inputAmount)
    {
        uint256 inputReserves = inputToken.balanceOf(address(this));
        uint256 outputReserves = outputToken.balanceOf(address(this));

        inputAmount = getInputAmountBasedOnOutput(outputAmount, inputReserves, outputReserves);

+        if(inputAmount > maxInputAmount){
+            revert();
+         }

        _swap(inputToken, inputAmount, outputToken, outputAmount);
    }


```

### [H-3] `TSwapPool::sellPoolTokens` mismatches input and output tokens, causing users to receive the incorrect amount of tokens.

**Description:** The `sellPoolTokens` function is intended to allow users to easily sell pool tokens and receive WETH in exchange. Users indicate how many pool tokens they're willing to sell in the `poolTokenAmount` parameter. However, the function currently miscalculates the swapped amunt.

This is due to the fact that `swapExactOutput` function is called whereas the `swapExactInput` function is the one that should be called. Because users specify the exact amount if input tokens, not output.

**Impact:** Users will swap the wrong amount of tokens, which is a severe disruption of the protocol functionality.

**Recommended Mitigation:**

Consider changing the implementation to use `swapExactInput` instead of `swapExactOutput`. Note that this would also require changing the `sellPoolTokens` function to accept a new parameter (ie `minWethToReceive` to be passed to `swapExactInput`).

```diff
function sellPoolTokens(
  uint256 poolTokenAmount,
+ uint256 minWethToReceive
  ) external returns (uint256 wethAmount) {
-        return swapExactOutput(i_poolToken, i_wethToken, poolTokenAmount, uint64(block.timestamp));
+        return swapExactInput(i_poolToken, poolTokenAmount, i_wethToken, minWethToReceive, uint64(block.timestamp));

    }
```

Additionally, it might be wise to add a deadline to the function as there is currently none.

### [H-4] In `TSwapPool::_swap` the extra tokens given to users after every `swapCount` breaks the protocol invariant of `x * y = k`

**Description:** The protocol follows a strict invariant of `x * y = k`. Where:

- `x`: The balance of pool token
- `y`: The balance of WETH
- `k`: The constant product of the two balances.

This means, that whenever the balances change in the protocl, the ration between the two amounts should remain constant, hence the `k`. However, this is broken due to the extra incentive in the `_swap` function. Meaning that over time the protocol funds will be drained.

The following block of code is responsible for the issue:

```javascript
swap_count++;
if (swap_count >= SWAP_COUNT_MAX) {
	swap_count = 0;
	outputToken.safeTransfer(msg.sender, 1_000_000_000_000_000_000);
}
```

**Impact:** An attacker could drain the protocl of funds by doing a lot of swaps and collecting the extra incentive given out by the protocol. The protocol core invariant is broken!

**Proof of Concept:**

1. A user swaps 10 times, and collects the extra incentive of `1_000_000_000_000_000_000` tokens.
2. That user continues to swap utill all the protocl funds are drained.

<details>
<summary>Proof of Code</summary>

Place the following into `TSwapPool.t.sol`

```solidity
function testInvariantBroken() public {
        vm.startPrank(liquidityProvider);
        weth.approve(address(pool), 100e18);
        poolToken.approve(address(pool), 100e18);
        pool.deposit(100e18, 100e18, 100e18, uint64(block.timestamp));
        vm.stopPrank();
        uint256 outputWeth = 1e17;

        vm.startPrank(user);
        poolToken.approve(address(pool), type(uint256).max);
        poolToken.mint(user, 100e18);
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));

        int256 startingY = int256(weth.balanceOf(address(pool)));
        int256 expectedDeltaY = int256(-1) * int256(outputWeth);

        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        vm.stopPrank();

        uint256 endingY = weth.balanceOf(address(pool)); // Final WETH reserve
        int256 actualDeltaY = int256(endingY) - int256(startingY);
        assertEq(actualDeltaY, expectedDeltaY);
    }
```

</details>

**Recommended Mitigation:** Remove the extra incentive mechanism. If you want to keep this in, we should account for the change in the `x * y = k` protocol invariant. Or, we should set aside tokens in the same way we do with fees.

```diff
-        swap_count++;
-        if (swap_count >= SWAP_COUNT_MAX) {
-            swap_count = 0;
-            outputToken.safeTransfer(msg.sender, 1_000_000_000_000_000_000);
-        }
```

# Medium Severity

### [M-1] `TSwapPool::deposit` is missing deadline check causing transactions to complete even after the deadline

**Description:** The `deposit` function accepts a deadline parameter, which according to the NatSpec is "The deadline for the transaction to be completed by". However, this parameter is never used. As a consequence, operations that add liquidity to the pool could be executed at unexpected times, im market conditions where the deposit rate is unfavorable.

**Impact:** Transactions could be sent when market conditions are unfavorable to deposit, even when adding a deadline parameter

**Proof of Concept:** The `deadline` parameter is not used.

<details><summary>Compiler Error</summary>

```bash
  Unused function parameter. Remove or comment out the variable name to silence this warning.solidity(5667)
```

</details>
<br />
**Recommended Mitigation:** Add deadline check modifier

```diff
 function deposit(
        uint256 wethToDeposit,
        uint256 minimumLiquidityTokensToMint,
        uint256 maximumPoolTokensToDeposit,
        uint64 deadline
    )
        external
+       revertIfDeadlinePassed(deadline)
        revertIfZero(wethToDeposit)
        returns (uint256 liquidityTokensToMint)
```

### [M-2] Rebase, fee-on-transfer, and ERC777 tokens break protocol invariant

**Description:** Creating pools with particular tokens might end up breaking the protocol invariant due to the way those tokens might be handled. IE: If a token has a fee-on-transfer mechanism it could potentially break the protocol invar

# Low Severity

### [L-1] `TSwapPool::LiquidityAdded` event has parameters out of order

**Description:** When the `LiquidityAdded` event is emmitted in the `TSwapPool::_addLiquidityMintAndTransfer` function, it logs values in an incorrect order. The `poolTokensToDeposit` should go third in the parameter position, whereas the `wethToDeposit` value should be second.

**Impact:** Incorrect event emisssion, leading to off-chain functions potentially malfunctioning.

**Recommended Mitigation:**

```diff
-        emit LiquidityAdded(msg.sender, poolTokensToDeposit, wethToDeposit);
+        emit LiquidityAdded(msg.sender, wethToDeposit, poolTokensToDeposit);

```

### [L-2] Default value returned by `TSwapPool::swapExactInput` results in incorrect return value given

**Description:** The `swapExactInput` function es expected to return the actual amount of tokens bought by the caller. However, while it declares the named return value `output` it is never assigned a value, nor uses an explicit return statement.

**Impact:** The return value will always be 0, giving incorrect information to the caller.

**Recommended Mitigation:**

```diff
{
        uint256 inputReserves = inputToken.balanceOf(address(this));
        uint256 outputReserves = outputToken.balanceOf(address(this));

-        uint256 outputAmount = getOutputAmountBasedOnInput(inputAmount, inputReserves, outputReserves);
+        output = getOutputAmountBasedOnInput(inputAmount, inputReserves, outputReserves);


-        if (outputAmount < minOutputAmount) {
-            revert TSwapPool__OutputTooLow(outputAmount, minOutputAmount);
-        }
+        if (output < minOutputAmount) {
+            revert TSwapPool__OutputTooLow(output, minOutputAmount);
+        }

-        _swap(inputToken, inputAmount, outputToken, outputAmount);
+        _swap(inputToken, inputAmount, outputToken, output);

    }
```

# Informational

### [I-1] `PoolFactory::PoolFactory__PoolDoesNotExist` is not used and should be removed.

```diff
-    error PoolFactory__PoolDoesNotExist(address tokenAddress);
```

### [I-2] Lacking Zero address checks on both contracts constructors

<details><summary>Recommended fix</summary>

```diff
constructor(address wethToken) {
+        if(wethToken == address(0)){
+           revert();
+ }
        i_wethToken = wethToken;
    }
```

```diff
constructor(
        address poolToken,
        address wethToken,
        string memory liquidityTokenName,
        string memory liquidityTokenSymbol
    )
        ERC20(liquidityTokenName, liquidityTokenSymbol)
    {
+        if(wethToken == address(0) || poolToken == address(0)){
+           revert();
+ }
        i_wethToken = IERC20(wethToken);
        i_poolToken = IERC20(poolToken);
    }
```

</details>

### [I-3]] `PoolFactory::createPool` should use `.symbol()` instead of `.name()`

```diff
-        string memory liquidityTokenSymbol = string.concat("ts", IERC20(tokenAddress).name());
+        string memory liquidityTokenSymbol = string.concat("ts", IERC20(tokenAddress).symbol());
```

### [I-4]: Event is missing `indexed` fields

Index event fields make the field more quickly accessible to off-chain tools that parse events. However, note that each index field costs extra gas during emission, so it's not necessarily best to index the maximum allowed per event (three fields). Each event should use three indexed fields if there are three or more fields, and gas usage is not particularly of concern for the events in question. If there are fewer than three fields, all of the fields should be indexed.

<details><summary>4 Found Instances</summary>

- Found in src/PoolFactory.sol [Line: 35](src/PoolFactory.sol#L35)

  ```solidity
      event PoolCreated(address tokenAddress, address poolAddress);
  ```

- Found in src/TSwapPool.sol [Line: 43](src/TSwapPool.sol#L43)

  ```solidity
      event LiquidityAdded(address indexed liquidityProvider, uint256 wethDeposited, uint256 poolTokensDeposited);
  ```

- Found in src/TSwapPool.sol [Line: 44](src/TSwapPool.sol#L44)

  ```solidity
      event LiquidityRemoved(address indexed liquidityProvider, uint256 wethWithdrawn, uint256 poolTokensWithdrawn);
  ```

- Found in src/TSwapPool.sol [Line: 45](src/TSwapPool.sol#L45)

  ```solidity
      event Swap(address indexed swapper, IERC20 tokenIn, uint256 amountTokenIn, IERC20 tokenOut, uint256 amountTokenOut);
  ```

</details>

### [I-5]: Define and use `constant` variables instead of using literals

If the same constant literal value is used multiple times, create a constant state variable and reference it throughout the contract.

<details><summary>4 Found Instances</summary>

- Found in src/TSwapPool.sol [Line: 232](src/TSwapPool.sol#L232)

  ```solidity
          uint256 inputAmountMinusFee = inputAmount * 997;
  ```

- Found in src/TSwapPool.sol [Line: 249](src/TSwapPool.sol#L249)

  ```solidity
          return ((inputReserves * outputAmount) * 10000) / ((outputReserves - outputAmount) * 997);
  ```

- Found in src/TSwapPool.sol [Line: 374](src/TSwapPool.sol#L374)

  ```solidity
              1e18, i_wethToken.balanceOf(address(this)), i_poolToken.balanceOf(address(this))
  ```

- Found in src/TSwapPool.sol [Line: 380](src/TSwapPool.sol#L380)

  ```solidity
              1e18, i_poolToken.balanceOf(address(this)), i_wethToken.balanceOf(address(this))
  ```

</details>

### [I-6] `TSwapPool::deposit` variable `poolTokenReserves` is never used and should be removed

```diff
-            uint256 poolTokenReserves = i_poolToken.balanceOf(address(this));
```

### [I-7] `TSwapPool::deposit` has external call before updating state. Does not follow CEI patter.

```diff
else {
+            liquidityTokensToMint = wethToDeposit;

            _addLiquidityMintAndTransfer(wethToDeposit, maximumPoolTokensToDeposit, wethToDeposit);

-            liquidityTokensToMint = wethToDeposit;
}
```

### [I-8] Shouldn't use Magic numbers. Replace input values with constant variables for better gas efficiency and code readability.

```diff
-        uint256 inputAmountMinusFee = inputAmount * 997;
+        uint256 inputAmountMinusFee = inputAmount * CONSTANT_VARIABLE;
```

### [I-9] Function `swapExactInput` does not have any NatSpec documentation.

### [I-10]: `public` functions not used internally could be marked `external`

Instead of marking a function as `public`, consider marking it as `external` if it is not used internally.

<details><summary>1 Found Instances</summary>

- Found in src/TSwapPool.sol [Line: 252](src/TSwapPool.sol#L252)

  ```solidity
      function swapExactInput(
  ```

</details>
