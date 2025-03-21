# ETAAcademy-Adudit: 2. Uniswap

<table>
  <tr>
    <th>title</th>
    <th>tags</th>
  </tr>
  <tr>
    <td>02. Uniswap</td>
    <td>
      <table>
        <tr>
          <th>audit</th>
          <th>basic</th>
          <th>library</th>
          <td>uniswap</td>
        </tr>
      </table>
    </td>
  </tr>
</table>

[Github](https://github.com/ETAAcademy)｜[Twitter](https://twitter.com/ETAAcademy)｜[ETA-Audit](https://github.com/ETAAcademy/ETAAcademy-Audit)

Authors: [Evta](https://twitter.com/pwhattie), looking forward to your joining

## 1.[High] `_getReferencePoolPriceX96()` will show incorrect price for negative tick deltas in current implementation cause it doesn’t round up for them

### Negative tickCumulativesDelta

- Summary: while the protocol accurately calculates tick deltas using `tickCumulativesDelta =  tickCumulatives[0] - tickCumulatives[1]`, it fails to round down ticks when the delta is negative, as it’s done in the [uniswap library](https://github.com/Uniswap/v3-periphery/blob/main/contracts/libraries/OracleLibrary.sol#L36). This oversight could lead to incorrect prices, reverting, unavailable `[_checkLoanIsHealthy()](https://github.com/code-423n4/2024-03-revert-lend/blob/457230945a49878eefdc1001796b10638c1e7584/src/V3Vault.sol#L702-L703)` , wrong tick and potential arbitrage opportunities.

- Impact & Recommendation: When tick deltas are negative, the protocol should rectify the rounding issue by adding `if (tickCumulatives[0] - tickCumulatives[1] < 0 && (tickCumulatives[0] - tickCumulatives[1]) % secondsAgo != 0) timeWeightedTick --;`**.**
  <br> 🐬: [Source](https://code4rena.com/reports/2024-03-revert-lend#h-05--_getreferencepoolpricex96-will-show-incorrect-price-for-negative-tick-deltas-in-current-implementation-cause-it-doesnt-round-up-for-them) & [Report](https://code4rena.com/reports/2024-03-revert-lend)

  <details><summary>POC</summary>

  ```solidity
      function _getReferencePoolPriceX96(IUniswapV3Pool pool, uint32 twapSeconds) internal view returns (uint256) {
        uint160 sqrtPriceX96;
        // if twap seconds set to 0 just use pool price
        if (twapSeconds == 0) {
            (sqrtPriceX96,,,,,,) = pool.slot0();
        } else {
            uint32[] memory secondsAgos = new uint32[](2);
            secondsAgos[0] = 0; // from (before)
            secondsAgos[1] = twapSeconds; // from (before)
            (int56[] memory tickCumulatives,) = pool.observe(secondsAgos); // pool observe may fail when there is not enough history available (only use pool with enough history!)
            //@audit
            int24 tick = int24((tickCumulatives[0] - tickCumulatives[1]) / int56(uint56(twapSeconds)));
            sqrtPriceX96 = TickMath.getSqrtRatioAtTick(tick);
        }
        return FullMath.mulDiv(sqrtPriceX96, sqrtPriceX96, Q96);
    }

  ```

  </details>

## 2.[High] Reallocation depends on the slot0 price, which can be manipulated

### Slot0 price

- Summary: The vulnerability in Perp.sol's reallocate function allows users to manipulate the slot0 price from Uniswap V3, potentially causing the LP position to be reallocated outside the desired range, leading to yield loss and additional swap fees for the protocol. This issue can be exploited to repeatedly drain funds from liquidity providers.

- Impact & Recommendation: The recommended mitigation is to use the TWAP price instead of the slot0 price. However, this solution might not fully address the issue since both the square root price and the current tick can be manipulated, suggesting that a more robust redesign, potentially involving a privileged role for re-allocations, is necessary.
  <br> 🐬: [Source](https://code4rena.com/reports/2024-05-predy#h-01-reallocation-depends-on-the-slot0-price,-which-can-be-manipulated) & [Report](https://code4rena.com/reports/2024-05-predy)

<details><summary>POC</summary>

```solidity

    function swapForOutOfRange(
        DataType.PairStatus storage pairStatus,
        uint160 _currentSqrtPrice,
        int24 _tick,
        uint128 _totalLiquidityAmount
    ) internal returns (int256 deltaPositionBase, int256 deltaPositionQuote) {
        uint160 tickSqrtPrice = TickMath.getSqrtRatioAtTick(_tick);
        // 1/_currentSqrtPrice - 1/tickSqrtPrice
        int256 deltaPosition0 =
            LPMath.calculateAmount0ForLiquidity(_currentSqrtPrice, tickSqrtPrice, _totalLiquidityAmount, true);
        // _currentSqrtPrice - tickSqrtPrice
        int256 deltaPosition1 =
            LPMath.calculateAmount1ForLiquidity(_currentSqrtPrice, tickSqrtPrice, _totalLiquidityAmount, true);
        if (pairStatus.isQuoteZero) {
            deltaPositionQuote = -deltaPosition0;
            deltaPositionBase = -deltaPosition1;
        } else {
            deltaPositionBase = -deltaPosition0;
            deltaPositionQuote = -deltaPosition1;
        }
        updateRebalancePosition(pairStatus, deltaPosition0, deltaPosition1);
    }

```

</details>

## 3.[High] One pair can steal another pair’s Uniswap liquidity during reallocate() call if both pairs operate on the same Uniswap pool and both have the same upper and lower tick during reallocation

### Liquidity tracking

- Summary: The Predy protocol's reallocation function can inadvertently steal liquidity from one pair to another if both pairs share the same Uniswap pool and have the same upper and lower ticks during reallocation. This vulnerability arises because the protocol does not verify that the liquidity belongs exclusively to a specific pair. As a result, if a user trades gamma on one pair and the price moves outside the threshold of a second pair, reallocating the second pair can wrongfully divert liquidity from the first, disrupting internal accounting and preventing proper position closure. The recommended mitigation involves tracking and reallocating only the liquidity mined within each pair.

- Impact & Recommendation: The recommended mitigation involves tracking and reallocating only the liquidity mined within each pair.
  <br> 🐬: [Source](<https://code4rena.com/reports/2024-05-predy#h-03-One-pair-can-steal-another-pair’s-uniswap-liquidity-during-reallocate()-call-if-both-pairs-operate-on-the-same-uniswap-pool-and-both-have-the-same-upper-and-lower-tick-during-reallocation>) & [Report](https://code4rena.com/reports/2024-05-predy)

<details><summary>POC</summary>

```solidity

// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;
import {TestPool} from "../pool/Setup.t.sol";
import {TestTradeMarket} from "../mocks/TestTradeMarket.sol";
import {IPredyPool} from "../../src/interfaces/IPredyPool.sol";
import {IFillerMarket} from "../../src/interfaces/IFillerMarket.sol";
import {Constants} from "../../src/libraries/Constants.sol";
contract TestPoCReallocate is TestPool {
    TestTradeMarket private tradeMarket;
    address private filler;
    function setUp() public override {
        TestPool.setUp();
        registerPair(address(currency1), address(0));
        registerPair(address(currency1), address(0));
        predyPool.supply(1, true, 1e8);
        predyPool.supply(1, false, 1e8);
        predyPool.supply(2, true, 1e8);
        predyPool.supply(2, false, 1e8);
        tradeMarket = new TestTradeMarket(predyPool);
        filler = vm.addr(12);
        currency0.transfer(address(tradeMarket), 1e8);
        currency1.transfer(address(tradeMarket), 1e8);
        currency0.approve(address(tradeMarket), 1e8);
        currency1.approve(address(tradeMarket), 1e8);
        currency0.mint(filler, 1e10);
        currency1.mint(filler, 1e10);
        vm.startPrank(filler);
        currency0.approve(address(tradeMarket), 1e10);
        currency1.approve(address(tradeMarket), 1e10);
        vm.stopPrank();
    }
    function testPoCReallocateStealFromOtherPair() public {
        // user opens gamma position on pair with id = 1
        {
            IPredyPool.TradeParams memory tradeParams =
                IPredyPool.TradeParams(1, 0, -90000, 100000, abi.encode(_getTradeAfterParams(2 * 1e6)));
            tradeMarket.trade(tradeParams, _getSettlementData(Constants.Q96));
        }
        _movePrice(true, 5 * 1e16);
        // reallocation on pair id = 2 steals from pair id = 1
        assertTrue(tradeMarket.reallocate(2, _getSettlementData(Constants.Q96 * 15000 / 10000)));
        _movePrice(true, 5 * 1e16);
        // reallocation on pair id = 1 can be done but there is 0 liquidity
        assertTrue(tradeMarket.reallocate(1, _getSettlementData(Constants.Q96 * 15000 / 10000)));
        // user can't close his position on pair with id = 1, internal accounting is broken
        {
            IPredyPool.TradeParams memory tradeParams =
                IPredyPool.TradeParams(1, 1, 90000, -100000, abi.encode(_getTradeAfterParams(2 * 1e6)));
            tradeMarket.trade(tradeParams, _getSettlementData(Constants.Q96));
        }
    }
}

```

</details>
