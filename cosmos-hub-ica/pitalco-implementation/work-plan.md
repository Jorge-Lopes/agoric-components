# Work Plan

Repository: https://github.com/pitalco/interaccounts

Component page: https://components.agoric.com/smart-contracts/cross-chain/cosmos-hub-ica

## Research

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

## Cosmos testnet

Name: theta-testnet-001

- https://github.com/cosmos/testnets/blob/master/public/README.md
- https://testnet.cosmos.directory/cosmoshubtestnet

## ICA implementations

- https://github.com/srdtrk/cw-ica-controller/tree/main
- https://github.com/cosmos/interchain-accounts-demo


# Debug issues

## verify rpc and grpc address

### rpc

Address: http://104.245.147.200:26657

```shell
curl -X POST -H "Content-Type: application/json" -d '{     
  "jsonrpc": "2.0",
  "method": "abci_info",
  "params": {},
  "id": 1
}' http://104.245.147.200:26657
```

```shell
{"jsonrpc":"2.0","id":1,"result":{"response":{"data":"GaiaApp","version":"11.0.0-rc0","app_version":"5","last_block_height":"17136892","last_block_app_hash":"OuTJ/nsIRK5NLFCY5VZZ62OGHZmjVAwAtkCRfeQV+Zs="}}}% 
```

### grpc

Address: http://104.245.147.200:9090

```shell
grpcurl -plaintext 104.245.147.200:9090 list
```

```shell
cosmos.auth.v1beta1.Query
cosmos.authz.v1beta1.Query
cosmos.bank.v1beta1.Query
cosmos.base.node.v1beta1.Service
cosmos.base.reflection.v1beta1.ReflectionService
cosmos.base.reflection.v2alpha1.ReflectionService
cosmos.base.tendermint.v1beta1.Service
cosmos.distribution.v1beta1.Query
cosmos.evidence.v1beta1.Query
cosmos.feegrant.v1beta1.Query
cosmos.gov.v1beta1.Query
cosmos.mint.v1beta1.Query
cosmos.params.v1beta1.Query
cosmos.slashing.v1beta1.Query
cosmos.staking.v1beta1.Query
cosmos.tx.v1beta1.Service
cosmos.upgrade.v1beta1.Query
gaia.globalfee.v1beta1.Query
grpc.reflection.v1alpha.ServerReflection
ibc.applications.interchain_accounts.host.v1.Query
ibc.applications.transfer.v1.Query
ibc.core.channel.v1.Query
ibc.core.client.v1.Query
ibc.core.connection.v1.Query
interchain_security.ccv.provider.v1.Query
router.v1.Query
tendermint.liquidity.v1beta1.Query
```
