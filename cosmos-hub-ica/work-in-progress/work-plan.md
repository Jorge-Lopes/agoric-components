# Work Plan

- Identify research topics relevant for this component
- Explore the topics identified
- Review the bounty code
- Build a diagram to improve visualization
- Build components page structure
- Build tutorial page structure
- Identify the required steps to build the tutorial
- Record the issues being faced

# Research Topics

## IBC

https://ibc.cosmos.network/

https://tutorials.cosmos.network/academy/3-ibc/1-what-is-ibc.html

## Hermes

https://hermes.informal.systems/index.html

https://github.com/informalsystems/hermes/blob/master/config.toml

## Gaia

https://hub.cosmos.network/main/getting-started/what-is-gaia.html

## IBC demo

https://github.com/anilhelvaci/agoric-sdk/blob/ibc-example-scripts/packages/pegasus/localDemo/localDemo.md

https://github.com/Agoric/agoric-sdk/pull/7533

https://gist.github.com/tgrecojs

## ICA

https://github.com/cosmos/ibc/blob/main/spec/app/ics-027-interchain-accounts/README.md

https://github.com/cosmos/interchain-accounts-demo

https://github.com/schnetzlerjoe/interaccounts

https://github.com/srdtrk/cw-ica-controller

## Governance

https://github.com/Agoric/agoric-sdk/tree/master/packages/governance


# Build Tutorial

There are three implementations found to establish the ICA:

- Run schnetzlerjoe/interaccounts demo

- Run agoric IBC demo

  - https://github.com/anilhelvaci/agoric-sdk/blob/ibc-example-scripts/packages/pegasus/localDemo/localDemo.md

  Note: initially I thought I could configure this demo to the current bounty. But it is not suitable because we are not going to use pegasus but icarus instead.
  Where one regards the ICS20 standard and the other ICS27.

- Run cosmos ICA demo

  - https://github.com/informalsystems/hermes

  Note: the goal is to configure this demo to create an ICA between agoric and cosmos, and then deploy the bounty contracts on agoric and execute the scripts

## First approach (execute the bounty code)

- Questions raised regarding the local-ibc module
- It use a hermes image to run a docker container
- Follows a structure similar with the IBC transfer demo that DAN has developed (outdated)
  - This makes me believe that this configuration is outdated as well, but I can be wrong
  - Demo https://gist.github.com/dckc/9a336a983a30a20751b38380e3308fab

When executing the following command at /local-ibc :

> make start

This error is triggered:

```
MNEMONIC="$(cat ibc-relay-mnemonic)"; \
echo $MNEMONIC | sha1sum ; \
docker run --rm -vhermes-home:/home/hermes:z -v$PWD:/config hermes:v1.0.0-rc.0 --config /config/hermes.config keys add --chain sim-agoric-7 --mnemonic-file /config/ibc-relay-mnemonic --hd-path "m/44'/564'/0'/0/0"; \
docker run --rm -vhermes-home:/home/hermes:z -v$PWD:/config hermes:v1.0.0-rc.0 --config /config/hermes.config keys add --chain theta-testnet-001 --mnemonic-file /config/ibc-relay-mnemonic;
6fe694d6bf408fbc46c6146953d9feb5298ba120 -
Success: Restored key 'agdevkey' (agoric12t2yqeg4pdne7w7fadacvp8l8afevdsumhtswr) on chain sim-agoric-7
Success: Restored key 'cosmoshub' (cosmos1h68l7uqw255w4m2v82rqwsl6p2qmkrg08u5mye) on chain theta-testnet-001
mkdir -p task && touch task/restore-keys
tapping faucet
goto discord channel https://discord.com/channels/669268347736686612/953697793476821092
paste equest cosmos1h68l7uqw255w4m2v82rqwsl6p2qmkrg08u5mye theta
mkdir -p task && touch task/tap-cosmos-faucet
tapping agoric faucet
agoric address agoric12t2yqeg4pdne7w7fadacvp8l8afevdsumhtswr
agd --home=../\_agstate/keys tx bank send -ojson --keyring-backend=test --gas=auto --gas-adjustment=1.2 --broadcast-mode=block --yes --chain-id=sim-agoric-7 --node=tcp://localhost:26657 provision agoric12t2yqeg4pdne7w7fadacvp8l8afevdsumhtswr 13000000ubld,50000000uist
Error: provision.info: key not found
```

## Second approach (Use the CLI to configure the IBC connection)

- Use this demo as a guide:

  - https://github.com/cosmos/interchain-accounts-demo

- changes made to original repo:

  - init.sh

    - add-genesis-account -> genesis add-genesis-account
    - gentx -> genesis gentx
    - collect-gentxs -> genesis collect-gentxs

  - create-conn.sh & restore-keys.sh

    - -c -> --config

  - restore-keys.sh

    - keys restore -> keys add --chain

  - hermes/config.toml
    - gas_adjustment = 0.1 -> gas_multiplier = 1.1

- an error will be triggered when executing ` make init-hermes`
  solution:
  - manually execute the scripts one by one
  - after running the `init.sh` manually update test-1 app.toml tcp address to `address = "tcp://localhost:1316" ` before executing the `start.sh` script
  - update `/data/test-2/config/genesis.json` "allow_messages": to [ "*" ]

## Steps

### Spin chain nodes

1. Agoric

[ibc-demo]

> cd agoric-sdk/packages/cosmic-swingset
> make scenario2-setup && make scenario2-run-chain
> make scenario2-run-client

or

[schnetzlerjoe/interaccounts]

> agoric start local-chain

2. Cosmos

[ibc-demo]

> gm start

or

[schnetzlerjoe/interaccounts]

> gaiad init test --home $CHAIN_DIR/$CHAINID_1 --chain-id=$CHAINID_1

or

[cosmos/interchain-accounts-demo]

> icad init test --home $CHAIN_DIR/$CHAINID_1 --chain-id=$CHAINID_1

### Add accounts

1. Agoric

[ibc-demo]

> cd agoric-sdk/packages/cosmic-swingset

> agd --home t1/8000/ag-cosmos-helper-statedir/ --keyring-backend=test keys show ag-solo --output json > $HOME/.hermes/ag-solo_key_seed.json

> jq --arg mnemonic "$(cat t1/8000/ag-solo-mnemonic)" '. + {mnemonic: $mnemonic}' $HOME/.hermes/ag-solo_key_seed.json > $HOME/.hermes/ag-solo_tmp.json && cat $HOME/.hermes ag-solo_tmp.json > $HOME/.hermes/ag-solo_key_seed.json && rm $HOME/.hermes/ag-solo_tmp.json jq . $HOME/.hermes/ag-solo_key_seed.json

or

[schnetzlerjoe/interaccounts]

> echo $RLY_MNEMONIC_1 | ~/go/bin/agd keys add rly1 --home=\_agstate/keys --keyring-backend test --recover

2. Cosmos

[ibc-demo]

> gm keys
> gm hermes keys

or

[schnetzlerjoe/interaccounts]

> echo $DEMO_MNEMONIC_1 | gaiad keys add demowallet --home $CHAIN_DIR/$CHAINID_1 --recover --keyring-backend=test
> echo $RLY_MNEMONIC_1 | $BINARY keys add rly --home $CHAIN_DIR/$CHAINID_1 --recover --keyring-backend=test

or

[cosmos/interchain-accounts-demo]

> echo $VAL_MNEMONIC_1 | icad keys add val1 --home $CHAIN_DIR/$CHAINID_1 --recover --keyring-backend=test
> echo $RLY_MNEMONIC_1 | icad keys add rly1 --home $CHAIN_DIR/$CHAINID_1 --recover --keyring-backend=test

> icad genesis add-genesis-account $(icad --home $CHAIN_DIR/$CHAINID_1 keys show val1 --keyring-backend test -a) 100000000000stake --home $CHAIN_DIR/$CHAINID_1
> icad genesis add-genesis-account $(icad --home $CHAIN_DIR/$CHAINID_1 keys show rly1 --keyring-backend test -a) 100000000000stake --home $CHAIN_DIR/$CHAINID_1

> icad genesis gentx val1 7000000000stake --home $CHAIN_DIR/$CHAINID_1 --chain-id $CHAINID_1 --keyring-backend test
>icad genesis collect-gentxs --home $CHAIN_DIR/$CHAINID_1

### Restore keys

[ibc-demo]

> > hermes keys add --chain agoriclocal --key-file .hermes/ag-solo_key_seed.json --hd-path "m/44'/564'/0'/0/0" --overwrite

or

[schnetzlerjoe/interaccounts]

> hermes --config ./network/hermes/config.toml keys add --key-name agoric --hd-path "m/44'/564'/0'/0/0" --mnemonic-file $HERMES_DIRECTORY/mnemonic.txt --chain agoric --overwrite
> sleep 5s

> hermes --config ./network/hermes/config.toml keys add --key-name theta-testnet-001 --mnemonic-file $HERMES_DIRECTORY/mnemonic.txt --chain theta-testnet-001 --overwrite
> sleep 5s

or

[cosmos/interchain-accounts-demo]

> hermes --config ./network/hermes/config.toml keys add --chain test-1 --mnemonic-file ./network/hermes/mnemonic-file-1.md
> sleep 5

> hermes --config ./network/hermes/config.toml keys add --chain test-2 --mnemonic-file ./network/hermes/mnemonic-file-2.md
> sleep 5

### Create connection/channel

[ibc-demo]

> hermes create channel --a-chain ibc-0 --b-chain agoriclocal --a-port transfer --b-port pegasus --new-client-connection

`NOTE: how to identify chain port?`

or

[schnetzlerjoe/interaccounts]

> hermes --config ./network/hermes/config.toml create connection --a-chain test-1 --b-chain test-2

or

[cosmos/interchain-accounts-demo]

> hermes --config ./network/hermes/config.toml create connection --a-chain test-1 --b-chain test-2

### Start hermes

[ibc-demo]

> hermes start

`Note: check your relay path: > hermes query channels --show-counterparty --chain ibc-0`

or

[schnetzlerjoe/interaccounts]

> hermes --config $CONFIG_DIR start >& hermes.log &

or

[cosmos/interchain-accounts-demo]

> hermes --config $CONFIG_DIR start

### Deploy scripts

1- install-contracts
2- start-icarus
3- start-committee
4- start-dao

expected (currently triggering an error)

- create-voter-facet
- send-invitations
- pose-api-question
- vote-question

or

use icad tx intertx to make ICA requests

# Open questions:

- What is Dragonberry?
  Mentioned on local-chain logs

Open questions

how the makeFarGovernorFacet makes to only reveal the limitedCreatorFacet

1. Check differences between hermes configuration on the IBC demo and ICA bounty, specially the:

- [telemetry]
- [chains.packet_filter]
- [chains.trust_threshold]

2. How the keys are being generated and stored, specially this command:

   `agd --home=../\_agstate/keys tx bank send -ojson --keyring-backend=test --gas=auto --gas-adjustment=1.2 --broadcast-mode=block --yes --chain-id=$(CHAIN_AG) --node=tcp://localhost:26657 provision $(ADDR_AG) 13000000ubld,50000000uist`

3.
