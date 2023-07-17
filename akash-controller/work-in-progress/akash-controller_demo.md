# Prerequesites

* [agoric-sdk](https://docs.agoric.com/guides/getting-started/)
    - checkout to the agoric-sdk `community-dev` branch
* [akash-controller-demo repository]( ToDo )
    - clone this repository
* [Hermes](https://hermes.informal.systems/quick-start/installation.html)
* [jq](https://stedolan.github.io/jq/download/)

# Setup test environment

This chapter will set the necessaries modules for Hermes relayer, and the Agoric and Akash networks.  
The steps here described were based on this Pegasus [Local IBC Demo](https://github.com/anilhelvaci/agoric-sdk/blob/ibc-example-scripts/packages/pegasus/localDemo/localDemo.md) and this guide [Akash CLI for GPU Testnet](https://docs.akash.network/testnet/gpu-testnet-client-instructions/detailed-steps).  
We advise you to explore both of these implementations to have more detailed descriptions of each step followed next.

## Update the Pegasus package
At the time of writing this(2023-07-21), there are a few adjustments to `agoric-sdk` that needs to be done. Specifically to `agoric-sdk/packages/pegasus/src/proposals/core-proposal.js` and `agoric-sdk/packages/vats/decentral-devnet-config.json`.

1. Add the code below, `publishConnections`, at the end of `core-proposal.js`.
    ```js
    export const publishConnections = async ({
    consume: { pegasusConnections: pegasusConnectionsP, client },
    }) => {
    const pegasusConnections = await pegasusConnectionsP;
    // FIXME: Be sure only to give the client the connections if _addr is on the
    // allowlist.
    return E(client).assignBundle([_addr => ({ pegasusConnections })]);
    };
    harden(publishConnections);
    ```

2. Update `getManifestForPegasus` to the version below.
    ```js
    export const getManifestForPegasus = ({ restoreRef }, { pegasusRef }) => ({
    manifest: {
        startPegasus: {
        consume: { board: t, namesByAddress: t, zoe: t },
        installation: {
            consume: { [CONTRACT_NAME]: t },
        },
        instance: {
            produce: { [CONTRACT_NAME]: t },
        },
        },
        listenPegasus: {
        consume: { networkVat: t, pegasusConnectionsAdmin: t, zoe: t },
        produce: { pegasusConnections: t, pegasusConnectionsAdmin: t },
        instance: {
            consume: { [CONTRACT_NAME]: t },
        },
        },
        [publishConnections.name]: {
        consume: { pegasusConnections: t, client: t },
        },
    },
    installations: {
        [CONTRACT_NAME]: restoreRef(pegasusRef),
    },
    });
    ```
  
3. Update `addPegasusTransferPort` to the version below.

    ```js
    export const addPegasusTransferPort = async (
    port,
    pegasus,
    pegasusConnectionsAdmin,
    ) => {
    const { handler, subscription } = await E(pegasus).makePegasusConnectionKit();
    observeIteration(subscription, {
        updateState(connectionState) {
        const { localAddr, actions } = connectionState;
        if (actions) {
            // We're open and ready for business.
            E(pegasusConnectionsAdmin).update(localAddr, connectionState); // !!! Wrap around E()
        } else {
            // We're closed.
            E(pegasusConnectionsAdmin).delete(localAddr); // !!! Wrap around E()
        }
        },
    });
    return E(port).addListener(
        Far('listener', {
        async onAccept(_port, _localAddr, _remoteAddr, _listenHandler) {
            return handler;
        },
        async onListen(p, _listenHandler) {
            console.debug(`Listening on Pegasus transfer port: ${p}`);
        },
        }),
    );
    };
    ```

4. Update `decentral-devnet-config.json` by adding `"@agoric/pegasus/scripts/init-core.js"` as the first element of the array with the `"coreProposals"` key.

    ```json
    {
    "bootstrap": "bootstrap",
    "defaultReapInterval": 1000,
    "snapshotInterval": 1000,
    "coreProposals": [
        "@agoric/pegasus/scripts/init-core.js",
        "@agoric/vats/scripts/init-core.js",
        {
        
        .....

    }
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

## Start Akash Testnet Client

1. Install Akash Binary.
    
    In a new terminal, download Akash Binary
    ```shell
    cd ~/Downloads
    wget https://github.com/akash-network/provider/releases/download/v0.3.1-rc1/provider-services_0.3.1-rc1_darwin_all.zip
    unzip provider-services_0.3.1-rc1_darwin_all.zip
    ```

    Move the binary file into a directory included in your path
    ```shell
    sudo mv provider-services /usr/local/bin
    ```

    Verify the installation by using a simple command to check the Akash version
    ```shell
    provider-services version
    ```

    Should output something like this:
    ```shell
    v0.3.1-rc1
    ```

2. Configure the name of your key.
    ```shell
    cd akash-controller-demo/akash
    AKASH_KEY_NAME=myWallet
    ```

    Verify you have the shell variables set up 
    ```shell
    echo $AKASH_KEY_NAME
    ```

3. Set the AKASH_KEYRING_BACKEND environmental variable.
    ```shell
    AKASH_KEYRING_BACKEND=os
    ```

4. Create an Akash account:
    ```shell
    provider-services keys add $AKASH_KEY_NAME
    ```
    <mark>Important:</mark> save your mnemonic phrase in a safe place.
    

5. Set a Shell Variable in Terminal AKASH_ACCOUNT_ADDRESS to save your account address for later.
    ```shell
    export AKASH_ACCOUNT_ADDRESS="$(provider-services keys show $AKASH_KEY_NAME -a)"
    echo $AKASH_ACCOUNT_ADDRESS
    ```

6. Fund your account

    Copy your wallet address from the terminal and paste it on this faucet:
    https://faucet.testnet-02.aksh.pw/  
    Wait to see the transaction hash confirmation.


7. Configure the Testnet Chain ID and RPC Node
    ```shell
    export AKASH_CHAIN_ID=testnet-02
    export AKASH_NODE=http://rpc.testnet-02.aksh.pw:26657
    ```

    Confirm your network variables are setup
    ```shell
    echo $AKASH_NODE $AKASH_CHAIN_ID $AKASH_KEYRING_BACKEND
    ```

    You should see something similar to:
    ```shell
    http://rpc.testnet-02.aksh.pw:26657 testnet-02 os
    ```

8. Set Additional Environment Variables
    ```shell
    export AKASH_GAS=auto
    export AKASH_GAS_ADJUSTMENT=1.25
    export AKASH_GAS_PRICES=0.025uakt
    export AKASH_SIGN_MODE=amino-json
    ```

9. Check your Account Balance
    ```shell
    provider-services query bank balances --node $AKASH_NODE $AKASH_ACCOUNT_ADDRESS
    ```

    You should see a response similar to:
    ```shell
    balances:
    - amount: "25000000"
    denom: uakt
    pagination:
    next_key: null
    total: "0"
    ```

10. Create your Certificate

    ```shell
    provider-services tx cert generate client --from $AKASH_KEY_NAME
    provider-services tx cert publish client --from $AKASH_KEY_NAME
    ```

11. Create your Deployment

    Go to the same directory of your deployment configuration. For this demo, we are using the same [configuration](https://docs.akash.network/testnet/gpu-testnet-client-instructions/detailed-steps/part-5.-create-your-configuration) as the one provided by the testnet deployment guide.

    ```shell
    cd akash-controller-demo/akash
    provider-services tx deployment create deploy.yml --from $AKASH_KEY_NAME 
    ```

12. Find your Deployment

    Find the Deployment Sequence (DSEQ) in the deployment you just created, it will be printed on the output of the previous command. You will need to replace `CHANGETHIS` with that value.

    ```shell
    export AKASH_DSEQ=CHANGETHIS
    ```

    ```shell
    AKASH_OSEQ=1
    AKASH_GSEQ=1
    ```

    Verify we have the right values
    ```shell
    echo $AKASH_DSEQ $AKASH_OSEQ $AKASH_GSEQ
    ```

13. View your Bids

    <mark>Important:</mark> to receive bids on the Akash testnet, you need to whitelist your Akash address.  
    For this purpose you can either open a PR to include your address on this [document](https://github.com/akash-network/net/blob/master/testnet-02/whitelist.txt), or you can send a message to a `Akash Core Team` member on their [discord server](https://discord.gg/RMbRm4nW).

    ```shell
    provider-services query market bid list --owner=$AKASH_ACCOUNT_ADDRESS --node $AKASH_NODE --dseq $AKASH_DSEQ --state=open
    ```


## Add myWallet keys to Hermes

1. Create a seed file under `$HOME/.hermes/` containing `myWallet`'s key info
    ```shell
    cd akash-controller-demo/akash
    provider-services keys show myWallet --output json > $HOME/.hermes/myWallet_key_seed.json
    ```

2. Add myWallet's mnemonic to myWallet_key_seed.json

    Paste the mnemonic generated on `step 4` of Start Akash Testnet Client section, and replace it in the command below with `<Paste the printed mnemonic here>`.
    ```shell
    jq --arg mnemonic "<Paste the printed mnemonic here>"  '. + {mnemonic: $mnemonic}'  $HOME/.hermes/myWallet_key_seed.json > $HOME/.hermes/myWallet_tmp.json && cat $HOME/.hermes/myWallet_tmp.json > $HOME/.hermes/myWallet_key_seed.json && rm $HOME/.hermes/myWallet_tmp.json

    jq . $HOME/.hermes/myWallet_key_seed.json
    ```

    Should output something like this:
    ```text
    {
    "name": "myWallet",
    "type": "local",
    "address": "akash1ncruylun2ltreurh3xevxpdtmk8jr9sn83wq0g",
    "pubkey": "{\"@type\":\"/cosmos.crypto.secp256k1.PubKey\",\"key\":\"A0WxwYs71k4AazmDq8hOGrp2mWUoz97qJeA39pGSE5nL\"}",
    "mnemonic": "idea else cancel enhance artwork hobby pluck swim dove library rally street region run occur attack fuel acquire plunge this aunt great embrace ranch"
    }
    ```

3. In a new terminal, add myWallet keys to Hermes
    ```shell
    cd ~
    hermes keys add --chain testnet-02 --key-file .hermes/myWallet_key_seed.json --overwrite
    ```

## Add New Relay Path and Start Hermes

1. Add the relay path

    ```shell
    cd ~
    hermes create channel --a-chain testnet-02 --b-chain agoriclocal --a-port transfer --b-port pegasus --new-client-connection
    ```

    Should output similar to:

    ```text
    SUCCESS Channel {
        ordering: Unordered,
        a_side: ChannelSide {
            chain: BaseChainHandle {
                chain_id: ChainId {
                    id: "testnet-02",
                    version: 0,
                },
                runtime_sender: Sender { .. },
            },
            client_id: ClientId(
                "07-tendermint-0",
            ),
            connection_id: ConnectionId(
                "connection-0",
            ),
            port_id: PortId(
                "transfer",
            ),
            channel_id: Some(
                ChannelId(
                    "channel-0",
                ),
            ),
            version: None,
        },
        b_side: ChannelSide {
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
            connection_id: ConnectionId(
                "connection-0",
            ),
            port_id: PortId(
                "pegasus",
            ),
            channel_id: Some(
                ChannelId(
                    "channel-0",
                ),
            ),
            version: None,
        },
        connection_delay: 0ns,
    }
    ```

    You can check your relay path like this
    ```shell
    hermes query channels --show-counterparty --chain testnet-02
    ```

    You can also check your Pegasus connection by running the following command on your wallet REPL
    ```shell
    E(home.pegasusConnections).entries()
    ```

2. Start your Hermes instance

    ```shell
    hermes start
    ```

## Send uAKT to Agoric


1. Run the deploy-peg-remote
    
    To receive and view assets from other chains you must create your peg using the Pegasus `pegRemote` method. On the `deploy-peg-remote` script we are pegging the uAKT asset.

    ```shell
    cd akash-controller-demo
    agoric deploy api/deploy-peg-remote.js
    ```

2. Execute a transaction to send uAKT from Akash to Agoric 

    ```shell
    hermes tx ft-transfer --timeout-seconds 1000 --dst-chain agoriclocal --src-chain testnet-02 --src-port transfer --src-channel channel-3 --amount 1_000_000 --denom uakt
    ```

# Run the Demo 

## Deploy the akashController contract

1. Run the `deploy.js` script. 
    
    This script takes the `akashController` contract code, installs it on Zoe, and makes the installation available on the newly created `installationConstants.js` file.

    ```shell
    cd akash-controller-demo
    agoric deploy contract/deploy.js
    ```

2. Update the Akash client variables

    ToDo: specify exactly what should be updated.
    Find the best configuration for the demo

3. Run the `deploy-onchain-agent.js` script.

    This script will initiate the Akash client using the 
    ```shell
    agoric deploy api/deploy-onchain-agent.js
    ```

## Verify that it is working

ToDo: see the best approach for this step