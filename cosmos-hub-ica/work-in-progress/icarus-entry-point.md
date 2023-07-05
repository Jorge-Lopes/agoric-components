deploy-icarus.js

```js
  const bundle = await bundleSource(pathResolve(`./src/icarus/icarus.js`));
  const installation = await E(zoe).install(bundle);

  const privateArgs = harden({
    networkVat: await networkVat,
  });

  // start instance
  console.log('Starting instance');
  const { instance, publicFacet } = await E(zoe).startInstance(
    installation,
    undefined,
    {},
    privateArgs,
  );

    E(scratch).set('icarusPublicFacet', publicFacet),
    E(scratch).set('icarusInstance', instance),
```

deploy-register.js
the output of this script is:

```js
    E(scratch).set('icaController', controller),
    E(scratch).set('cosmoshubSubscription', subscription),
    E(scratch).set('cosmoshubIcaActions', icaActions),

```

E(scratch).set('icaController', controller),

```js
let controller = await E(scratch).get("icaController"); // I cannot find the stage where icaController is stored in scratch

if (!controller || overrideController) {
  // overrideController is set to 'true', so it will alwayes create a new controller
  console.log("Creating new controller");
  controller = await E(icarus).newController();
}
```

E(scratch).set('cosmoshubSubscription', subscription) && E(scratch).set('cosmoshubIcaActions', icaActions),

```js
const specs = {
  hostChainId: "ibc-0",
  hostConnectionId: "connection-828",
  hostPortId: "icahost",
  controllerConnectionId: "connection-0",
};

console.log("Creating new account", portId, specs);
const { icaActions, subscription } = await E(controller).makeRemoteAccount(
  specs
);
```

The icaActions returned by makeRemoteAccount

```js
      icaActions: Far('icarusConnectionActions', {
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
            .then(ack => E(icaProtocol).assertICAPacketAck(ack))
            .catch(handleConnectionError);
        },
        async reconnect() {
          if (getState().isConnecting) {
            throw Error('Another connecting attempt is in progress');
          }
          // update the connection
          conn = await doConnect();
        },
      }),
```

For the subscription returned by makeRemoteAccount

```js
const makeIcaConnectionKit = () => {
    const { subscription, publication } = makeSubscriptionKit();
    let state = {
      isReady: false,
      isConnecting: false,
      icaAddr: null,
    };

    const updateState = newAttrs => {
      // update current state with new attrs
      state = {
        ...state,
        ...newAttrs,
      };

      // publish to all subscribers
      publication.updateState(harden(state));
    };
```

The newController method of icarus.js returns the buildControllerActions()

```js
    async newController() {
      // XXX should we try to bindport until success
      const portId = nextIcaPortId();
      const icaPort = await makeIcaPort(portId);

      return buildControllerActions(icaPort, portId);
    },
```

This method is used for cosmoshubSubscription and cosmoshubIcaActions

```js
const buildControllerActions = async (icaPort) => {
  const localAddr = await E(icaPort).getLocalAddress();
  const portId = parsePortId(localAddr);

  /**
   * @type {import('./types.js').IcarusControllerActions}
   */
  return Far("icarusControllerActions", {
    async getPort() {
      return icaPort;
    },
    async getPortId() {
      return portId;
    },
    async makeRemoteAccount({
      hostConnectionId,
      hostPortId = "icahost",
      controllerConnectionId,
    }) {
      assert(hostPortId, X`Host port id is required, got ${hostPortId}`);
      assert(
        hostConnectionId,
        X`Host connection id is required, got ${hostConnectionId}`
      );
      assert(
        controllerConnectionId,
        X`Controller connection id is required, got ${controllerConnectionId}`
      );

      return makeIcaActions({
        icaPort,
        portId,
        controllerConnectionId,
        hostConnectionId,
        hostPortId,
      });
    },
  });
};
```







```js

  const openIcaChannel = async ({
    icaPort,
    handler,
    publication,
    hostConnectionId,
    controllerConnectionId,
    hostPortId,
    address = '',
  }) => {
    const version = JSON.stringify({
      version: 'ics27-1',
      hostConnectionId,
      controllerConnectionId,
      address,
      encoding: 'proto3',
      txType: 'sdk_multi_msg',
    });
    const connStr = `/ibc-hop/${controllerConnectionId}/ibc-port/${hostPortId}/ordered/${version}`;
    return E(icaPort)
      .connect(connStr, handler)
      .catch(publication.fail);
  };




    const makeIcaActions = async ({


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


    return {
        async reconnect() {
          if (getState().isConnecting) {
            throw Error('Another connecting attempt is in progress');
          }
          // update the connection
          conn = await doConnect();
        },
      },
    };
  };
```