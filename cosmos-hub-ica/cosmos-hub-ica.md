# Agoric Components: Cosmos Hub ICA

## Summary

The ICA-DAO component revolutionizes cross-chain account management and governance by enabling seamless control of Agoric interchain accounts on the Cosmos Hub. This contract empowers decentralized organizations to manage ATOM delegation, staking, and governance actions, by facilitating interactions between controller chains and host chains. With the ICS-27 protocol and IBC integration, interchain accounts can be programmatically controlled via IBC packets, eliminating the need for private keys. This innovative solution opens up new possibilities for DAOs to stake, vote, and participate in governance processes across chains, enhancing decentralization and efficiency.

## Details

The [Interchain Accounts (ICA)](https://ibc.cosmos.network/main/apps/interchain-accounts/overview.html) is the implementation of the [ICS-27 protocol](https://github.com/cosmos/ibc/blob/main/spec/app/ics-027-interchain-accounts/README.md) that enables cross-chain account management using the [IBC protocol](https://tutorials.cosmos.network/academy/3-ibc/1-what-is-ibc.html). Interchain accounts are programmatically controlled via IBC packets instead of using private keys for transaction signing. The main concepts of ICA include the host chain (where the interchain accounts are registered), the controller chain (managing and controlling the interchain account), the interchain account itself (with similar capabilities to regular accounts), and an authentication module on the controller chain that defines custom logic for creating and managing interchain accounts.

The [Icarus contract](https://github.com/simpletrontdip/agoric-ica-dao/blob/master/contract/src/icarus/icarus.js) is designed to facilitate interchain communication and execute transactions between Agoric and a different chain, in this implementation it will be used the Cosmos Hub as host chain. The contract includes functions and utilities for managing interchain connections and sending transactions. It has its own [ICS-27 implementation](https://github.com/simpletrontdip/agoric-ica-dao/blob/master/contract/src/icarus/ics27.js) that supports methods for creating and managing ICA ports, establishing connections, and sending transaction messages in conformity with the ICS-27 standard. It handles error messaging and provides mechanisms for recovering from connection failures. The contract also imports functions for encoding and decoding transaction messages.

The [Agoric Governance package](https://github.com/Agoric/agoric-sdk/tree/community-dev/packages/governance) provides a framework for creating and managing governance processes. It includes Electorates, which represent sets of voters, and VoteCounters, which tally votes. The package supports various voting methods and provides transparency to voters and observers. A ContractGovernor allows contracts to be governed publicly, with controlled parameter changes and method invocations. The package enables users to create and vote on questions, ensuring the integrity of the process. Electorate members receive invitations to vote, and the package facilitates the verification of votes and the governance structure.

The [Ica-dao contract](https://github.com/simpletrontdip/agoric-ica-dao/blob/master/contract/src/ica-dao/ica-dao.js) provides a secure and decentralized mechanism for managing interchain accounts within a DAO environment, leveraging the capabilities of the Icarus contract and the governance package.
The contract provides functionalities for registering, connecting, and controlling ICAs on a host chain. It supports various transaction types such as delegation, undelegation, redelegation, voting, and claiming rewards. These transactions are encoded and sent from the ICA controller chain through IBC packets.
It also provides access to information such as the readiness of the ICA, the ICA address, and the port ID associated with the ICA controller.
The contract follows a governance model, where certain actions are governed by a set of predefined committee that establish a set of rules. The governance APIs enable users to perform actions within the defined governance framework.

## Dependencies

There are some previous considerations to have before exploring this component.

First, it is recommended to have some previous knowledge of Inter-Blockchain Communication, Interchain Accounts, and the Agoric Governance package. As a source of information, we advise you to visit the links provided above, in the Details section.

Regarding the agoric-sdk version used at the moment of its development. The tag returned by running the command `git describe --tags --always` is `?????`, so it is advised to checkout to the same state and test if any major update is required to be implemented at the desired agoric-sdk version.

`git checkout ?????`

As mentioned before, the Ica-dao module relies on the Icarus contract and the governance package. This means that before instantiating the Ica-dao contract some other contracts need to be instantiated first, being them:

- Icarus;
- Committee (representing a type of electorate that is elected or appointed);
- Governor (representing the ContractGovernor).

The path for the contracts mentioned above are:

```js
    [icarus]: './src/icarus/icarus.js',
    [committee]: '@agoric/governance/src/committee.js',
    [governor]: '@agoric/governance/src/contractGovernor.js',
```

### Icarus

To create an instance of the Icarus contract, it is required to pass the networkVat object on the privateArgs, which can be retrieved from the home object. The example shown below was extracted from the [start-icarus script](https://github.com/simpletrontdip/agoric-ica-dao/blob/master/contract/deploy/start-icarus.js).

```js
const { zoe, scratch, networkVat } = await homeP;

const installation = await E(scratch).get(`installation.${icarus}`);

const privateArgs = harden({
  networkVat: await networkVat,
});
```

### Committee

To create an instance of the Committee contract, it is required to pass the committee name and size on the terms, as well as the storageNode and marshaller on the privateArgs. Although we can see the issuerKeywordRecord being declared, it is assigned an empty object. Example extracted from the [start-committee script](https://github.com/simpletrontdip/agoric-ica-dao/blob/master/contract/deploy/start-committee.js).

```js
const options = {
  name: "ICA committee",
  size: 3,
};

const { zoe, scratch, board, personalStorage } = await homeP;

const storageNode = await E(personalStorage).makeChildNode(
  options.name.replace(/[ ,]/g, "_")
);
const marshaller = await E(board).getReadonlyMarshaller();

const installation = await E(scratch).get(`installation.${committee}`);

const terms = harden({
  committeeName: options.name,
  committeeSize: options.size,
});

const issuerKeywordRecord = harden({});

const privateArgs = harden({
  storageNode,
  marshaller,
});
```

### Governor

To create an instance of the Governor's contract, it is required to specify the contract terms and privateArgs.
The contract terms, declared as icaGovernorTerms, have to include a timer (which can be retrieved by the chainTimerService from the home object), the electorateInstance (the instance of the Committee contract instantiated above), the governedContractInstallation (the installation object of the Ica-dao contract) and lastly the governed object, which has two attributes, the terms and the empty issuerKeywordRecord.
The following code snippets were extracted from the [start-dao script](https://github.com/simpletrontdip/agoric-ica-dao/blob/master/contract/deploy/start-dao.js).

```js
const icaGovernorTerms = {
  timer,
  electorateInstance,
  governedContractInstallation: icaContractInstall,
  governed: {
    terms: icaTerms,
    issuerKeywordRecord: {},
  },
};
```

The icaTerms mentioned above is composed of an object named governedParams with two attributes, Electorate and IcarusInstance.
The Electorate has two properties, being them the type, assigned with the string "invitation", and the value, assigned with the poserInvitationAmount. The poserInvitationAmount is obtained by calling the getPoserInvitation method on the committee creatorFacet and passing that promise to the getAmountOf method of the invitationIssuer.
The IcarusInstance has the same two properties as the Electorate, but with different values assigned, being the type the string "instance" and the value the instance of the Icarus contract.

```js
const icaTerms = harden({
  governedParams: {
    Electorate: {
      type: ParamTypes.INVITATION,
      value: poserInvitationAmount,
    },
    IcarusInstance: {
      type: ParamTypes.INSTANCE,
      value: icarusInstance,
    },
  },
});
```

As for privateArgs, it is required to provide the creatorFacet of the governor contract, declared as electorateCreatorFacet, and the resolved promise of the poserInvitation returned by the getPoserInvitation method, declared as initialPoserInvitation.

```js
{
    electorateCreatorFacet: committeeCreator,
    governed: {
      initialPoserInvitation: poserInvitation,
    },
  }
```

### Ica-dao

It is very important to highlight that the Ica-dao contract is instantiated through the Governor contract.

The Ica-dao installation and IssuerKeywordRecord are obtained through the icaGovernorTerms, declared as governedContractInstallation and governedIssuerKeywordRecord.
The augmentedTerms is the combination of the governed attribute of the icaGovernorTerms and the Governor contract instance.
Finally, the privateArgs is the initialPoserInvitation mentioned on the Governor contract privateArgs.

```js
const {
  creatorFacet: governedCF,
  instance: governedInstance,
  publicFacet: governedPF,
  adminFacet,
} = await E(zoe).startInstance(
  governedContractInstallation,
  governedIssuerKeywordRecord,
  augmentedTerms,
  privateArgs.governed
);
```

## Contract Facets

### Icarus

The Icarus contract exports only the publicFacet, which exposes two functions, being them the newController and makeController.

```js
return {
  async newController() {
    const portId = nextIcaPortId();
    const icaPort = await makeIcaPort(portId);

    return buildControllerActions(icaPort, portId);
  },

  async makeController(icaPort) {
    return buildControllerActions(icaPort);
  },
};
```

### Ica-dao

The Ica-dao contract has two export two facets, being them the creatorFacet and publicFacet

The **creatorFacet** is the result of makeGovernorFacet, a function provided by the governance package, that receives as arguments the creatorApis and governedApis. The makeGovernorFacet function will be described later in this section.

```js
const creatorApis = {
  async claimReward(args) {
    return sendIcaTxMsg("claimReward", args);
  },
  reconnectAccount,
};

const governedApis = {
  async delegate(args) {
    return sendIcaTxMsg("delegate", args);
  },
  async redelegate(args) {
    return sendIcaTxMsg("redelegate", args);
  },
  async undelegate(args) {
    return sendIcaTxMsg("undelegate", args);
  },
  async vote(args) {
    return sendIcaTxMsg("vote", args);
  },
  async sendRawTxMsg(msgs) {
    return E(icaActions).sendTxMsgs(msgs);
  },
  registerAccount,
};
```

The **publicFacet** is the result of augmentPublicFacet, a function provided by the governance package as well, that receives as arguments the publicApis.The augmentPublicFacet function will also be described later in this section.

```js
const publicApis = {
  isReady() {
    return E(icaActions).isReady();
  },
  getAddress() {
    return E(icaActions).getAddress();
  },
  async getPortId() {
    const controller = await getUpdatedController();
    return E(controller).getPortId();
  },
};
```

### Governance package

The handleParamGovernance function and the ParamTypes are imported from the [contractHelper](https://github.com/Agoric/agoric-sdk/blob/community-dev/packages/governance/src/contractHelper.js), which belongs to the Agoric governance package. They are used to retrieve the augmentPublicFacet and makeGovernorFacet functions mentioned above.

```js
const { augmentPublicFacet, makeGovernorFacet, params } =
  await handleParamGovernance(zcf, privateArgs.initialPoserInvitation, {
    [ICARUS_INSTANCE]: ParamTypes.INSTANCE,
  });
```

#### makeGovernorFacet

The makeGovernorFacet function takes two arguments: originalCreatorFacet, which in this case is the creatorApis, and governedApis.
It first calls the makeLimitedCreatorFacet function with originalCreatorFacet to obtain the limitedCreatorFacet, and then, it calls the makeFarGovernorFacet function with the limitedCreatorFacet and governedApis.

```js
const makeGovernorFacet = (originalCreatorFacet, governedApis = {}) => {
  const limitedCreatorFacet = makeLimitedCreatorFacet(originalCreatorFacet);
  return makeFarGovernorFacet(limitedCreatorFacet, governedApis);
};
```

The makeLimitedCreatorFacet function returns a remotable object called governedContract creator facet, which includes all the methods of the originalCreatorFacet and adds a new method called getContractGovernor that returns the electionManager. In this case, the electionManager is assigned with the instance of the contractGovernor used to instantiate the Ica-dao contract.
The governedContract creator facet object will be assigned to the limitedCreatorFacet constant.
The makeFarGovernorFacet function takes two arguments: limitedCreatorFacet and governedApis and returns a remotable object called governorFacet, with various methods and properties useful to interact with the governed contract, that is exclusively for the contractGovernor, which only reveals limitedCreatorFacet.
In conclusion, the object assigned to the Ica-dao creatorFacet is the limitedCreatorFacet.

```js
const makeFarGovernorFacet = (limitedCreatorFacet, governedApis = {}) => {
  const governorFacet = Far("governorFacet", {
    getParamMgrRetriever: () =>
      Far("paramRetriever", { get: () => paramManager }),
    getInvitation: (name) => paramManager.getInternalParamValue(name),
    getLimitedCreatorFacet: () => limitedCreatorFacet,
    // The contract provides a facet with the APIs that can be invoked by
    // governance
    /** @type {() => GovernedApis} */
    // @ts-expect-error TS think this is a RemotableBrand??
    getGovernedApis: () => Far("governedAPIs", governedApis),
    // The facet returned by getGovernedApis is Far, so we can't see what
    // methods it has. There's no clean way to have contracts specify the APIs
    // without also separately providing their names.
    getGovernedApiNames: () => Object.keys(governedApis),
    setOfferFilter: (strings) => zcf.setOfferFilter(strings),
  });

  //
  return governorFacet;
};
```

#### augmentPublicFacet

The augmentPublicFacet function takes the originalPublicFacet as an argument, in this case is the publicApis, and creates a remotable object named 'publicFacet' that includes the properties of the originalPublicFacet plus the methods from commonPublicMethods and typedAccessors.
The typedAccessors object defines a set of methods that retrieve specific types of parameters from the paramManager. Each method corresponds to a specific parameter type and is associated with a corresponding method from the paramManager.
The commonPublicMethods object defines three methods that returns the paramManager subscription and parameters and lastly the electionManager.

The ParamManager is a facility that governed contracts can use to manage their visible state in a way that allows the ContractGovernor to update values using governance. When paramManager is instantiated inside the contract, the contract has synchronous access to the values, and clients of the contract can verify that a ContractGovernor can change the values in a legible way.

```js
const typedAccessors = {
  getAmount: paramManager.getAmount,
  getBrand: paramManager.getBrand,
  getInstance: paramManager.getInstance,
  getInstallation: paramManager.getInstallation,
  getInvitationAmount: paramManager.getInvitationAmount,
  getNat: paramManager.getNat,
  getRatio: paramManager.getRatio,
  getString: paramManager.getString,
  getUnknown: paramManager.getUnknown,
};

const commonPublicMethods = {
  getSubscription: () => paramManager.getSubscription(),
  getContractGovernor: () => electionManager,
  getGovernedParams: () => paramManager.getParams(),
};
```

## Functionalities

### Icarus

#### makeController

The makeController function triggers the execution of the buildControllerActions function and returns its result. The buildControllerActions receives as an argument an icaPort and returns a remotable object called icarusControllerActions that has three methods, being them getPort, getPortId and lastly the makeRemoteAccount.

Within the buildControllerActions, the icaPort passed as an  argument is used to retrieve the locally bound name of that port, declared as localAddr. The localAddress will then be used to retrieve its respective ibc-port, declared as portId.
The getPort method returns the icaPort provided while the method getPortId will return the portId.

The makeRemoteAccount method receives as arguments the hostConnectionId, hostPortId and controllerConnectionId. It will assert that all arguments were given and will return the makeIcaActions function, passing as arguments all the variables mentioned above.
The makeIcaActions function will start by calling makeIcaConnectionKit, which will return the required methods to retrieve the current connection state, update the state, access the connection handler, as well as the subscription and publication objects.

```js
const { getState, updateState, subscription, publication, handler } =
  makeIcaConnectionKit();
```

The makeIcaConnectionKit function initializes a subscription service and a state object that holds the current state of the connection kit. It initially has properties such as isReady (indicating if the connection is ready), isConnecting (indicating if the connection is in progress), and icaAddr (representing the ICA address). These properties are set to default values. The updateState function defined within makeIcaConnectionKit is used to update the state object with new attributes. It merges the new attributes with the existing state using the spread operator and then publishes the updated state to all subscribers via a subscription service.

A handler variable, which holds a remotable object called icarusConnectionHandler, has two methods: onOpen and onClose. The onOpen method is called when the connection is successfully opened, and it updates the state with relevant connection details and marks the connection as ready. The onClose method is called when the connection is closed, and it updates the state to indicate that the connection is no longer ready.

After retrieving the IcaConnectionKit, the next step of makeIcaActions is to initialize a variable called conn, which value assigned is the result of the promise returned by the doConnect function. The doConnect is an asynchronous function responsible for connecting to an ICA channel, it uses the openIcaChannel function to establish the connection and updates the state to reflect the connection status.

```js
const doConnect = async () => {
  updateState({
    isConnecting: true,
  });

  return openIcaChannel({
    controllerConnectionId,
    hostConnectionId,
    hostPortId,
    icaPort,
    handler,
    publication,
  }).finally(() => {
    updateState({
      isConnecting: false,
    });
  });
};
```

The doConnect function begins by updating the state to indicate that the connection process is starting by setting the isConnecting property to true. Next, it calls the openIcaChannel function to establish the ICA channel connection. It provides the necessary parameters identified in the code snippet above.
The openIcaChannel creates a version string in JSON format containing information about the ICA connection, including the version, host and controller connection IDs, address, encoding, and transaction type. It constructs the connection string (connStr) based on the provided parameters, following a specific format with IBC components.
After, it will establish the ICA channel connection by calling the connect method on the icaPort object, passing the connStr and handler as arguments. The connect method returns a promise representing the connection. If any error occurs during the connection process, the catch block is executed to handle the error.
The last step of doConnect is to update the state to indicate that the connection process has been completed by setting the isConnecting property to false.

In the end, makeIcaActions will return a remotable object called icarusConnectionActions, which has five methods, as you can see in the code snippet below.
The state, isReady and getAddress methods will return the state object or one of its attributes, described on the makeIcaConnectionKit function.

```js
icaActions: Far("icarusConnectionActions", {
  state() {
    return getState();
  },
  isReady() {
    return getState().isReady;
  },
  getAddress() {
    return getState().icaAddr;
  },
  async sendTxMsgs(msgs) {
    const icaTxPackage = await E(icaProtocol).makeICAPacket(msgs);
    return E(conn)
      .send(icaTxPackage)
      .then((ack) => E(icaProtocol).assertICAPacketAck(ack))
      .catch(handleConnectionError);
  },
  async reconnect() {
    if (getState().isConnecting) {
      throw Error("Another connecting attempt is in progress");
    }
    // update the connection
    conn = await doConnect();
  },
});
```

The sendTxMsgs method handles the process of sending transaction messages using the ICS27 protocol. It creates an interchain packet from the provided messages, sends the packet over a connection, validates and extracts data from the acknowledgment, and handles any errors that may occur during the process.
For this purpose, two icaProtocol (Icarus ICS-27 implementation) methods are used:
The makeICS27ICAPacket is used to create an interchain transaction packet in the ICS27 format. It takes a list of JSON transactions (msgs) and an optional memo as input parameters, validates them, then returns a promise that resolves to a string representing the ICS27 packet.
The assertICS27ICAPacketAck is used to assert the validity of an ICS27 interchain packet acknowledgment. It takes an ack parameter representing the acknowledgment and returns either the data field or the responses field.

The reconnect method will verify that there is no connecting attempt is in progress, if so, it will call the doConnect function described above.

#### newController

The difference between the newController and makeController functions is that for makeController, an already existing icaPort is provided as an argument, while for new controller an icaPort needs to be registered on the controller chain.
Having that in mind, the first step is to create the portId. This is accomplished by calling the nextIcaPortId function, which will generate a unique ID string with the prefix 'icacontroller-' followed by the value of icaPortPrefix, which is hard-coded as 'icarus', and a number that increments each time the function is called.
By having the portId, it is now possible to get the icaPort by calling the makeIcaPort function.

```js
const makeIcaPort = async (portId) => {
  return E(networkVat).bind(`/ibc-port/${portId}`);
};
```

Within the makeIcaPort function, the bind method of the networkVat object is used to establish a connection with a specific IBC port identified by the portId. It takes a string argument representing the path to the IBC port, which is constructed by appending the portId to the /ibc-port/ prefix.
The function makeIcaPort returns a promise that resolves to the IcaPort, which will then be used for the buildControllerActions function.

More information on the bind method is described in the [agoric documentation](https://docs.agoric.com/reference/repl/networking.html#opening-a-listening-port-and-accepting-an-inbound-connection).

<mark> return buildControllerActions(icaPort, portId); portId not used </mark>

### Ica-Dao

#### Delegate, redelegate, undelegate, vote and sendRawTxMsg (governedApis)

The reason we address all these methods of governedApis in a single section is because they all serve similar purposes, that is sending transaction messages using the ICS27 protocol, as described in sendTxMsgs function of the makeController section.
Each of the methods described in this section will return the sendIcaTxMsg function with two parameters, the type and the args.

```js
const sendIcaTxMsg = (type, args) => {
  assert(isRegistered, "Not registered, please `registerAccount` first");

  const buildFn = supportedTxMsgs[type];
  assert(buildFn, `Msg type ${type} is not supported`);

  const msg = buildFn(args);
  return E(icaActions).sendTxMsgs([msg]);
};
```

The sendIcaTxMsg first checks whether the account is registered by using the isRegistered flag and throws an error if it's not registered. It then retrieves the appropriate build function based on the provided type from the supportedTxMsgs object. If a build function is found, it uses the buildFn to construct the message by passing the args as an argument. Finally, it calls the sendTxMsgs method on the icaActions object, passing an array containing the constructed msg as the argument.

The types accepted by the supportedTxMsgs are delegate, undelegate, redelegate, vote and claimReward. For each one of them it is assigned the respective function responsible to build the transaction message. These functions belong to the encoder module of the Ica-dao contract.
Let's use as an example the delegate type.

```js
const makeBuildCosmosTxMsg = (MsgBuilder, typeUrl) => (params) => {
  const msg = MsgBuilder.fromPartial(params);
  const msgBytes = MsgBuilder.encode(msg).finish();

  return Any.toJSON({
    typeUrl,
    value: msgBytes,
  });
};

const buildDelegateMsg = makeBuildCosmosTxMsg(
  MsgDelegate,
  "/cosmos.staking.v1beta1.MsgDelegate"
);
```

When the type delegate is passed to sendIcaTxMsg function, the supportedTxMsgs will return the buildDelegateMsg which is created by invoking makeBuildCosmosTxMsg with two arguments: MsgDelegate and '/cosmos.staking.v1beta1.MsgDelegate'. It utilizes the higher-order function to build a specific delegate message for Cosmos network staking.
The purpose of buildDelegateMsg is to simplify the construction of a delegate message by providing a pre-configured makeBuildCosmosTxMsg function. It abstracts away the details of constructing the message and converting it to a JSON representation with the appropriate type URL.

The sendRawTxMsg method is very similar to the above methods with one exception, it allows sending a message without the predefined structure built by the makeBuildCosmosTxMsg function.

#### registerAccount (governedApis)

The purpose of the registerAccount function is to facilitate the registration of an account by obtaining an updated controller and setting the corresponding actions. The registerAccount function ensures that the account is not already registered, sets the registration flag, retrieves an updated controller, and assigns the obtained actions to the icaActions variable by calling the setIcaActions function.

```js
const registerAccount = async ({
  hostPortId,
  hostConnectionId,
  controllerConnectionId,
}) => {
  if (isRegistered) {
    throw Error("Already registered");
  }

  isRegistered = true;
  const controller = await getUpdatedController();

  const { icaActions: actions } = await E(controller).makeRemoteAccount({
    hostPortId,
    hostConnectionId,
    controllerConnectionId,
  });

  // update the ref
  setIcaActions(actions);
};
```

The getUpdatedController asynchronous function first retrieves the icarusInstance, if the icaController variable is not already set, it creates a new controller by calling newController() on the public facet of icarusInstance obtained from zoe. If the icaController exists and the lastIcarusInstance is the same as the current icarusInstance, it simply returns the existing icaController. If the lastIcarusInstance differs from the current icarusInstance, it retrieves the icaPort from the existing icaController and creates a new controller by calling makeController() on the public facet of the updated icarusInstance, passing the icaPort. The updated icaController is then returned.

The icaActions returned by the makeRemoteAccount method of the icaController object is described above in the makeController section of the Icarus contract, since it is one of the icarusControllerActions methods.

#### ClaimReward (creatorApis)

The claimReward method of the creatorApis has similar behavior has the governedApis methods described above that return the result of sendIcaTxMsg, but for this case, the supportedTxMsgs type is `claimReward`.

#### ReconnectAccount (creatorApis)

The reconnectAccount method will verify that an account was previously registered, and if so, will call the icarusConnectionActions reconnect method described above on the makeController section of the Icarus contract.

```js
const reconnectAccount = async () => {
  if (!isRegistered) {
    throw Error("Not registered");
  }

  return E(icaActions).reconnect();
};
```

#### isReady & getAddress (publicApis)

The isReady and getAddress methods of the publicApis will call the icarusConnectionActions isReady and getAddress method described above on the makeController section of the Icarus contract.

#### getPortId (publicApis)

The getPortId method will get the icaController as described in the registerAccount (governedApis) section. The the icaController object will call the icarusControllerActions getPortId method described above on the makeController section of the Icarus contract.

```js
async getPortId() {
  const controller = await getUpdatedController();
  return E(controller).getPortId();
},
```

## Notifiers & Subscriptions

As mentioned above, the makeIcaActions function of the Icarus contract returns a subscription object. The subscriptionKit is created when the makeIcaConnectionKit is called. It will publish the state object every time it is updated, it will also publish the error message as a fail state when a connection is opened or closed.

```js
const updateState = (newAttrs) => {
  // update current state with new attrs
  state = {
    ...state,
    ...newAttrs,
  };

  // publish to all subscribers
  publication.updateState(harden(state));
};
```

## Usage and Integration

A step-by-step guide on how this module can be used in practice and the dependencies that must be installed can be found in the [cosmos-hub-ica-tutorial file](addLink...).
There you will find how to setup and run a specific scenario that is executed with the help of pre-built scripts that can be updated according to your preferences.

## Link

https://bounties.gitcoin.co/issue/28955
https://github.com/simpletrontdip/agoric-ica-dao
