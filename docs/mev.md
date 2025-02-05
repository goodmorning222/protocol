# MEV

This document is intended to serve as a resource for MEV searchers and others looking to interact with the deployed protocol programatically.

## Overview

Like any protocol, the Reserve Protocol causes some amount of MEV. While the full extent is not necessarily described here, there are two obvious examples of MEV opportunities in the protocol:

1. Issuance/Redemption
2. Auctions

### 1. Issuance/Redemption

MEV searchers can arb an RToken's issuance/redemption price against the broader market, whether that be AMM pools or CEX prices. This is a fairly standard MEV opportunity and it works the way an MEV searcher would expect. All that one needs to be able to do to participate is execute `issue()` or `redeem()` on the `RToken.sol` contract. The issuance requires approvals in advance, while the `redeem()` does not. You can find more documentation elsewhere in the repo about the properties of our `issue()`/`redeem()`/`redeemCustom()` functions. In short, they are atomic and work the way a searcher would expect, with the caveat that `redeem()` will revert during rebalancing (`redeemCustom()` does not).

### 2. Auctions

To bid in the protocol's single-lot, atomic, falling-price dutch auctions, an MEV searcher needs to monitor all `Broker` instances associated with RTokens. Whenever a `Broker` emits a `TradeStarted(ITrade indexed trade, IERC20 indexed sell, IERC20 indexed buy, uint256 sellAmount, uint256 minBuyAmount)` event, the `trade.KIND()` can be checked to see what kind of trade it is.

- if trade.KIND() == 0, then it is a [DutchTrade](../contracts/plugins/trading/DutchTrade.sol)
- if trade.KIND() == 1, then it is a [GnosisTrade](../contracts/plugins/trading/GnosisTrade.sol)

#### DutchTrade

Bidding instructions from the `DutchTrade` contract:

`DutchTrade` (relevant) interface:

```solidity
function bid() external; // execute a bid at the current block number

function sell() external view returns (IERC20);

function buy() external view returns (IERC20);

function status() external view returns (uint8); // 0: not_started, 1: active, 2: closed, 3: mid-tx only

function lot() external view returns (uint256); // {qSellTok} the number of tokens being sold

function bidAmount(uint256 blockNumber) external view returns (uint256); // {qBuyTok} the number of tokens required to buy the lot, at a particular block number

```

To participate:

1. Call `status()` view; the auction is ongoing if return value is 1
2. Call `lot()` to see the number of tokens being sold
3. Call `bidAmount()` to see the number of tokens required to buy the lot, at various block numbers
4. After finding an attractive bidAmount, provide an approval for the `buy()` token. The spender should be the `DutchTrade` contract.
   **Note**: it is very important to set tight approvals! Do not set more than the `bidAmount()` for the desired bidding block else reorgs present risk.
5. Wait until the desired block is reached (hopefully not in the first 40% of the auction)
6. Call `bid()`. If someone else completes the auction first, this will revert with the error message "bid already received". Approvals do not have to be revoked in the event that another MEV searcher wins the auction. (Though ideally the searcher includes the approval in the same tx they `bid()`)

For a sample price curve, see [docs/system-design.md](./system-design.md#sample-price-curve)

#### GnosisTrade

`GnosisTrade.sol` implements a batch auction on top of Gnosis's [EasyAuction](https://github.com/gnosis/ido-contracts/blob/main/contracts/EasyAuction.sol) platform. In general a batch auction is designed to minimize MEV, and indeed that's why it was chosen in the first place. Both types of auctions (batch + dutch) can be opened at anytime, but the expectation is that dutch auctions will be preferred by MEV searchers because they are more likely to be profitable.

However, if a batch auction is launched, an MEV searcher may still be able to profit. In order to bid in the auction, the searcher must call `function placeSellOrders(uint256 auctionId, uint96[] memory _minBuyAmounts, uint96[] memory _sellAmounts, bytes32[] memory _prevSellOrders, bytes calldata allowListCallData)`, providing an approval in advance. This call will escrow `_sellAmounts` tokens in EasyAuction for the remaining duration of the auction. Once the auction is over, anyone can settle the auction directly in EasyAuction via `settleAuction(uint256 auctionId)`, or by calling `settleTrade(IERC20 erc20)` on the `ITrading` instance in our system that started the trade (either BackingManager or a RevenueTrader).

Since the opportunity is not atomic, it is not likely MEV searchers will be very interested in this option. Still, there may be batch auctions that clear with money left on the table, so it is worth mentioning.

**Note**: Atomic settlement will always be set to disabled in EasyAuction, which makes the MEV opportunity further unattractive.
