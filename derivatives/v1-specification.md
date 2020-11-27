# Equity Protocol 1.0 specification

## Table of contents

1. [Overview](#overview)
1. [Key Terms](#key-terms)
	1. [Perpetual Swap](#perpetual-swap)
	1. [CFD](#cfd)
	1. [Address](#address)
	1. [Account](#account)
	1. [Deposits](#deposits)
	1. [Market](#market)
	1. [Direction](#direction)
	1. [Index Price](#index-price)
	1. [Contract Price](#contract-price)
	1. [Quantity](#quantity)
	1. [Position](#position)
	1. [Margin](#margin)
	1. [Contract](#contract)
	1. [Order](#order)
	1. [Funding](#funding)
	1. [Funding Rate](#funding-rate)
	1. [Funding Fee](#funding-fee)
	1. [Liquidation](#liquidation)
	1. [Initial Margin Requirement](#initial-margin-requirement)
	1. [Maintenance Margin Requirement](#maintenance-margin-requirement)
	1. [Clawback](#clawback)
	1. [Leverage](#lLeverage)
1. [Orders](#make-orders)
	1. [Order Message Format](#order-message-format)
	1. [Order Validation](#order-validation)
	1. [Filling Orders](#take-orders)
	1. [Matching Orders](#matching-orders)
1. [Positions](#positions)
	1. [Closing a Position](#closePosition)
	1. [Liquidating a Position](#liquidatepositionwithorders)
1. [Transaction Fees](#transaction-fees)
1. [Oracle](#oracle)
# Overview
This protocol enables traders to create, enter into, and execute decentralized perpetual swap contracts on any arbitrary market.

Our design adopts an account balance model where all positions are publicly recorded with their respective accounts. Architecturally, the core logic is implemented on on-chain smart contracts while the order matching is done through on the Equity Chain \(in an off-chain relay on-chain settlement fashion\).

In our system, users have **accounts** which manage their **positions** in one or more futures **markets.** Each futures market specifies the bilateral futures contract parameters and terms, including most notably the margin ratio and oracle. Buyers and sellers of contracts in a futures market coordinate off-chain and then enter into contracts through an on-chain transaction. At all times, the payout \(NPV\) of long positions are balanced with corresponding short positions. The positions held by an account are subject to liquidation when the NAV of the account becomes negative.

While a market is live, market-wide actions may affect all positions in the market such as dynamic funding rate adjustment \(to balance market long/shorts\) as well as clawbacks in rare scenarios.

# Key Terms

## **Perpetual Swap**

A derivative providing exposure to the value of an underlying asset which mimics a margin-based spot market and has no expiry/settlement.

## **CFD**

Contract for difference which is a derivative stipulating that the sellers will pay the buyers the difference between the current value of an asset and its value at the contract's closing time.

## **Address**

A secp256k1 based address. It's important to differentiate an address from an account in the futures protocol.

## **Account**

An address can own one or more accounts which hold positions. Isolated margin positions are each held by their own account while cross margined positions are held by the same account.

## **Deposits**

The value of an account's aggregate deposits denominated in a specified base currency \(e.g. USDT\).

## **Market**

A market is defined by its supported base currency, oracle, and minimum margin ratio / margin requirement. The market may also support a unique funding rates calculation, which is a distinctive feature of a perpetual swap futures market.

## **Direction**

Long or Short.

## **Index Price**

The reference spot index price \(`indexPrice`\) of the underlying asset that the derivative contract derives from. This value is obtained from an oracle.


## **Contract Price**

The value of the futures contract the position entered/created \(`contractPrice`\). This is purely market driven and is set by individuals, independent of the oracle price.


## **Quantity**

The quantity of contracts (`quantity`). Must be a whole number.

## **Position**

Executing a contract creates two opposing positions \(one long and one short\) which reflects the ownership conditions of some quantity of contracts.

A cross margined account can hold multiple positions while an isolated margined account holds one position.

## **Margin**

Margin refers to the amount of base currency that a trader posts as collateral for a given position.

## **Contract**

A contract refers to the single unit of account for a position in a given direction in a given market. There are two sides of a contract \(long or short\) that one can enter into.

## **Order**

A cryptographically signed message expressing an unilateral agreement to enter into a position under certain specified terms \(i.e. make order or take order\).

## **Funding**

Funding refers to the periodic payments exchanged between the traders that are long or short of a perpetual contract at the end of every funding epoch \(e.g. every 8 hours\). When the funding rate is positive, longs pay shorts. When negative, shorts pay longs.

## **Funding Rate**

The funding rate value determines the funding fee that positions pay and is based on the difference between the price of the contract in the perpetual swap markets and spot markets.

## **Funding Fee**

The funding fee `f_i` refers to the funding payment that is made for a **single contract** in the market for a given epoch `i`. When a position is created, the position records the current epoch which is noted as the `entry` epoch. 

Cumulative funding `F` refers to the cumulative funding for the contract since position entry. `F_entry  = f_entry + ... + f_current`

## **NPV**

Net Present Value of a single contract. For long and short perpetual contracts, the NPV calculation is:
- `NPV_long = indexPrice - contractPrice - F_entry`
- `NPV_short = - NPV_long = contractPrice - indexPrice + F_entry`

## **NAV**

The Net Asset Value for an account. Equals the sum of the net position value over all of the account's positions. 

`NAV = sum(quantity * NPV + margin - indexPrice * maintenanceMarginRatio)`

## **Liquidation**

When an account's **NAV** becomes negative, all of its positions are subject to liquidation, i.e. the forced closure of the position due to a maintenance margin requirement being breached.

<!-- ## **Liquidation Penalty**

The liquidation penalty (`liquidationPenalty`) is a fixed percentage defining the value of the liquidated position that is paid out. Each market can define its own liquidation penalty \(e.g. 3%\) but every market's liquidation penalty must be greater than the global minimum liquidation penalty.
 -->

## **Maintenance Margin Requirement**

The maintenance margin requirement refers to the minimum amount of margin that a position must maintain after being established. If this requirement is breached, the position is subject to liquidation.

Throughout the lifetime of a position, each contract must satisfy the following margin requirement:

`margin / quantity >= indexPrice * maintenanceMarginRatio - NPV`

## **Initial Margin Requirement**

When a position is first created, the amount of collateral supplied as margin must satisfy the initial margin requirement. This margin requirement is stricter than the maintenance margin requirement and exists in order to reduce the risk of immediate liquidation.

`initialMarginRatio = maintenanceMarginRatio + initialMarginRatioFactor`

Upon position creation, each contract must satisfy the following margin requirement:

`margin / quantity >= max(contractPrice * initialMarginRatio, indexPrice * initialMarginRatio - NPV`

## Liquidation Price

The liquidation price is the price at which a position can be liquidated and can be derived from the maintenance margin requirement formula. 

For longs:

`contractPrice = indexPrice - indexPrice * maintenanceMarginRatio + margin / quantity - F_entry`

For shorts:

`contractPrice =  indexPrice + indexPrice * maintenanceMarginRatio - margin / quantity  - F_entry`

## **Clawback**

A clawback event occurs when a threshold number of accounts cannot be liquidated with a net-zero final payout. In practice, this happens when a chain reaction of unidirectional liquidation cannot be liquidated with the position margin due to volatility or lack of liquidity.

## **Leverage**

Although leverage is not explicitly defined in our protocol, the amount of margin a trader uses to collateralize a position is a function of leverage according to the following formula:

`margin = quantity * max(contractPrice / leverage, indexPrice / leverage - NPV)`


For new positions, the funding fee component of the NPV formula can removed since the position has not been created yet, resulting in the following NPV calculations:

- `NPV_long = indexPrice - contractPrice` 
- `NPV_short = contractPrice - indexPrice`

# Make Orders

In the Equity Perpetuals Protocol, there are two main types of orders: maker orders and taker orders. **Make orders** are stored on Equity's decentralized orderbook on the Equity Chain while **Take orders** are immediately executed against make orders on the Equity Perpetuals Contract.

## **Order Message Format**

The Equity Perpetuals Protocol leverages the [0x Order Message format](https://github.com/0xProject/0x-protocol-specification/blob/master/v3/v3-specification.md#order-message-format) for the external interface to represent a make order for a derivative position, i.e. a cryptographically signed message expressing an agreement to enter into a derivative position under specified parameters.

A make order message consists of the following parameters:

| Parameter | Type | Description |
| :--- | :--- | :--- |
| makerAddress | address | Address that created the order. |
| takerAddress | address | Empty. |
| feeRecipientAddress | address | Address of the recipient of the order transaction fee. |
| senderAddress | address | Empty. |
| makerAssetAmount | uint256 | The contract price \(`contractPrice`\), i.e. the price of 1 contract denominated in base currency. |
| takerAssetAmount | uint256 | The `quantity` of contracts the maker seeks to obtain. |
| makerFee | uint256 | The amount of `margin` denoted in base currency the maker would like to post/risk for the order. |
| takerFee | uint256 | Empty. |
| expirationTimeSeconds | uint256 | Timestamp in seconds at which order expires. |
| salt | uint256 | Arbitrary number to facilitate uniqueness of the order's hash. |
| makerAssetData | bytes | The first 32 bytes contain the `marketID` of the market for the position if the order is LONG, empty otherwise.  Right padded with 0's to be 36 bytes |
| takerAssetData | bytes | The first 32 bytes contain the `marketID` of the market for the position if the order is SHORT, empty otherwise.  Right padded with 0's to be 36 bytes |
| makerFeeAssetData | bytes | Empty. |
| takerFeeAssetData | bytes | Empty. |

In a given perpetual market specified by `marketID`, an order encodes the willingness to purchase `quantity` contracts in a given direction \(long or short\) at a specified contract price `contractPrice`  using a specified amount of `margin` of base currency as collateral.

## Order Validation

### getOrderRelevantState

`getOrderRelevantState` can be used to validate an order before use with a given index price. 

```solidity
/// @dev Fetches all order-relevant information needed to validate if the supplied order is fillable.
/// @param order The order structure
/// @param signature Signature provided by maker that proves the order's authenticity.
/// @param indexPrice The index price to use as a reference. If 0, use the market's existing index price.
/// @return The orderInfo (hash, status, and `takerAssetAmount` already filled for the given order),
/// fillableTakerAssetAmount (amount of the order's `takerAssetAmount` that is fillable given all on-chain state),
/// and isValidSignature (validity of the provided signature).
function getOrderRelevantState(
    LibOrder.Order memory order,
    bytes memory signature,
    uint256 indexPrice
)
    public
    view
    returns (
        LibOrder.OrderInfo memory orderInfo,
        uint256 fillableTakerAssetAmount,
        bool isValidSignature
    )
{
```

Logic:

Calling `getOrderRelevantState` will perform the following steps:

1. Set the hash of the `order`  in `orderInfo.orderHash`
2. Set the quantity of contracts of the `order` that have been filled in `orderInfo.orderTakerAssetFilledAmount`
3. Set the order status of the order in `orderInfo.orderStatus` according to the following conditions:
   1. If the order has been cancelled, the `orderStatus` will be `CANCELLED`.
   2. If the order has been fully filled \(i.e. `orderInfo.orderTakerAssetFilledAmount` equals `order.takerAssetAmount`, the `orderStatus` will be `FULLY_FILLED`.
   3. If the order has expired, the `orderStatus` will be `EXPIRED`.
   4. If the order's margin \(`order.makerFee`\) does not satisfy the [initial margin requirement](keyterms.md#initial-margin-requirement), the `orderStatus` will be `INVALID_MAKER_ASSET_AMOUNT`. Note: the index price used in this calculation is the inputted `indexPrice`.
   5. Otherwise if the order is fillable, the `orderStatus` will be `FILLABLE` and the `fillableTakerAssetAmount` to equal the quantity of contracts that remain fillable \(i.e. `order.takerAssetAmount - orderInfo.orderTakerAssetFilledAmount`\).
4. Check whether or not the signature is valid and set `isValidSignature` to true if valid. Otherwise, set to false and set the `orderStatus` to `INVALID`.
5. If the `orderStatus` from the previous steps is `FILLABLE`, check that the order maker has sufficient balance of baseCurrency in his freeDeposits \(his `availableMargin`\) to fill `fillableTakerAssetAmount` contracts of the order.
   * If the maker does not have at least `order.makerFee * fillableTakerAssetAmount / order.takerAssetAmount + fillableTakerAssetAmount * order.makerAssetAmount * makerTxFee` amount of balance to the fill the order, `fillableTakerAssetAmount` is set to the maximum quantity of contracts fillable which equals `availableMargin * order.TakerAssetAmount / order.MakerFee + order.TakerAssetAmount * makerTxFee`. If this value is zero, the `orderStatus` will be `INVALID_TAKER_ASSET_AMOUNT`.

### getOrderRelevantStates

`getOrderRelevantStates` can be used to validate multiple orders before use. 

```solidity
/// @dev Fetches all order-relevant information needed to validate if the supplied orders are fillable.
/// @param orders Array of order structures
/// @param signatures Array of signatures provided by makers that prove the authenticity of the orders.
/// @return The ordersInfo (array of the hash, status, and `takerAssetAmount` already filled for each order),
/// fillableTakerAssetAmounts (array of amounts for each order's `takerAssetAmount` that is fillable given all on-chain state),
/// and isValidSignature (array containing the validity of each provided signature).
/// NOTE: Expects each of the orders to be of the same marketID, otherwise may potentially return relevant states for orders of differing marketID's using a stale price
function getOrderRelevantStates(LibOrder.Order[] memory orders, bytes[] memory signatures)
  public
  view
  returns (
    LibOrder.OrderInfo[] memory ordersInfo,
    uint256[] memory fillableTakerAssetAmounts,
    bool[] memory isValidSignature
    );
```

**Logic**

Calling `getOrderRelevantStates` will result in sequentially calling `getOrderRelevantState` with the current index price obtained from the oracle.

### getMakerOrderRelevantStates

`getMakerOrderRelevantStates` can be used to validate multiple orders from a single maker before use. 

```solidity
/// @dev Fetches all order-relevant information needed to validate if the supplied orders are fillable.
/// @param orders Array of order structures
/// @param signatures Array of signatures provided by makers that prove the authenticity of the orders.
/// @param makerAddress Address of maker to check.
/// @return The ordersInfo (array of the hash, status, and `takerAssetAmount` already filled for each order),
/// fillableTakerAssetAmounts (array of amounts for each order's `takerAssetAmount` that is fillable given all on-chain state),
/// isValidSignature (array containing the validity of each provided signature), and availableMargin (amount of available
/// base currency usable as margin after margin needs of the `orders` are satisfied).
/// NOTE: Expects each of the orders to be of the same marketID, otherwise may potentially return relevant states for orders of differing marketID's using a stale price
function getMakerOrderRelevantStates(
    LibOrder.Order[] memory orders,
    bytes[] memory signatures,
    address makerAddress
)
    public
    view
    returns (
        LibOrder.OrderInfo[] memory ordersInfo,
        uint256[] memory fillableTakerAssetAmounts,
        bool[] memory isValidSignature,
        uint256 availableMargin
    )
```

### OrderInfo

```solidity
struct OrderInfo {
    uint8 orderStatus;                    // Status that describes order's validity and fillability.
    bytes32 orderHash;                    // EIP712 hash of the order (see LibOrder.getOrderHash).
    uint256 orderTakerAssetFilledAmount;  // Amount of order that has already been filled.
}
```

### Order Status

```solidity
enum OrderStatus {
    INVALID,
    INVALID_MAKER_ASSET_AMOUNT,
    INVALID_TAKER_ASSET_AMOUNT,
    FILLABLE,
    EXPIRED,
    FULLY_FILLED,
    CANCELLED
}
```


# Take Orders

Orders can be filled by calling the following methods on the `EquityFutures` contract

## fillOrder

This is the most basic way to fill an order. All of the other methods call `fillOrder` under the hood with additional logic. This function will attempt to execute `quantity` contracts of the `order` specified by the caller. However, if the remaining fillable amount is less than the `quantity` specified, the remaining amount will be filled. Partial fills are allowed when filling orders.

```solidity
/// @dev Fills the input order.
/// @param order The make order to be executed.
/// @param quantity Desired quantity of contracts to execute.
/// @param margin Desired amount of margin (denoted in baseCurrency) to use to fill the order.
/// @param signature The signature of the order signed by maker.
/// @return fillResults
function fillOrder(
    LibOrder.Order memory order,
    uint256 quantity,
    uint256 margin,
    bytes memory signature
) external returns (FillResults memory);
```

**Logic**

Calling `fillOrder` will perform the following steps:

1. Query the oracle to obtain the most recent price and funding fee.
2. Query the state and status of the order with [`getOrderRelevantState`](keyterms.md#getorderrelevantstate).
3. Revert if the orderStatus is not `FILLABLE`. 
4. Create the Maker's Position.
   1. If the order has been used previously, execute funding payments on the existing position and then update the existing position state. Otherwise, create a new account with a corresponding new position for the maker and log a `FuturesPosition` event.
   2. Transfer `fillResults.makerMarginUsed + fillResults.feePaid` of base currency from the maker to the contract to create \(or add to\) the maker's position.
   3. Allocate `fillResults.makerMarginUsed` for the new position margin and allocate `relayerFeePercentage` of the `fillResults.makerFeePaid` to the fee recipient \(if specified\) and the remaining `fillResults.feePaid` to the insurance pool.
5. Create the Taker's position.
   1. Create a new account with a corresponding new position for the taker and log a `FuturesPosition` event.
   2. Transfer `margin + fillResults.takerFeePaid` of base currency from the taker to the contract to create the taker's position.  
   3. Allocate  `margin`  for the new position margin and allocate `relayerFeePercentage` of the `fillResults.takerFeePaid` to the fee recipient \(if specified\) and the remaining `fillResults.takerFeePaid` to the insurance pool.

## fillOrKillOrder

`fillOrKillOrder` can be used to fill an order while guaranteeing that the specified amount will either be filled or the call will revert.

```solidity
/// @dev Fills the input order. Reverts if exact quantity not filled
/// @param order The make order to be executed.
/// @param quantity Desired quantity of contracts to execute.
/// @param margin Desired amount of margin (denoted in baseCurrency) to use to fill the order.
/// @param signature The signature of the order signed by maker.
/// return results The fillResults
function fillOrKillOrder(
  LibOrder.Order memory order,
  uint256 quantity,
  uint256 margin,
  bytes memory signature
) external returns (FillResults memory fillResults);
```

**Logic**

Calling `fillOrKillOrder` will perform the following steps:

1. Call `fillOrder` with the passed in inputs
2. Revert if `fillResults.quantityFilled` does not equal the passed in `quantity`

## batchFillOrders

`batchFillOrders` can be used to fill multiple orders in a single transaction.

```text
/// @dev Executes multiple calls of fillOrder.
/// @param orders The make order to be executed.
/// @param quantities Desired quantity of contracts to execute.
/// @param margins Desired amount of margin (denoted in baseCurrency) to use to fill the order.
/// @param signatures The signature of the order signed by maker.
/// return results The fillResults
function batchFillOrders(
  LibOrder.Order[] memory orders,
  uint256[] memory quantities,
  uint256[] memory margins,
  bytes[] memory signatures
) external returns (FillResults[] memory results); 
```

**Logic**

Calling `batchFillOrders` will perform the following steps:

1. Sequentially call `fillOrder` for each element of `orders`, passing in the order, fill amount, and signature at the same index.

## batchFillOrKillOrders

`batchFillOrKillOrders`can be used to fill multiple orders in a single transaction while guaranteeing that the specified amounts will either be filled or the call will revert.

```solidity
/// @dev Executes multiple calls of fillOrKill orders.
/// @param orders The make order to be executed.
/// @param quantities Desired quantity of contracts to execute.
/// @param margins Desired amount of margin (denoted in baseCurrency) to use to fill the order.
/// @param signatures The signature of the order signed by maker.
/// return results The fillResults
function batchFillOrKillOrders(
  LibOrder.Order[] memory orders,
  uint256[] memory quantities,
  uint256[] memory margins,
  bytes[] memory signatures
) external returns (FillResults[] memory results)
```

**Logic**

Calling `batchFillOrKillOrders` will perform the following steps:

1. Sequentially call `fillOrder` for each element of `orders`, passing in the order, fill amount, and signature at the same index. 
2. Revert if any of the `fillOrder` calls do not fill the entire quantity passed. 

## batchFillOrdersSinglePosition

`batchFillOrdersSinglePosition` can be used to fill multiple orders in a single transaction while creating just one opposing position for the taker. 

```solidity
/// @dev Executes multiple calls of fillOrder but creates only one position for the taker.
/// @param orders The make order to be executed.
/// @param quantities Desired quantity of contracts to execute.
/// @param margins Desired amount of margin (denoted in baseCurrency) to use to fill the order.
/// @param signatures The signature of the order signed by maker.
/// return results The fillResults
function batchFillOrdersSinglePosition(
  LibOrder.Order[] memory orders,
  uint256[] memory quantities,
  uint256[] memory margins,
  bytes[] memory signatures
) external returns (FillResults[] memory results)
```

**Logic**

Calling `batchFillOrdersSinglePosition` will perform the same steps as `batchFillOrders` but will reuse the same taker position to create the opposing position for each order. 

## batchFillOrKillOrdersSinglePosition

`batchFillOrKillOrdersSinglePosition` can be used to

```solidity
/// @dev Executes batchFillOrKillOrders but creates only one position for the taker.
/// @param orders The make order to be executed.
/// @param quantities Desired quantity of contracts to execute.
/// @param margins Desired amount of margin (denoted in baseCurrency) to use to fill the order.
/// @param signatures The signature of the order signed by maker.
/// return results The fillResults
function batchFillOrKillOrdersSinglePosition(
  LibOrder.Order[] memory orders,
  uint256[] memory quantities,
  uint256[] memory margins,
  bytes[] memory signatures
) external returns (FillResults[] memory results)
```

**Logic**

Calling `batchFillOrKillOrdersSinglePosition` will perform the same steps as `batchFillOrdersSinglePosition` but will revert if each of the quantities specified is not filled. 

## **marketOrders**

`marketOrders` can be used to can be used to purchase a specified quantity of contracts of a derivative by filling multiple orders while guaranteeing that no individual fill throws an error. Note that this function does not enforce that the entire quantity is filled. The input orders should be sorted from best to worst price.

```solidity
/// @dev marketOrders executes the orders sequentially using `fillOrder` until the desired `quantity` is reached or until all of the margin provided is used.
/// @param orders Array of order specifications.
/// @param quantity Desired quantity of contracts to execute.
/// @param margin Desired amount of margin (denoted in baseCurrency) to use to fill the orders.
/// @param signatures Proofs that orders have been signed by makers.
function marketOrders(
	LibOrder.Order[] memory orders,
	uint256 quantity,
	uint256 margin,
	bytes[] memory signatures
) public returns (FillResults[] memory results)
```

**Logic**

Calling `marketOrders` will perform the following steps:

1. Sequentially call `fillOrder` while decrementing the `quantity` and `margin` executed after each fill until the quantity is fully filled, the margin is exhausted, or all of the orders have been executed. 

## **marketOrdersOrKill**

`marketOrders` can be used to can be used to purchase a specified quantity of contracts of a derivative by filling multiple orders while guaranteeing that no individual fill throws an error. Note that this function enforces that the entire quantity is filled. The input orders should be sorted from best to worst price.

```solidity
/// @dev marketOrdersOrKill performs the same steps as `marketOrders` but reverts if the inputted `quantity` of contracts are not filled
/// @param orders Array of order specifications.
/// @param quantity Desired quantity of contracts to execute.
/// @param margin Desired amount of margin (denoted in baseCurrency) to use to fill the orders.
/// @param signatures Proofs that orders have been signed by makers.
function marketOrdersOrKill(
  LibOrder.Order[] memory orders,
  uint256 quantity,
  uint256 margin,
  bytes[] memory signatures
) public returns (FillResults[] memory results);
```

**Logic**

Calling `marketOrdersOrKill` will perform the same steps as `marketOrders` but will revert if the entire `quantity` is not filled. 

# Matching Orders

Two orders of opposing directions can directly be matched if they have a negative spread.

## matchOrders

`matchOrders` can be used to atomically fill 2 orders without requiring the taker to hold any capital. This function is optimized for creator of the `rightOrder`.

```solidity
/// @dev Matches the input orders.
/// @param leftOrder The order to be settled.
/// @param rightOrder The order to be settled.
/// @param leftSignature The signature of the order signed by maker.
/// @param rightSignature The signature of the order signed by maker.
function matchOrders(
  LibOrder.Order memory leftOrder,
  LibOrder.Order memory rightOrder,
  bytes memory leftSignature,
  bytes memory rightSignature
) external;
```

**Logic**

Calling `matchOrders` will perform the following steps: TODO

## multiMatchOrders

`multiMatchOrders` can be used to match a set of orders with another opposing order with negative spread, resulting in the creation of just one position for the creator of the `rightOrder`. This function is optimized for creator of the `rightOrder`.

```solidity
/// @dev Matches the input orders and only creates one position for the `rightOrder` maker.
/// @param leftOrders The orders to be settled.
/// @param rightOrder The order to be settled.
/// @param leftSignatures The signatures of the order signed by maker.
/// @param rightSignature The signature of the order signed by maker.
function multiMatchOrders(
  LibOrder.Order[] memory leftOrders,
  LibOrder.Order memory rightOrder,
  bytes[] memory leftSignatures,
  bytes memory rightSignature
) external;
```

**Logic**

Calling `multiMatchOrders` will sequentially call `matchOrders`.

## batchMatchOrders

`batchMatchOrders` can be used to match 2 sets of an arbitrary number of orders with each other using the same matching strategy as [`matchOrders`](https://github.com/0xProject/0x-protocol-specification/blob/master/v3/v3-specification.md#matchorders).

```solidity
/// @dev Matches the input orders.
/// @param leftOrders The orders to be settled.
/// @param rightOrder The order to be settled.
/// @param leftSignatures The signatures of the order signed by maker.
/// @param rightSignature The signature of the order signed by maker.
function batchMatchOrders(
  LibOrder.Order[] memory leftOrders,
  LibOrder.Order[] memory rightOrders,
  bytes[] memory leftSignatures,
  bytes[] memory rightSignatures
) external;
```

**Logic**

Calling `batchMatchOrders` will sequentially call `matchOrders`. 

# Transaction Fees

The business model for traditional derivatives exchanges \(e.g. BitMEX, Binance, FTX, etc.\) is to take a transaction fee from the each trade. However, because Equity Protocol is designed as a fully decentralized network, we do not follow this approach.

Instead, transaction fees paid in Equity Protocol are used for two purposes.

1. As compensation for relayers for driving liquidity to the protocol
2. As auction collateral for a "buyback and burn" process. In this token economic model, transaction fees collected over a fixed period of time \(e.g. 2 weeks\) are aggregated and then auctioned off to the public for the INJ token. The INJ received from this auction is then permanently burned. 

There are two types of transaction fees that should be introduced: **maker fees** \(for limit or "maker" orders\) and **taker fees** \(for market or "taker" orders\). For both maker and taker fees, transaction fees are to be extracted from the **notional value** of the trade.

## Maker Fees

Maker fees are fees that makers of limit orders pay. These are the fees paid by the order makers \(found in the `makerAddress` parameter of each order\). Limit orders \(i.e. the orders from which maker fees apply\) are `leftOrders` and `rightOrder` in the `multiMatchOrders` function, `orders` in the `marketOrders` function, `order` in the `fillOrder` function, `order` in the `closePosition` function, and `orders` in the `closePositionWithOrders` function.

**Notional Value of Maker Orders**

Recall that within a given perpetual market, an order encodes the willingness to purchase up to `quantity` contracts in a given direction \(long or short\) at a specified `contractPrice` using a specified amount of `margin` of base currency as collateral.

The notional value of a single contract is simply the `contractPrice` \(or $P\_{contract}$\). Hence, the notional value of $n$ contracts is simply `n * contractPrice`. Note that orders can be partially filled so `n` must be less than or equal to `quantity`.

This the calculation for notional value that you will need to use for`leftOrders` and `rightOrder`\* in `multiMatchOrders`, `orders` in the `marketOrders` function, `order` in the `fillOrder` function, `order` in the `closePosition` function, and `orders` in the `closePositionWithOrders` function.

* Note: the `contractPrice` for the `rightOrder` should be the weighted average `contractPrice` of the `leftOrders`. 

## Taker Fees

Taker fees are the fees paid by the `msg.sender` \(the taker\) in the `fillOrder` and `marketOrders` calls.

**Notional Value for Taker Orders**

The notional value calculation in `fillOrder` for the taker is simply `notional = quantity * contractPrice`.

The notional value calculation in `marketOrders` is the sum of the notional values for the quantity of contracts taken in each of the individual orders i.e. `notional = quantity_1 * contractPrice_1 + quantity_2 * contractPrice_2 + ... + quantity_n * contractPrice_n`. Note that $\sum \limits\_{i=1}^{n} quantity\_i$ must equal `quantity`.

The notional value calculation in `closePosition` and `closePositionWithOrders` for the taker \(i.e. the closer/msg.sender\) is `notional = quantity * avgContractPrice` \(`avgContractPrice` is already defined in the contract\).

**Maker Orders Transaction Fees**

Currently, whenever `n` contracts of a maker `order` are executed, the proportional amount of `margin` \(where `margin = order.makerFee * n / order.takerAssetAmount`\) is transferred from the user's base currency ERC-20 balance to the perpetuals contract.

With the introduction of the maker order fee, executing `n` contracts of the `order` should result in the trader to post `margin + txFee` where `txFee = notional * MAKER_FEE_PERCENT / 10000` where `notional = n * order.makerAssetAmount`and where `MAKER_FEE_PERCENT` refers to the digits of the maker fee percentage scaled by 10000 \(i.e a 0.15% maker fee would be 15\).

**Taker Orders Transaction Fees**

Similarly, with the introduction of the taker order fee, executing `n` contracts of the `order` should result in the trader posting `margin + txFee` where `txFee = notional * TAKER_FEE_PERCENT / 10000` where `TAKER_FEE_PERCENT` refers to the digits of the taker fee percentage scaled by 10000 \(i.e a 0.25% taker fee would be 25\) and where notional is described in the **Notional Value for Taker Orders** section.

**Transaction Fee Distribution**

For a given `order` with a transaction fee of `txFee`, if the `feeRecipientAddress` is defined, `txFee * RELAYER_FEE_PROPORTION / 100` should be distributed to the `feeRecipientAddress`'s balance. and the remainder should be distributed to the auction contract's balance. Note: "distributed to balance" in this case does not mean an ERC-20 transfer but rather incrementing the recipient's internal balance in the perpetuals contract from which the recipient can withdraw from in the future \(this withdrawal functionality already exists\).


# Positions

## closePosition

```solidity
/// @dev Closes the input position.
/// @param positionID The positionID of the position being closed
/// @param orders The orders to use to settle the position
/// @param quantity The quantity of contracts being used in the order
/// @param signatures The signatures of the orders signed by makers.
function closePosition(
    uint256 positionID,
    LibOrder.Order[] memory orders,
    uint256 quantity,
    bytes[] memory signatures
) external (PositionResults[] memory pResults, CloseResults memory cResults); 
```

**Logic**

Calling `closePosition` will perform the following steps:

1. Query the oracle to obtain the most recent price and funding fee.
2. Execute funding payments on the existing position and then update the existing position state.
3. Check that the existing `position` (referenced by `positionID`) is valid and can be closed. 
4. Create the Makers' Positions.
   1.  For each order `i`:
      1. If the order has been used previously, execute funding payments on the existing position and then update the existing position state. Otherwise, create a new account with a corresponding new position with the `pResults[i].quantity` contracts for the maker and log a `FuturesPosition` event.
      2. Transfer `pResults[i].marginUsed + pResults[i].fee` of base currency from the maker to the contract to create (or add to) the maker's position.
      3. Allocate `pResults.marginUsed` for the new position margin and allocate `relayerFeePercentage` of the `pResults.fee` to the fee recipient (if specified) and the remaining `pResults.fee` to the insurance pool.
5. Close the `cResults.quantity` quantity conracts of the existing position. 
   1. Calculate the PNL per contract (`contractPNL`) which equals `averageClosingPrice - position.contractPrice` for longs and ` position.contractPrice - averageClosingPrice ` for shorts, where:
      * `averageClosingPrice = (orders[i].contractPrice * results[0].quantity + orders[n-1].contractPrice * results[n-1].quantity)/(cResults.quantity)` 
      * `cResults.quantity = results[0].quantity + ... + results[n-1].quantity` . Note that `cResults.quantity <= quantity`. 
   2. Transfer `cResults.payout` to the owner of the `position` and update the position state. 
      * `cResults.payout = cResults.quantity * (position.margin / position.quantity + contractPNL) ` 
   3. Log a `FuturesClose` event. 

## closePositionOrKill
```solidity
/// @dev Closes the input position and revert if the entire `quantity` of contracts cannot be closed.
/// @param positionID The positionID of the position being closed
/// @param orders The orders to use to settle the position
/// @param quantity The quantity of contracts being used in the order
/// @param signatures The signatures of the orders signed by makers.
function closePositionOrKill(
  uint256 positionID,
  LibOrder.Order[] memory orders,
  uint256 quantity,
  bytes[] memory signatures
) external returns (PositionResults[] memory pResults, CloseResults memory cResults);
```

**Logic**

Calling `closePositionOrKill` will perform the same steps as `closePosition` but will revert if the entire `quantity` of contracts inputted cannot be closed. 

# Liquidation

## liquidatePositionWithOrders 

```jsx
/// @dev Closes the input position.
/// @param positionID The ID of the position to liquidate.
/// @param quantity The quantity of contracts of the position to liquidate.
/// @param orders The orders to use to liquidate the position.
/// @param signatures The signatures of the orders signed by makers.
function liquidatePositionWithOrders(
  uint256 positionID,
  uint256 quantity,
  LibOrder.Order[] memory orders,
  bytes[] memory signatures
) external returns (PositionResults[] memory pResults, LiquidateResults memory lResults){
```

**Logic**

Calling `liquidatePositionWithOrders` will perform the following steps:

1. Query the oracle to obtain the most recent price and funding fee.
2. Execute funding payments on the existing position and then update the existing position state.
3. Check that the existing `position` (referenced by `positionID`) is valid and can be liquidated (i.e. that the [maintenance margin requirement](keyterms.md#maintenance-margin-requirement) is breached. 
4. Create the Makers' Positions. 
	1. For each order `i`:
		1. If the order has been used previously, execute funding payments on the existing position and then update the existing position state. Otherwise, create a new account with a corresponding new position with the `pResults[i].quantity` contracts for the maker and log a `FuturesPosition` event.`
		2. Transfer `pResults[i].marginUsed + pResults[i].fee` of base currency from the maker to the contract to create (or add to) the maker's position.
		3. Allocate `pResults.marginUsed` for the new position margin and allocate `relayerFeePercentage` of the `pResults.fee` to the fee recipient (if specified) and the remaining `pResults.fee` to the insurance pool.
	2. `lResults.quantity` equals the total quantity of contracts created across the orders from the previous step, i.e. `lResults.quantity = pResults[0].quantity + ... + pResults[n-1].quantity` . Note that `lResults.quantity <= quantity`. 
	3.  `lResults.liquidationPrice` equals the weighted average price, i.e. `lResults.liquidationPrice = (orders[i].contractPrice * pResults[0].quantity + orders[n-1].contractPrice * pResults[n-1].quantity)/(lResults.quantity)` 
		* To liquidate a long position, `lResults.liquidationPrice` must be greater than or equal to the index price. 
		* To liquidate a short position, `lResults.liquidationPrice` must be less than or equal to the index price. 

5. Close the `lResults.quantity` quantity contracts of the existing position. 
	1. Calculate the trader's loss (`position.margin / position.quantity * quantity`) and update the state of the trader's position. 
	2. Calculate the PNL per contract (`contractPNL`) which equals `lResults.liquidationPrice - position.contractPrice` for longs and ` position.contractPrice - averageClosingPrice ` for shorts.
	3. Calculate the total payout from the position`cResults.payout = cResults.quantity * (position.margin / position.quantity + contractPNL) `
		1. Allocate half of the payout to the liquidator and half to the insurance fund
	4. Decrement the position's remaining margin by `position.margin / position.quantity * (position.quantity - quantity)`
	5. Emit a `FuturesLiquidation` event. 


# Oracle

## General concept

The oracle serves two functions:

1. Update the index price of an asset
2. Set the new funding rate

### 1. Update the index price

The index price is used to calculate the NPV of positions. It will be periodically updated by the oracle.

### 2. Set the new funding rate

The funding rate is a critical piece to ensure convergence of market prices to the real underlying asset price. It consists of two components:

1. Premium Index: The premium index measures the difference between the perpetual swap’s price and the underlying asset it’s tracking.
2. Interest Rate: The interest rate is a function of interest rates between base and quote currency where each currency has its own defined rate.

The exact formula are:

`Premium = (max(0, Impact Bid Price — Index Price) — max(0, Index Price — Impact Ask Price)) / Index Price`

`Funding Rate = Premium + clamp(Interest Rate — Premium, 0.05%, -0.05%)`

The impact prices are weighted averages over the first 3000 long or respective short orders.

## Testnet setup

For the testnet we are setting up a centralized oracle service. In later versions this will be replaced by a decentralized mechanism.

### Testnet config

* Pair = XAU/USDT
* Gold Interest Rate = 0.03%
* USD Interest Rate = 0.06%
* Funding Interval = 8 hours

### Oracle service

The oracle service updates the ticker prices every 5 minutes. If any asynchronous requests fail, we deploy an [exponential backoff](https://cloud.google.com/iot/docs/how-tos/exponential-backoff) strategy. Prices are taken from [https://metals-api.com/](https://metals-api.com/).

Three times per day the funding rate is calculated according to the formula from above.





