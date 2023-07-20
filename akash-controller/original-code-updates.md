# Changes made to the original code

## IBC module

- The `local-ibc` folder was removed, this folder had an outdated implementation of hermes that would run in a docker container. For this demo we will use the Hermes relayer CLI to execute the same required operations.

- changes made to hermes configuration, replacing the outdated akash test network edgenet-1 with testnet-02.
  Note that the IP address for `testnet-02.aksh.pw` is `216.153.52.237`
  ```
  [[chains]]
  id = 'testnet-02'
  rpc_addr = 'http://216.153.52.237:443/'
  grpc_addr = 'http://216.153.52.237:9090/'
  websocket_addr = 'ws://216.153.52.237:443//websocket'
  ```

## Contract

- Replace `@agoric/eventual-send` with `@endo/eventual-send` on the package.json, contract and test.

- Replace `getCurrentAllocationJig` with `getCurrentAllocationJig` in `const remains = await E(transferSeatP).getCurrentAllocationJig();`.

- Add error handling for akashClient initialize method.

- Add creator and public facets as empty objects to remove the warning being triggered.

## Tests

- Replacing test object import to fix `The expression evaluated to a falsy value` error

  ```js
  import { test } from "@agoric/zoe/tools/prepare-test-env-ava.js";
  ```

  with

  ```js
  import "@agoric/zoe/tools/prepare-test-env.js";
  import test from "ava";
  ```

- Update the condition on the depositDeployment method of the makeFakeAkashClient function
  from:

  ```js
  t.is(amount, `${akash.deployment.value}uakt`, "Deposit amount did not match");
  ```

  to:

  ```js
  t.is(
    amount.amount,
    `${akash.deployment.value}`,
    "Deposit amount did not match"
  );
  ```

- On the `zoe - watch Akash deployment, maxCheck=1, current Fund is sufficient` test
  - Replace the value of the last assertion from 5000000n to 4980000n
  - Replace the t(1) with t(4)

## Api scripts

- replace the outdated deploy-peg.js with deploy-peg-remote.js, which was configured for uAKT.

- replace the outdated deploy-send-ibc.js with deploy-ibc-send.js, which was configured for uAKT.

- remove agent.js and deploy-offchain-agent.js, they use a different implementation that excludes the akashController contract.

- create deploy-config.js to store the information related with the agoric-akash channel, remote pegged asset and the akash account.

- update deploy-onchain-agent.js:
  - replace `akt` object with `akash` and `akashAccount` objects imported from deploy-config.js
  - await for the `chainTimerService` before including it on the contract terms to be hardned
  - replace `getNotifier` method with `getAllocationNotifierJig`
  - replace `E(E(wallet).getAdminFacet()).getPurse()` with `E(wallet).getPurse()`

## Akash

- add akash deployment configuration, deploy.yml, to ./akash directory.

## Package.json

- add ` "type": "module",` on the package.json at the project root