# Code Analysis

## Components

- plugin.js
- osmosis.js
- agoric.js
- math library
- bot.js

### plugin.js

#### bootPlugin()

The bootPlugin function initializes the plugin, setting up the necessary client connections and methods for interacting with the Osmosis blockchain. It returns an interface that include querying balances, retrieving pool data, getting spot prices for token swaps, and executing token swaps In and Out.

### osmosis.js

#### makeOsmosisPool()

It returns an interface to execute operations trade between 2 assets (central = USDC and secondary = OSMO) on an Osmosis pool, as well as monitor methods to track the pool balances and spot prices.
In its arguments it is provided the osmosisClient, which is returned by the bootPlugin(), along with the assets denoms and the poolId.

### agoric.js

#### makeAgoricFund()

It returns an interface to manage the funds of 2 assets (central = RUN/IST and secondary = OSMO). By providing as arguments the assets Issuer, Payments and DepositFacets, the makeAgoricFund function allows the withdrawal and deposit of payments for each specific asset, as well as monitor methods to track their balances.

#### makeAgoricPool()

It returns an interface to execute operations trade the secondary asset against the central on the agoric amm, as well as monitor methods to track the amm pool balances.
In its arguments it is provided the ammPublicFacet and terms, along with the assets brands and the tool set returned by the makeAgoricFund.

### math library

A set of methods that execute the required calculations for the arbitrage bot.

### bot.js

Bot that monitors the spot prices of two different token pools (Agoric and Osmosis), calculates whether arbitrage opportunities are present based on certain price thresholds, and executes arbitrage trades if profitable. The bot is designed to run either in debug mode (for a single check) or in production mode (with periodic checks) 


## Code execution flow

- Entry point: 
deploy script will create an arbitrage object, create an osmosisClient using the plugin.js, then it will bundle the bot.js and install it using the spawner and passing the required arguments. 

### bot.js

1. The bot contract will receive as terms the following arguments:

```js
  {
    zoe,
    ammAPI,
    ammTerms,
    centralBrand,
    secondaryBrand,
    centralIssuer,
    secondaryIssuer,
    centralPayment,
    secondaryPayment,
    centralDepositFacetP,
    secondaryDepositFacetP,
    timeAuthority: chainTimerService,
    // osmosis config
    poolId,
    centralDenom,
    secondaryDenom,
    osmosisClient,
  }
```

And declare the following constants:

```js
  const oneDec = new Dec(1); // 1 * 10 ^ 18
  const oneHundred = new Dec(100); // 1 * 10 ^ 20
  const oneExp6 = new Dec(1_000_000n); // 1 * 10 ^ 24
  const isDebugging = true;
```

2. The entry point for the contract is the startBot function.

The startBot creates an agoricFund which will return the following interface:

```js
  async balances() 
  async getAmountOf(brand)
  async withdraw(amount) 
  async deposit(payment) 
  async cleanup() 
```

Then it will create an osmosisPool and agoricPool. Both pools have an interface similar to this:

```js
  async balances()
  async getSpotPrice(includeSwapFee = true) 
  async sellToken(inAmount, minReturn)
  async buyToken(outAmount, maxSpend)
  async shutdown()
```


3. The startBot will then declare the following variable/constants that will be used in the calculateTradeParams() function:

```js
  let count = 0;
  const priceDiffThreshold = new Dec(50n, 4); // 0.05% diff 
  const minProfitThreshold = new Dec(50, 2); // 0.5 usdc
  const smoothTradeRate = arbitrageOptions.smoothTradeRate || new Dec(5, 3); 
```

The `arbitrageOptions` are not declared in the bot, neither is provided in the contract terms. So we can assume it will be equal to `new Dec(5, 3)`.

4. The bot contract will check if the `isDebugging` flag is set to true or false. Currently, it is hard-coded as true.

```js
 console.log('Starting the bot, debug:', isDebugging);

  if (isDebugging) {
    await checkAndActOnPriceChanges();
    await shutdown();
  } else {
    await registerNextWakeupCheck();
  }
```

5. The checkAndActOnPriceChanges() will retrieve the spot price for both osmosisPool and agoricPool and print the returned values.

```js
    const [osmosisPrice, agoricPrice] = await Promise.all([
      E(osmosisPool).getSpotPrice(),
      E(agoricPool).getSpotPrice(),
    ]);
```

Then it will check if a trade is profitable and if so, what amount and order should be traded.

```js
    const {
      shouldTrade,
      secondaryAmount,
      centralBuyMaxAmount,
      centralSellMinAmount,
      buyPool,
      sellPool,
    } = await calculateTradeParams(osmosisPrice, agoricPrice);
```

If the shouldTrade flag is set to true, it will execute the buy and sell operations

```js
    const success = await doArbitrage(
      secondaryAmount,
      centralBuyMaxAmount,
      centralSellMinAmount,
      buyPool,
      sellPool,
    );
```

At the end, it will print the prices updates.

6. If the isDebugging flag is set to false, we assume a production environment. In this scenario the registerNextWakeupCheck function is invoked

The maxRunCount is set to 3, and the checkInterval is set to 10. This means that the registerNextWakeupCheck will do 3 cycles separated by 10 units of time, and at each cycle invoke the checkAndActOnPriceChanges() and set the wakeup call for the next cycle. 

