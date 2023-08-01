# Agoric Components: Cosmos Hub ICA

## Summary

The Cosmos Hub ICA component provides a seamless solution to establish cross-chain accounts and transactions by facilitating interactions between Agoric and any desired host chains that support interchain accounts.

## Details

The Cosmos Hub ICA component is designed to facilitate the creation of [Interchain Accounts (ICA)](https://ibc.cosmos.network/main/apps/interchain-accounts/overview.html) and execute transactions between Agoric and a different chain, using the [IBC protocol](https://tutorials.cosmos.network/academy/3-ibc/1-what-is-ibc.html).

With the integration of the [ICS-27 standard](https://github.com/cosmos/ibc/blob/main/spec/app/ics-027-interchain-accounts/README.md) and IBC protocol, interchain accounts can be programmatically controlled via IBC packets, rather than signing transactions with a private key.

The Cosmos Hub ICA component is composed of a [Contract](https://github.com/pitalco/interaccounts/blob/master/contract/src/contract.js) and the [ICA module](https://github.com/pitalco/interaccounts/blob/master/contract/src/ica.js). The contract acts as a proxy for the ICA module, which has its own ICS-27 implementation, called ICS27ICAProtocol, that supports methods for establishing connections and sending transaction messages in conformity with the ICS-27 standard.

## Dependencies

There are some previous considerations to have before instantiating the Cosmos Hub ICA contract.  
The first one is related to the agoric-sdk version used at the moment of its development. The tag returned by running the command `git describe --tags --always` is `todo`, so it is advised to checkout to the same state when exploring this component and test if any major update is required in order to be implemented at the desired agoric-sdk version.

`git checkout todo`

Note that to start an instance of this contract it is not required any issuerKeywordRecord, terms, or privateArgs. Only the installation reference.

## Contract Facets

The contract developed for this module exports both creatorFacet and publicFacet. Although, as you can see from the code snippet below, the creatorFacet has no methods. On the other hand, the publicFacet has the createICAAccount and the sendICATxPacket methods.

```js
const creatorFacet = Far("creatorFacet", {});

const publicFacet = Far("publicFacet", {
  createICAAccount: () =>
    ICS27ICAProtocol.createICS27Account(
      port,
      connectionHandler,
      controllerConnectionId,
      hostConnectionId
    ),
  sendICATxPacket: () => ICS27ICAProtocol.sendICATx(msgs, connection),
});
```

The ICA module holds all the logic behind this component. It exports a single remotable object called ICS27ICAProtocol that has two methods with the same names as the ones contract publicFacet, createICAAccount and the sendICATxPacket.
Since the logic behind these methods is implemented in the ICA module, that will be the focus of the next section, Functionalities.

```js
export const ICS27ICAProtocol = Far("ics27-1 ICA protocol", {
  sendICATx: sendICAPacket,
  createICS27Account: createICAAccount,
});
```

## Functionalities

### createICAAccount

The createICAAccount function serves the purpose of creating a new ICA, returning a Promise that resolves to a Connection object.  
As for parameters, this function expects an IBC listening port, an object acting as the connection handler, and lastly, two connection IDs, one representing the controller chain and the other representing the host chain.

You can access these Port objects in the home.ibcport, which is an array of personal IBC listening ports where each element represents an individual port. The connection handle is equipped with a set of methods that are triggered when specific events occur within the connection. The controllerConnectionId and the hostConnectionId can be retrieved from the IBC relayer being used. Using [Hermes](https://hermes.informal.systems/quick-start/installation.html) as an example, when you start the relayer, it will print a message on your console with the connection ID of both clients, and you can also use the Hermes query connection command on the Hermes CLI

Note: we recommend reading the Agoric [Network API documentation](https://docs.agoric.com/reference/repl/networking.html#connecting-to-a-remote-port) for a more detailed description.

```js
/**
 * Create an ICA account/channel on the connection provided
 *
 * @param {Port} port
 * @param {object} connectionHandler
 * @param {string} controllerConnectionId
 * @param {string} hostConnectionId
 * @returns {Promise<Connection>}
 */
export const createICAAccount = async (
  port,
  connectionHandler,
  controllerConnectionId,
  hostConnectionId
) => {
  const connString = JSON.stringify({
    version: "ics27-1",
    controllerConnectionId,
    hostConnectionId,
    address: "",
    encoding: "proto3",
    txType: "sdk_multi_msg",
  });

  const connection = await E(port).connect(
    `/ibc-hop/${controllerConnectionId}/ibc-port/icahost/ordered/${connString}`,
    connectionHandler
  );

  return connection;
};
```

One of the available methods of the Port object is the connect, which enables users to establish a connection to a remote IBC port, facilitating communication with other chain-like entities. To establish a connection you must provide the remote endpoint as the first parameter, which will be a string similar to /ibc-hop/$HOPNAME/ibc-port/$PORTNAME/ordered/$VERSION. For this configuration, the $PORTNAME is already predefined as "icahost".

### sendICATxPacket

The main purpose of the sendICAPacket function is to send a set of messages through an ICA channel, where the messages represent specific data or operations to be transmitted between chain-like entities. These messages are packaged and transmitted over the IBC channel to a remote endpoint. This function returns a Promise that resolves to the acknowledgment data sent by the other side of the connection, represented in the same format as inbound data.

As parameters, it is expected an array of messages, where each message represents a specific data or operation, and a connection object like the one returned by the above createICAAccount function.

The sendICAPacket starts by validating the messages provided in the msgs array, ensuring that each message contains the required data and typeUrl properties and that the data property is a base64 encoded string. For each message in the msgs array, the base64 encoded data string is converted into a Uint8Array format to be used in creating the message. Then, it assembles all the converted messages into a single packet conforming to the ICS27 protocol.  
Finally, the packet is sent over the ICA channel using the send method of the connection object, which will be serialized and transmitted to the remote endpoint.

Note: the ica.js module imports some interfaces and functions external to Agoric. from the cosmjs and cosmjs-types packages that provide essential functionalities for handling ICA protocol-related data. TxBody defines transaction body structure, Any enables serialization of protocol buffer messages, toBase64 and fromBase64 encode/decode data in base64 format.

```js
/**
 * Provide a connection object and a list of msgs and send them through the ICA channel.
 *
 * @param {[Msg]} msgs
 * @param {Connection} connection
 * @returns {Promise<string>}
 */
export const sendICAPacket = async (msgs, connection) => {
  var allMsgs = [];
  // Asserts/checks
  for (let msg of msgs) {
    assert.typeof(
      msg.data,
      "string",
      X`data within object must be a base64 encoded string`
    );
    assert.typeof(
      msg.typeUrl,
      "string",
      X`typeUrl within object must be a string of the type`
    );

    // Convert the base64 string into a uint8array
    let valueBytes = fromBase64(msg.data);

    // Generate the msg.
    const txmsg = Any.fromPartial({
      typeUrl: msg.typeUrl,
      value: valueBytes,
    });

    // add the new message to all msg array
    allMsgs.push(txmsg);
  }
  const body = TxBody.fromPartial({
    messages: Array.from(allMsgs),
  });

  const buf = TxBody.encode(body).finish();

  // Generate the ics27-1 packet.
  /** @type {ICS27ICAPacket} */
  const ics27 = {
    type: 1,
    data: toBase64(buf),
    memo: "",
  };

  /** @type {Data} */
  const packet = JSON.stringify(ics27);

  const res = await E(connection).send(packet);

  return res;
};
```

In the snippet below, you can see an example of a packet being sent through the sendICATxPacket method. 
In this scenario, a connection was previously created, being the Cosmos chain the host and the Agoric chain the controller. The operation being executed is a transaction of 450000 atoms between two cosmos addresses.

```js
  const rawMsg = {
    amount: [{ denom: 'uatom', amount: '450000' }],
    fromAddress: 'cosmos1sdk5kxmjej8yqtcp29murh0xxjl90j7h34s6te',
    toAddress: 'cosmos14y0mj9j2l982dkkl8j9xws9v5vs35u8utj8386',
  };
  const msgType = MsgSend.fromPartial(rawMsg);

  const msgBytes = MsgSend.encode(msgType).finish();

  const bytesBase64 = encodeBase64(msgBytes);

  const res = await E(publicFacet).sendICATxPacket(
    [
      {
        typeUrl: '/cosmos.bank.v1beta1.MsgSend',
        data: bytesBase64,
      },
    ],
    connection,
  );
```

## Usage and Integration

To help you effectively utilize and integrate the Cosmos Hub ICA component into your projects, we offer a comprehensive guide that walks you through the necessary steps for setting up the testing environment and running a demonstration scenario. This guide will enable you to better understand the code's functionality and make the necessary configurations, so you can seamlessly integrate it into your existing applications.  
[Cosmos Hub ICA Demo]()

## Link

https://github.com/pitalco/interaccounts
