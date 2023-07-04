# Agoric Components: Stop-Loss AMM

## Summary

This component allows a liquidity provider to an AMM pool to define a specific price (ratio of supplied assets) at which they would like to have their liquidity removed. This feature acts as a stop-loss contract which would allow LPs to define liquidity ranges.

## Details

After providing liquidity to an AMM liquidity pool and receiving in return the corresponding amount of LP tokens, the liquidity provider can instantiate a stopLoss contract that will allow him to lock the respective amount of LP tokens and specify the boundaries for a price range.
When the price of the respective AMM pool hits one of the boundaries (upper or lower), it will trigger the removal of the user assets (central and secondary tokens) from the AMM pool, in exchange for his LP tokens. Then he will be able to withdraw his assets from this contract to his purse.
At any moment the user is allowed to withdraw his locked LP tokens, remove liquidity from the AMM pool and update the price range boundaries.
When updating the boundaries, if the user specifies a range outside the current AMM pool price, it will trigger the removal of the assets from the AMM pool.

## Dependencies

There are some previous considerations to have before instantiating this contract.
The first one is related to the agoric-sdk version used at the moment of its development. The tag returned by running the command `git describe --tags --always` is `agoricxnet-7-914-gfedf04943`, so it is advised to check out to the same state when exploring this component and test if any major update is required in order to be implemented at the desired agoric-sdk version.

`git checkout fedf049435d7307311219fbab1b2b342ec6acce8`

When the contract is instantiated, the terms should specify the AMM publicFacet, the secondary issuer, the LP token issuer, the central issuer, and the initial boundaries.
The issuerKeywordRecord should also be specified with Central, Secondary and LpToken, being each one related to his corresponding issuer.

```js
 const {
   /** @type XYKAMMPublicFacet */ ammPublicFacet,
   /** @type Issuer */ centralIssuer,
   /** @type Issuer */ secondaryIssuer,
   /** @type Issuer */ lpTokenIssuer,
   boundaries,
   /** @type PriceAuthority */ devPriceAuthority = undefined,
 } = zcf.getTerms();


 assertIssuerKeywords(zcf, ['Central', 'Secondary', 'LpToken']);
```

The third consideration regards one of the contract terms, the ammPublicFacet.
For a production environment, the ammPublicFacet should be the respective public facet of a deployed AMM instance on zoe that the user provided liquidity to.
In a development environment, as a support for your unit tests, you can either create an instance of the AMM contract `@agoric/inter-protocol/src/vpool-xyk-amm/multipoolMarketMaker.js` and add the initial liquidity to the pool, or you can use Agoric priceAuthority to generate the desired price quotes.

```js
/**
*
* @param {XYKAMMPublicFacet} ammPublicFacet
* @param {PriceAuthority} devPriceAuthority
*/
export const assertExecutionMode = (ammPublicFacet, devPriceAuthority) => {
 const checkExecutionModeValid = () => {
   return (ammPublicFacet && !devPriceAuthority) || (!ammPublicFacet && devPriceAuthority);
 };
 tracer('assertExecutionMode', { ammPublicFacet, devPriceAuthority });
 assert(checkExecutionModeValid(),
   X`You can either run this contract with a ammPublicFacet for prod mode or with a priceAuthority for dev mode`);
};
```

## Contract Facets

The stopLoss contract exports two remotable objects, publicFacet and creatorFacet.
The publicFacet has a single method that allows any user with access to a reference of the stopLoss publicFacet to monitor the balance of the assets held by the contract.
The creator facet has multiple methods that are accessible exclusively to the contract owner, which allows him to take advantage of the features implemented by the contract such as implementing a stop-loss condition on a liquidity staking, withdrawing the locked assets and updating the price boundaries.
For each method mentioned, a more detailed description will be provided in the next section

```js
   const publicFacet = Far('public facet', {
     getBalanceByBrand,
   });


   const creatorFacet = Far('creator facet', {
     makeUpdateConfigurationInvitation,
     makeLockLPTokensInvitation,
     makeWithdrawLpTokensInvitation,
     makeWithdrawLiquidityInvitation,
     getNotifier: () => notifier,
   });
```

## Functionality

### getBalanceByBrand

When requesting a balance of an asset held by the stopLossSeat, the caller must specify the asset keyword and issuer, which will depend on the terms and IssuerKeywords defined when the contract was instantiated.

```js
 const getBalanceByBrand = (keyword, issuer) => {
   return stopLossSeat.getAmountAllocated(
     keyword,
     zcf.getBrandForIssuer(issuer),
   );
 };
```

### makeUpdateConfigurationInvitation

The contract owner can update the previously defined boundaries by exercising the invitation returned by makeUpdateConfigurationInvitation().
Before implementing the new update, the contract will verify that some conditions are not violated, such as the current allocation phase is set to ACTIVE or SCHEDULED and the offerArgs has the new boundaries object.

```js
 const makeUpdateConfigurationInvitation = () => {
   /** @type OfferHandler */
   const updateConfiguration = async (seat, offerArgs) => {
     assertScheduledOrActive(phaseSnapshot);
     assertUpdateConfigOfferArgs(offerArgs);
     const { boundaries } = offerArgs;


     const updateBoundaryResult = await updateBoundaries(boundaries);
     assertUpdateSucceeded(updateBoundaryResult);
     boundariesSnapshot = boundaries;
     updateAllocationState(ALLOCATION_PHASE.ACTIVE);


     return UPDATED_BOUNDARY_MESSAGE;
   };


   return zcf.makeInvitation(updateConfiguration, 'Update boundary configuration')
 };
```

The invitation offer needs to include the new lower and upper boundaries in the offerArgs as a price ratio. Note that if the creator specifies a range outside the current AMM pool price, it will trigger the removal of the assets from the pool.
After the offer result promise is resolved, the price boundaries will be updated, the allocation phase will be set to active, and it will return a message confirming the success of the operation.

```js
 const newBoundaries = {
   lower: boundaries.lower,
   upper: widerBoundaries.upper,
 };
  const userSeat = await E(zoe).offer(
   E(creatorFacet).makeUpdateConfigurationInvitation(),
   undefined,
   undefined,
   harden({ boundaries: newBoundaries }),
 );


 const offerResult = await E(userSeat).getOfferResult();
 t.deepEqual(offerResult, 'Successfully updated boundaries');
```

### makeLockLPTokensInvitation

After the contract owner provides liquidity to a pool and receives its LP tokens in exchange, he can lock them on the contract by exercising the invitation returned by makeLockLPTokensInvitation().
After reallocating the assets to the contract seat, the allocation phase will be updated to active.

```js
 const makeLockLPTokensInvitation = () => {
   const lockLPTokens = (creatorSeat) => {
     assertProposalShape(creatorSeat, {
       give: { LpToken: null },
     });


     assertScheduledOrActive(phaseSnapshot);


     const {
       give: { LpToken: lpTokenAmount },
     } = creatorSeat.getProposal();


     stopLossSeat.incrementBy(
       creatorSeat.decrementBy(harden({ LpToken: lpTokenAmount })),
     );


     zcf.reallocate(stopLossSeat, creatorSeat);


     creatorSeat.exit();


     updateAllocationState(ALLOCATION_PHASE.ACTIVE);


     return `LP Tokens locked in the value of ${lpTokenAmount.value}`;
   };


   return zcf.makeInvitation(
     lockLPTokens,
     'Lock LP Tokens in stopLoss contract',
   );
 };
```

The offer proposal has to specify the LpToken as a keyword identifier and the amount of tokens to be locked, as well as provide the respective payment.
After the offer result promise is resolved, it will return a message declaring the amount of tokens locked on the contract.

```js
 const lockLpTokensInvitation =
   E(creatorFacet).makeLockLPTokensInvitation();
 const proposal = harden({ give: { LpToken: lpTokenAmount } });
 const paymentKeywordRecord = harden({ LpToken: lpTokenPayment });


 const lockLpTokenSeat = await E(zoe).offer(
   lockLpTokensInvitation,
   proposal,
   paymentKeywordRecord,
 );
 const lockLpTokensMessage = await E(lockLpTokenSeat).getOfferResult();
 t.deepEqual(lockLpTokensMessage, `LP Tokens locked in the value of ${lpTokenAmount.value}`);
```

### makeWithdrawLpTokensInvitation

If at any moment the contract owner wishes to withdraw their locked LP tokens from the stopLoss contract to his seat, he can do it by exercising the invitation returned by makeWithdrawLpTokensInvitation().
When called, the current allocation phase needs to be active or error, being after updated to withdraw.

```js
 const makeWithdrawLpTokensInvitation = () => {
   const withdrawLpTokens = (creatorSeat) => {
     assertProposalShape(creatorSeat, {
       want: { LpToken: null },
     });


     assertActiveOrError(phaseSnapshot);


     const lpTokenAmountAllocated = stopLossSeat.getAmountAllocated(
       'LpToken',
       lpTokenBrand,
     );


     creatorSeat.incrementBy(
       stopLossSeat.decrementBy(harden({ LpToken: lpTokenAmountAllocated })),
     );


     zcf.reallocate(creatorSeat, stopLossSeat);


     creatorSeat.exit();


     updateAllocationState(ALLOCATION_PHASE.WITHDRAWN);


     return `LP Tokens withdraw to creator seat`;
   };


   return zcf.makeInvitation(withdrawLpTokens, 'withdraw Lp Tokens');
 };
```

When exercising this invitation, the offer proposal has to specify the LpToken as a keyword identifier. Since the owner will not give anything in this operation, there is no need for a payment.
After the offer result promise is resolved, it will return a message declaring that the tokens were withdrawn to the creator seat.

```js
 const withdrawLpTokensInvitation = await E(creatorFacet).makeWithdrawLpTokensInvitation();
 const withdrawProposal = harden({want: { LpToken: AmountMath.makeEmpty(lpTokenBrand)}});


 /** @type UserSeat */
 const withdrawLpSeat = E(zoe).offer(
   withdrawLpTokensInvitation,
   withdrawProposal,
 );


 const withdrawLpTokenMessage = await E(withdrawLpSeat).getOfferResult();
 t.deepEqual(withdrawLpTokenMessage, 'LP Tokens withdraw to creator seat');
```

### makeWithdrawLiquidityInvitation

There are two scenarios where the owner would want to call this method to withdraw his assets to his seat.
The first scenario is when the AMM pool price quote went out of the defined boundaries, and the owner assets were removed in exchange for the LP tokens.
The second scenario is when the contract owner decides to remove his liquidity from the AMM pool before the price quote of the AMM pool goes outside any boundary.

```js
 const makeWithdrawLiquidityInvitation = () => {
   const withdrawLiquidity = async (creatorSeat) => {
     assertProposalShape(creatorSeat, {
       want: {
         Central: null,
         Secondary: null,
       },
     });


     await removeLiquidityFromAmm();
     assertAllocationStatePhase(phaseSnapshot, ALLOCATION_PHASE.REMOVED);


     const centralAmountAllocated = stopLossSeat.getAmountAllocated(
       'Central',
       centralBrand,
     );
     const secondaryAmountAllocated = stopLossSeat.getAmountAllocated(
       'Secondary',
       secondaryBrand,
     );


     creatorSeat.incrementBy(
       stopLossSeat.decrementBy(
         harden({
           Central: centralAmountAllocated,
           Secondary: secondaryAmountAllocated,
         }),
       ),
     );


     zcf.reallocate(creatorSeat, stopLossSeat);


     creatorSeat.exit();


     updateAllocationState(ALLOCATION_PHASE.WITHDRAWN);


     return `Liquidity withdraw to creator seat`;
   };


   return zcf.makeInvitation(withdrawLiquidity, 'withdraw Liquidity');
 };
```

When exercising this invitation, the offer proposal has to specify the keywords identifiers of the assets expected to receive, being Central and Secondary.
After the offer result promise is resolved, the stopLoss contract will no longer hold any asset, and the contract owner will lose his LP tokens in exchange for the asked Central and Secondary tokens. It will return as well a message declaring that the assets were withdrawn to the creator seat.

```js
 const withdrawLiquidityInvitation = await E(creatorFacet).makeWithdrawLiquidityInvitation();
 const withdrawProposal = harden({
   want: {
     Central: AmountMath.makeEmpty(centralR.brand),
     Secondary: AmountMath.makeEmpty(secondaryR.brand),
   },
 });


 /** @type UserSeat */
 const withdrawSeat = E(zoe).offer(
   withdrawLiquidityInvitation,
   withdrawProposal,
 );


 const withdrawLiquidityMessage = await E(withdrawSeat).getOfferResult();
 t.deepEqual(withdrawLiquidityMessage, 'Liquidity withdraw to creator seat');
```

## Notifiers

The stopLoss contract creates a notifierKit that keeps track of the current allocation phase, balances and boundaries. The contract owner has access to the notifier state through the exposed method on the creatorFacet.

```js
 const getStateSnapshot = (phase) => {
   return harden({
     phase: phase,
     lpBalance: stopLossSeat.getAmountAllocated('LpToken', lpTokenBrand),
     liquidityBalance: {
       central: stopLossSeat.getAmountAllocated('Central', centralBrand),
       secondary: stopLossSeat.getAmountAllocated('Secondary', secondaryBrand),
     },
     boundaries: boundariesSnapshot,
   });
 };


 const { updater, notifier } = makeNotifierKit(
   getStateSnapshot(ALLOCATION_PHASE.IDLE),
 );
```

## Usage and Integration

A step-by-step guide on how the contract can be used in practice, and dependencies that must be installed can be found in the README file in the project repository.
There you will find 5 different scenarios that are executed with the help of pre-built scripts that can be updated according to your preferences.
The list of unit tests built on test-stopLoss.js is also a good way to understand how to interact with the different features implemented on the stopLoss contract.

## Link
https://github.com/Jorge-Lopes/stop-loss-amm
