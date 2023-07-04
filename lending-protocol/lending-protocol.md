# Agoric Components: Lending Protocol

## Summary

The LendingPool Protocol component is a pool-based loan protocol on Agoric that offers users five main operations that they can perform: deposit money, borrow money, redeem deposited money, adjust loans, and close loans.
These functionalities are distributed among several components that make up the protocol.

## Details

The LendingPool Protocol is based on Compound Finance, with several pools containing underlying assets and accepting multiple collateral types to lend the underlying asset. Liquidity providers fund the pools and receive a protocol token that is minted when they deposit money into the pool. The protocol token is exchanged for the underlying asset at an exchange rate, which increases as interest accrues. The interest rate for borrowers is dynamically calculated using pre-determined parameters and variables within a pool.

Loans lent from this protocol are over-collateralized, meaning that the value of the collateral must be greater than the value of the debt being requested by a predetermined margin called the 'liquidationMargin.' When creating a new pool, the liquidationMargin is passed as a variable. If a loan falls below the liquidationMargin, it gets liquidated by selling the collateral in Agoric's native AMM. Only the amount of collateral sufficient to cover the value of the debt is sold, while the remaining collateral is available for the borrower to withdraw.

LendingPool is similar to the VaultFactory in a way that they both accept a collateral and lend money for that collateral. The difference is that VaultFactory mints IST and lends it whereas LendingPool uses its own liquidity to lend money. Another important difference is that LendingPool lends multiple types of assets whereas VaultFactory only lends IST.

Two important modules implemented on the LendingPool are the liquidationObserver and debtsPerCollateral.

The liquidationObserver has a method called checkLiquidation, which detects when the loan's Collateral/Debt ratio exceeds the allowed limit. It uses a generator function to track the latest debt and collateral prices and checks if it triggers a liquidation. If so, it resolves a promise with the specific prices that caused the liquidation and terminates the control flow.

The debtsPerCollateral gathers loans with the same collateral type and provides functions to add new loans and set up a liquidator contract. It also utilizes a liquidationObserver object to schedule and execute liquidation when the closest loan to the liquidation margin reaches its threshold.

## Dependencies

There are some previous considerations to have before instantiating this contract.
The first one is related to the agoric-sdk version used at the moment of its development. The tag returned by running the command `git describe --tags --always` is `agoric-upgrade-8-326-g65d3f14c8`, so it is advised to checkout to the same state when exploring this component and test if any major update is required in order to be implemented at the desired agoric-sdk version.

`git checkout 65d3f14c8102993168d2568eed5e6acbcba0c48a`

The lendingPool module relies on the Agoric AMM (Automated Market Maker) to have pools like {lending_pool_underlying_currency} / IST in order to facilitate its liquidation process. By having these specific pools available, the lendingPool module can effectively execute its liquidation operations.

When the contract is instantiated, the initialPoserInvitation should be included on the privateArgs, which is accessible through the lendingPoolElectorate creatorFacet.
The contract terms should specify the following attributes:

```js
 const {
   ammPublicFacet,
   priceManager,
   timerService,
   liquidationInstall,
   loanTimingParams,
   compareCurrencyBrand,
   governance: {
     keyword,
     units,
     decimals,
     committeeSize
   },
 } = terms;
```

## Contract Facets

The LendingPool contract exports two remotable objects, publicFacet and the lendingPoolWrapper.
The publicFacet has a list of methods that allows any user to monitor and interact with lending pools, as well as some governance related methods.

The lendingPoolWrapper has multiple methods that are accessible exclusively to the contract owner through the lendingPoolElectionManager contract for monitoring and governance purposes.
For each method mentioned, a more detailed description will be provided in the next section

```js
 const publicFacet = Far('lending pool public facet', {
   helloWorld: () => 'Hello World',
   hasPool,
   hasKeyword,
   getPool: brand => poolTypes.get(brand),
   makeBorrowInvitation,
   makeRedeemInvitation,
   makeDepositInvitation,
   getPoolNotifier: () => poolNotifier,
   getGovernanceBrand: () => govBrand,
   getGovernanceIssuer: () => govIssuer,
   getGovernanceKeyword: () => keyword,
   getTotalSupply: () => totalSupply,
   getProposalTreshold: () => proposalThreshold,
   getGovBalance: () => govSeat.getAmountAllocated(keyword, govBrand),
   getMemberSupplyAmount: () => memberSupplyAmount,
   getCommitteeSize: () => committeeSize,
   getParamsSubscription: underlyingBrand => E(poolParamManagers.get(underlyingBrand)).getSubscription(),
   getCollateralBalance: brand => balanceTracer.getBalance(brand),
 });


 const lendingPoolWrapper = Far('powerful lendingPool wrapper', {
   getParamMgrRetriever,
   getLimitedCreatorFacet: () => lendingPool,
   getGovernedApis: () => Far('GovernedApis', { addPoolType }),
   getGovernedApiNames: () => harden(['addPoolType']),
 });
```

## Functionalities

### hasPool

The hasPool function receives a brand as an argument and will verify if a pool was previously created for that brand by checking the poolTypes and poolParamManagers maps.
It will return true if it exists and false if not.

```js
 const hasPool = brand => {
   const result = poolTypes.has(brand) && poolParamManagers.has(brand);
   return result;
 };
```

### hasKeyword

The hasKeyword function receives a keyword as an argument and will check if it is valid and was not already used as a brand in this Instance.
It will return undefined if it is unique or throw an appropriate error if it's not.

```js
 const hasKeyword = keyword => {
   return zcf.assertUniqueKeyword(keyword);
 };
```

### getPool

The getPool function receives a brand as an argument and it will return the respective poolManager for the brand provided.

```js
getPool: brand => poolTypes.get(brand),
```

### makeBorrowInvitation

When the user wishes to borrow an asset in exchange for a collateral, he can do it by exercising the makeBorrowInvitation. The offer proposal needs to define the amount of Debt requested and the amount of Collateral that will be provided, which should correspond to the payment.
In addition to the proposal and payment, the user needs to specify the collateralUnderlyingBrand in the offerArgs.

A few assertions will be conducted, verifying that the collateralUnderlyingBrand provided on the offerArgs exists on the poolTypes map, if the Collateral and Debt brands are supported and if the collateral does not exceed the allowed limit.

```js
 const makeBorrowInvitation = () => {
   /**
    * @type OfferHandler
    * */
   const borrowHook = async (borrowerSeat, offerArgs) => {
     assertProposalShape(borrowerSeat, {
       give: { Collateral: null },
       want: { Debt: null },
     });


     const collateralUnderlyingBrand = assertBorrowOfferArgs(offerArgs, poolTypes);
     /** @type PoolManager */
     const collateralUnderlyingPool = poolTypes.get(collateralUnderlyingBrand);


     const borrowBrand = assertBorrowProposal(poolTypes, borrowerSeat, collateralUnderlyingPool);
     assertColLimitNotExceeded(balanceTracer, getCollateralLimit, borrowerSeat.getProposal(), collateralUnderlyingBrand);


     const currentCollateralExchangeRate = collateralUnderlyingPool.getExchangeRate();
     const pool = poolTypes.get(borrowBrand);
     assertAssetsUsableInLoan(pool, collateralUnderlyingPool);


     return pool.makeBorrowKit(borrowerSeat, currentCollateralExchangeRate);
   };


   return zcf.makeInvitation(borrowHook, 'Borrow');
 };
```

Once all the assertions have been passed, the poolManager's makeBorrowKit method is called. The borrowerSeat and currentCollateralExchangeRate are passed as arguments and upon execution, makeBorrowKit creates a new loan based on the type of collateral and assigns a liquidator contract to handle various types of liquidation behaviors.
Once the new loan has been added, the borrower can monitor and interact with it using the loanKit obtained by exercising the invitation.

```js
export const makeLoanKit = (inner, assetNotifier) => {
 const { loan, loanUpdater } = wrapLoan(inner);
 return harden({
   uiNotifier: {
     assetNotifier,
     loanNotifier: loan.getNotifier(),
   },
   invitationMakers: Far('invitation makers', {
     AdjustBalances: loan.makeAdjustBalancesInvitation,
     CloseLoan: loan.makeCloseInvitation,
   }),
   loan,
   loanUpdater,
 });
};
```

### makeDepositInvitation

When the user wishes to deposit an underlying collateral, he will receive in return the corresponding amount of protocolToken. For that purpose the user will call makeDepositInvitation providing as an argument the underlyingBrand and receiving the desired invitation.
When exercising this invitation, the user will define on the proposalShape the Underlying amount he will be giving and the protocolToken he is expecting to receive. The payment should be according to the proposal. No offerArgs are expected for this offer.

The protocolAmountToMint will be calculated based on the amount of underlying asset provided and it will be increased that amount to the totalProtocolSupply
The calculated amount of protocolAmountToMint will be minted and reallocated from the protocolAssetSeat to the fundHolderSeat. The Underlying amount provided is removed from the fundHolderSeat to the underlyingAssetSeat and the asset state is updated

```js
 const makeDepositInvitation = () => {
   /**
    * @type {OfferHandler}
    * @param {ZCFSeat} fundHolderSeat*/
   const depositHook = async fundHolderSeat => {
     console.log('[DEPSOSIT]: Icerdeyim');
     assertProposalShape(fundHolderSeat, {
       give: { Underlying: null },
       want: { Protocol: null },
     });


     const {
       give: { Underlying: fundAmount },
     } = fundHolderSeat.getProposal();


     const protocolAmountToMint = shared.getProtocolAmountOut(fundAmount);
     protocolMint.mintGains(
       harden({ Protocol: protocolAmountToMint }),
       protocolAssetSeat,
     );
     totalProtocolSupply = AmountMath.add(
       totalProtocolSupply,
       protocolAmountToMint,
     );
     fundHolderSeat.incrementBy(
       protocolAssetSeat.decrementBy(
         harden({ Protocol: protocolAmountToMint }),
       ),
     );


     underlyingAssetSeat.incrementBy(
       fundHolderSeat.decrementBy(harden({ Underlying: fundAmount })),
     );


     zcf.reallocate(fundHolderSeat, underlyingAssetSeat, protocolAssetSeat);
     fundHolderSeat.exit();


     updateAssetState(UPDATE_ASSET_STATE_OPERATION.DEPOSIT);


     return 'Finished';
   };


   return zcf.makeInvitation(depositHook, 'depositFund');
 };
```

### makeRedeemInvitation

When the user that has previously deposited some underlying funds to the lendingPool wishes to redeem his loan, he can do it by calling makeRedeemInvitation. Based on the underlying brand, the respective poolManager is fetched and his redeemHook is called.

When exercising makeRedeemInvitation, the offer needs to define at the offerProposal the amount of protocolTokens to be given and Underlying to be received. The payment should be according to the proposal. No offerArgs are expected for this offer.

The underlying amount to be redeemed is calculated, considering the exchange rate and returned to the user. In exchange, the given redeemProtocolAmount is subtracted from the totalProtocolSupply and burned.

```js
 const redeemHook = async seat => {
   assertProposalShape(seat, {
     give: { Protocol: null },
     want: { Underlying: null },
   });


   const {
     give: { Protocol: redeemProtocolAmount },
     want: { Underlying: askedAmount },
   } = seat.getProposal();


   const redeemUnderlyingAmount = ceilMultiplyBy(
     redeemProtocolAmount,
     getExchangeRate(),
   );
   trace('RedeemAmounts', {
     redeemProtocolAmount,
     redeemUnderlyingAmount,
     askedAmount,
   });
   assertEnoughLiquidtyExists(
     redeemUnderlyingAmount,
     underlyingAssetSeat,
     underlyingBrand,
   );
   totalProtocolSupply = AmountMath.subtract(
     totalProtocolSupply,
     redeemProtocolAmount,
   );
   seat.decrementBy(
     protocolAssetSeat.incrementBy(harden({ Protocol: redeemProtocolAmount })),
   );
   seat.incrementBy(
     underlyingAssetSeat.decrementBy(
       harden({ Underlying: redeemUnderlyingAmount }),
     ),
   );
   zcf.reallocate(seat, underlyingAssetSeat, protocolAssetSeat);
   seat.exit();
   protocolMint.burnLosses(
     { Protocol: redeemProtocolAmount },
     protocolAssetSeat,
   );


   updateAssetState(UPDATE_ASSET_STATE_OPERATION.REDEEM);


   return 'Success, thanks for doing business with us';
 };
```

### getGovernanceBrand & getGovernanceIssuer & getGovernanceKeyword & getCommitteeSize

The governance object is one of the attributes provided on the contract terms, it has the following structure:
governance: {keyword, units, decimals, committeeSize }

The keyword and decimals are used to create a ZCFMint and consequently retrieve the respective brand and issuer. These values, along with the keyword and committeeSize, can be retrieved through the respective methods of the lendingPool publicFacet.

```js
 const [govMint, electorateParamManager] = await Promise.all([
   zcf.makeZCFMint(keyword, AssetKind.NAT, { decimalPlaces: decimals }),
	. . .
 ]);
 const { brand: govBrand, issuer: govIssuer } = govMint.getIssuerRecord();
```

### getTotalSupply & getProposalTreshold

The totalSupply is calculated based on the number of units and decimals defined on the governance object on the contract terms. The proposalThreshold is the round up value of 2% of the previously calculated totalSupply.
These values can be retrieved through the respective methods of the lendingPool publicFacet.

```js
const totalSupply = AmountMath.make(govBrand, units * 10n ** BigInt(decimals));
const proposalThreshold = ceilMultiplyBy(totalSupply, makeRatio(2n, govBrand));
```

### getMemberSupplyAmount

The committeeSize is a value passed on the governance object which represents the total number of members of the governance committee, and it is used to calculate the supplyRatio. The memberSupplyAmount is the result of splitting, in equal parts, the totalSupply of governance tokens through the members of the committee.
The calculated amount for a single member can be retrieved through the respective method of the lendingPool publicFacet.

```js
 const supplyRatio = makeRatio(1n, govBrand, BigInt(committeeSize), govBrand);
 const memberSupplyAmount = floorMultiplyBy(totalSupply, supplyRatio);
```

### getGovBalance

The govSeat is a ZCFSeat instantiated on the lendingPool contract, where it is initially allocated the totalSupply amount of governance tokens. The getGovBalance lets us know, at any given time, the remaining governance tokens allocated on the govSeat.

```js
  getGovBalance: () => govSeat.getAmountAllocated(keyword, govBrand)
```

### getCollateralBalance

The getCollateralBalance function receives a brand as an argument and it will return the current balance for the brand provided. The balanceTracer is a component of the lendingPool that keeps track of the balances of all the different protocolBrand

```js
getCollateralBalance: brand => balanceTracer.getBalance(brand)
```

### getParamMgrRetriever

The getParamMgrRetriever function, consumed by the lendingPoolElectionManager module, returns a remotable object with one get method, which receives the paramDesc as an argument.
If the paramDesc key is 'governedParams' it will return the electorateParamManager, which was instantiated by calling the makeElectorateParamManager function. If not, it will return the poolManager corresponding to the paramDesc respective collateralBrand.

```js
 const getParamMgrRetriever = () =>
   Far('paramManagerRetriever', {
     get: paramDesc => {
       if (paramDesc.key === 'governedParams') {
         return electorateParamManager;
       } else {
         return poolParamManagers.get(paramDesc.collateralBrand);
       }
     },
   });
```

### getLimitedCreatorFacet

The getLimitedCreatorFacet method of the lendingPool returns the lendingPool remote object. The exposed methods of this object will be now described, except for addPoolType method, which will be addressed in the next functionality.

```js
 const lendingPool = Far('Lending Pool Creator Facet', {
   helloFromCreator: () => 'Hello From the creator',
   addPoolType,
   getGovernanceInvitation: index => governanceInvitations[index],
   makeUpdateRiskControlsInvitation,
 });
```

The getGovernanceInvitation method will receive as an argument an index, which represents a committee member, and return an makeFetchGovInvitation. When this invitation is exercised, the respective committee member receives his corresponding memberSupplyAmount of governance tokens.

```js
 const governanceInvitations = harden([...Array(committeeSize)].map(makeFetchGovInvitation));
```

The makeUpdateRiskControlsInvitation method returns an invitation, that when exercised will update the risk controls, provided on the offerArgs, of the paramManager corresponding to the provided underlyingBrand.

```js
 const makeUpdateRiskControlsInvitation = () => {
   /**
    * @type OfferHandler
    */``
   const updateRiskControls = async (creatorSeat, offerArgs) => {


     const {
       underlyingBrand,
       changes
     } = offerArgs;
     creatorSeat.exit();


     const paramManager = poolParamManagers.get(underlyingBrand);
     await E(paramManager).updateParams(changes);


     return 'Params successfully updated!';
   };


    return zcf.makeInvitation(updateRiskControls, 'UpdateRiskControls');
 };
```

### getGovernedApis

The remote object returned by getGovernedApis encapsulates the function addPoolType, which allows a new pool to be created based on its underlyingBrand. It will also create and return the respective poolManager, after updating the poolTypes map and the pool state.

```js
 const addPoolType = async (
   underlyingIssuer,
   underlyingKeyword,
   params,
   priceAuthority,
 ) => {
   const {
     poolParamManager,
     underlyingBrand,
     protocolMint
   } = await setUpPoolParams(underlyingIssuer, underlyingKeyword, params);
   poolParamManagers.init(underlyingBrand, poolParamManager);


   const [startTimeStamp, priceAuthNotifier] = await Promise.all([
     E(timerService).getCurrentTimestamp(),
     E(priceManager).addNewWrappedPriceAuthority(
       underlyingBrand,
       priceAuthority,
       compareCurrencyBrand,
     ),
   ]);


   /** @type {ERef<PoolManager>} */
   const pm = makePoolManager(
     zcf,
     protocolMint,
     underlyingBrand,
     underlyingBrand,
     compareCurrencyBrand,
     underlyingKeyword,
     priceAuthority,
     priceAuthNotifier,
     priceManager,
     loanTimingParams,
     poolParamManager.getParams,
     timerService,
     startTimeStamp,
     getExchangeRateForPool,
     makeRedeemInvitation,
     liquidationInstall,
     ammPublicFacet,
     balanceTracer,
     getCollateralLimit,
   );
   poolTypes.init(underlyingBrand, pm);
   updatePoolState();
   return pm;
 };
```

### getGovernedApiNames

This method returns an hardened single entry array with the name of the GovernedApi. It is not being currently used by the lendingPool but it is a requirement of the Agoric governance package.

```js
  getGovernedApiNames: () => harden(['addPoolType'])
```

## Notifiers & Subscriptions

The lendingPool contract creates a notifierKit that keeps track of all poolManagers that are created and stored on the poolTypes map. The access to the notifier is exposed on the publicFacet.

```js
 const updatePoolState = () => {
   poolUpdater.updateState(
     [...poolTypes.values()].map(getPmAttributes),
   );
 };
```

At the lendingPool publicFacet there is also a subscriptionKit, which has the subscriber exposed through the method getParamsSubscription. It returns the state of PoolParamManager to the corresponding underlyingBrand provided.

```js
 return makeParamManagerSync(getSubscriptionKit(), {
   [LIQUIDATION_MARGIN_KEY]: [ParamTypes.RATIO, rates.liquidationMargin],
   [INITIAL_EXCHANGE_RATE_KEY]: [ParamTypes.RATIO, rates.initialExchangeRate],
   [BASE_RATE_KEY]: [ParamTypes.RATIO, rates.baseRate],
   [MULTIPILIER_RATE_KEY]: [ParamTypes.RATIO, rates.multipilierRate],
   [PENALTY_RATE_KEY]: [ParamTypes.RATIO, rates.penaltyRate],
   [BORROWABLE]: [ParamTypes.UNKNOWN, riskControls.borrowable],
   [USABLE_AS_COLLATERAL]: [ParamTypes.UNKNOWN, riskControls.usableAsCol],
   [COLLATERAL_LIMIT]: [ParamTypes.AMOUNT, riskControls.colLimit],
 })
```

## Usage and Integration

A step-by-step guide on how the contract can be used in practice and the dependencies that must be installed can be found in the README file in the project repository.
There you will find how to setup and run a specific scenario that is executed with the help of pre-built scripts that can be updated according to your preferences.

The list of unit tests built on test-lendingPool.js and test-expandedLendingPool.js is also a good way to understand how to interact with the different features implemented on the lendingProtool contract.

Another source of information where the LendingPool Protocol logic and flow are thoroughly described, with the support of code snippets and UI screenshots, is in the following papers:
https://bytepitch.com/blog/pool-based-lending-protocol-agoric
https://agoric.com/blog/guest-post/how-a-javascript-novice-won-the-liquidity-pool-bounty

## Link

https://github.com/anilhelvaci/dapp-pool-lending-protocol
