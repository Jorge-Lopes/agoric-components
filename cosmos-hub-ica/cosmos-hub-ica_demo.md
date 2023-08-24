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
   - Important: instead of using the community-dev branch, you need to check out to [mainnet1B-rc3](https://github.com/Agoric/agoric-sdk/releases/tag/mainnet1B-rc3)
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

## Agoric-sdk configuration

   1. Update the ibc.go, keeper.go and expected_keepers.go files on the agoric-sdk repository

   Currently, there is an issue when trying to create a new connection to a remote Port, one of the necessary actions on  **Deploy contract and execute ICA transactions** section.  
   It is expected that this issue will be fixed, but in the meanwhile you need to follow [this instructions](https://github.com/Agoric/agoric-sdk/pull/8127) and make the suggested updates on the files changed to the agoric-sdk, more specifically to the ibc.go, /keeper/keeper.go and /types/expected_keepers.go of the golang/cosmos/x/vibc/ directory.

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

3. Verify if the account was successfully created

   Note: When you query an account balance with zero tokens, you will get this error: No account with that address was found in the state. This can also happen if you fund the account before your node has fully synced with the chain. These are both normal.

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

   Note: replace the `<mnemonic phrase>` in the command below with your mnemonic phrase returned on step 2.

   ```shell
   jq --arg mnemonic '<mnemonic phrase>'  '. + {mnemonic: $mnemonic}'  $HOME/.hermes/cosmoswallet_key_seed.json > $HOME/.hermes/cosmoswallet_tmp.json && cat $HOME/.hermes/cosmoswallet_tmp.json > $HOME/.hermes/cosmoswallet_key_seed.json && rm $HOME/.hermes/cosmoswallet_tmp.json

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
   agoric deploy deploy.js
   ```

2. Choose how you wish to deploy and interact with the contract

   - Follow the approach `A` if you wish to use deploy scripts;
   - Follow the approach `B` if you wish to use agoric REPL;

### Script (approach A)

1. Run the following script.

   Create a new script in a directory of your preference (e.g. /interaccounts/contract/deploy/) and paste the following code.  

   Note: replace the `<controllerConnectionId>` with the agoric connection ID and the `<hostConnectionId>` with cosmos connection ID retrieved in the first step of the `Start Hermes` section. Replace as well the `<toAddress>` with a cosmos address that you wish to send the atom to.  


   ```js
   import '@agoric/zoe/exported.js';
   import { E, Far } from '@endo/far';
   import { MsgSend } from 'cosmjs-types/cosmos/bank/v1beta1/tx.js';
   import { encodeBase64 } from '@endo/base64';
   import installationConstants from '../ui/public/conf/installationConstants.js';

   const createICAAccountAndExecuteTransaction = async (homePromise) => {
   const home = await homePromise;
   const { board, zoe, ibcport } = home;

   const { INSTALLATION_BOARD_ID } = installationConstants;
   const installation = await E(board).getValue(INSTALLATION_BOARD_ID);
   const { publicFacet } = await E(zoe).startInstance(installation);

   assert(publicFacet, 'publicFacet must not be null');

   console.log('Contract instance started');

   const portArray = await ibcport;
   const port = portArray[2];

   const address = await E(port).getLocalAddress();
   assert(address, '/ibc-port/icacontroller-1', 'Invalid listening port');
   console.log('IBC port successfully found');

   const connectionHandler = Far('handler', {
      infoMessage: (...args) => {
         console.log(...args);
      },
      onReceive: (c, p) => {
         console.log('received packet: ', p);
      },
      onOpen: (c) => {
         console.log('opened');
      },
   });

   const controllerConnectionId = '<controllerConnectionId>';
   const hostConnectionId = '<hostConnectionId>';

   const connection = await E(publicFacet).createICAAccount(
      port,
      connectionHandler,
      controllerConnectionId,
      hostConnectionId,
   );

   const remoteAddr = await E(connection).getRemoteAddress();
   console.log('Connection opened:', remoteAddr);

   const start = remoteAddr.indexOf('{');
   const end = remoteAddr.lastIndexOf('}') + 1;
   const versionJson = remoteAddr.substring(start, end);
   const { address: controlledAddress } = JSON.parse(versionJson);
   console.log('remote controlled address', controlledAddress);

      const rawMsg = {
         amount: [{ denom: 'uatom', amount: '450000' }],
         fromAddress: controlledAddress,
         toAddress: '<toAddress>',
      };
      const msgType = MsgSend.fromPartial(rawMsg);

      const msgBytes = MsgSend.encode(msgType).finish();

      const bytesBase64 = encodeBase64(msgBytes);

      const akn = await E(publicFacet).sendICATxPacket(
         [
            {
            typeUrl: '/cosmos.bank.v1beta1.MsgSend',
            data: bytesBase64,
            },
         ],
         connection,
      );

      console.log(akn);

      console.log('Transaction concluded');
      };
      export default createICAAccountAndExecuteTransaction;
   ```

2. Update package.json

   On the project root, add the following line to the package.json file

   ```json
   "type": "module",
   ```

3. Execute the script

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
   publicFacet = history[1].publicFacet
   ```

3. Get an IBC listening port

   Identify the home.ibcport that has the icacontroller prefix
   ```shell
   home.ibcport.then(ps => Promise.all(ps.map(p => E(p).getLocalAddress())))
   ```
   
   Replace the position with the number of the icacontroller prefix above, most likely will be the position 2. Remember that it is 0 indexed.
   ```shell
   home.ibcport

   port = history[4][<position>]
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

   When the promise is resolved, an 'opened' message should be returned on REPL. Now run the following command on REPL which will print the connection remote address You should also be able to see on the agoric chain terminal a similar message.

   ```shell
   E(connection).getRemoteAddress()
   ```

   It should return the following

   ```
   "/ibc-hop/connection-0/ibc-port/icahost/ordered/{\"version\":\"ics27-1\",\"controller_connection_id\":\"connection-0\",\"host_connection_id\":\"connection-2563\",\"address\":\"cosmos13tx86qk672ezn7xte5ue7k9pnz5fqjshlgnr5le5x2789jz7r7zqlqpmat\",\"encoding\":\"proto3\",\"tx_type\":\"sdk_multi_msg\"}/ibc-channel/channel-3098"
   ```
   

   <mark>Important:</mark> the address printed on the message above correspond to the newly created interchain account hosted on Cosmos chain. It will be used in the next steps, declared as `fromAddress`.


7. Send atom to the interchain account

   In order to execute a transaction through the interchain account, you will need to provide it with some atom first.
   For this purpose you can use the same [Cosmos Faucet](https://discord.com/channels/669268347736686612/953697793476821092) as before.

8. Build message structure

   Run the following script, which will encode the rawMsg object using Protocol Buffers and then converting it to a base64-encoded format. 

   Note: replace the `<fromAddress>` with the one retrieved on the previous step and `<toAddress>` with a different cosmos address that you wish to send the atom to. The string returned on the console will be used in the next step.  

   ```js
   import '@agoric/zoe/exported.js';
   import { MsgSend } from 'cosmjs-types/cosmos/bank/v1beta1/tx.js';
   import { encodeBase64 } from '@endo/base64';

   const encodeRawMsg = async () => {
   const rawMsg = {
      amount: [{ denom: 'uatom', amount: '450000' }],
      fromAddress: '<fromAddress>',
      toAddress: '<toAddress>',
   };
   const msgType = MsgSend.fromPartial(rawMsg);

   const msgBytes = MsgSend.encode(msgType).finish();

   const bytesBase64 = encodeBase64(msgBytes);

   console.log(bytesBase64);
   };
   export default encodeRawMsg;
   ```

   ToDo: (update this step to be executed on the REPL)
   
9. Send a new packet through the connection previously created

   Replace the `<bytesBase64>` with the string returned by the script

   ```shell
   E(publicFacet).sendICATxPacket(
      [
      {
         typeUrl: '/cosmos.bank.v1beta1.MsgSend',
         data: '<bytesBase64>',
      },
      ],
      connection,
   )
   ```

   When the promise is resolved, it should return the following acknowledgement

   ```
   "{\"result\":\"Ch4KHC9jb3Ntb3MuYmFuay52MWJldGExLk1zZ1NlbmQ=\"}"
   ```

   Also, you should be able to see on the agoric chain terminal a similar message to this on the logs:

   ```
   SwingSet: vat: v16: IBC fromBridge { acknowledgement: 'eyJyZXN1bHQiOiJDaDRLSEM5amIzTnRiM011WW1GdWF5NTJNV0psZEdFeExrMXpaMU5sYm1RPSJ9', blockHeight: 517, blockTime: 1692632702, event: 'acknowledgementPacket', packet: { data: 'eyJ0eXBlIjoxLCJkYXRhIjoiQ3FRQkNod3ZZMjl6Ylc5ekxtSmhibXN1ZGpGaVpYUmhNUzVOYzJkVFpXNWtFb01CQ2tGamIzTnRiM014YkhsbGN6TjVNbU4xZGpWb2NqVmpOR05yWldRM2NUZHFaamh1YlhGcWRHWXpiVzVzYTJ0MWR6UTBjM1JsY21jeWVIUm1jMnMwZURjeWR4SXRZMjl6Ylc5ek1USjNibk56YldZNWEyNDVjbkF5WmpNellYSTROMlJ4ZWpabE5IVnJkbVJ3Ym5ZemVEZGtHZzhLQlhWaGRHOXRFZ1kwTlRBd01EQT0iLCJtZW1vIjoiIn0=', destination_channel: 'channel-3075', destination_port: 'icahost', sequence: 1, source_channel: 'channel-0', source_port: 'icacontroller-1', timeout_height: {}, timeout_timestamp: 1692633281756360000 }, type: 'IBC_EVENT' }
   ```

## See transaction result

1. See transaction in the block explorer

   Go to the [Cosmos testnet explorer](https://explorer.theta-testnet.polypore.xyz/) and search for both Cosmos addresses used as sender and receiver and see the atom being moved, you can also use gaiad to query their balances.

   Also, you can use the explorer and search for the cosmos address provided to the Hermes relayer on `Prepare Cosmos chain` section and see the transactions made to establish the ICA connection.


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


### Cosmos testnet

Name: theta-testnet-001

- https://github.com/cosmos/testnets/blob/master/public/README.md
- https://testnet.cosmos.directory/cosmoshubtestnet
