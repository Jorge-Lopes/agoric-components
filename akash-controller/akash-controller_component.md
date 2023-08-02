# Agoric Components: Akash Lease Mgmt.

## Summary

The Akash Lease Mgmt. module aims to automate the continuous monitoring and funding process for an Akash deployment. It verifies the funding status and triggers the funding process when the available funds fall below a specified threshold. The contract utilizes the AkashClient and Pegasus APIs to interact with the Akash blockchain and maintain the required funding level to ensure the deployment remains functional.

## Details

The [Akash Network](https://akash.network/) is an open-source decentralized cloud computing platform that allows users to bid and lease computing and storage resources. It uses blockchain technology and a marketplace to enable peer-to-peer resource leasing, providing a cost-efficient and decentralized cloud infrastructure for application deployment.
To be able to interact with the Akash Network, an [akashClient](https://github.com/simpletrontdip/dapp-akash-controller/blob/main/api/src/akash.js) library was created. The akashClient facilitates deployment management, account balance checks, and depositing uAKT tokens to fund specific deployments.

The Agoric [Pegasus package](https://github.com/Agoric/agoric-sdk/tree/master/packages/pegasus) enables seamless and secure communication and token transfers between different blockchain networks using the [ICS20 standard](https://github.com/cosmos/ibc/blob/main/spec/app/ics-020-fungible-token-transfer/README.md) and [IBC protocol](https://tutorials.cosmos.network/academy/3-ibc/1-what-is-ibc.html). It facilitates the creation and management of pegged fungible tokens, allowing their exchange between local and remote blockchains. Users can create invitations for asset transfers over the network, simplifying cross-chain interactions and enhancing the interoperability and accessibility of assets across various blockchains.

The [AkashController](https://github.com/simpletrontdip/dapp-akash-controller/blob/main/contract/src/contract.js) contract automates the funding process for Akash deployments by monitoring the account balance of the deployed application. The contract implements periodic checks, and if the account balance falls below a specified threshold, the contract automatically triggers the deposit of uAKT tokens to ensure the continuous operation of the application on the Akash Network.

## Dependencies

There are some previous considerations to have before instantiating the akashController contract.  
The first one is related to the agoric-sdk version used at the moment of its development. The tag returned by running the command `git describe --tags --always` is `agoric-upgrade-8-524-gf4143fe13`, so it is advised to checkout to the same state when exploring this component and test if any major update is required in order to be implemented at the desired agoric-sdk version.

`git checkout f4143fe13363a763e37f163225f4684028786663`

The Akash Lease Mgmt. module relies on the Pegasus contract for IBC transactions and an Akash client to monitor and manage the respective Akash deployment. The first step that should be addressed is to create a remote peg for uAKT, using the Pegasus method pegRemote. The second step is to boot an akashClient, for this, you can use the akash.js bootPlugin function and pass your Akash deployment account mnemonic and the Akash network rpcEndpoint as parameters. In return, you will receive a remotable object with the methods detailed in the Contract Facets section.
Both objects returned, the uAKT peg and the akashClient, will be required for the contract terms.

```js
const akashClient = await installUnsafePlugin("./src/akash.js", {
  mnemonic,
  rpcEndpoint,
}).catch((e) => console.error(`${e}`));
```

<mark>Important:</mark> the current implementation of the akashClient plugin is not working properly, so we advise to consider this when trying to implement this component into your applications. More details can be found [here](https://github.com/anilhelvaci/dapp-akash-controller/blob/main/README.md).

To instantiate the akashController contract, you need to provide the contract installation, the issuerKeywordRecord and lastly the contract terms.  
For the issuerKeywordRecord you need to specify the keyword 'Fund', being the value of the issuer of the pegged asset, in this case, it is the uAKT.

```js
const issuerKeywordRecord = harden({
  Fund: aktIssuer,
});
```

Regarding the contract terms, the snippet below lists all the required attributes that should be included in the terms.  
For the timeAuthority, you can pass the chainTimerService retrieved from the home object. Regarding the Pegasus attribute, you can use the home.agoricNames to lookup for the Pegasus instance, and from there retrieve the contract publicFacet. The deploymentId refers to the Akash deployment sequence identifier (DSEQ).

```js
const {
  akashClient,
  timeAuthority,
  checkInterval = 15n,
  deploymentId,
  maxCheck = 2,
  depositValue = 5_000n,
  minimalFundThreshold = 100_000n,
  aktPeg,
  pegasus,
  brands,
} = zcf.getTerms();
```

## Contract Facets

As mentioned in the Dependencies section, an akashClient needs to be provided on the contract terms. The start method of the bootPlugin object returns an akash-client, which is a remotable object that has a set of methods required to interact with the Akash deployment. Those methods are listed bellow, and will be described in more detail in the Functionalities section.

```js
return Far('akash-client', {
    initialize(),
    getAddress(),
    async getDeploymentList(),
    async getDeploymentDetail(dseq),
    async getDeploymentFund(dseq),
    async depositDeployment(dseq, amount),
    });
```

The akashController contract returns a creatorInvitation at the moment of its instantiation. The creatorInvitation is a zoe invitation that receives as offerHandler the watchAkashDeployment function. The akashController functions will be described as well on the next section.

```js
const creatorInvitation = zcf.makeInvitation(
  watchAkashDeployment,
  "watchAkashDeployment"
);

return harden({
  creatorInvitation,
});
```

## Functionalities

### Akash Client

#### initialize

The purpose of the initialize method is to ensure that the Akash client is properly set up before any subsequent method calls are made. It initializes the Akash instance and retrieves the account's address, which is required for various operations.  
It first checks if the Akash variable is already set, indicating that the client has already been initialized. If so, it logs a warning message and returns early to avoid re-initialization. If the client has not been initialized, it retrieves the mnemonic and RPC endpoint from the opts object. If not provided, it falls back to the default values specified at the beginning of the code. Then, it calls the initClient function, passing the mnemonic and RPC endpoint as parameters.  
The returned Akash instance and address are assigned to the respective variables.

```js
const initialize = async () => {
  if (akash) {
    console.warn("Client initialized, ignoring...");
    return;
  }
  const mnemonic = opts.mnemonic || DEFAULT_MNEMONIC;
  const rpcEndpoint = opts.rpcEndpoint || DEFAULT_AKASH_RPC;
  const result = await initClient(mnemonic, rpcEndpoint);
  akash = result.akash;
  address = result.address;
};
```

The initClient function, called within the initialize method, initiates the Akash client by creating an instance of a DirectSecp256k1HdWallet using the provided mnemonic and connects to the Akash network using the provided rpcEndpoint. It retrieves the address associated with the mnemonic and returns an object containing the Akash instance and the address.

```js
const initClient = async (mnemonic, rpcEndpoint) => {
  const offlineSigner = await DirectSecp256k1HdWallet.fromMnemonic(mnemonic, {
    prefix: "akash",
  });
  const accounts = await offlineSigner.getAccounts();
  const address = accounts[0].address;

  const akash = await Akash.connect(rpcEndpoint, offlineSigner);
  console.log("Akash address", address);
  return { akash, address };
};
```

#### getAddress

The getAddress method returns the address associated with the initialized Akash client, derived from the mnemonic phrase.

```js
getAddress: () => address,
```

#### balance

The purpose of the balance method is to provide an interface to query the account balance of the Akash client. It relies on the initialized akash instance to query the account balance by sending a request to the Akash network via the query.bank.balance method. It includes the client's address and the token denom (uakt) as parameters in the request.

```js
async balance() {
    assert(akash, 'Client need to be initalized');
    return akash.query.bank.balance(address, 'uakt');
},
```

#### getDeploymentList

The getDeploymentList method retrieves a list of deployments associated with the Akash client. It depends on the initialized Akash instance to query the deployment list by sending a request to the Akash network via the query.bank.list.params method, with the client's address as a parameter.

```js
async getDeploymentList() {
    assert(akash, 'Client need to be initalized');
    console.log('Getting deployment list');
    return akash.query.deployment.list.params({
    owner: address,
    });
},
```

#### getDeploymentDetail

The getDeploymentDetail method retrieves the detailed information of a specific deployment associated with the Akash client. It relies on the initialized Akash instance to query the deployment detail. The method sends a request to the Akash network, via the query.deployment.get.params method, with the client's address and the dseq parameter as inputs.

```js
async getDeploymentDetail(dseq) {
    console.log('Getting deployment detail', dseq);
    assert(akash, 'Client need to be initalized');
    return akash.query.deployment.get.params({
    owner: address,
    dseq,
    });
},
```

#### getDeploymentFund

The getDeploymentFund method retrieves the funding balance of a specific deployment associated with the Akash client. It takes only dseq as a parameter, and relies on the initialized Akash instance and the getDeploymentDetail method to query the deployment funding balance.
The method first calls the getDeploymentDetail method internally, passing the dseq parameter, to retrieve the detailed information of the deployment.
Once the deployment detail is obtained, the method accesses the escrowAccount.balance property of the detail to retrieve the funding balance.

```js
async getDeploymentFund(dseq) {
    console.log('Getting deployment fund', dseq);
    const detail = await this.getDeploymentDetail(dseq);
    return detail.escrowAccount.balance;
},
```

#### depositDeployment

The purpose of the depositDeployment method is to provide an interface for depositing funds into a specific deployment associated with the Akash client. It takes two parameters, the dseq, and the amount of funds to deposit, and relies on the initialized Akash instance to execute the deposit transaction.
The method sends a request to the Akash network, via the tx.deployment.deposit.params method, providing the client's address, dseq, and amount as parameters.
The result is an asynchronous operation that executes the deposit transaction, transferring the specified amount of funds into the deployment.

```js
async depositDeployment(dseq, amount) {
    assert(akash, 'Client need to be initalized');
    console.log('Depositing deployment', dseq, amount);
    return akash.tx.deployment.deposit.params({
    owner: address,
    dseq,
    amount,
    });
},
```

### Akash Controller

#### watchAkashDeployment

When the creatorInvitation is exercised by the contract owner, the watchAkashDeployment function serves as the entry point for the contract's logic. It starts by verifying the offer proposal shape, ensuring it matches the expected shape. Then, it assigns the ZCFseat parameter to the controllerSeat variable for future reference. Finally, the startWatchingDeployment function is called to initiate the monitoring process for the Akash deployment.  
Upon successful execution, the watchAkashDeployment function returns a default acceptance message: "The offer has been accepted. Once the contract has been completed, please check your payout." This message indicates that the offer has been accepted.

```js
const watchAkashDeployment = (seat) => {
  assertProposalShape(seat, {
    give: { Fund: null },
  });

  controllerSeat = seat;
  // start watching deployment
  startWatchingDeployment();

  return defaultAcceptanceMsg;
};
```

#### startWatchingDeployment

The startWatchingDeployment function begins by calling the initialize method on the akashClient, described in the section above, to set up the client and establish the necessary connections. After the client initialization, the function proceeds to register the first wake-up call using the registerNextWakeupCheck function. If an error occurs during the registration process, it is caught and logged.

```js
const startWatchingDeployment = async () => {
  // init the client
  await E(akashClient).initialize();

  // register next call
  await registerNextWakeupCheck().catch((err) => {
    controllerSeat.fail(err);
  });
};
```

#### registerNextWakeupCheck

The purpose of the registerNextWakeupCheck function is to calculate the next wake-up time and register a wake-up callback for that specific time, ensuring that the monitoring and funding cycles for the Akash deployment occur at the desired intervals.  
The function begins by incrementing the count variable, which keeps track of the number of monitoring cycles that have occurred. If the value of count exceeds the limit, indicating that the maximum number of checks has been reached, the function logs a message and exits the monitoring process. Otherwise, it retrieves the current timestamp from the timeAuthority, and then it calculates the next wake-up time, checkAfter, and sets the wake-up callback. The callback is defined as a remotable object with a wake function that triggers the next monitoring and funding cycle.  
If an error occurs during the registration process, it is caught and logged.

```js
const registerNextWakeupCheck = async () => {
  count += 1;
  if (count > maxCheck) {
    console.log("Max check reached, exiting");
    controllerSeat.exit();
    return;
  }

  const currentTs = await E(timeAuthority).getCurrentTimestamp();
  const checkAfter = currentTs + checkInterval;
  console.log("Registering next wakeup call at", checkAfter);

  E(timeAuthority)
    .setWakeup(
      checkAfter,
      Far("wakeObj", {
        wake: async () => {
          await checkAndFund();
          registerNextWakeupCheck();
        },
      })
    )
    .catch((err) => {
      console.error(
        `Could not schedule the nextWakeupCheck at the deadline ${checkAfter} using this timer ${timeAuthority}`
      );
      console.error(err);
      throw err;
    });
};
```

#### checkAndFund

The checkAndFund function monitors the funding status of the Akash deployment and triggers the funding process when the available funds fall below the specified threshold. This helps to maintain the required funding level for the deployment's operation and ensures its continued functionality.  
First, this function queries the current deployment balance using akashClient's getDeploymentFund method. The balance is represented as a DecCoin object, containing the amount and denomination. Next, it compares the balance amount against the predefined minimalFundThreshold. If it is below the threshold, it proceeds with the funding process by calling fundAkashAccount. If the amount meets or exceeds the threshold, it indicates sufficient funds are available, and the function moves to the next funding cycle without executing the funding process.

```js
const checkAndFund = async () => {
  // deployment balance type DecCoin
  const balance = await E(akashClient).getDeploymentFund(deploymentId);
  const amount = BigInt(balance.amount) / 1_000_000_000_000_000_000n;

  console.log("Details here", deploymentId, amount, minimalFundThreshold);

  if (amount < minimalFundThreshold) {
    // funding account and deposit the watch deployment
    await fundAkashAccount();
  }
};
```

#### fundAkashAccount

The fundAkashAccount function is used to fund the Akash account. It first obtains the Akash address using the akashClient.getAddress() function. Then, it creates a transfer invitation using the pegasus makeInvitationToTransfer method, which allows it to transfer tokens from the Pegasus contract to the Akash account. The amount to be transferred is specified as aktDepositAmount, which is a predetermined value in the contract's terms.  
The fundAkashAccount uses the offerTo function, and provides the necessary parameters, including the zcf (Zoe contract facet), the transferInvitation, the keyword mapping, the offer proposal and the from and to Seat.  
To ensure that the deposit is completed before proceeding further, the function calls the waitForPendingDeposit function. Once the deposit is completed, the function logs a message indicating that the transfer has been completed and checks the remaining allocation of the Transfer keyword in the user seat. If the remaining allocation is empty, it signifies that the transfer was successful. In this case, the function proceeds to execute the depositAkashDeployment function, which finalizes the deposit of the Akash deployment.
If an error occurs during the offer process or while waiting for the pending deposit, the function catches the error, logs an error message, and handles any necessary error handling or termination of the monitoring process.

```js
const fundAkashAccount = async () => {
  console.log("Funding Akash account");
  const akashAddr = await E(akashClient).getAddress();
  const transferInvitation = await E(pegasus).makeInvitationToTransfer(
    aktPeg,
    akashAddr
  );

  console.log("Offering transfer invitation...");
  const { userSeatPromise: transferSeatP, deposited } = await offerTo(
    zcf,
    transferInvitation,
    harden({
      Fund: "Transfer",
    }),
    harden({
      give: {
        Transfer: aktDepositAmount,
      },
    }),
    controllerSeat,
    controllerSeat
  );

  // register callback for deposited promise
  pendingDeposit = deposited.then(async () => {
    console.log("Transfer completed, checking result...");
    const remains = await E(transferSeatP).getCurrentAllocationJig();
    const transferOk = AmountMath.isEmpty(remains.Transfer);

    if (transferOk) {
      console.log("IBC transfer completed");
      // XXX should we recheck the balance?
      await depositAkashDeployment();
    } else {
      console.log("IBC transfer failed");
    }
  });

  const result = await E(transferSeatP)
    .getOfferResult()
    .catch((err) => {
      console.error("Error while offering result", err);
      throw err;
    });
  console.log("Offer completed, result:", result);

  await waitForPendingDeposit();
  console.log("Done");
};
```

#### depositAkashDeployment

The depositAkashDeployment function is responsible for depositing funds into the Akash deployment.
It begins by creating a response variable and awaits the result of the depositDeployment method of the akashClient. This method sends a deposit request to the Akash blockchain, specifying the dseq (deploymentId), the amount to be deposited (depositValue), and the token denomination (uakt).
After the deposit request is made, the function logs a message indicating that the deposit is completed. The response variable may contain relevant information about the deposit process, which can also be logged for reference if needed.

```js
const depositAkashDeployment = async () => {
  console.log("Depositing akash deployment", deploymentId);
  const response = await E(akashClient).depositDeployment(deploymentId, {
    // amount type Coin
    amount: String(depositValue),
    denom: "uakt",
  });
  console.log("Deposit, done", response);
};
```

#### waitForPendingDeposit

After waiting for the pending deposit, which represents the completion of the deposit process, the waitForPendingDeposit function, called in the fundAkashAccount function described above, has two possible outcomes. If the deposit process is successful and the promise resolves, the function completes without any further action. If an error occurs during the deposit process and the promise is rejected, the function logs an error message and calls the exit method on the controllerSeat. This action terminates the monitoring process and exits the contract.  
By awaiting the completion of the deposit, it guarantees that subsequent actions related to the deposit are executed only when the funds are confirmed on the blockchain.

```js
const waitForPendingDeposit = async () => {
  try {
    console.log("Waiting for pending deposit");
    await pendingDeposit;
  } catch (err) {
    // always exit if exception occur
    console.log("Error while depositing", err);
    controllerSeat.exit();
  }
};
```

## Usage and Integration

**Usage and Integration Guide**

To help you effectively utilize and integrate the akashController component into your projects, we offer a comprehensive guide that walks you through the necessary steps for setting up the testing environment and running a demonstration scenario. This guide will enable you to better understand the code's functionality, make necessary configurations, and seamlessly integrate it into your existing systems.
[Demo](https://github.com/Jorge-Lopes/agoric-components/blob/main/akash-controller/akash-controller_demo.md)


## Start Today

https://github.com/simpletrontdip/dapp-akash-controller
