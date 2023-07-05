start-dao.js
this script will instantiate the governor contract: '@agoric/governance/src/contractGovernor.js'

```js
const icaContractBundle = await bundleSource(
  pathResolve(`../src/ica-dao/ica-dao.js`)
);
const icaContractInstall = await E(zoe).install(icaContractBundle);

const [
  governorInstall,
  committeeCreator,
  electorateInstance,
  icarusInstance,
  timer,
] = await Promise.all([
  E(scratch).get(`installation.${governor}`),
  E(scratch).get(`creatorFacet.${committee}`),
  E(scratch).get(`instance.${committee}`),
  E(scratch).get(`instance.${icarus}`),
  chainTimerService,
]);

const poserInvitationP = E(committeeCreator).getPoserInvitation();
const [poserInvitation, poserInvitationAmount] = await Promise.all([
  poserInvitationP,
  E(E(zoe).getInvitationIssuer()).getAmountOf(poserInvitationP),
]);

const icaTerms = harden({
  governedParams: {
    Electorate: {
      type: ParamTypes.INVITATION,
      value: poserInvitationAmount,
    },
    IcarusInstance: {
      type: ParamTypes.INSTANCE,
      value: icarusInstance,
    },
  },
});

const icaGovernorTerms = {
  timer,
  electorateInstance,
  governedContractInstallation: icaContractInstall,
  governed: {
    terms: icaTerms,
    issuerKeywordRecord: {},
  },
};

// start instance
console.log("Starting governor");
const g = await E(zoe).startInstance(governorInstall, {}, icaGovernorTerms, {
  electorateCreatorFacet: committeeCreator,
  governed: {
    initialPoserInvitation: poserInvitation,
  },
});
```

contractGovernor.js
The governor will instantiate the ica-dao.js contract

```js

  const augmentedTerms = harden({
    ...contractTerms,   // = icaTerms
    electionManager: zcf.getInstance(),  // governor instance


  const {
    creatorFacet: governedCF,
    instance: governedInstance,
    publicFacet: governedPF,
    adminFacet,
  } = await E(zoe).startInstance(
    governedContractInstallation,
    governedIssuerKeywordRecord,  // = {}
    // @ts-expect-error XXX governance types
    augmentedTerms,
    privateArgs.governed, // = poserInvitation
  );



    const creatorFacet = Far('governor creatorFacet', {
    replaceElectorate,
    voteOnParamChanges,
    voteOnApiInvocation,
    voteOnOfferFilter: voteOnFilter,
    getCreatorFacet: () => limitedCreatorFacet,
    getAdminFacet: () => adminFacet,
    getInstance: () => governedInstance,
    getPublicFacet: () => governedPF,
  });

  const publicFacet = Far('contract governor public', {
    getElectorate: getElectorateInstance,
    getGovernedContract: () => governedInstance,
    validateVoteCounter,
    validateElectorate,
    validateTimer,
  });

  return { creatorFacet, publicFacet };

```

the script start-dao will store the following values to scratch

```js
    E(scratch).set(`creatorFacet.${governor}`, g.creatorFacet),
    E(scratch).set(`creatorFacet.${governedIca}`, creatorFacet),
    E(scratch).set(`publicFacet.${governedIca}`, publicFacet),
    E(scratch).set(`instance.${governedIca}`, instance),

    // ---- creatorFacet.${governor} -----

    const creatorFacet = Far('governor creatorFacet', {
    replaceElectorate,
    voteOnParamChanges,
    voteOnApiInvocation,
    voteOnOfferFilter: voteOnFilter,
    getCreatorFacet: () => limitedCreatorFacet,
    getAdminFacet: () => adminFacet,
    getInstance: () => governedInstance,
    getPublicFacet: () => governedPF,
  });

    // ---- creatorFacet.${governedIca} -----

const limitedCreatorFacet = E(governedCF).getLimitedCreatorFacet();

const creatorApis = {
  async claimReward(args) {
    return sendIcaTxMsg("claimReward", args);
  },
  reconnectAccount,
};

    // ---- `publicFacet.${governedIca}`-----

  const augmentPublicFacet = originalPublicFacet => {
    return Far('publicFacet', {
      ...originalPublicFacet,
      ...commonPublicMethods,
      ...typedAccessors,
    });

    // see bellow the methods exposed
    // ---- `instance.${governedIca}`-----

// returns ica-dao.js instance
```

The next snippet explains the output of creatorFacet.${governedIca}

```js
// ica-dao.js

const creatorApis = {
  async claimReward(args) {
    return sendIcaTxMsg("claimReward", args);
  },
  reconnectAccount,
};

// the handleParamGovernance is a method imported from '@agoric/governance';
const { augmentPublicFacet, makeGovernorFacet, params } =
  await handleParamGovernance(zcf, privateArgs.initialPoserInvitation, {
    [ICARUS_INSTANCE]: ParamTypes.INSTANCE,
  });

const creatorFacet = makeGovernorFacet(creatorApis, governedApis);

const makeFarGovernorFacet = (limitedCreatorFacet, governedApis = {}) => {
  const governorFacet = Far("governorFacet", {
    getParamMgrRetriever: () =>
      Far("paramRetriever", { get: () => paramManager }),
    getInvitation: (name) => paramManager.getInternalParamValue(name),
    getLimitedCreatorFacet: () => limitedCreatorFacet,
    // The contract provides a facet with the APIs that can be invoked by
    // governance
    /** @type {() => GovernedApis} */
    // @ts-expect-error TS think this is a RemotableBrand??
    getGovernedApis: () => Far("governedAPIs", governedApis),
    // The facet returned by getGovernedApis is Far, so we can't see what
    // methods it has. There's no clean way to have contracts specify the APIs
    // without also separately providing their names.
    getGovernedApiNames: () => Object.keys(governedApis),
    setOfferFilter: (strings) => zcf.setOfferFilter(strings),
  });

  // exclusively for contractGovernor, which only reveals limitedCreatorFacet
  return governorFacet;
};

/**
 * @template {{}} CF
 * @param {CF} originalCreatorFacet
 * @param {{}} [governedApis]
 * @returns {GovernorFacet<CF>}
 */
const makeGovernorFacet = (originalCreatorFacet, governedApis = {}) => {
  const limitedCreatorFacet = makeLimitedCreatorFacet(originalCreatorFacet);
  return makeFarGovernorFacet(limitedCreatorFacet, governedApis);
};

// this means that When the governor contract calls E(governedCF).getLimitedCreatorFacet();
// the result will be the ica-dao.js creatorApis
```

```js
const publicApis = {
  isReady() {
    return E(icaActions).isReady();
  },
  getAddress() {
    return E(icaActions).getAddress();
  },
  async getPortId() {
    const controller = await getUpdatedController();
    return E(controller).getPortId();
  },
};

const publicFacet = augmentPublicFacet(publicApis);

// contractHelper. same as above
const typedAccessors = {
  getAmount: paramManager.getAmount,
  getBrand: paramManager.getBrand,
  getInstance: paramManager.getInstance,
  getInstallation: paramManager.getInstallation,
  getInvitationAmount: paramManager.getInvitationAmount,
  getNat: paramManager.getNat,
  getRatio: paramManager.getRatio,
  getString: paramManager.getString,
  getUnknown: paramManager.getUnknown,
};

const { electionManager } = terms;

const commonPublicMethods = {
  getSubscription: () => paramManager.getSubscription(),
  getContractGovernor: () => electionManager,
  getGovernedParams: () => paramManager.getParams(),
};

/**
 * Add required methods to publicFacet
 *
 * @template {{}} PF public facet
 * @param {PF} originalPublicFacet
 * @returns {GovernedPublicFacet<PF>}
 */
const augmentPublicFacet = (originalPublicFacet) => {
  return Far("publicFacet", {
    ...originalPublicFacet,
    ...commonPublicMethods,
    ...typedAccessors,
  });
};
```

# LimitedCreatorFacet

**ica-dao**
```js
  const creatorFacet = makeGovernorFacet(creatorApis, governedApis);
```

**contractHelper**
```js
  /**
   * @template {{}} CF
   * @param {CF} originalCreatorFacet
   * @param {{}} [governedApis]
   * @returns {GovernorFacet<CF>}
   */
  const makeGovernorFacet = (originalCreatorFacet, governedApis = {}) => {
    const limitedCreatorFacet = makeLimitedCreatorFacet(originalCreatorFacet);
    return makeFarGovernorFacet(limitedCreatorFacet, governedApis);
  };


  /**
   * @template {{}} CF creator facet
   * @param {CF} originalCreatorFacet
   * @returns {LimitedCreatorFacet<CF>}
   */
  const makeLimitedCreatorFacet = originalCreatorFacet => {
    return Far('governedContract creator facet', {
      ...originalCreatorFacet,
      getContractGovernor: () => electionManager,
    });
  };

  const makeFarGovernorFacet = (limitedCreatorFacet, governedApis = {}) => {
    const governorFacet = Far('governorFacet', {
      getParamMgrRetriever: () =>
        Far('paramRetriever', { get: () => paramManager }),
      getInvitation: name => paramManager.getInternalParamValue(name),
      getLimitedCreatorFacet: () => limitedCreatorFacet,
      // The contract provides a facet with the APIs that can be invoked by
      // governance
      /** @type {() => GovernedApis} */
      // @ts-expect-error TS think this is a RemotableBrand??
      getGovernedApis: () => Far('governedAPIs', governedApis),
      // The facet returned by getGovernedApis is Far, so we can't see what
      // methods it has. There's no clean way to have contracts specify the APIs
      // without also separately providing their names.
      getGovernedApiNames: () => Object.keys(governedApis),
      setOfferFilter: strings => zcf.setOfferFilter(strings),
    });

```

**contractGovernor**
```js
// CRUCIAL: only contractGovernor should get the ability to update params
const limitedCreatorFacet = E(governedCF).getLimitedCreatorFacet();

const creatorFacet = Far("governor creatorFacet", {
  replaceElectorate,
  voteOnParamChanges,
  voteOnApiInvocation,
  voteOnOfferFilter: voteOnFilter,
  getCreatorFacet: () => limitedCreatorFacet,
  getAdminFacet: () => adminFacet,
  getInstance: () => governedInstance,
  getPublicFacet: () => governedPF,
});
```

# publicFacet

```js
const augmentPublicFacet = (originalPublicFacet) => {
  return Far("publicFacet", {
    ...originalPublicFacet,
    ...commonPublicMethods,
    ...typedAccessors,
  });
};

// -- originalPublicFacet
const publicApis = {
  isReady() {
    return E(icaActions).isReady();
  },
  getAddress() {
    return E(icaActions).getAddress();
  },
  async getPortId() {
    const controller = await getUpdatedController();
    return E(controller).getPortId();
  },
};

// -- commonPublicMethods
const commonPublicMethods = {
  getSubscription: () => paramManager.getSubscription(),
  getContractGovernor: () => electionManager,
  getGovernedParams: () => paramManager.getParams(),
};

// -- typedAccessors
const typedAccessors = {
  getAmount: paramManager.getAmount,
  getBrand: paramManager.getBrand,
  getInstance: paramManager.getInstance,
  getInstallation: paramManager.getInstallation,
  getInvitationAmount: paramManager.getInvitationAmount,
  getNat: paramManager.getNat,
  getRatio: paramManager.getRatio,
  getString: paramManager.getString,
  getUnknown: paramManager.getUnknown,
};

```

ToDo: understand what is the paramManager and where it comes from!
What is the const poserInvitationP = E(committeeCreator).getPoserInvitation();

```js
  const {
    augmentPublicFacet,
    makeGovernorFacet,
    params,
  } = await handleParamGovernance(zcf, privateArgs.initialPoserInvitation, {
    [ICARUS_INSTANCE]: ParamTypes.INSTANCE,
  });


  -----------------------------

  const handleParamGovernance = async (
  zcf,
  initialPoserInvitation,
  paramTypesMap,
  storageNode,
  marshaller,
) => {
  /** @type {import('@agoric/notifier').StoredPublisherKit<GovernanceSubscriptionState>} */
  const publisherKit = makeStoredPublisherKit(
    storageNode,
    marshaller,
    GOVERNANCE_STORAGE_KEY,
  );
  const paramManager = await makeParamManagerFromTerms(
    publisherKit,
    zcf,
    initialPoserInvitation,
    paramTypesMap,
  );

  return facetHelpers(zcf, paramManager);
};

```

methods exported by paramManager.js, which I think it is related to the committee
Every governed contract has an Electorate param that starts as `initialPoserInvitation` private arg

