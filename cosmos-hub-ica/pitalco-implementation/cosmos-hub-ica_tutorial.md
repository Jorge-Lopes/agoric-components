# Tutorial

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

gaiad init test --chain-id=theta-testnet-001

gaiad keys add cosmosWallet -i --keyring-backend=test

```text
- name: cosmosWallet
  type: local
  address: cosmos1qadwqhf49zfztygpm8gur7v0zlcm5h9p45uwcw
  pubkey: '{"@type":"/cosmos.crypto.secp256k1.PubKey","key":"AycnhE9URXBLwSDCGnuU6L4LfoMTS/0fdgl5OFHZJ3b4"}'
  mnemonic: ""


**Important** write this mnemonic phrase in a safe place.
It is the only way to recover your account if you ever forget your password.

busy wage range hungry legend hard digital idle about build snow grant tunnel reward jewel puppy wealth hammer stable cloth amused toward result oyster
```

gaiad keys show cosmosWallet --output json > $HOME/.hermes/cosmosWallet_key_seed.json

jq --arg mnemonic "busy wage range hungry legend hard digital idle about build snow grant tunnel reward jewel puppy wealth hammer stable cloth amused toward result oyster"  '. + {mnemonic: $mnemonic}'  $HOME/.hermes/cosmosWallet_key_seed.json > $HOME/.hermes/myWallet_tmp.json && cat $HOME/.hermes/myWallet_tmp.json > $HOME/.hermes/cosmosWallet_key_seed.json && rm $HOME/.hermes/myWallet_tmp.json

jq . $HOME/.hermes/cosmosWallet_key_seed.json

cd ~
hermes keys add --chain theta-testnet-001 --key-file .hermes/cosmosWallet_key_seed.json --overwrite


cd ~
hermes create connection --a-chain agoriclocal --b-chain theta-testnet-001 