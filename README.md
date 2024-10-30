# Issue H-1: Boost buyback burns incorrect amount of liquidity 

Source: https://github.com/sherlock-audit/2024-10-axion-judging/issues/114 

## Found by 
carrotsmuggler
### Summary

The function `_unfarmBuyBurn` in the V3AMO contract is a public function open to everyone and calculates the amount of liquidity to burn from the pool. This function basically burns LP positions to take out liquidity and uses the usd to buy up boost tokens and burns them to raise the price of boost tokens.

The issue is in the `_unfarmBuyBurn` function when it tries to estimate how much liquidity needs to be taken out.

https://github.com/sherlock-audit/2024-10-axion/blob/main/liquidity-amo/contracts/SolidlyV3AMO.sol#L320-L326

As seen above, first the token reserves of the pool are checked. Then, the `liquidity` to be burnt is calculated from the difference of the reserves.

```solidity
liquidity = (totalLiquidity * (boostBalance - usdBalance)) / (boostBalance + usdBalance);
liquidity = (liquidity * LIQUIDITY_COEFF) / FACTOR;
```

However, this calculation is not valid for V3/CL pools. This is because in V3 pools, single sided liquidity is allowed which adds to the `totalLiquidity` count, but increases the reserves of only 1 token. If a user adds liquidity at a tick lower than the current price, they will be adding only usd to the pool.

For example, lets say the price currently is below peg, at 0.9. Say there are 1000 boost and 900 usd tokens in the pool, similar to a V2 composition. Now, since its a V3 pool, a user can come in and add 100 usd to the pool at a price of 0.5. Since this price is lower than the spot price, only usd will be needed to open this position. Now, the total reserves of both boost and usd are 1000 each, so the calculated `liquidity` amount to be removed will be 0.

Thus the `liquidity` calculated in the contract has absolutely no meaning since it uses the reserves to calculate it, which is not valid for V3 pools. In the best case scenario, this will cause the function to revert and not work. In the worst case scenario, the liquidity calculated will be overestimated and the price will be pushed up even above the peg price. This is possible if users add single sided `boost` to the pool, increasing the `liquidity` amount calculated without changing the price. In this case, the contract assets will be used to force the boost token above peg, and malicious actors can buy the boost token before and sell it after for a handy profit.

### Root Cause

The main cause is that `liquidity` is calculated from the reserves. This is not valid for V3, since it can have single sided liquidity, and thus the reserves does not serve as an indicator of price or in this case the deviation from the peg.

### Internal pre-conditions

None

### External pre-conditions

Any user can add boost-only liquidity to make the contract overestimate the amount of liquidity it needs to burn

### Attack Path

Users can add boost-only liquidity to make the contract overestimate the amount of liquidity it needs to burn. When extra liquidity is burnt and extra boost is bought back and burnt, the price will be pushed up even above the peg price. Users can buy before trigerring this and sell after for profit.

### Impact

Price can be pushed above the peg price

### PoC

None

### Mitigation

Use the `quoteSwap` function to calculate how much needs to be swapped until the target price is hit.


# Issue H-2: SolidlyV2AMO is not compatible with Velodrome/Aerodrome. 

Source: https://github.com/sherlock-audit/2024-10-axion-judging/issues/239 

## Found by 
0x37, Greese, KupiaSec, PNS, carrotsmuggler, pkqs90, y4y

### Summary

According to the docs, the Dex scope for SolidlyV2 includes Velodrome/Aerodrome.

> We expect the SolidlyV2 tech-implementation work with the “classic” pools on the following Dexes: Velodrome, Aerodrome, Thena, Equalizer (Fantom/Sonic/Base), Ramses and forks (legacy pools), Tokan

However, for Velodrome/Aerodrome implementations, the current `SolidlyV2AMO` is not compatible.

### Root Cause

There are two parts of integration with Velodrome/Aerodrome that are buggy:

1. Gauge
2. Router

Let's go through them one by one (Note, since Velodrome and Aerodrome have basically the same code, I will only post Aerodrome code):

#### 1. Gauge

The main difference is in the `getReward()` function.

Aerodrome interface: https://github.com/aerodrome-finance/contracts/blob/main/contracts/interfaces/IGauge.sol

```solidity
interface IGauge {
    ...
    function getReward(address _account) external;
    ...
}
```

SolidiyV2AMO interface: https://github.com/sherlock-audit/2024-10-axion/blob/main/liquidity-amo/contracts/interfaces/v2/IGauge.sol#L4

```solidity
interface IGauge {
    ...
    function getReward(address account, address[] memory tokens) external;

    function getReward(uint256 tokenId) external;

    function getReward() external;
    ...
}
```

#### 2. Router

The main difference is:

1. Aerodrome uses `poolFor` instead of `pairFor` when querying a pool/pair.
2. The `Route` struct is implemented differently, and is used when performing swap

Aerodrome interface: https://github.com/aerodrome-finance/contracts/blob/main/contracts/interfaces/IRouter.sol#L6

```solidity
interface IRouter {
    struct Route {
        address from;
        address to;
        bool stable;
        address factory;
    }

    function poolFor(
        address tokenA,
        address tokenB,
        bool stable,
        address _factory
    ) external view returns (address pool);

    function swapExactTokensForTokens(
        uint256 amountIn,
        uint256 amountOutMin,
        Route[] calldata routes,
        address to,
        uint256 deadline
    ) external returns (uint256[] memory amounts);
    ...
}
```

SolidiyV2AMO interface: https://github.com/sherlock-audit/2024-10-axion/blob/main/liquidity-amo/contracts/interfaces/v2/ISolidlyRouter.sol#L4

```solidity
interface ISolidlyRouter {
    struct route {
        address from;
        address to;
        bool stable;
    }

    function pairFor(address tokenA, address tokenB, bool stable) external view returns (address pair);

    function swapExactTokensForTokens(
        uint256 amountIn,
        uint256 amountOutMin,
        route[] memory routes,
        address to,
        uint256 deadline
    ) external returns (uint256[] memory amounts);
    ...
}
```

### Internal pre-conditions

N/A

### External pre-conditions

N/A

### Attack Path

N/A

### Impact

SolidlyV2AMO does not work with Aerodrome/Velodrome as expected.

### PoC

N/A

### Mitigation

N/A

# Issue M-1: SolidlyV2AMO and SolidlyV3AMO: USDT Approval Logic Causes Reversion 

Source: https://github.com/sherlock-audit/2024-10-axion-judging/issues/47 

## Found by 
0x37, Naresh, RadCet, ZanyBonzy, unnamed, wellbyt3
### Summary

As the documentation mentions, the contracts are designed to be compatible with any EVM chain and support USDT:
>The smart contracts can potentially be implemented on any full-EVM chain

>USD is a generic name for a reference stable coin paired with BOOST in the AMO ( USDC and USDT are the first natural candidates )

However, both the `SolidlyV2AMO` and `SolidlyV3AMO` contracts will not work with USDT, as they will revert during the `_addLiquidity()` and `_unfarmBuyBurn()` functions.


### Root Cause
Both `SolidlyV2AMO` and `SolidlyV3AMO` use OpenZeppelin's IERC20Upgradeable interface, which expects a boolean return value when calling the `approve()` function. However, [USDT's implementation of the approve() function](https://vscode.blockscan.com/ethereum/0xdac17f958d2ee523a2206206994597c13d831ec7) does not return a boolean value, which causes the contract to revert during execution.

```solidity
    /**
    * @dev Approve the passed address to spend the specified amount of tokens on behalf of msg.sender.
    * @param _spender The address which will spend the funds.
    * @param _value The amount of tokens to be spent.
    */
    function approve(address _spender, uint _value) public onlyPayloadSize(2 * 32) {
```

The functions `_addLiquidity()` and `_unfarmBuyBurn()` in both contracts expect a boolean return value, causing them to revert when interacting with USDT.

example [SolidlyV3AMO::_addLiquidity](https://github.com/sherlock-audit/2024-10-axion/blob/d75df3636ea28dd627d548d5d473c5d5477b0dc6/liquidity-amo/contracts/SolidlyV3AMO.sol#L182-L202) and [SolidlyV3AMO::_unfarmBuyBurn](https://github.com/sherlock-audit/2024-10-axion/blob/d75df3636ea28dd627d548d5d473c5d5477b0dc6/liquidity-amo/contracts/SolidlyV3AMO.sol#L254-L267):
```solidity
 function _addLiquidity(uint256 usdAmount, uint256 minBoostSpend, uint256 minUsdSpend, uint256 deadline)
        internal
        override
        returns (uint256 boostSpent, uint256 usdSpent, uint256 liquidity)
    {
        //....

        // Approve the transfer of BOOST and USD tokens to the pool
        IERC20Upgradeable(boost).approve(pool, boostAmount);
        
@>      IERC20Upgradeable(usd).approve(pool, usdAmount);

        (uint256 amount0Min, uint256 amount1Min) = sortAmounts(minBoostSpend, minUsdSpend);

        uint128 currentLiquidity = ISolidlyV3Pool(pool).liquidity();
        liquidity = (usdAmount * currentLiquidity) / IERC20Upgradeable(usd).balanceOf(pool);

        // Add liquidity to the BOOST-USD pool within the specified tick range
        (uint256 amount0, uint256 amount1) = ISolidlyV3Pool(pool).mint(
            address(this), tickLower, tickUpper, uint128(liquidity), amount0Min, amount1Min, deadline
        );

        // Revoke approval from the pool
        IERC20Upgradeable(boost).approve(pool, 0);
@>      IERC20Upgradeable(usd).approve(pool, 0);

      //....
    }
    
     function _unfarmBuyBurn(
        uint256 liquidity,
        uint256 minBoostRemove,
        uint256 minUsdRemove,
        uint256 minBoostAmountOut,
        uint256 deadline
    )
        internal
        override
        returns (uint256 boostRemoved, uint256 usdRemoved, uint256 usdAmountIn, uint256 boostAmountOut)
    {
       //....
        // Approve the transfer of usd tokens to the pool
@>      IERC20Upgradeable(usd).approve(pool, usdRemoved);

        // Execute the swap and store the amounts of tokens involved
        (int256 amount0, int256 amount1) = ISolidlyV3Pool(pool).swap(
            address(this),
            boost > usd, // Determines if we are swapping USD for BOOST (true) or BOOST for USD (false)
            int256(usdRemoved),
            targetSqrtPriceX96,
            minBoostAmountOut,
            deadline
        );

        // Revoke approval from the pool
@>      IERC20Upgradeable(usd).approve(pool, 0);

        //...
    }
```
[SolidlyV2AMO::_addLiquidity](https://github.com/sherlock-audit/2024-10-axion/blob/d75df3636ea28dd627d548d5d473c5d5477b0dc6/liquidity-amo/contracts/SolidlyV2AMO.sol#L214-L236) and [SolidlyV2AMO::_unfarmBuyBurn](https://github.com/sherlock-audit/2024-10-axion/blob/d75df3636ea28dd627d548d5d473c5d5477b0dc6/liquidity-amo/contracts/SolidlyV2AMO.sol#L296) faces the same issue

### Internal pre-conditions

_No response_

### External pre-conditions

_No response_

### Attack Path

_No response_

### Impact

Adding liquidity and farming will fail due to a revert on USDT approvals

### Mitigation

Use `safeApprove` instead of `approve`

# Issue M-2: Public mintSellFarm can be permanently DOSed due to bad check of V3 pool in _addLiquidity 

Source: https://github.com/sherlock-audit/2024-10-axion-judging/issues/64 

## Found by 
spark1
### Summary

Public mintSellFarm can be permanently DOSed due to bad check of V3 pool in _addLiquidity. The title is similar to https://github.com/sherlock-audit/2024-10-axion-testing549/issues/8 but this is not a duplicate according to the rules. The code implementation and impact and fix are all different.

> Duplication rules
> ...
> The exception to this would be if underlying code implementations OR impact OR the fixes are different, then they may be treated separately.

### Root Cause

https://github.com/sherlock-audit/2024-10-axion/blob/main/liquidity-amo/contracts/SolidlyV3AMO.sol#L186-L187

```solidity
        uint128 currentLiquidity = ISolidlyV3Pool(pool).liquidity();
        liquidity = (usdAmount * currentLiquidity) / IERC20Upgradeable(usd).balanceOf(pool);
```

The amount of liquidity to mint depends on the usd balance of the pool. This is wrong. The problem is similar to https://github.com/sherlock-audit/2024-10-axion-testing549/issues/8. Copied from there:

> Uni/Solidly V3 pools have concentrated liquidity and the balance of the tokens of the pool is not related to the price. When liquidity is added out of range, tokens of only one type can be added to the pool balance without moving the price. See uniswap docs. https://support.uniswap.org/hc/en-us/articles/20902968738317-What-is-single-sided-liquidity
> 
> Example: Uni v3 DAI/USDC pool has 36m dai and 37.2m usdc but the price is 1:1. https://app.uniswap.org/explore/pools/ethereum/0x5777d92f208679DB4b9778590Fa3CAB3aC9e2168

Here the problem is not adding single sided liquidity but that the same tokens can be used to add more liquidity units if they are added in a concentrated range than over the whole range.

Uni/Solidly V3 has invariant: adding same amount of tokens (A, B) over same range gives same amount of liquidity (check whitepaper page 2 https://app.uniswap.org/whitepaper-v3.pdf). The tokens of the pool do not matter. This calculation is wrong because that invariant is broken. If concentrated liquidity is added, the `.liquidity()` increases for less usd tokens than it took to add over the full range, which is what is done in the tests and presumably on mainnet. The calculated liquidity to add will be increased more than it should. This can cause permanent DOS because the required usd to obtain that much liquidity can be more than usdAmount. (This calculation will be wrong even if other users add normal liquidity with usd/boost in differing ratios.)

mintSellFarm will be DOSed as it adds liquidity with the full usd balance of the contract. https://github.com/sherlock-audit/2024-10-axion/blob/main/liquidity-amo/contracts/MasterAMO.sol#L213-L214
```solidity
            uint256 usdBalance = IERC20Upgradeable(usd).balanceOf(address(this));
            (boostSpent, usdSpent, liquidity) = _addLiquidity(usdBalance, minBoostSpend, minUsdSpend, deadline);
```

The admin can call addLiquidity manually and specify a lower usdAmount than what is calculated in mintSellFarm, which is the full balance. That involves difficult math or trial and error.

### Internal pre-conditions

_No response_

### External pre-conditions

Someone adds concentrated liquidity to V3 pool.

### Attack Path

Not an attack, but normal usage.

1. Start with example pool initially at 100 boost and 100 usd of liquidity over the full range owned by the protocol. Protocol receives some liquidity amount. At this point if _addLiquidity is called it will calculate that it needs the received liquidity amount (= total amount) if it wants to add 100 usd of liquidity over the full range.
2. User adds liquidity to pool over the 0.5-1.5 usd. The liquidity units here are cheaper per token than over the full range.
3. User buys boost and increases boost price to 1.1 usd.
4. _addLiquidity will calculate that it needs more liquidity than before if it wants to add 100 usd of liquidity over the full range because of `liquidity = (usdAmount * currentLiquidity) / IERC20Upgradeable(usd).balanceOf(pool);`, and disproportionate increase in currentLiquidity. This is greater than in step 1. The actual usd required to add this liquidity will be > 100. The problem is currentLiquidity has been inflated for cheaper than the protocol added due to concentrated liquidity.
5. mintSellFarm is now DOSed because it tries to add more usd than it has.

### Impact

Users cannot call mintSellFarm to restore the peg. The admin can call addLiquidity manually and specify a lower usdAmount than what is calculated in mintSellFarm. That involves difficult math or trial and error as well as repeatedly paying unnecessary gas fees. The admin/AMO bot may not know or be unable to do this math and the peg may be lost.

### PoC

Edit the test https://github.com/sherlock-audit/2024-10-axion/blob/main/liquidity-amo/test/SolidlyV3AMO.test.ts#L522-L553. See section with `// BEGIN ADDED LINES`.

```javascript
    describe("mintSellFarm", function() {
      it("Should execute mintSellFarm successfully when price is above 1", async function() {
        let limitSqrtPriceX96: bigint;
        const zeroForOne = usd2boost;
        if (zeroForOne) {
          limitSqrtPriceX96 = MIN_SQRT_RATIO + BigInt(10);
        } else {
          limitSqrtPriceX96 = MAX_SQRT_RATIO - BigInt(10);
        }
        const usdToBuy = ethers.parseUnits("1000000", 6);
        await testUSD.connect(admin).mint(user.address, usdToBuy);
        await testUSD.connect(user).approve(poolAddress, usdToBuy);
        await pool.connect(user).swap(
          user.address,
          zeroForOne,
          usdToBuy,
          limitSqrtPriceX96,
          1,
          deadline
        );

        
        // BEGIN ADDED LINES

        const boostToAdd = ethers.parseUnits("100000000", 18);
        await boost.connect(boostMinter).mint(user.address, boostToAdd);
        await boost.connect(user).approve(poolAddress, boostToAdd);
        const usdToAdd = ethers.parseUnits("100000000", 6);
        await testUSD.connect(boostMinter).mint(user.address, usdToAdd);
        await testUSD.connect(user).approve(poolAddress, usdToAdd);

        console.log(await pool.connect(user).liquidity());

        // add liquidity in the smaller range to increase uint128 currentLiquidity = ISolidlyV3Pool(pool).liquidity();
        // because .liquidity() only gives liquidity in range

        await pool.connect(user).mint(
          user.address,
          tickLower / 2,
          tickUpper / 2,
          ethers.parseUnits("10000000", 12),
          1,
          1,
          deadline
        );

        console.log(await pool.connect(user).liquidity());

        console.log(" user usd " + (await testUSD.connect(user).balanceOf(user)));
        // END ADDED LINES
        

        expect(await solidlyV3AMO.boostPrice()).to.be.gt(ethers.parseUnits("1.1", 6));
        await expect(solidlyV3AMO.connect(amo).mintSellFarm()).to.emit(solidlyV3AMO, "PublicMintSellFarmExecuted");
        expect(await solidlyV3AMO.boostPrice()).to.be.approximately(ethers.parseUnits(price, 6), 10);
        expect(await boost.balanceOf(amoAddress)).to.be.equal(0);
        expect(await testUSD.balanceOf(amoAddress)).to.be.lt(Math.floor(Number(usdToBuy) * errorTolerance));
      });
```

Result

```text
    Public AMO Functions
      mintSellFarm
10000000000000000000n
10000000000000000000n
20000000000000000000n
 user usd 89002432544798
        1) Should execute mintSellFarm successfully when price is above 1

  1) SolidlyV3AMO
       Public AMO Functions
         mintSellFarm
           Should execute mintSellFarm successfully when price is above 1:
     Error: VM Exception while processing transaction: reverted with reason string 'TF'
    at <UnrecognizedContract>.<unknown> (0xa0152cdfc8a8bcf1291570afec409f3a81c2d734)
    at SolidlyV3AMO._addLiquidity (contracts/SolidlyV3AMO.sol:190)
...
    at async Context.<anonymous> (SolidlyV3AMO.test.ts:585:9)
```

_addLiquidity failed because of this bug and mintSellFarm is DOSed. Line 585 is `await expect(solidlyV3AMO.connect(amo).mintSellFarm()).to.emit(solidlyV3AMO, "PublicMintSellFarmExecuted");`

The test should pass because adding liquidity does not change the price and so mintSellFarm should still be callable.

### Mitigation

Read the whitepaper and do heavy math. https://app.uniswap.org/whitepaper-v3.pdf

# Issue M-3: Contracts of the codebase will not strictly compliant with the ERC-1504. 

Source: https://github.com/sherlock-audit/2024-10-axion-judging/issues/155 

## Found by 
0xnbvc, Atharv, AuditorPraise, HackTrace, Kirkeelee, dany.armstrong90, isagixyz, pkqs90
### Summary

Contracts of the codebase isn't strictly compliant with the ERC-1504. This breaks the readme.

### Root Cause

As per readme:
> Is the codebase expected to comply with any EIPs? Can there be/are there any deviations from the specification?
> 
> Strictly compliant: ERC-1504: Upgradable Smart Contract

But the contracts of the codebase uses openzepplin upgradable contracts as base contract which are not compliant with ERC-1504. As per [ERC-1504](https://eips.ethereum.org/EIPS/eip-1504), the upgradable contract should consists of have handler contract, data contract and optionally the upgrader contract.
But the contracts of the codebase are not compliant with ERC-1504 because they has no data contract and has data inside the handler contract.


### Internal pre-conditions

_No response_

### External pre-conditions

_No response_

### Attack Path

_No response_

### Impact

Break the readme.

### PoC

_No response_

### Mitigation

Make the contracts strictly compliant with ERC-1504.


# Issue M-4: Precision loss in `boostPrice` in `SolidlyV3AMO` 

Source: https://github.com/sherlock-audit/2024-10-axion-judging/issues/191 

## Found by 
0x37, Bigsam, durov, s1ce, yuza101
### Summary

There is precision loss in a specific case in `SolidlyV3AMO`, which leads to incorrect value of `boostPrice` being reported. This leads to issues with upstream functions that use it. 

### Root Cause

https://github.com/sherlock-audit/2024-10-axion/blob/main/liquidity-amo/contracts/SolidlyV3AMO.sol#L343

Consider the following code:

```solidity
function boostPrice() public view override returns (uint256 price) {
        (uint160 _sqrtPriceX96, , , ) = ISolidlyV3Pool(pool).slot0();
        uint256 sqrtPriceX96 = uint256(_sqrtPriceX96);
        if (boost < usd) {
            price = (10 ** (boostDecimals - usdDecimals + PRICE_DECIMALS) * sqrtPriceX96 ** 2) / Q96 ** 2;
        } else {
            if (sqrtPriceX96 >= Q96) {
                // @audit: massive precision loss here
                price = 10 ** (boostDecimals - usdDecimals + PRICE_DECIMALS) / (sqrtPriceX96 ** 2 / Q96 ** 2);
            } else {
                price = (10 ** (boostDecimals - usdDecimals + PRICE_DECIMALS) * Q96 ** 2) / sqrtPriceX96 ** 2;
            }
        }
    }
```

Notice that in this specific clause:

```solidity
if (sqrtPriceX96 >= Q96) {
                // @audit: massive precision loss here
                price = 10 ** (boostDecimals - usdDecimals + PRICE_DECIMALS) / (sqrtPriceX96 ** 2 / Q96 ** 2);
            }
```

We divide `(sqrtPriceX96 ** 2 / Q96 ** 2)`. However, consider a case where the value of the stablecoin is 20% higher than the value of boost; in this case a price of around 0.8 should be reported by the function but because `(sqrtPriceX96 ** 2 / Q96 ** 2)` will round down to 1, the function will end up reporting a price of 1. 

### Internal pre-conditions

_No response_

### External pre-conditions

We need the price of the stablecoin to be at least a bit higher than the price of BOOST for this to be relevant

### Attack Path

Described above; precision loss leads to boostPrice calculation reported by `SolidlyV3AMO` being incorrect. 

### Impact

Impact is that, because the price is reported incorrect, public calls to `unfarmBuyBurn()` will fail because the following check will fail:

`if (newBoostPrice > boostUpperPriceBuy) revert PriceNotInRange(newBoostPrice);`


`newBoostPrice` will be reported as `1` even though it should be much lower. 

### PoC

First, set `sqrtPriceX96` to `87150978765690771352898345369` (10% above 2^96)      

```typescript
pool = await ethers.getContractAt("ISolidlyV3Pool", poolAddress);
await pool.initialize(sqrtPriceX96);
```

Then: 

```typescript
it("Showcases the incorrect price that is returned", async function() {
        console.log(await solidlyV3AMO.boostPrice()) // 1000000
})
```


### Mitigation

_No response_

# Issue M-5: SolidlyV3AMO integration with SolidlyV3 does not collect LP fees when burning liquidity. 

Source: https://github.com/sherlock-audit/2024-10-axion-judging/issues/242 

## Found by 
pkqs90

### Summary

SolidlyV3AMO supplies liquidity for the Boost-USD pool. When users call `unfarmBuyBurn` to burn liquidity from SolidlyV3AMO, the LP fees should be collected as well. However, the current integration does not correctly collect fees for SolidlyV3, and the fees are stuck forever.

### Root Cause

Let's see the SolidlyV3 implementation for `burnAndCollect()` function. https://ftmscan.com/address/0x54a571D91A5F8beD1D56fC09756F1714F0cd8aD9#code (This is taken from [notion docs](https://sticky-yarrow-52e.notion.site/AXION-Audit-Documentation-11bb3e443073802c949bd3700abad968).)


Even though SolidlyV3 is a fork of UniswapV3, there is a significant difference. In UniswapV3, the liquidity fees are calculated within the Pool, and can be collected via the Pool. However, in SolidlyV3, the fees are not updated at all (In the following code, where the fees should be updated in Position.sol as done by UniswapV3, you can see the fee update code is fully removed).

Also, according to the SolidlyV3 docs https://docs.solidly.com/v3/rewards-distributor, all fees (and bribes) distribution have been moved into the `RewardsDistributor.sol` contract, and distributed via merkel proof. This means calling `burnAndCollect()` does not allow us to collect the LP fees anymore, and that we need to call separate function for SolidlyV3AMO in order to retrieve the LP fees.

SolidlyV3Pool.sol
```solidity
    function burnAndCollect(
        address recipient,
        int24 tickLower,
        int24 tickUpper,
        uint128 amountToBurn,
        uint128 amount0ToCollect,
        uint128 amount1ToCollect
    )
        external
        override
        returns (uint256 amount0FromBurn, uint256 amount1FromBurn, uint128 amount0Collected, uint128 amount1Collected)
    {
        (amount0FromBurn, amount1FromBurn) = _burn(tickLower, tickUpper, amountToBurn);
        (amount0Collected, amount1Collected) = _collect(
            recipient,
            tickLower,
            tickUpper,
            amount0ToCollect,
            amount1ToCollect
        );
    }
    function _burn(
        int24 tickLower,
        int24 tickUpper,
        uint128 amount
    ) private lock returns (uint256 amount0, uint256 amount1) {
@>      (Position.Info storage position, int256 amount0Int, int256 amount1Int) = _modifyPosition(
            ModifyPositionParams({
                owner: msg.sender,
                tickLower: tickLower,
                tickUpper: tickUpper,
                liquidityDelta: -int256(amount).toInt128()
            })
        );

        amount0 = uint256(-amount0Int);
        amount1 = uint256(-amount1Int);

        if (amount0 > 0 || amount1 > 0) {
@>          (position.tokensOwed0, position.tokensOwed1) = (
                position.tokensOwed0 + uint128(amount0),
                position.tokensOwed1 + uint128(amount1)
            );
        }
        emit Burn(msg.sender, tickLower, tickUpper, amount, amount0, amount1);
    }
    function _modifyPosition(
        ModifyPositionParams memory params
    ) private returns (Position.Info storage position, int256 amount0, int256 amount1) {
        checkTicks(params.tickLower, params.tickUpper);

        Slot0 memory _slot0 = slot0; // SLOAD for gas optimization

@>      position = _updatePosition(params.owner, params.tickLower, params.tickUpper, params.liquidityDelta);
        ...
    }
    function _updatePosition(
        address owner,
        int24 tickLower,
        int24 tickUpper,
        int128 liquidityDelta
    ) private returns (Position.Info storage position) {
        ...
@>      position.update(liquidityDelta);
		..
    }
```

SolidlyV3 Position.sol
```solidity
    /// @notice Updates the liquidity amount associated with a user's position
    /// @param self The individual position to update
    /// @param liquidityDelta The change in pool liquidity as a result of the position update
    function update(Info storage self, int128 liquidityDelta) internal {
    	// @audit-note: Fees should be accumulated in UniswapV3. But in SolidlyV3, this is removed.
        if (liquidityDelta != 0) {
            self.liquidity = LiquidityMath.addDelta(self.liquidity, liquidityDelta);
        }
    }
```

SolidlyV3AMO implementation
```solidity
    function _unfarmBuyBurn(
        uint256 liquidity,
        uint256 minBoostRemove,
        uint256 minUsdRemove,
        uint256 minBoostAmountOut,
        uint256 deadline
    )
        internal
        override
        returns (uint256 boostRemoved, uint256 usdRemoved, uint256 usdAmountIn, uint256 boostAmountOut)
    {
        (uint256 amount0Min, uint256 amount1Min) = sortAmounts(minBoostRemove, minUsdRemove);
        // Remove liquidity and store the amounts of USD and BOOST tokens received
        (
            uint256 amount0FromBurn,
            uint256 amount1FromBurn,
            uint128 amount0Collected,
            uint128 amount1Collected
@>      ) = ISolidlyV3Pool(pool).burnAndCollect(
                address(this),
                tickLower,
                tickUpper,
                uint128(liquidity),
                amount0Min,
                amount1Min,
                type(uint128).max,
                type(uint128).max,
                deadline
            );
        ...
    }
```

- https://github.com/sherlock-audit/2024-10-axion/blob/main/liquidity-amo/contracts/SolidlyV3AMO.sol#L235

### Internal pre-conditions

N/A

### External pre-conditions

N/A

### Attack Path

N/A

### Impact

LP Fees are not retrievable for SolidlyV3AMO.

### PoC

N/A

### Mitigation

Add a function to call the `RewardsDistributor.sol` for SolidlyV3 to retrieve the LP fees. This can be an independent function, since not all SolidlyV3 forks may support this feature.

# Issue M-6: MasterAMO should not use the `initializer` modifier 

Source: https://github.com/sherlock-audit/2024-10-axion-judging/issues/244 

## Found by 
FonDevs, KupiaSec, UsmanAtique, calc1f4r, hunter\_w3b
### Summary

`MasterAMO` is a utils contract that intended to be inherited by SolidlyV2AMO, SolidlyV3AMO contracts, therefore. it's initializer function should not use the [initializer](https://github.com/sherlock-audit/2024-10-axion/blob/main/liquidity-amo/contracts/MasterAMO.sol#L104) modifier, instead, it should use onlyInitializing modifier.



### Root Cause

In the [MasterAMO.sol:104](https://github.com/sherlock-audit/2024-10-axion/blob/main/liquidity-amo/contracts/MasterAMO.sol#L104) contract, the initialize function uses the initializer modifier. This is incorrect for a contract like MasterAMO, which is meant to be inherited by other contracts, such as [SolidlyV2AMO](https://github.com/sherlock-audit/2024-10-axion/blob/main/liquidity-amo/contracts/SolidlyV2AMO.sol#L10) and [SolidlyV3AMO](https://github.com/sherlock-audit/2024-10-axion/blob/main/liquidity-amo/contracts/SolidlyV3AMO.sol#L8).


In this inheritance model, the [SolidlyV2AMO](https://github.com/sherlock-audit/2024-10-axion/blob/main/liquidity-amo/contracts/SolidlyV2AMO.sol#L62-L82) contract also has its own initialize function, which includes the initializer modifier and calls the initialize function of MasterAMO. The problem here is that both the parent contract [MasterAMO](https://github.com/sherlock-audit/2024-10-axion/blob/main/liquidity-amo/contracts/MasterAMO.sol#L104) and the child contracts [SolidlyV2AMO](https://github.com/sherlock-audit/2024-10-axion/blob/main/liquidity-amo/contracts/SolidlyV2AMO.sol#L79), [SolidlyV3AMO](https://github.com/sherlock-audit/2024-10-axion/blob/main/liquidity-amo/contracts/SolidlyV3AMO.sol#L67) are using the [initializer](https://docs.openzeppelin.com/contracts/4.x/api/proxy#Initializable-initializer--) modifier, which limits initialization to only one call. 

According to the [OpenZeppelin documentation](https://docs.openzeppelin.com/contracts/4.x/api/proxy#Initializable-initializer--), the [onlyInitializing](https://docs.openzeppelin.com/contracts/4.x/api/proxy#Initializable-onlyInitializing--) modifier should be used to allow initialization in both the parent and child contracts. The  [onlyInitializing](https://docs.openzeppelin.com/contracts/4.x/api/proxy#Initializable-onlyInitializing--) modifier ensures that when the initialize function is called, any contracts in its inheritance chain can still complete their own initialization.

https://docs.openzeppelin.com/contracts/4.x/api/proxy#Initializable-initializer--

>> A modifier that defines a protected initializer function that can be invoked at most once. In its scope, onlyInitializing functions can be used to initialize parent contracts.



### Internal pre-conditions

_No response_

### External pre-conditions

_No response_

### Attack Path

_No response_

### Impact

In this scenario, no direct attack or monetary loss is likely. However, the vulnerability causes a significant operational issue, preventing inheriting contracts from completing initialization. This could lead to a failure in the deployment of critical protocol components, affecting the overall system functionality.

### PoC

A simple POC in Remix.

![Screenshot from 2024-10-15 15-49-31](https://github.com/user-attachments/assets/c1fc0f96-c7cc-493b-aba3-71191edafa61)


### Mitigation


Replace the initializer modifier in the [MasterAMO](https://github.com/sherlock-audit/2024-10-axion/blob/main/liquidity-amo/contracts/MasterAMO.sol#L96C1-L104C27) contract with the onlyInitializing modifier. This allows the initialize function to be used by both the MasterAMO and any inheriting contracts during their initialization phase, without conflicting with their individual setup processes.

```diff
    function initialize(
        address admin, // // Address assigned the admin role (given exclusively to a multi-sig wallet)
        address boost_, // The Boost stablecoin address
        address usd_, // generic name for $1 collateral ( typically USDC or USDT )
        address pool_, // The pool where AMO logic applies for Boost-USD pair
        // On each chain where Boost is deployed, there will be a stable Boost-USD pool ensuring BOOST's peg.
        // Multiple Boost-USD pools can exist across different DEXes on the same chain, each with its own AMO, maintaining independent peg guarantees.
        address boostMinter_ // the minter contract
-   ) public initializer {
+   ) public onlyInitializing {
```

# Issue M-7: The `SolidlyV3AMO._mintAndSellBoost()` function does not work with Velodrome, Aerodrome, Fenix, Thena and Ramses 

Source: https://github.com/sherlock-audit/2024-10-axion-judging/issues/268 

## Found by 
ABA, Kirkeelee, KupiaSec, carrotsmuggler, pkqs90
### Summary
The protocol mints `BOOST` tokens and sells them for USD using the `_mintAndSellBoost()` function for SolidlyV3 DEXes. Since Velodrome, Aerodrome, Fenix, Thena and Ramses are parts of the SolidlyV3 DEXes, they should be compatible. However, the `_mintAndSellBoost()` function does not work with the DEXes due to incorrect function parameters.

### Root Cause
In the `_mintAndSellBoost()` function, it mints `BOOST` tokens and swaps them for USD tokens.

https://github.com/sherlock-audit/2024-10-axion/blob/c65e662999d0c79439703fc6713814b4ad023e01/liquidity-amo/contracts/SolidlyV3AMO.sol#L144-L151
```solidity
File: liquidity-amo\contracts\SolidlyV3AMO.sol
144: @>      (int256 amount0, int256 amount1) = ISolidlyV3Pool(pool).swap(
145:             address(this),
146:             boost < usd,
147:             int256(boostAmount), // Amount of BOOST tokens being swapped
148:             targetSqrtPriceX96, // The target square root price
149:             minUsdAmountOut, // Minimum acceptable amount of USD to receive from the swap
150:             deadline
151:         );
```

However, the `swap()` function of Velodrome, Aerodrome, Fenix, Thena and Ramses has different parameters.
The `swap()` functions of the DEXes are as follows:

Velodrome : https://optimistic.etherscan.io/address/0xCc0bDDB707055e04e497aB22a59c2aF4391cd12F#code:~:text=File%208%20of%2037%20%3A%20CLPool.sol L613-L619

Aerodrome : https://basescan.org/address/0x5e7BB104d84c7CB9B682AaC2F3d509f5F406809A#code:~:text=File%208%20of%2037%20%3A%20CLPool.sol L679-L685

Fenix : https://blastscan.io/address/0x5aCCAc55f692Ae2F065CEdDF5924C8f6B53cDaa8#code:~:text=File%202%20of%2044%20%3A%20AlgebraPool.sol L212-L218

Thena : https://bscscan.com/address/0xc89F69Baa3ff17a842AB2DE89E5Fc8a8e2cc7358#code:~:text=File%202%20of%2031%20%3A%20AlgebraPool.sol L591-L597

Ramses : https://arbiscan.io/address/0xf896d16fa56a625802b6013f9f9202790ec69908#code:~:text=File%2044%20of%2045%20%3A%20RamsesV2Pool.sol L944-L950

```solidity
  function swap(
    address recipient,
    bool zeroToOne,
    int256 amountRequired,
    uint160 limitSqrtPrice,
    bytes calldata data
  ) external override returns (int256 amount0, int256 amount1) {   
```

As a result, the `_mintAndSellBoost()` function does not work with Velodrome, Aerodrome, Fenix, Thena and Ramses due to incorrect function parameters.

### Internal pre-conditions

For convenience, let's assume that the USD token is `USDC` from this point forward.
- Protocol team is going to mint additional `BOOST` and sell them for `USDC` to bring the price back down to peg. 

### External pre-conditions

- The `BOOST-USDC` price diverges from peg and `BOOST` is trading above `$1` in Velodrome.

### Attack Path

- Alice(protocol team) calls the `mintAndSellBoost()` function.

It reverts.

### Impact

The `mintAndSellBoost()`, `mintSellFarm()` functions will be permanently DoSed for Velodrome, Aerodrome, Fenix, Thena and Ramses.
Protocol team can't mint additional `BOOST` and sell them for `USDC` to bring the price back down to peg.

### PoC

### Mitigation

Use the correct function parameters for Velodrome, Aerodrome, Fenix, Thena and Ramses.

# Issue M-8: Public unfarmBuyBurn may not rebalance the pool correctly 

Source: https://github.com/sherlock-audit/2024-10-axion-judging/issues/283 

## Found by 
vinica\_boy
### Summary

Public function `unfarmBuyBurn()` is expected to bring back the BOOST price close to $1 in case it is lower than $1. In AMO V3 it is done by removing liquidity and swapping the removed USD for BOOST. Current implementation wont work if the price is in a range with low liquidity.

### Root Cause

Wrong implementation in calculation of the to be removed liquidity when rebalancing a pool which BOOST price is < $1.
https://github.com/sherlock-audit/2024-10-axion/blob/c65e662999d0c79439703fc6713814b4ad023e01/liquidity-amo/contracts/SolidlyV3AMO.sol#L319

totalLiquidity is the amount of "in range" liquidity and it can be much smaller than the whole liquidity spread in the pool.
The whole liquidity is `boostBalance + usdBalance` in case there are not donated amounts of the both tokens. In case there are user donations to the pool, they can additionally prevent the correct pool rebalance, but as this is not profitable and also expensive for an attacker when there is enough liquidity in pool.

Calculated `liquidity` to be removed will be much smaller than the needed in order to move the BOOST price to $1 if the current in range liquidity is not high enough.
```solidity
function _unfarmBuyBurn() internal override returns (uint256 liquidity, uint256 newBoostPrice) {
        uint256 totalLiquidity = ISolidlyV3Pool(pool).liquidity();
        uint256 boostBalance = IERC20Upgradeable(boost).balanceOf(pool);
        uint256 usdBalance = toBoostAmount(IERC20Upgradeable(usd).balanceOf(pool)); // scaled
        if (boostBalance <= usdBalance) revert PriceNotInRange(boostPrice());

        liquidity = (totalLiquidity * (boostBalance - usdBalance)) / (boostBalance + usdBalance);
        liquidity = (liquidity * LIQUIDITY_COEFF) / FACTOR;

        _unfarmBuyBurn(
            liquidity,
            1, // minBoostRemove
            1, // minUsdRemove
            1, // minBoostAmountOut
            block.timestamp + 1 // deadline
        );
```

### Internal pre-conditions

In range liquidity for the current price is low.
### External pre-conditions

N/A
### Attack Path

N/A
### Impact

`unfarmBuyBurn()` wont make the price of BOOST close to 1$.

### PoC

```solidity
it.only("Should execute unfarmBuyBurn successfully when price is above 1", async function() {
        let limitSqrtPriceX96: bigint;
        const zeroForOne = boost2usd;
        if (zeroForOne) {
          limitSqrtPriceX96 = MIN_SQRT_RATIO + BigInt(10);
        } else {
          limitSqrtPriceX96 = MAX_SQRT_RATIO - BigInt(10);
        }
        const boostToBuy = ethers.parseUnits("1200000", 18); //1000000
        await boost.connect(boostMinter).mint(user.address, boostToBuy);
        await boost.connect(user).approve(poolAddress, boostToBuy);
        await pool.connect(user).swap(
          user.address,
          zeroForOne,
          boostToBuy,
          limitSqrtPriceX96,
          1,
          deadline
        );
        
		// Simulate bigger liquidity spread in the pool
        await boost.connect(boostMinter).mint(poolAddress, ethers.parseUnits("10000000", 18));
        await testUSD.mint(poolAddress, ethers.parseUnits("10000000", 6))
        
        // Boost price is lower than 0.9
        expect(await solidlyV3AMO.boostPrice()).to.be.lt(ethers.parseUnits("0.9", 6));
        
        await expect(solidlyV3AMO.connect(amo).unfarmBuyBurn()).to.emit(solidlyV3AMO, "PublicUnfarmBuyBurnExecuted");
        expect(await solidlyV3AMO.boostPrice()).to.be.approximately(ethers.parseUnits(price, 6), 10);
        expect(await boost.balanceOf(amoAddress)).to.be.equal(0);
        expect(await testUSD.balanceOf(amoAddress)).to.be.lt(
          Math.floor(Number(boostToBuy) * (10 ** 6 - Number(usdUsageRatio)) / 10 ** 18));
      });
```

```text
output:
AssertionError: expected 895194 to be close to 1000000 +/- 10
```
### Mitigation

Calculate the needed to be removed liquidity not based current in range liquidity.

# Issue M-9: Liquidity is incorrectly calculated during `unfarmBuyBurn()` for SolidlyV2AMO, causing DoS. 

Source: https://github.com/sherlock-audit/2024-10-axion-judging/issues/286 

## Found by 
pkqs90

### Summary

In SolidlyV2AMO, when performing `unfarmBuyBurn()` and calculating the amount of liquidity to remove, it assumes all the LP tokens are supplied by SolidlyV2AMO. This would result in a overestimated number of liquidity to burn, and may cause DoS or cause the protocol to burn most of it's LP.

### Root Cause

Users can call the public `unfarmBuyBurn()` function to burn liquidity, currently the amount of liquidity to burn is calculated by the following formula: `liquidity = (usdNeeded * totalLp) / usdReserve;`.

However, the main issue here is the `totalLp` means the amount of ALL LP tokens, and not the amount of LP tokens that SolidlyV2AMO owns. It could easily happen that other users are also providing LP for the pool, and the calculated `liquidity` to burn would be too large.

In this case, if the `liquidity` is too large, it would cause an DoS. Or even worse, an attacker can deposit LP, and trick the SolidlyV2AMO to burn nearly all of it's LP tokens, which would be very undesirable.

```solidity
    function _unfarmBuyBurn() internal override returns (uint256 liquidity, uint256 newBoostPrice) {
        (uint256 boostReserve, uint256 usdReserve) = getReserves();

        uint256 usdNeeded = (((boostReserve - usdReserve) / 2) * usdBuyRatio) / FACTOR;

        // @audit-bug: There may be other LP suppliers. Should use the LP tokens that AMO contract owns.

@>      uint256 totalLp = IERC20Upgradeable(pool).totalSupply();
        liquidity = (usdNeeded * totalLp) / usdReserve;

        // Readjust the LP amount and USD needed to balance price before removing LP
        // ( rationale: we first compute the amount of USD needed to rebalance the price in the pool; then first-order adjust for the fact that removing liquidity/totalLP fraction of the pool increases price impact —— less liquidity needs to be removed )
        // liquidity -= liquidity ** 2 / totalLp;

        _unfarmBuyBurn(
            liquidity,
            (liquidity * boostReserve) / totalLp, // the minBoostRemove argument
            toUsdAmount(usdNeeded), // the minUsdRemove argument
            usdNeeded, // the minBoostAmountOut argument
            block.timestamp + 1 // deadline is next block as the computation is valid instantly
        );

        newBoostPrice = boostPrice();
    }
```

- https://github.com/sherlock-audit/2024-10-axion/blob/main/liquidity-amo/contracts/SolidlyV2AMO.sol#L362

### Internal pre-conditions

N/A

### External pre-conditions

N/A

### Attack Path

Attackers can brick the public `unfarmBuyBurn()` function by depositing LP, or even force the SolidlyV2AMO to burn most of it's liquidity.

### Impact

Attackers can deposit LP to make `unfarmBuyBurn()`. This is an important feature to keep Boost/USD pegged, thus a high severity issue.

This is basically no cost for attackers since the Boost/USD will always go back to 1:1 so no impermanent loss is incurred.

### PoC

N/A

### Mitigation

Use the amount of LP tokens that the AMO contract owns instead of totalSupply of LP tokens.

