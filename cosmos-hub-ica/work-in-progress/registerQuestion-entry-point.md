

```js
  const [icaGovernorCreatorFacet, voteCounterInstall] = await Promise.all([
    E(scratch).get(`creatorFacet.${governor}`),
    E(scratch).get(`installation.${voteCounter}`),
  ]);

  const connectionParams = harden([
    {
      hostPortId: 'icahost',
      controllerConnectionId: 'connection-0',
      hostConnectionId: 'connection-855',
    },
  ]);

  const votingDuration = 120n;
  const now = await E(chainTimerService).getCurrentTimestamp();
  const deadline = now + votingDuration;


  const { instance, outcomeOfUpdate } = await E(
    icaGovernorCreatorFacet,
  ).voteOnApiInvocation(
    'registerAccount',
    connectionParams,
    voteCounterInstall,
    deadline,
  );

  console.log('Writing details');
  const publicFacet = E(zoe).getPublicFacet(instance);

  await E(scratch).set(`publicFacet.${registerQuestion}`, publicFacet);
```

icaGovernorCreatorFacet.voteOnApiInvocation

```js
  const { voteOnApiInvocation, createdQuestion: createdApiQuestion } =
    await initApiGovernance();

// ---------

  // this conditional was extracted so both sides are equally asynchronous
  /** @type {() => Promise<ApiGovernor>} */
  const initApiGovernance = async () => {
    const [governedApis, governedNames] = await Promise.all([
      E(governedCF).getGovernedApis(),
      E(governedCF).getGovernedApiNames(),
    ]);
    if (governedNames.length) {
      return setupApiGovernance(
        zoe,
        governedInstance,
        governedApis,
        governedNames,
        timer,
        getUpdatedPoserFacet,
      );
    }

    // if we aren't governing APIs, voteOnApiInvocation shouldn't be called
    return {
      voteOnApiInvocation: () => {
        throw Error('api governance not configured');
      },
      createdQuestion: () => false,
    };
  };
```

governAPI.setupApiGovernance.voteOnApiInvocation

```js
 /** @type {VoteOnApiInvocation} */
  const voteOnApiInvocation = async (
    apiMethodName,
    methodArgs,
    voteCounterInstallation,
    deadline,
  ) => {
    governedNames.includes(apiMethodName) ||
      Fail`${apiMethodName} is not a governed API.`;

    const { positive, negative } = makeApiInvocationPositions(
      apiMethodName,
      methodArgs,
    );

    /** @type {ApiInvocationIssue} */
    const issue = harden({ apiMethodName, methodArgs });
    const questionSpec = coerceQuestionSpec({
      method: ChoiceMethod.UNRANKED,
      issue,
      positions: [positive, negative],
      electionType: ElectionType.API_INVOCATION,
      maxChoices: 1,
      maxWinners: 1,
      closingRule: { timer, deadline },
      quorumRule: QuorumRule.MAJORITY,
      tieOutcome: negative,
    });

    const { publicFacet: counterPublicFacet, instance: voteCounter } = await E(
      getUpdatedPoserFacet(),
    ).addQuestion(voteCounterInstallation, questionSpec);

    voteCounters.add(voteCounter);

    // CRUCIAL: Here we wait for the voteCounter to declare an outcome, and then
    // attempt to invoke the API if that's what the vote called for. We need to
    // make sure that outcomeOfUpdateP is updated whatever happens.
    //
    // * If the vote passed, invoke the API, and return the positive position
    // * If the vote was negative, return the negative position
    // * If we can't do either, (the vote failed or the API invocation failed)
    //   return a broken promise.
    const outcomeOfUpdate = E(counterPublicFacet)
      .getOutcome()
      .then(
        /** @type {(outcome: Position) => ERef<Position>} */
        outcome => {
          if (keyEQ(positive, outcome)) {
            keyEQ(outcome, harden({ apiMethodName, methodArgs })) ||
              Fail`The question's method name (${q(
                apiMethodName,
              )}) and args (${methodArgs}) didn't match the outcome ${outcome}`;

            // E(remote)[name](args) invokes the method named 'name' on remote.
            return E(governedApis)
              [apiMethodName](...methodArgs)
              .then(() => {
                return positive;
              });
          } else {
            return negative;
          }
        },
      );

    return {
      outcomeOfUpdate,
      instance: voteCounter,
      details: E(counterPublicFacet).getDetails(),
    };
  };
```