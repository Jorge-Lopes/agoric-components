# Agoric Tutorial: Cosmos Hub ICA

## Prerequisites

## Components

- **Relayer**: Hermes
- **Host Chain**: CosmosHub
- **Controller Chain**: Agoric
- **Icarus**: IBC Authentication module in Agoric
- **ICA-DAO**: DAO controlling remote account

## Steps

### Spin chain nodes

1.  Agoric

        cd agoric-sdk/packages/cosmic-swingset
        make scenario2-setup && make scenario2-run-chain
        make scenario2-run-client
        agoric open -repl

2.  Cosmos

        gm start ibc-0 node-0

### Add accounts

1.  Cosmos

        gm keys
        gm hermes keys

2.  Agoric

        cd agoric-sdk/packages/cosmic-swingset

        agd --home t1/8000/ag-cosmos-helper-statedir/ --keyring-backend=test keys show ag-solo --output json > $HOME/.hermes/ag-solo_key_seed.json

        jq --arg mnemonic "$(cat t1/8000/ag-solo-mnemonic)" '. + {mnemonic: $mnemonic}' $HOME/.hermes/ag-solo_key_seed.json > $HOME/.hermes/ag-solo_tmp.json && cat $HOME/.hermes ag-solo_tmp.json > $HOME/.hermes/ag-solo_key_seed.json && rm $HOME/.hermes/ag-solo_tmp.json

        jq . $HOME/.hermes/ag-solo_key_seed.json

### Restore keys

        cd ~
        hermes keys add --chain agoriclocal --key-file .hermes/ag-solo_key_seed.json --hd-path "m/44'/564'/0'/0/0" --overwrite

### Create connection/channel

        hermes create connection --a-chain ibc-0 --b-chain agoriclocal

### Start hermes

        hermes start

### Deploy scripts

        cd agoric-ica-demo

        agoric deploy contract/deploy/install-contracts.js

        agoric deploy contract/deploy/start-icarus.js

        agoric deploy contract/deploy/start-committee.js

        agoric deploy contract/deploy/start-dao.js

        agoric deploy contract/deploy/create-voter-facet

        agoric deploy contract/deploy/pose-register-question

        or

        agoric deploy contract/deploy/pose-api-question 

        agoric deploy contract/deploy/vote-question

        agoric deploy contract/deploy/send-invitations

        agoric deploy contract/deploy/reconnect-account
