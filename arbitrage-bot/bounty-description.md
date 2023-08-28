# Agoric Component: Arbitrage Bot

## Objective: 
Provide an estimate of the number of hours required to update this component to work with the current Agoric version (mainnet-1b) and a pool on Osmosis.

## Context:
The current component is a bot that is able to arbitrage prices between a pool on Osmosis and a pool on the Agoric AMM. It is an off-chain bot that can make nearly simultaneous off-setting trades on the Agoric AMM and Osmosis DEX given a divergence in price. 

## Code repository:
https://github.com/simpletrontdip/agoric-osmosis-bot

## Code analysis

Link to file: (todo)

## Arbitrage strategy

### Current approach

Objective: Identify arbitrage opportunity by checking the value of OSMO against IST on agoric vs the value of OSMO against USDC on osmosis

Assets:

| Pool              | Central  | Secondary |
| :---------------- | :------: | --------: |
| Agoric AMM        |   RUN    |   OSMO    |
| Osmosis Pool      |   USDC   |   OSMO    |

### Desired approach

Assets:

| Pool              |   Central   | Secondary  |
| :---------------- |  :-------:  | ---------: |
| Agoric Vault      |     IST     | Collateral |
| Osmosis Pool      | Stable coin | Collateral |


## Open Questions

- Assuming the AMM will be replaced by a Vault. What should be done if the requested secondary asset is not currently supported as collateral by the agoric vaults?

- Identify the impact of the vaults collaterization ratio, liquidation ratio and stability fee on the arbitrage strategy and formulas.

- Considering that Agoric vaults are currently accepting ATOM as collateral. Should the bounty be tested with an IST/ATOM Agoric vault and the Osmosis pool #670 ATOM/USDC.grv?

- Will we require an IBC implementation to transact the asset in common between Agoric and Osmosis? I don't see on the code any suggestion on how the assets are currently being transacted from the Agoric to the Osmosis account, and vice-versa.

- Will we require an ICA implementation to execute the operations on the Osmosis pool or follow the current approach by using the osmosis plugin?

- Should the contract be durable?


## Components that require an update/development

- Agoric SDK:
    - update agoric packages to the latest release (mainnet1B-rc3)
    - fix any issue triggered by the update.
- Agoric client:
    - agoric AMM is deprecated, so it is necessary to replace it with Agoric vaults.
- bot contract:
    - hardcoded variables should be able to be updated accordingly to the desired arbitrage strategy, such as riceDiffThreshold, minProfitThreshold and maxRunCount, etc...
    - contract instantiates a single bot, it could follow an approach similar to vaults were a botFactory allows the creation and management of multiple bots with different terms
- math :
    - if required, update formulas to include vauls specifics. 
- IBC/ICA implementation


## Acceptance Criteria

- Contract:
    - botFactory that allows the creation of multiple arbitrage bots
    - bot that identifies arbitrage opportunities between an Agoric vault and an Osmosis pool
    - botManager to wrapper the desired managing tools
- Clients
    - agoricClient to execute operations on the Agoric vault
    - osmosisClient and plugin to execute operations on the Osmosis pool
    or
    - ICA implementation
- IBC implementation
- Unit tests
- Deploy scripts
- Demo
- Documentation

## Time estimate

| Task            | Hours |
| :-------------- | ----: |
| Contracts       |  22   |
| Clients *       |  16   |
| IBC             |  16   |
| ICA             |  16   |
| Unit tests      |  12   |
| Deploy scripts  |  12   |
| Demo            |  4    |
| Documentation   |  2    |
| :-------------- | ----: |
| Total           |  100  |

Contingency margin for unexpected delays -> 20%

- Clients * : using osmosis plugin instead of ICA transaction, which would decrease the 16 hours allocated to integrate the ICA implementation.