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

## Step 10 - Request Filters on the Provider

Because our pact file has static data in it, our bearer token is now out of date, so when Pact verification passes it to the Provider we get a `401`. There are multiple ways to resolve this - mocking or stubbing out the authentication component is a common one. In our use case, we are going to use a process referred to as _Request Filtering_, using a `RequestFilter`.

_NOTE_: This is an advanced concept and should be used carefully, as it has the potential to invalidate a contract by bypassing its constraints. See https://github.com/DiUS/pact-jvm/blob/master/provider/junit/README.md#modifying-the-requests-before-they-are-sent for more details on this.

The approach we are going to take to inject the header is as follows:

1. If we receive any Authorization header, we override the incoming request with a valid (in time) Authorization header, and continue with whatever call was being made
1. If we don't receive an Authorization header, we do nothing

_NOTE_: We are not considering the `403` scenario in this example.

In `provider/product/product.pact.test.js`:

```javascript
// add this to the Verifier opts
requestFilter: (req, res, next) => {
  if (!req.headers["authorization"]) {
    next();
    return;
  }
  req.headers["authorization"] = `Bearer ${ new Date().toISOString() }`;
  next();
},
```

We can now run the Provider tests

```console
❯ npm run test:pact --prefix provider


```

_Move on to [step 11](https://github.com/bookmd/pact-workshop-js/tree/step11#step-11---using-a-pact-broker)_
