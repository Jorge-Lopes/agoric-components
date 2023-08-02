# Tutorial Page

## Description

This demonstration showcases how to establish a cross-chain communication channel using the Interchain Accounts (ICA) protocol. The setup involves two blockchains: Agoric and Cosmos testnets.  
The process involves several steps:

- Install required tools and dependencies;
- Modify the Hermes configuration file to define the chains' details and connection parameters.
- Set up and run a local Agoric chain.
- Initialize and create a new Cosmos account.
- Create and open a connection between the Agoric and Cosmos chains using Hermes.
- Deploy the contract and execute transactions between Agoric and Cosmos, this step is demonstrated through two approaches: a script-based approach and a REPL-based approach;
- See the transaction results.

## Prerequisites

- Follow the [installing the Agoric SDK](https://docs.agoric.com/guides/getting-started/) guide to install the Agoric Software Development Kit (SDK);
- Clone the [Interchain Accounts' repository](https://github.com/pitalco/interaccounts) and run `agoric install` in the project directory;
- Follow the [installing Hermes](https://hermes.informal.systems/quick-start/installation.html) guide to install the Hermes relayer and Gaia;
- Download [jq](https://stedolan.github.io/jq/download/).

## Hermes configuration

1. Replace Hermes configuration at the config.toml file with the following structure

   ```text
   [global]
   log_level = 'info'

   [mode]

   [mode.clients]
   enabled = true
   refresh = true
   misbehaviour = true

   [mode.connections]
   enabled = true

   [mode.channels]
   enabled = true

   [mode.packets]
   enabled = true
   clear_interval = 100
   clear_on_start = true
   tx_confirmation = true

   [telemetry]
   enabled = true
   host = '127.0.0.1'
   port = 3001

   [[chains]]
   id = 'agoriclocal'
   rpc_addr = 'http://localhost:26657'
   grpc_addr = 'http://localhost:9090'
   websocket_addr = 'ws://localhost:26657/websocket'
   rpc_timeout = '15s'
   account_prefix = 'agoric'
   key_name = 'ag-solo'
   store_prefix = 'ibc'
   gas_price = { price = 0.001, denom = 'ubld' }
   gas_multiplier = 1.1
   default_gas = 1000000
   max_gas = 10000000
   max_msg_num = 30
   max_tx_size = 2097152
   clock_drift = '5s'
   max_block_time = '30s'
   trusting_period = '14days'
   trust_threshold = { numerator = '2', denominator = '3' }

   [[chains]]
   id = 'theta-testnet-001'
   rpc_addr = 'http://104.245.147.200:26657'
   grpc_addr = 'http://104.245.147.200:9090'
   websocket_addr = 'ws://104.245.147.200:26657/websocket'
   rpc_timeout = '15s'
   account_prefix = 'cosmos'
   key_name = 'cosmosWallet'
   store_prefix = 'ibc'
   max_gas = 3000000
   gas_price = { price = 0.10, denom = 'uatom' }
   gas_multiplier = 1.1
   clock_drift = '5s'
   trusting_period = '1days'
   trust_threshold = { numerator = '2', denominator = '3' }
   ```

## Prepare Agoric Chain

1. Open a new terminal and start a local Agoric chain by typing the below commands

   ```shell
   cd agoric-sdk/packages/cosmic-swingset
   make scenario2-setup && make scenario2-run-chain
   ```

2. Open another terminal and start ag-solo

   ```shell
   cd agoric-sdk/packages/cosmic-swingset
   make scenario2-run-client
   ```

3. In a new terminal, open your wallet UI

   ```shell
   cd agoric-sdk/packages/cosmic-swingset
   agoric open --repl
   ```

## Add ag-solo keys to Hermes

1. Create a seed file under `$HOME/.hermes/` containing `ag-solo`'s key info

   ```shell
   cd agoric-sdk/packages/cosmic-swingset
   agd --home t1/8000/ag-cosmos-helper-statedir/ --keyring-backend=test keys show ag-solo --output json > $HOME/.hermes/ag-solo_key_seed.json
   ```

2. Add `ag-solo`'s mnemonic to `ag-solo_key_seed.json`

   ```shell
   cd agoric-sdk/packages/cosmic-swingset
   jq --arg mnemonic "$(cat t1/8000/ag-solo-mnemonic)"  '. + {mnemonic: $mnemonic}'  $HOME/.hermes/ag-solo_key_seed.json > $HOME/.hermes/ag-solo_tmp.json && cat $HOME/.hermes/ag-solo_tmp.json > $HOME/.hermes/ag-solo_key_seed.json && rm $HOME/.hermes/ag-solo_tmp.json
   jq . $HOME/.hermes/ag-solo_key_seed.json
   ```

   Should output something like this:

   ```json
   {
     "name": "ag-solo",
     "type": "local",
     "address": "agoric1jy006w0fx3q5mqyz5as57vvm32yalc0j6tvmm2",
     "pubkey": "{\"@type\":\"/cosmos.crypto.secp256k1.PubKey\",\"key\":\"AxQ6Ci1etkkk/rHfwKcUeyfpTE8I+vuG+DsPuwnNkVpc\"}",
     "mnemonic": "shop ignore brand tonight maid eternal setup payment seek bus fuel bunker awful stick group year betray quote nice stable catalog access indoor eagle"
   }
   ```

3. Add `ag-solo` keys to `hermes`

   ```shell
   cd ~
   hermes keys add --chain agoriclocal --key-file .hermes/ag-solo_key_seed.json --hd-path "m/44'/564'/0'/0/0" --overwrite
   ```

## Prepare Cosmos chain

1. Initialize Cosmos validators's and node's configuration files.

   ```shell
   gaiad init test --chain-id=theta-testnet-001
   ```

2. Create a new Cosmos account

   Note: this command will generate a new key, if you wish to recover an existing one use the `--recover` flag.

   ```shell
   gaiad keys add cosmoswallet
   ```

   Should output something like this:

   ```text
   - name: cosmoswallet
   type: local
   address: cosmos1sdk5kxmjej8yqtcp29murh0xxjl90j7h34s6te
   pubkey: '{"@type":"/cosmos.crypto.secp256k1.PubKey","key":"AomHXaMKKJA5C/6fBNKVdho+JX6rRJ6ODRhNR2syYmxZ"}'
   mnemonic: ""

   **Important** write this mnemonic phrase in a safe place.
   It is the only way to recover your account if you ever forget your password.

   awful device zone news benefit hospital snap real benefit myth rate badge gain penalty trial green matter cute tackle accident harbor call library evolve
   ```

   <mark>Important:</mark> write this mnemonic phrase in a safe place.

3. Verify if the account was successfully created (optional)

   Note: When you query an account balance with zero tokens, you will get this error: No account with address <account_cosmos> was found in the state. This can also happen if you fund the account before your node has fully synced with the chain. These are both normal.

   You can request atom from the [Cosmos Faucet](https://discord.com/channels/669268347736686612/953697793476821092).

   ```shell
   gaiad q account <address> --node http://104.245.147.200:26657
   ```

   Should output something like this:

   ```shell
   '@type': /cosmos.auth.v1beta1.BaseAccount
   account_number: "726520"
   address: cosmos1sdk5kxmjej8yqtcp29murh0xxjl90j7h34s6te
   pub_key: null
   sequence: "0"
   ```

   ```shell
   gaiad q bank balances <address> --node http://104.245.147.200:26657
   ```

   ```shell
   balances:
   - amount: "3000000"
   denom: uatom
   pagination:
   next_key: null
   total: "0"
   ```

4. Create a seed file under `$HOME/.hermes/` containing `cosmos`'s key info

   ```shell
   gaiad keys show cosmoswallet --output json > $HOME/.hermes/cosmoswallet_key_seed.json
   ```

   Note: add your mnemonic phrase in the command below.

   ```shell
   jq --arg mnemonic "<mnemonic phrase>"  '. + {mnemonic: $mnemonic}'  $HOME/.hermes/cosmoswallet_key_seed.json > $HOME/.hermes/cosmoswallet_tmp.json && cat $HOME/.hermes/cosmoswallet_tmp.json > $HOME/.hermes/cosmoswallet_key_seed.json && rm $HOME/.hermes/cosmoswallet_tmp.json

   jq . $HOME/.hermes/cosmoswallet_key_seed.json
   ```

   Should output something like this:

   ```json
   {
     "name": "cosmoswallet",
     "type": "local",
     "address": "cosmos1sdk5kxmjej8yqtcp29murh0xxjl90j7h34s6te",
     "pubkey": "{\"@type\":\"/cosmos.crypto.secp256k1.PubKey\",\"key\":\"AomHXaMKKJA5C/6fBNKVdho+JX6rRJ6ODRhNR2syYmxZ\"}",
     "mnemonic": "awful device zone news benefit hospital snap real benefit myth rate badge gain penalty trial green matter cute tackle accident harbor call library evolve"
   }
   ```

5. Add `cosmos` keys to `hermes`

   ```shell
   cd ~
   hermes keys add --chain theta-testnet-001 --key-file .hermes/cosmoswallet_key_seed.json --overwrite
   ```

6. Create a new Cosmos account that will receive atom from the previously created account

   ```shell
   gaiad keys add receivercosmoswallet
   ```

## Start Hermes

1. Create a connection

   ```shell
   cd ~
   hermes create connection --a-chain agoriclocal --b-chain theta-testnet-001
   ```

   Should output something like this:

   ```text
   SUCCESS Connection {
      delay_period: 0ns,
      a_side: ConnectionSide {
         chain: BaseChainHandle {
            chain_id: ChainId {
                  id: "agoriclocal",
                  version: 0,
            },
            runtime_sender: Sender { .. },
         },
         client_id: ClientId(
            "07-tendermint-0",
         ),
         connection_id: Some(
            ConnectionId(
                  "connection-0",
            ),
         ),
      },
      b_side: ConnectionSide {
         chain: BaseChainHandle {
            chain_id: ChainId {
                  id: "theta-testnet-001",
                  version: 0,
            },
            runtime_sender: Sender { .. },
         },
         client_id: ClientId(
            "07-tendermint-2220",
         ),
         connection_id: Some(
            ConnectionId(
                  "connection-2563",
            ),
         ),
      },
   }
   ```

2. Start Hermes

   ```shell
   hermes start
   ```

   Should output something like this:

   ```text
   # Chain: agoriclocal
   - Client: 07-tendermint-0
      * Connection: connection-0
         | State: OPEN
         | Counterparty state: OPEN
   # Chain: theta-testnet-001
   - Client: 07-tendermint-2220
      * Connection: connection-2563
         | State: OPEN
         | Counterparty state: OPEN
   ```

   <mark>Important:</mark> The connections IDs returned above will be used in the next sections as `controllerConnectionId` and `hostConnectionId` respectively.

## Deploy contract and execute ICA transactions

1. Install contract bundle

   ```shell
   cd ~/interaccounts/contract
   agoric deploy ./deploy.js
   ```

2. Choose how you wish to deploy and interact with the contract

   - Follow the approach `A` if you wish to use deploy scripts;
   - Follow the approach `B` if you wish to use agoric REPL;

### Script (approach A)

1. Copy this deploy script to the interaccounts project

   Create a new deploy script in a directory of your preference (e.g. /interaccounts/contract/deploy/) and paste the following code.

   Note: update `rawMsg fromAddress` with the `cosmoswallet` address and the `rawMsg toAddress` with the `receivercosmoswallet` address. As well as the `controllerConnectionId` and `hostConnectionId`.

   ```js
   import "@agoric/zoe/exported.js";
   import { E, Far } from "@endo/far";
   import { MsgSend } from "cosmjs-types/cosmos/bank/v1beta1/tx.js";
   import { encodeBase64 } from "@endo/base64";
   import installationConstants from "../ui/public/conf/installationConstants.js";

   const createSendICA = async (homePromise) => {
     const home = await homePromise;

     const { wallet, board, zoe, ibcport: untypedPorts } = home;

     const { INSTALLATION_BOARD_ID } = installationConstants;
     const installation = await E(board).getValue(INSTALLATION_BOARD_ID);
     const { publicFacet } = await E(zoe).startInstance(installation);

     assert(publicFacet, "publicFacet must not be null");
     console.log("Contract instance started");

     /** @type {Port[]} */
     const ibcport = untypedPorts;
     const port = E(ibcport[0]);

     const connectionHandler = Far("handler", {
       infoMessage: (...args) => {
         console.log(...args);
       },
       onReceive: (c, p) => {
         console.log("received packet: ", p);
       },
       onOpen: (c) => {
         console.log("opened");
       },
     });

     const controllerConnectionId = "connection-0";
     const hostConnectionId = "connection-2563";

     console.log("createICAAccount started");
     const connection = await E(publicFacet).createICAAccount(
       port,
       connectionHandler,
       controllerConnectionId,
       hostConnectionId
     );

     console.log("createICAAccount finished");

     const rawMsg = {
       amount: [{ denom: "uatom", amount: "450000" }],
       fromAddress: "cosmos1sdk5kxmjej8yqtcp29murh0xxjl90j7h34s6te",
       toAddress: "cosmos14y0mj9j2l982dkkl8j9xws9v5vs35u8utj8386",
     };
     const msgType = MsgSend.fromPartial(rawMsg);

     const msgBytes = MsgSend.encode(msgType).finish();

     const bytesBase64 = encodeBase64(msgBytes);

     const res = await E(publicFacet).sendICATxPacket(
       [
         {
           typeUrl: "/cosmos.bank.v1beta1.MsgSend",
           data: bytesBase64,
         },
       ],
       connection
     );

     console.log(res);
     console.log("ICA established");
   };
   export default createSendICA;
   ```

2. Execute the script

   ```shell
   cd ~/interaccounts/<script directory>
   agoric deploy <script name>
   ```

### REPL (approach B)

1. Start a new instance

   Note: replace <Installation_ID> with the `INSTALLATION_BOARD_ID` in the file /`interaccounts/ui/public/conf/installationConstants.js` generated from the deploy.js script.

   ```shell
   installation = E(home.board).getValue("<Installation_ID>")

   E(home.zoe).startInstance(installation)
   ```

2. Get the contract publicFacet

   ```shell
   publicFacet = history[2].publicFacet
   ```

3. Get an IBC listening port

   ```shell
   home.ibcport

   port = history[4][0]
   ```

4. Create a connection handler

   ```shell
   connectionHandler = Far('handler', { "infoMessage": (...args) => { console.log(...args) }, "onReceive": (c, p) => { console.log('received packet: ', p); }, "onOpen": (c) => { console.log('opened') } })
   ```

5. Declare the controller and host connection IDs

   ```shell
   controllerConnectionId = "connection-0"

   hostConnectionId = "connection-2563"
   ```

6. Create a connection using the given port

   ```shell
   connection = E(publicFacet).createICAAccount(port, connectionHandler, controllerConnectionId, hostConnectionId)
   ```

7. Build message structure

   (ToDO: not working properly, meaning of typeUrl: '/cosmos.bank.v1beta1.MsgSend')

   ```shell
   rawMsg = {
         amount: [{ denom: 'uatom', amount: '450000' }],
         fromAddress: 'cosmos1sdk5kxmjej8yqtcp29murh0xxjl90j7h34s6te',
         toAddress: 'cosmos14y0mj9j2l982dkkl8j9xws9v5vs35u8utj8386',
   };

   msgType = MsgSend.fromPartial(rawMsg);

   msgBytes = MsgSend.encode(msgType).finish();

   bytesBase64 = encodeBase64(msgBytes);
   ```

8. Send a new packet through the connection previously created

   ```shell
   E(publicFacet).sendICATxPacket(
      [
      {
         typeUrl: '/cosmos.bank.v1beta1.MsgSend',
         data: bytesBase64,
      },
      ],
      connection,
   )
   ```

## See transaction result

1. See transaction in the block explorer

   Go to the [Cosmos testnet explorer](https://explorer.theta-testnet.polypore.xyz/) and search for your Cosmos account address
   See the transactions made to establish the ICA connection and assets being moved

2. See balance update

   Use Gaiad to query both sender and receiver account balances.

   ```shell
   gaiad q bank balances <address> --node http://104.245.147.200:26657
   ```

## Relevant links

### IBC

- https://ibc.cosmos.network/
- https://tutorials.cosmos.network/academy/3-ibc/1-what-is-ibc.html

### Hermes

- https://hermes.informal.systems/index.html
- https://github.com/informalsystems/hermes/blob/master/config.toml

### Gaia

- https://hub.cosmos.network/main/getting-started/what-is-gaia.html

### ICA

- https://github.com/cosmos/ibc/blob/main/spec/app/ics-027-interchain-accounts/README.md

### ICA implementations

- https://github.com/srdtrk/cw-ica-controller/tree/main
- https://github.com/cosmos/interchain-accounts-demo

### Cosmos testnet

Name: theta-testnet-001

- https://github.com/cosmos/testnets/blob/master/public/README.md
- https://testnet.cosmos.directory/cosmoshubtestnet
