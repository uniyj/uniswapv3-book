---
title: "Swap Fees"
weight: 2
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

{{< katex display >}} {{</ katex >}}

# Swap Fees

As I mentioned in the introduction, swap fees is a core mechanism of Uniswap. Liquidity providers need to get paid for
the liquidity they provide, otherwise they'll just use it somewhere else. To incentivize them, trades pay a small fee
during each swap. These fees then distributed among all liquidity providers pro rata (proportionally to their share
in total pool liquidity).

To better understand the mechanism of fees collection and distribution, let's see how they work.

## How Swap Fees are Collected

![Liquidity ranges and fees](/images/milestone_5/liquidity_ranges_fees.png)

Swap fees are collected only when a price range is engaged (used in trades). So we need to track the moments when price
range boundaries get crossed. This is when a price range gets engaged and this is when we want to start collecting
fees for it:
1. when price is increasing and a tick is crossed from left to right;
1. when price is decreasing and a tick is crossed from right to left.

This is when a price range gets disengaged:
1. when price is increasing and a tick is crossed from right to left;
1. when price is decreasing and a tick is crossed from left to right.

![Liquidity range engaged/disengaged](/images/milestone_5/liquidity_range_engaged.png)

Besides knowing when a price range gets engaged/disengaged, we also want to keep track of how much fees each price
range accumulated.

To make fees accounting simpler, Uniswap V3 tracks **the global fees generated by 1 unit of liquidity**. Price range
fees are then calculated based on the global ones: fees accumulated outside of a price range are subtracted from the
global fees. Fees accumulated outside of a price range are tracked when a tick is crossed (and ticks are crossed when
swaps move the price; fees are collected during swaps). With this approach, we don't need to update fees accumulated
by each position on very swap–this allows to save a lot of gas and make interaction with pools cheaper.

Let's recap so we have a clear picture before moving on:
1. Fees are paid by users who swap tokens. A small amount is subtracted from input token and accumulated on pool's
balance.
1. Each pool has `feeGrowthGlobal0X128` and `feeGrowthGlobal1X128` state variables that track total accumulated fees per
unit of liquidity (that is, fee amount divided by pool's liquidity).
1. Notice that at this point actual positions are not updated to optimize gas usage.
1. Ticks keep record of fees accumulated outside of them. When adding a new position and activating a tick (adding
liquidity to a previously empty tick), the tick records how much fees were accumulated outside of it (by convention,
we assume all fees were accumulated **below the tick**).
1. Whenever a tick is activated, fees accumulated outside of the tick are updated as the difference between global fees
accumulated outside of the tick and the fees accumulated outside of the tick since the last time it was crossed.
1. Having ticks that know how much fees were accumulated outside of them will allow us to calculated how much fees were
accumulated inside of a position (position is a range between two ticks).
1. Knowing how much fees were accumulated inside a position will allow us to calculate the shares of fees liquidity
providers are eligible for. If a position wasn't involved in swapping, it'll have zero fees accumulated inside of it and
the liquidity providers who provided liquidity into this range will have no profits from it.

Now, let's see how to calculate fees accumulated by a position (step 6).

## Calculating Position Accumulated Fees

To calculated total fees accumulated by a position, we need to consider two cases: when current price is inside the
position and when it's outside of the position. In both cases, we subtract fees collected outside of the lower and the
upper ticks of the position from fees collected globally. However, we calculate those fees differently depending on
current price.

When current price is inside the position, we subtract the fees that have been collected outside of ticks by this moment:

![Fees accrued inside and outside of a price range](/images/milestone_5/fees_inside_and_outside_price_range.png)

When current price is outside of the position, we need to update fees collected by either upper or lower ticks before
subtracting them from fees collecting globally. We update them only for the calculations and don't overwrite them in
ticks because the ticks don't get crossed.

This is how we update fees collected outside of a tick:

$$f_{o}(i) = f_{g} - f_{o}(i)$$

Fees collected outside of a tick ($f_{o}(i)$) is the difference between fees collected globally ($f_{g}$) and fees
collected outside of the tick when it crossed last time. We kind of reset the counter when a tick is crossed.

To calculate fees collected inside a position:

$$f_{r} = f_{g} - f_{b}(i_{l}) - f_{a}(i_{u})$$

We subtract fees collected below its lower tick ($f_{b}(i_{l})$) and above its upper tick ($f_{a}(i_{u})$) from fees
collected globally from all price ranges ($f_{g}$). This is what we saw on the illustration above.

Now, when current price is above the lower tick (i.e. the position is engaged), we don't need to update fees accumulated
below the lower tick and can simply take them from the lower tick. The same is true for fees collected outside of the
upper tick when current price is below upper tick. In the two other cases, we need to consider updated fees:
1. when taking fees collected below the lower tick and current price is also below the tick (the lower tick hasn't been
crossed recently);
1. when taking fees above the upper tick and current price is also above the tick (the upper tick hasn't been crossed
recently).

I hope this all is not too confusing. Luckily, we now know everything to start coding!

## Accruing Swap Fees

To keep it simple, we'll add fees to our codebase step by step. And we'll begin with accruing swap fees.

### Adding Required State Variables
First thing we need to do is to add the fee amount parameter to Pool–every pool will have a fixed and immutable fee
configured during deployment. In the previous chapter, we added Factory contract that unified and simplified pools
deployment. One of the required pool parameters was tick spacing. Now, we're going to replace it with fee amount and
we'll tie fee amounts to tick spacing: the bigger the fee amount, the larger the tick spacing. This is so that low
volatility pools (stablecoin ones) have lower fees.

Let's update Factory:
```solidity
// src/UniswapV3Factory.sol
contract UniswapV3Factory is IUniswapV3PoolDeployer {
    ...
    mapping(uint24 => uint24) public fees; // `tickSpacings` replaced by `fees`

    constructor() {
        fees[500] = 10;
        fees[3000] = 60;
    }

    function createPool(
        address tokenX,
        address tokenY,
        uint24 fee
    ) public returns (address pool) {
        ...
        parameters = PoolParameters({
            factory: address(this),
            token0: tokenX,
            token1: tokenY,
            tickSpacing: fees[fee],
            fee: fee
        });
        ...
    }
}
```

Fee amounts are hundredths of the basis point. That is, 1 fee unit is 0.0001%, 500 is 0.05%, and 3000 is 0.3%.

Next step is to start accumulating fees in Pool. For that, we'll add two global fee accumulator variables:
```solidity
// src/UniswapV3Pool.sol
contract UniswapV3Pool is IUniswapV3Pool {
    ...
    uint24 public immutable fee;
    uint256 public feeGrowthGlobal0X128;
    uint256 public feeGrowthGlobal1X128;
}
```

The one with index 0 tracks fees accumulated in `token0`, the one with index 1 tracks fees accumulated in `token1`.

### Collecting Fees

Now we need to update `SwapMath.computeSwapStep`–this is where we calculate swap amounts and this is also where
we'll calculate and subtract swap fees. In the function, we replace all occurrences of `amountRemaining` with
`amountRemainingLessFee`:
```solidity
uint256 amountRemainingLessFee = PRBMath.mulDiv(
    amountRemaining,
    1e6 - fee,
    1e6
);
```

Thus, we subtract the fee from input token amount and calculate output amount from a smaller input amount.

The function now also returns the fee amount collected during the step–it's calculated differently depending on whether
the upper limit of the range was reached or not:
```solidity
bool max = sqrtPriceNextX96 == sqrtPriceTargetX96;
if (!max) {
    feeAmount = amountRemaining - amountIn;
} else {
    feeAmount = Math.mulDivRoundingUp(amountIn, fee, 1e6 - fee);
}
```
When it's not reached, the current price range has enough liquidity to fulfill the swap, thus we simply return the
difference between the amount we needed to fulfill and the actual amount fulfilled. Notice that `amountRemainingLessFee`
is not involved here since the actual final amount was calculated in `amountIn` (it's calculated based on available
liquidity).

When the target price is reached, we cannot subtract fees from the entire `amountRemaining` because the current price
range doesn't have enough liquidity to fulfill the swap. Thus, fee amount is subtracted from the amount the current
price range has fulfilled (`amountIn`).

After `SwapMath.computeSwapStep` has returned, we need to update fees accumulated by the swap. Notice that there's only
one variable to track them because, when staring a swap, we already know the input token (during a swap, fees are collected
in either `token0` or `token1`, not both of them):
```solidity
SwapState memory state = SwapState({
    ...
    feeGrowthGlobalX128: zeroForOne
        ? feeGrowthGlobal0X128
        : feeGrowthGlobal1X128
});

(...) = SwapMath.computeSwapStep(...);

state.feeGrowthGlobalX128 += PRBMath.mulDiv(
    step.feeAmount,
    FixedPoint128.Q128,
    state.liquidity
);
```

This is where we adjust accrued fees by the amount of liquidity to later distribute fees among liquidity providers in a
fair way.

### Updating Fee Trackers in Ticks

Next, we need to update the fee trackers in a tick, if it was crossed during a swap (crossing a tick means we're entering
a new price range):
```solidity
if (state.sqrtPriceX96 == step.sqrtPriceNextX96) {
    int128 liquidityDelta = ticks.cross(
        step.nextTick,
        (
            zeroForOne
                ? state.feeGrowthGlobalX128
                : feeGrowthGlobal0X128
        ),
        (
            zeroForOne
                ? feeGrowthGlobal1X128
                : state.feeGrowthGlobalX128
        )
    );
    ...
}
```

Since we haven't yet updated `feeGrowthGlobal0X128/feeGrowthGlobal1X128` state variables at this moment, we pass
`state.feeGrowthGlobalX128` as either of the fee parameters depending on swap direction. `cross` function updates the
fee trackers as we discussed above:
```solidity
// src/lib/Tick.sol
function cross(
    mapping(int24 => Tick.Info) storage self,
    int24 tick,
    uint256 feeGrowthGlobal0X128,
    uint256 feeGrowthGlobal1X128
) internal returns (int128 liquidityDelta) {
    Tick.Info storage info = self[tick];
    info.feeGrowthOutside0X128 =
        feeGrowthGlobal0X128 -
        info.feeGrowthOutside0X128;
    info.feeGrowthOutside1X128 =
        feeGrowthGlobal1X128 -
        info.feeGrowthOutside1X128;
    liquidityDelta = info.liquidityNet;
}
```

> We haven't added the initialization of `feeGrowthOutside0X128/feeGrowthOutside1X128` variables–we'll do this in a later
step.

### Updating Global Fee Trackers

And, finally, after the swap is fulfilled, we can update the global fee trackers:
```solidity
if (zeroForOne) {
    feeGrowthGlobal0X128 = state.feeGrowthGlobalX128;
} else {
    feeGrowthGlobal1X128 = state.feeGrowthGlobalX128;
}
```
Again, during a swap, only one of them is updated because fees are taken from the input token, which is either of `token0`
or `token1` depending on swap direction.

That's it for swapping! Let's now see what happens to fees when liquidity is added.

## Fee Tracking in Positions Management

When adding or removing liquidity (we haven't implemented the latter yet), we also need to initialize or update fees.
Fees need to be tracked both in ticks (fees accumulated outside of ticks–the `feeGrowthOutside` variables we added just
now) and positions (fees accumulated inside of positions). In case of positions, we also need to keep track of and update
the amounts of tokens collected as fees–or in other words, we convert fees per liquidity to token amounts. The latter is
needed so that when a liquidity provider removes liquidity, they get extra tokens collected as swap fees.

Let's do it step by step again.

### Initialization of Fee Trackers in Ticks

In `Tick.update` function, whenever a tick is initialized (adding liquidity to a previously empty tick), we initialize
its fee trackers. However, we're only doing so when the tick is below current price, i.e. when it's inside of the current
price range:

```solidity
// src/lib/Tick.sol
function update(
    mapping(int24 => Tick.Info) storage self,
    int24 tick,
    int24 currentTick,
    int128 liquidityDelta,
    uint256 feeGrowthGlobal0X128,
    uint256 feeGrowthGlobal1X128,
    bool upper
) internal returns (bool flipped) {
    ...
    if (liquidityBefore == 0) {
        // by convention, assume that all previous fees were collected below
        // the tick
        if (tick <= currentTick) {
            tickInfo.feeGrowthOutside0X128 = feeGrowthGlobal0X128;
            tickInfo.feeGrowthOutside1X128 = feeGrowthGlobal1X128;
        }

        tickInfo.initialized = true;
    }
    ...
}
```

If it's not inside of the current price range, its fee trackers will be 0 and they'll be update when the tick is crossed
next time (see the `cross` function we updated above).

### Updating Position Fees and Token Amounts

Next step is to calculate the fees and tokens accumulated by a position. Since a position is a range between two ticks,
we'll calculate these values using the fee trackers we added to ticks on the previous step. The next function might
look messy, but it implements the exact price range fee formulas we saw earlier:
```solidity
// src/lib/Tick.sol
function getFeeGrowthInside(
    mapping(int24 => Tick.Info) storage self,
    int24 lowerTick_,
    int24 upperTick_,
    int24 currentTick,
    uint256 feeGrowthGlobal0X128,
    uint256 feeGrowthGlobal1X128
)
    internal
    view
    returns (uint256 feeGrowthInside0X128, uint256 feeGrowthInside1X128)
{
    Tick.Info storage lowerTick = self[lowerTick_];
    Tick.Info storage upperTick = self[upperTick_];

    uint256 feeGrowthBelow0X128;
    uint256 feeGrowthBelow1X128;
    if (currentTick >= lowerTick_) {
        feeGrowthBelow0X128 = lowerTick.feeGrowthOutside0X128;
        feeGrowthBelow1X128 = lowerTick.feeGrowthOutside1X128;
    } else {
        feeGrowthBelow0X128 =
            feeGrowthGlobal0X128 -
            lowerTick.feeGrowthOutside0X128;
        feeGrowthBelow1X128 =
            feeGrowthGlobal0X128 -
            lowerTick.feeGrowthOutside1X128;
    }

    uint256 feeGrowthAbove0X128;
    uint256 feeGrowthAbove1X128;
    if (currentTick < upperTick_) {
        feeGrowthAbove0X128 = upperTick.feeGrowthOutside0X128;
        feeGrowthAbove1X128 = upperTick.feeGrowthOutside1X128;
    } else {
        feeGrowthAbove0X128 =
            feeGrowthGlobal0X128 -
            upperTick.feeGrowthOutside0X128;
        feeGrowthAbove1X128 =
            feeGrowthGlobal0X128 -
            upperTick.feeGrowthOutside1X128;
    }

    feeGrowthInside0X128 =
        feeGrowthGlobal0X128 -
        feeGrowthBelow0X128 -
        feeGrowthAbove0X128;
    feeGrowthInside1X128 =
        feeGrowthGlobal1X128 -
        feeGrowthBelow1X128 -
        feeGrowthAbove1X128;
}
```

Here, we're calculating fees accumulated between two ticks (inside a price range). For this, we first calculate fees
accumulated below the lower tick and then fees calculated above the upper tick. In the end, we subtract those fees from
the globally accumulated ones. This is the formula we saw earlier:

$$f_{r} = f_{g} - f_{b}(i_{l}) - f_{a}(i_{u})$$

When calculating fees collected above and below a tick, we do it differently depending on whether the price range is
engaged or not (whether the current price is between the boundary ticks of the price range). When it's engaged we simply
use the current fee trackers of a tick; when it's not engaged we need to take updated fee trackers of a tick–you can see
these calculations in the two `else` branches in the code above.

After finding the fees accumulated inside of a position, we're ready to update fee and token amount trackers of the
position:
```solidity
// src/lib/Position.sol
function update(
    Info storage self,
    int128 liquidityDelta,
    uint256 feeGrowthInside0X128,
    uint256 feeGrowthInside1X128
) internal {
    uint128 tokensOwed0 = uint128(
        PRBMath.mulDiv(
            feeGrowthInside0X128 - self.feeGrowthInside0LastX128,
            self.liquidity,
            FixedPoint128.Q128
        )
    );
    uint128 tokensOwed1 = uint128(
        PRBMath.mulDiv(
            feeGrowthInside1X128 - self.feeGrowthInside1LastX128,
            self.liquidity,
            FixedPoint128.Q128
        )
    );

    self.liquidity = LiquidityMath.addLiquidity(
        self.liquidity,
        liquidityDelta
    );
    self.feeGrowthInside0LastX128 = feeGrowthInside0X128;
    self.feeGrowthInside1LastX128 = feeGrowthInside1X128;

    if (tokensOwed0 > 0 || tokensOwed1 > 0) {
        self.tokensOwed0 += tokensOwed0;
        self.tokensOwed1 += tokensOwed1;
    }
}
```

When calculating owed tokens, we multiply fees accumulated by the position by liquidity–the reverse of what we did
during swapping. In the end, we update the fee trackers and add the token amounts to the previously tracked ones.

Now, whenever a position is modified (during addition or removal of liquidity), we calculate fees collected by a
position and update the position:
```solidity
// src/UniswapV3Pool.sol
function mint(...) {
    ...
    bool flippedLower = ticks.update(params.lowerTick, ...);
    bool flippedUpper = ticks.update(params.upperTick, ...);
    ...
    (uint256 feeGrowthInside0X128, uint256 feeGrowthInside1X128) = ticks
        .getFeeGrowthInside(
            params.lowerTick,
            params.upperTick,
            slot0_.tick,
            feeGrowthGlobal0X128_,
            feeGrowthGlobal1X128_
        );

    position.update(
        params.liquidityDelta,
        feeGrowthInside0X128,
        feeGrowthInside1X128
    );
    ...
}
```

## Removing Liquidity

We're now ready to add the only core feature we haven't implemented yet–removal of liquidity. As opposed to minting,
we'll call this function `burn`. This is the function that will let liquidity providers remove a fraction or whole
liquidity from a position they previously added liquidity to. In addition to that, it'll also calculate the fee tokens
liquidity providers are eligible for. However, actual transferring of tokens will be done in a separate function–`collect`.

### Burning Liquidity

Burning liquidity is opposed to minting. Our current design and implementation makes it a hassle-free task: burning
liquidity is simply minting with the negative sign. It's like adding a negative amount of liquidity.

> To implement `burn`,  I needed to refactor the code and extract everything related to position management (updating
ticks and position, and token amounts calculation) into `_modifyPosition` function, which is used by both `mint` and
`burn` function.

```solidity
function burn(
    int24 lowerTick,
    int24 upperTick,
    uint128 amount
) public returns (uint256 amount0, uint256 amount1) {
    (
        Position.Info storage position,
        int256 amount0Int,
        int256 amount1Int
    ) = _modifyPosition(
            ModifyPositionParams({
                owner: msg.sender,
                lowerTick: lowerTick,
                upperTick: upperTick,
                liquidityDelta: -(int128(amount))
            })
        );

    amount0 = uint256(-amount0Int);
    amount1 = uint256(-amount1Int);

    if (amount0 > 0 || amount1 > 0) {
        (position.tokensOwed0, position.tokensOwed1) = (
            position.tokensOwed0 + uint128(amount0),
            position.tokensOwed1 + uint128(amount1)
        );
    }

    emit Burn(msg.sender, lowerTick, upperTick, amount, amount0, amount1);
}
```

In `burn` function, we first update a position and remove some amount of liquidity from it. Then, we update the token
amount owed by the position–they now include amounts accumulated via fees as well as amounts that were previously
provided as liquidity. We can also see this as conversion of position liquidity into token amounts owed by the position–
these amounts won't be used as liquidity anymore and can be freely redeemed by calling the `collect` function:

```solidity
function collect(
    address recipient,
    int24 lowerTick,
    int24 upperTick,
    uint128 amount0Requested,
    uint128 amount1Requested
) public returns (uint128 amount0, uint128 amount1) {
    Position.Info memory position = positions.get(
        msg.sender,
        lowerTick,
        upperTick
    );

    amount0 = amount0Requested > position.tokensOwed0
        ? position.tokensOwed0
        : amount0Requested;
    amount1 = amount1Requested > position.tokensOwed1
        ? position.tokensOwed1
        : amount1Requested;

    if (amount0 > 0) {
        position.tokensOwed0 -= amount0;
        IERC20(token0).transfer(recipient, amount0);
    }

    if (amount1 > 0) {
        position.tokensOwed1 -= amount1;
        IERC20(token1).transfer(recipient, amount1);
    }

    emit Collect(
        msg.sender,
        recipient,
        lowerTick,
        upperTick,
        amount0,
        amount1
    );
}
```

This function simply transfers tokens from a pool and ensures that only valid amounts can be transferred (one cannot
transfer out more than they burned + fees they earned).

There's also a way to collect fees only without burning liquidity: burn 0 amount of liquidity and then call `collect`.
During burning, the position will be updated and token amounts it owes will be updated as well.

And, that's it! Our pool implementation is complete now!