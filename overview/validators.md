# Getting Started

To become a validator of the Equity sidechain, the participants have to accumulate more tokens than the minimum stake amount outlined in the protocol. The validator has the responsibility of proposing and verifying the sidechain block, facilitating the propagation, matching, and submission of orders. We utilize [Tendermint](https://cosmos.network/docs/cosmos-hub/validators/validator-faq.html#becoming-a-validator) consensus for the block production and validator selection process.

## Registration and Stake

The participants will submit a request `create-validator` on the sidechain to become validators. If the participants satisfy the requirements \(most notably the `minimum self-delegation` amount or generally known as the minimum stake amount\), they will be elidgible to become validators after a predefined number of blocks are mined. Once a participant becomes a validator, other participants in the sidechain can also delegate stakes to the validator's staking pool. Since our sidechain interacts with Ethereum, the validators and delegators alike will need to include their Ethereum address in their requests as well.

## Validator Duties

Beyond participating in the consensus of the Equity sidechain, the validators also have the responsibility of verifying incoming orders, submitting orders to the Ethereum filter contract, and approving orders before it was submitted to 0x for settlement. When a trader submits an order with a VDF time proof to a validtor, the validator need to verify the proof and check for existing time proofs for the same order. If an existing order with shorter time proof is found, the validator can update the same order with the longer time proof. If no identical orders are found, the validator will verify and include the order in the upcoming block.

If the validator is the block proposer of a checkpoint block \(one that aggregates orders since the last checkpoint block and submits to Ethereum \), the block proposer has the responsibility of submitting the orders within the block to Ethereum. Although the block proposers need to provide the gas cost for the entire block, they can recoup the cost from staking reward and earnings from exchange fee.

## Validator Rewards

Depending on the final implementation of the token economic model. The validator will either recieve both staking reward and exchange fee distribution in Equity's native token or solely the staking reward. Exchange fees are collected from the market pairs via negative spread model and are periodically auctioned off to market makers in exchange for Equity's native token. These tokens are either burned or distributed proportional by stake to the validators' staking pools depending on the final implementation.

## Slashing

If validators fail to fulfill their responsibilities or perform maclicious activities, their entire staking pool is forfeited or slashed. Beyond Tendermint's standard slashing condition, Equity will also slash a block proposer's stake if he/she fails to submit the checkpoint block to Ethereum. This can be detected when other validators have yet to observe a confirmed Ethereum transaction containing the checkpoint block in question after a threshold of Ethereum blocks has been mined.

