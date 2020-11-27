# Architecture

Equity Protocol is comprised of four principal components:

1. Equity Chain
2. Equity's smart contracts on Ethereum
3. Equity API nodes
4. Front-end interface

![](../.gitbook/assets/architecture.png)

## Equity Chain

The Equity Chain is an fully-decentralized sidechain relayer network which serves as a layer-2 derivatives platform, trade execution coordinator \(TEC\), and decentralized orderbook. The core consensus is Tendermint-based.

### Layer-2 Derivatives Platform

The Equity Chain supports building generalized derivatives/DeFi applications through two avenues: the Equity Futures Protocol and general smart contracts.

**Equity Futures Protocol**   
The Equity Futures Protocol is deployed on the Equity Chain as a Cosmos-SDK based application. This protocol enables traders to create, enter into, and execute decentralized perpetual swap contracts and CFDs on any arbitrary market.

**Smart Contracts**   
The Equity Chain provides a two-way Ethereum peg-zone for Ether and ERC-20 tokens to be transferred to the Equity Chain as well as an EVM-compatible execution environment for DeFi applications. The peg-zone is based off [Peggy](https://github.com/cosmos/peggy) and the EVM execution is based off [Ethermint](https://github.com/chainsafe/ethermint).

### Trade Execution Coordinator

The Equity Trade Execution Coordinator \(TEC\) is a decentralized coordinator implementation based off the [0x 3.0 Coordinator](https://github.com/0xProject/0x-protocol-specification/blob/master/v3/coordinator-specification.md) specification. The Equity TEC safeguards trades from front-running using Verifiable Delay Functions and enables lower-latency trading through soft-cancellations.

### Decentralized Orderbook

Equity's Decentralized Orderbook is a fully decentralized 0x-based orderbook enabling **sidechain order relay with on-chain settlement** - a decentralized implementation of the traditionally centralized [off-chain order relay](https://github.com/0xProject/0x-protocol-specification/blob/master/v2/v2-specification.md#architecture) used by nearly all central limit order book decentralized exchanges.

Nodes of the Equity Chain host a decentralized, censorship-resistant orderbook which stores and relays orders.

## Equity API

Equity API nodes have two purposes: 1\) providing transaction relay services and 2\) serving as a data layer for the protocol.

**Transaction Relay Service**   
Although users can directly interact with the Equity Chain by broadcasting a compatible Tendermint transaction encoding a compatible message type, doing so would be cumbersome for most users. To this end, API nodes provide users a simple HTTP and Websocket API to interact with the protocol. The API nodes then formulate the appropriate transactions and relay them to the Equity Chain.

The Equity API supports the Equity Futures API, the 0x Standard Relayer API version 3 \(SRAv3\), and the 0x Standard Coordinator API.

It also provides abstractions for protocol actions including staking, voting and governance. The full specification for these actions can be found [here](architecture.md).

**Data Layer**   
Equity API nodes also serve as a data layer for external clients. Equity provides a data and analytics API which is out-of-the-box compatible with Equity's sample frontend interface. Although Equity provides the API server as an in-process service based off BadgerDB communicating over gRPC with the Equity Chain node, developers can provide their own implementions for their custom needs \(e.g. using a relational database for indexed queries\).

The specification for this API can be found [here](architecture.md).

## Equity Contracts

Equity Protocol is token-based protocol which is inextricably tied to the INJ token \(an ERC-20 token\). As such, key components of protocol interactions and token economics are implemented through the following smart contracts: 

**Equity Coordinator Contract**   
The Equity's Coordinator Contract services both 0x-based orders as well as Equity's derivative transactions on Ethereum as well as on the Equity Chain. 

**Staking Contract**  
The Equity Staking Contract manages the core functions for stakers in Equity Protocol including slashing, rewards, delegation and governance. 

**Equity Futures Contracts**  
The Equity Futures Protocol encompasses a suite of smart contracts. Comprehensive details can be found [here](https://github.com/EquityLabs/Equity-futures). 

**Equity Bridge Contracts**  
The Equity Bridge Contracts encompass a suite of smart contracts managing the two-way peg between Ethereum and the Equity Chain. More details can be found [here](https://github.com/EquityLabs/Equity-core). 

**Equity Token Contract**  
The Equity Token Contract is an ERC-20 contract for the INJ token.

## Frontend Interface

Equity Protocol is a fully decentralized protocol which allows for individuals to access the protocol in a permisssionless manner. Equity provides an open-source sample frontend interface which individuals or companies can run locally or host on a web-server to interface with the protocol. This interface is also deployed on IPFS.

