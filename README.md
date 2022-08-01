# Pact JS workshop

## Introduction

This workshop is aimed at demonstrating core features and benefits of contract testing with Pact.

Whilst contract testing can be applied retrospectively to systems, we will follow the [consumer driven contracts](https://martinfowler.com/articles/consumerDrivenContracts.html) approach in this workshop - where a new consumer and provider are created in parallel to evolve a service over time, especially where there is some uncertainty with what is to be built.

This workshop should take from 1 to 2 hours, depending on how deep you want to go into each topic.

**Workshop outline**:

- [step 1: **create consumer**](https://github.com/bookmd/pact-workshop-js/tree/step1#step-1---simple-consumer-calling-provider): Create our consumer before the Provider API even exists
- [step 2: **unit test**](https://github.com/bookmd/pact-workshop-js/tree/step2#step-2---client-tested-but-integration-fails): Write a unit test for our consumer
- [step 3: **pact test**](https://github.com/bookmd/pact-workshop-js/tree/step3#step-3---pact-to-the-rescue): Write a Pact test for our consumer
- [step 4: **pact verification**](https://github.com/bookmd/pact-workshop-js/tree/step4#step-4---verify-the-provider): Verify the consumer pact with the Provider API
- [step 5: **fix consumer**](https://github.com/bookmd/pact-workshop-js/tree/step5#step-5---back-to-the-client-we-go): Fix the consumer's bad assumptions about the Provider
- [step 6: **pact test**](https://github.com/bookmd/pact-workshop-js/tree/step6#step-6---consumer-updates-contract-for-missing-products): Write a pact test for `404` (missing User) in consumer
- [step 7: **provider states**](https://github.com/bookmd/pact-workshop-js/tree/step7#step-7---adding-the-missing-states): Update API to handle `404` case
- [step 8: **pact test**](https://github.com/bookmd/pact-workshop-js/tree/step8#step-8---authorization): Write a pact test for the `401` case
- [step 9: **pact test**](https://github.com/bookmd/pact-workshop-js/tree/step9#step-9---implement-authorisation-on-the-provider): Update API to handle `401` case
- [step 10: **request filters**](https://github.com/bookmd/pact-workshop-js/tree/step10#step-10---request-filters-on-the-provider): Fix the provider to support the `401` case
- [step 11: **pact broker**](https://github.com/bookmd/pact-workshop-js/tree/step11#step-11---using-a-pact-broker): Implement a broker workflow for integration with CI/CD
- [step 12: **simulate CI**](https://github.com/bookmd/pact-workshop-js/tree/step12#step-12---simulate-ci): Run a simulation of a CI pipeline to understand how pact interacts with CI

_NOTE: Each step is tied to, and must be run within, a git branch, allowing you to progress through each stage incrementally. For example, to move to step 2 run the following: `git checkout step2`_

## Learning objectives

If running this as a team workshop format, you may want to take a look through the [learning objectives](./LEARNING.md).

## Requirements

[Docker](https://www.docker.com)

[Docker Compose](https://docs.docker.com/compose/install/)

[Node + NPM](https://nodejs.org/en/)

## Scenario

There are two components in scope for our workshop.

1. Product Catalog website. It provides an interface to query the Product service for product information.
1. Product Service (Provider). Provides useful things about products, such as listing all products and getting the details of an individual product.

## Step 11 - Using a Pact Broker

![Broker collaboration Workflow](diagrams/workshop_step10_broker.svg)

We've been publishing our pacts from the consumer project by essentially sharing the file system with the provider. But this is not very manageable when you have multiple teams contributing to the code base, and pushing to CI. We can use a [Pact Broker](https://pactflow.io) to do this instead.

Using a broker simplifies the management of pacts and adds a number of useful features, including some safety enhancements for continuous delivery which we'll see shortly.

In this workshop we will be using the open source Pact broker.

### Running the Pact Broker with docker-compose

In the root directory, run:

```console
docker-compose up
```

### Publish contracts from consumer

First, in the consumer project we need to tell Pact about our broker. We can use the in built `pact-broker` CLI command to do this:

```javascript
// add this under scripts
"pact:publish": "pact-broker publish ./pacts --consumer-app-version=\"$(npx @pact-foundation/absolute-version)\" --auto-detect-version-properties --broker-base-url=http://localhost:8000 --broker-username pact_workshop --broker-password pact_workshop"
```

Now run

```console
❯ npm run test:pact --prefix consumer

PASS src/api.pact.spec.js
  API Pact test
    getting all products
      ✓ products exists (22ms)
      ✓ no products exists (12ms)
      ✓ no auth token (13ms)
    getting one product
      ✓ ID 10 exists (11ms)
      ✓ product does not exist (12ms)
      ✓ no auth token (14ms)

Test Suites: 1 passed, 1 total
Tests:       6 passed, 6 total
Snapshots:   0 total
Time:        2.653s
Ran all test suites matching /pact.spec.js/i.
```

To publish the pacts:

```
❯ npm run pact:publish --prefix consumer

Created FrontendWebsite version 24c0e1-step11+24c0e1.SNAPSHOT.SB-AS-G7GM9F7 with branch step11
Pact successfully published for FrontendWebsite version 24c0e1-step11+24c0e1.SNAPSHOT.SB-AS-G7GM9F7 and provider ProductService.
  View the published pact at http://localhost:8000/pacts/provider/ProductService/consumer/FrontendWebsite/version/24c0e1-step11%2B24c0e1.SNAPSHOT.SB-AS-G7GM9F7
  Events detected: contract_published (pact content is the same as previous versions with tags  and no new tags were applied)
  Next steps:
    * Configure separate ProductService pact verification build and webhook to trigger it when the pact content changes. See https://docs.pact.io/go/webhooks
```

_NOTE: you would usually only publish pacts from CI. _

Have a browse around the broker on http://localhost:8000 (with username/password: `pact_workshop`/`pact_workshop`) and see your newly published contract!

### Verify contracts on Provider

All we need to do for the provider is update where it finds its pacts, from local URLs, to one from a broker.

In `provider/product/product.pact.test.js`:

```javascript
//replace
pactUrls: [
  path.resolve(__dirname, '../pacts/frontendwebsite-productservice.json')
],

// with
pactBrokerUrl: process.env.PACT_BROKER_BASE_URL || "http://localhost:8000",
pactBrokerUsername: process.env.PACT_BROKER_USERNAME || "pact_workshop",
pactBrokerPassword: process.env.PACT_BROKER_PASSWORD || "pact_workshop",
```

```javascript
// add to the opts {...}
publishVerificationResult: process.env.CI ||
  process.env.PACT_BROKER_PUBLISH_VERIFICATION_RESULTS;
```

Let's run the provider verification one last time after this change. It should print a few notices showing which pact(s) it has found from the broker, and why they were selected:

```console
❯ PACT_BROKER_PUBLISH_VERIFICATION_RESULTS=true npm run test:pact --prefix provider

The pact at http://localhost:8000/pacts/provider/ProductService/consumer/FrontendWebsite/pact-version/80d8e7379fc7d5cfe503665ec1776bfb139aa8cf is being verified because the pact content belongs to the consumer version matching the following criterion:
    * latest version of FrontendWebsite that has a pact with ProductService (9cd950-step10+9cd950.SNAPSHOT.SB-AS-G7GM9F7)

Verifying a pact between FrontendWebsite and ProductService

  get all products
    returns a response which
      has status code 200 (OK)
      includes headers
        "Content-Type" with value "application/json; charset=utf-8" (OK)
      has a matching body (OK)

  get product by ID 10 with no auth token
    returns a response which
      has status code 401 (OK)
      has a matching body (OK)

  get product with ID 10
    returns a response which
      has status code 200 (OK)
      includes headers
        "Content-Type" with value "application/json; charset=utf-8" (OK)
      has a matching body (OK)

  get product with ID 11
    returns a response which
      has status code 404 (OK)
      has a matching body (OK)

  get all products
    returns a response which
      has status code 401 (OK)
      has a matching body (OK)
```

As part of this process, the results of the verification - the outcome (boolean) and the detailed information about the failures at the interaction level - are published to the Broker also.

This is one of the Broker's more powerful features. Referred to as [Verifications](https://docs.pact.io/pact_broker/advanced_topics/provider_verification_results), it allows providers to report back the status of a verification to the broker. You'll get a quick view of the status of each consumer and provider on a nice dashboard. But, it is much more important than this!

### Can I deploy?

With just a simple use of the `pact-broker` [can-i-deploy tool](https://docs.pact.io/pact_broker/advanced_topics/provider_verification_results) - the Broker will determine if a consumer or provider is safe to release to the specified environment.

You can run the `pact-broker can-i-deploy` checks as follows:

```console
❯ cd consumer
❯ npx pact-broker can-i-deploy \
               --pacticipant FrontendWebsite \
               --broker-base-url http://localhost:8000 \
               --broker-username pact_workshop \
               --broker-password pact_workshop \
               --latest

Computer says yes \o/

CONSUMER        | C.VERSION | PROVIDER       | P.VERSION | SUCCESS?
----------------|-----------|----------------|-----------|---------
FrontendWebsite | fe0b6a3   | ProductService | 1.0.0     | true

All required verification results are published and successful

----------------------------

❯ cd ../provider
❯ npx pact-broker can-i-deploy \
                --pacticipant ProductService \
                --broker-base-url http://localhost:8000 \
                --broker-username pact_workshop \
                --broker-password pact_workshop \
                --latest

Computer says yes \o/

CONSUMER        | C.VERSION | PROVIDER       | P.VERSION | SUCCESS?
----------------|-----------|----------------|-----------|---------
FrontendWebsite | fe0b6a3   | ProductService | 1.0.0     | true

All required verification results are published and successful
```

That's it - you're now a Pact pro. Go build 🔨

For an advanced step goto [step 12](https://github.com/bookmd/pact-workshop-js/tree/step12#step-12---simulate-ci)
